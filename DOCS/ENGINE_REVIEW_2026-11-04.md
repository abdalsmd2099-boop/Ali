# Engine review 2026-11-04 – how a move works, how it ends, the two open errors, the smooth scan

Written against the files as they stand in this folder: `FB_RobotMove` v0.6, `FC_MoveSelect` v0.2, `FB_MoveQueue` v0.1, `FB_ScanCubeJob` v0.6, `FB_CubeButtons` v0.1, `OB30_complete.scl` and `OB1_Stage2_call.scl` of 2026-11-04, `PROGRAM20.md`, and the manual 33956901 as text (`ROBOT/MOVIKIT_33956901.txt`, the numbers in brackets are its lines). Every claim carries a file line or a manual line. Nothing in `PLC/` was changed for this document.

How it was made: six independent readings of the code (engine, cycle timing, one writer, bus contract, cube job, error diagnosis), every finding then attacked by three refuters (code, manual, a cycle-by-cycle failure sequence), and the nine changes I proposed on 2026-11-04 judged by three readers each. Section 6 has the result, section 7 the change set in the order to type it.

Sections: 1 the engine in one picture, 2 one move step by step, 3 how a move ends and how "program finished" becomes trustworthy, 4 the two open errors, 5 the smooth scan, 6 the review, 7 the change set, 8 what I need from the machine.

---

## 1. The engine in one picture

One OB1 cycle, in the order of `OB1_Stage2_call.scl`. Every arrow is a DB tag with one writer.

```
screen buttons -> "DB_RobotHand".Btn -> FB_CubeButtons -> Teach.* (pulse)  ->  FB_Teach -> "DB_Poses"
                                                       -> Man.udtReq, Man.xStart (go to, pulse)
                                                       -> Man.xReset, Man.xAbort (ack, pulse)

auto:  FB_ScanCubeJob --push one per cycle--> FC_JobMux --> FB_MoveQueue --head-->+
                                                                                  |
                                                                   FC_MoveSelect  +--> Req, xReqStart --> FB_RobotMove
hand:  Man.udtReq, Man.xStart ----------------------------------------------------+                           |
                                                                                                              v
                                                   "DB_App".Srl.Pose/Par/Gate, Cmd.Start/ProgramNumber/Override, "DB_RobotPath"
                                                                                                              |
                                    OB30 every 10 ms: NW1 handshake, NW2 module 1, NW3 module 2 ----> robot   |
                                    OB30 NW4: PI words -> "DB_Robot".Status ------------------------> engine, job, select
                                    module 2: SrlRealRaw[1..2] = REAL[71], REAL[72] ----------------> engine
```

| Layer | Block | Decides | Never touches |
|---|---|---|---|
| job | `FB_ScanCubeJob` | which poses, in which order; checks all of them first; pushes one per cycle; counts the done pulses | the bus, the robot |
| queue | `FB_MoveQueue` | keeps the list, head = the next move; pops on the engine's done pulse | the content of a move |
| select | `FC_MoveSelect` | the one request the engine sees. Auto: the head with a level start (head valid, engine idle, not done, no error). Hand: `Man.udtReq` with the pulse | the bus |
| engine | `FB_RobotMove` | one move of program 20: check, write the bus, stop if needed, settle, start, watch running, watch finished, check the echo, report | which pose, which job, which source |
| OB30 | three FB calls and the map | nothing. Ships what the DBs hold every 10 ms and maps the PI words | the decisions |
| robot | program 20 | the motion of the block whose gate is TRUE | everything else |

The engine serves auto and hand the same way: `FC_MoveSelect` is the only one that knows the source (`FC_MoveSelect.scl:26-38`).

---

## 2. One move, step by step

The steps of `FB_RobotMove` v0.6 with what the robot sees. Step numbers are `State.intStep`, the `:n` are lines of `FB_RobotMove.scl`.

| Step | The PLC does (one cycle unless it waits) | On the bus | The robot | Goes on when | Limit → error |
|---|---|---|---|---|---|
| 0 idle | busy FALSE | nothing changes | finished (PI 2.5), the rest state | start edge AND enable AND no error (`:167`) | – |
| 5 check | block 1..8 and ≤ Cfg (1); `FC_PoseCheck` on the target (2, 3); path 0..7 and every path pose (16, 2, 3); radius for 3, 4, 6, 7 (4); comm (7); ready, referenced, automatic, access, no fault, setpoints active (5); not moving, not running, not paused (6); override (8). Latches block and id, busy TRUE (`:171-221`) | nothing | – | all pass | – |
| 10 write | run number + 1 → `Par[4]`; program 20; pose → `Pose[1..5]`; `aorPar[1..3]` → `Par[1..3]`; override clamped 1..100 when given; the path → `DB_RobotPath` when `intPathLen` > 0; all gates FALSE, then gate n TRUE; gate enable TRUE; start FALSE (`:223-260`) | POSE[51], REAL[51..54], BOOL[50+n], PO 4 within 10 ms | takes the data (the consistent block, handshake equal) but uses nothing yet: a variable takes effect when a command uses it (manual 10821) | PI 2.6 "repositioning required" → 15 with stop TRUE, else → 20 (`:261-266`) | – |
| 15 program stop | `State.xPrgStop` TRUE; OB30 NW2 `stop := Cmd.Stop OR State.xPrgStop` (`OB30_complete.scl:52`) | PO 2.4 | cancels the way back to the old path, program goes to initialized (9.8, 10834-10850) | initialized AND NOT 2.6 AND NOT 2.7 → stop FALSE → 20 (`:271-274`) | T_STOP 2 s → 15 |
| 20 settle | waits 100 ms with pose and gate on the bus, and program 20 initialized or finished (`:133`, `:283`) | – | shows PI 4 = 20 and 2.2 or 2.5 | both → 30 | T_PRG 3 s → 14 |
| 30 go | start TRUE (`:291`) | PO 2.2 rises, if `ReadyToStart AND MachineOn` (`OB30_complete.scl:50`) | basic state, interprets program 20 from the top | at once → 40 | – |
| 40 wait running | – | start stays TRUE | `IF BOOL n` true → SET_MOTIONSET, LIN_VAR → PI 2.3 running, PI 1.4 moving | 2.3 seen → 50. Or 2.5 AND REAL[72] = run → 50 (a move too short to be seen running) (`:297-300`) | T_RUN 3 s, not counting while 2.7 → 11 |
| 50 wait finished | – | – | moves; END_PROG waits for the motion; then 2.5 finished, 1.4 off | 2.5 AND NOT 1.4 AND remaining 0, stable 200 ms → 60 (`:130`, `:143`, `:308`) | T_MOVE 60 s, path 300 s → 12 |
| 60 end | start FALSE, gates FALSE, pose latched; REAL[71] = block else 10; REAL[72] = run else 13; done pulse, busy FALSE (`:315-332`) | PO 2.2 falls, word 123 = 0 | finished, rests there | → 0 | – |
| 90 error | start FALSE, gates FALSE, busy FALSE, error TRUE, message id latched (`:334-345`) | as 60 | whatever it was doing (see section 6, finding 2) | reset → 0 (`:121-127`) | – |

Abort in steps 10..50, checked before the CASE every cycle (`:147-162`): comm lost 7, robot fault 21, access lost 22, enable fell 20.

### 2.1 The timeline of one single move (block 1)

```
cycle N      step 5   check. nothing on the bus.
cycle N+1    step 10  pose, par, run number, gate n on the bus. start FALSE.
                      OB30 ships within 10 ms. the robot has the data, uses nothing.
             step 20  the settle timer runs.
+100 ms      step 30  start TRUE. OB30 ships: PO 2.2 rises.
+?           robot    basic state (6.3.2), program pointer runs: IF BOOL n, SET_MOTIONSET, LIN_VAR.
                      PI 2.3 running, PI 1.4 moving.  the ? is the robot's start latency, never measured yet.
+?           robot    REAL[71] := n and REAL[72] := REAL[54] are written NOW, milliseconds after the start,
                      because the program pointer does not wait for the motion (manual 2570-2576).
             step 50  waits.
arrival      robot    the motion ends. END_PROG "wait" holds the program end until then (14112-14116).
                      PI 1.4 off, PI 2.3 off, PI 2.5 on.
+200 ms      step 60  echo checked, done pulse, start FALSE, gates FALSE.
cycle +1     queue pops the head (i_xPop = xDone), job counts the move.
cycle +2     FC_MoveSelect: head valid, idle, not done -> start level TRUE -> engine edge -> step 5.
cycle +3     step 10 of the next move.
```

So between two moves of a scan the robot stands for: 200 ms (still) + 3 OB1 cycles + 100 ms (settle) + its own start latency. Plus the exact-stop deceleration of every LIN, which is the robot's, not ours.

### 2.2 Why "program finished" is the normal rest state

Every run of program 20 ends in "finished" (PI 2.5). The state diagram (2089-2116) goes finished → program start ** → running, with ** = control type inactive. So the next start works from finished, nothing clears it, no machine off/on is needed. Step 20 accepts initialized or finished (`:133`). The 2026-11-03 confusion is in `ERROR_LOG.md` as solved: what matters after a jog is not "finished" but "repositioning required" (PI 2.6), section 4.2.

---

## 3. How a move ends, and how "program finished" becomes trustworthy

### 3.1 The problem

PI 2.5 "finished" is TRUE before the move (left from the last run) and TRUE after it. Finished alone proves nothing. That is rule 5 of `GOALS.md` 2.1: running must be seen first in this move.

### 3.2 What the robot really does with the echo lines

Manual 2570-2576, in substance: variables or expressions in the SRL program are not synchronous with the movement; when the program pointer reaches the expression, it is processed; expressions can be synchronized by a WAIT_MOTION command. Manual 12968-12975 separates the program pointer P (the command being processed) from the motion pointer M (the segment being traveled). Manual 14112-14116: END_PROG either waits for the movement or ends the program the moment the program pointer reaches it.

For program 20 this means:

| Line | Written when | Proves |
|---|---|---|
| `REAL 71 := n` (line 4, 9, 14, 19) | milliseconds after the start, while the robot still moves | block n was interpreted in this run. Not: arrived |
| `REAL 72 := REAL 54` (line 21) | also at once, it is reached right after the END_IF of the last block | the run reached the last line. Not: arrived |
| `END_PROG` with "wait for the movement to end" ON (line 22) | the program end waits for the motion | PI 2.5 comes when the robot stands. **This line is the only thing that ties "finished" to arrival** |

`PROGRAM20.md` line 79 says the opposite about line 21 ("the PLC takes program finished as the robot stands at the target"); the conclusion there is right, the reason is END_PROG wait, not the echo line. The engine does not depend on the echo for arrival: step 50 also needs PI 1.4 off and remaining segments 0 for 200 ms (`:130`, `:143`), so the code is safe either way.

### 3.3 The three proofs the engine uses today

| Proof | Where | Worth |
|---|---|---|
| running seen (PI 2.3) | step 40 `:297` | the robot took the start. Good |
| finished AND not moving AND remaining 0, 200 ms | step 50 `:308` | standstill after a start. Good, because the program states are exclusive (manual 15110-15125): once running was seen, finished means the program ended, and with END_PROG wait that is arrival |
| REAL[71] = block, REAL[72] = run | step 60 `:322-327` | the right block ran in this run. Good, with the two holes below |

### 3.4 Where it is weak, and the fix on each side

**Hole 1, the step 40 shortcut after a download.** `intRunNr` is a plain static (`:78`), it restarts at 1 after a download or a PLC power cycle. The robot keeps REAL[72] from the last run of the previous session. When the two are equal, step 40 passes at once on the stale finished (`:299-300`) without running, 200 ms later step 60 reports done, the robot never moved, the queue pops, the job drives the next pose from the wrong place. It needs the previous session to have ended after exactly one move, which is a normal commissioning loop (download, one MOVE, download, MOVE). Found by three readers independently, verified in section 6.

Fix in the PLC, step 10, after `:227`:

```scl
         // the number on the bus must never equal what the robot still echoes from before
         IF #intRunNr = #i_intEchoRun THEN
            #intRunNr := #intRunNr + 1;
            IF #intRunNr > #RUNNR_MAX THEN
               #intRunNr := 1;
            END_IF;
         END_IF;
```

**Hole 2, a run in which no block ran looks like a success.** REAL[71] is never cleared. Every cube move is block 1, so the stale REAL[71] = 1 satisfies step 60. A run that skipped all IFs (gate word 123 not taken for the one cycle the IF is evaluated, a wrong BOOL mapping after a MOVISUITE change, BoolWriteEnable off) still writes REAL[72] := run at line 21, "finished" comes after 200 ms, the engine says done, the robot stands. Error 10 catches "a different block ran", never "no block ran".

Fix on the robot, one line, no PLC change: the first line of program 20, before the first IF:

| Line | New Block | Fields | Means |
|---|---|---|---|
| 0 | `CALC_REALVAR` | target: REAL, index **71**. Operand 1: val, **0** | clear the block echo at every run |

Then REAL[71] = n proves block n was interpreted in this run, and error 10 fires when no block ran.

**Hole 3, the echo is written before arrival.** My proposal C4 of 2026-11-04 (require REAL[72] = run inside the step 50 condition) rests on the echo being path-synchronous. It is not (3.2). Without a wait line it would change nothing in the normal case and could end a move early in the stale case. Do not add it. If you want the run echo to mean "arrived", the robot can say so itself:

| Line | New Block | Fields | Means |
|---|---|---|---|
| 20a | `WAIT_MOTION` | argument **Motion_Done** | the program pointer waits here until the motion is done (14022-14031; 7817-7819 lists it with END_PROG WaitMotion as a command that expects the end of the movement) |
| 21 | `CALC_REALVAR` | REAL 72 := var REAL 54, unchanged | now written at standstill |

With that line REAL[72] = run is a true "arrived", the step 40 shortcut is a real proof for very short moves, and the PLC no longer depends on END_PROG wait being ON. Cost: one line in RobotMonitor. The exact stop is there anyway (blending 0).

**What stays as it is:** the 200 ms of step 50. It filters a torn status read between two OB30 frames; two frames are 20 ms, 200 ms is generous. It could go to 100 ms later for 100 ms per move. Not now, not before the open errors are closed.

### 3.5 The end of a move seen from the job

`FB_ScanCubeJob` step 30 counts the done pulses (`FB_ScanCubeJob.scl:369-371`) and ends when the queue is empty, the engine is not busy and not done (`:372-376`). The done pulse is one cycle (`FB_RobotMove.scl:118`, `:329`); job, queue and select see it in the same cycle because OB1 runs them before the engine (`OB1_Stage2_call.scl:87`, `:147`, `:162`, `:172`). The end of the job comes two cycles after the last done, after the pop. No pulse is lost, no entry is popped twice, no edge is missed. This was walked cycle by cycle by the timing reading (section 6, "checked").

---

## 4. The two open errors

### 4.1 Engine 11 after a download or a machine off/on, cured once by a hand-set `Cmd.Start`

What the code does: step 30 sets `Cmd.Start` 100 ms after the pose and gate went out and the robot showed program 20 initialized or finished (`:283`, `:291`). OB30 ANDs it with two tags whose logic is not in this folder: `start := Cmd.Start AND ReadyToStart AND MachineOn` (`OB30_complete.scl:50`). Step 40 waits 3 s for PI 2.3 (`:301`). The engine cannot see whether PO 2.2 ever rose.

What the manual demands for a start (9.4, 10672-10687): setpoints active, override > 0, pause and stop deactivated, a rising edge of program start at least one cycle after the rising edge of "setpoint active" and after the falling edges of pause and stop. The state diagram (2089-2090) adds the start conditions: type of control inactive, enable state position hold control. 16.1.1 (19188-19195): pause activated, stop activated, no rising edge of program start detected, target same as the current position.

The hypotheses, ranked by how well they fit "only after a download or power cycle, cured by one hand-set start, PI words all good on 11-03 with 125.57 'start edge ignored, robot not yet ready for movement'":

Five facts every hypothesis must fit: F1 only after a download or power cycle (`ERROR_LOG.md:12`). F2 cured by one hand-set `Cmd.Start` with the engine idle, then every start works. F3 on 2026-11-03 the fault came with the OB30 pin on `Cmd.Start` alone (`GOALS.md:415-416`), so `ReadyToStart` and `MachineOn` were not the cause that day. F4 the robot answered PI 3 = 32057 = 125.57 "rising edge of program start ignored, robot not yet ready for movement" while PI showed automatic, initialized, program 20, ready, referenced, setpoints active, access granted, control type inactive (`GOALS.md:417`). F5 the error is 11 and not 14, so step 20 saw program 20 initialized or finished.

| Rank | Hypothesis | Fits | Discriminating check (watch table, the moment) |
|---|---|---|---|
| 1 | **The first start after a restart is the only one that follows a change of PO 4 and starts from "Initialized".** Step 0 and step 5 never write the bus (`:165-171`), so after a download PO 4 carries the start values of `DB_App` (number, override) until the first step 10. That step 10 is then a real program-number change on the bus, the robot re-initializes program 20, and the start edge follows it by at most 100 ms (settle and ready are checked in parallel, `:134`, `:283`), from the state "Initialized", whose start condition is the stricter one (* = type of control inactive AND enable state position hold control, 2089-2093), not the one from "Finished" (** = control type inactive, 2090, 2107-2111). The robot refuses: 125.57. Every later move writes the same number (no change) and starts from "Finished". The hand-set start works because by then 20 has stood on PO 4 for seconds and its run leaves the program "Finished". Also explains 2026-10-29 (`GOALS.md:448-449`) | F1 to F5 | Before the first MOVE after a download: `"DB_App".Cmd.ProgramNumber`, `"DB_MOVIKIT_Robotics-5AX".PO[4]` (low byte number, high byte override), `PI[4]`, `"DB_Robot".Status.intPrgNr/.xPrgInitialized/.xPrgFinished`. The hypothesis needs PO 4 low byte ≠ 16#14 there. At the error: `State.uiMsgIdAtError` = 32057 means the edge reached the robot and was refused. Cheap proof: after the next download write 20 into `Cmd.ProgramNumber` by hand, wait 2 s, MOVE. And: RESET and a second MOVE without any hand-set must also work under this hypothesis (the failed attempt left 20 on PO 4) |
| 2 | `ReadyToStart` or `MachineOn` is FALSE after a download or power-up until something sets it; the engine's start never reaches PO 2.2 | F1, F2. Not F3, not F4 | `"DB_App".ReadyToStart`, `"DB_App".MachineOn`, `PO[2]` bit 2 and `State.intStep` during steps 30..40 of the first MOVE. PO 2.2 stays 0 while `Cmd.Start` = 1 → this one. Fix in the logic that writes the two tags (not in the folder): TRUE whenever enable, automatic and setpoints active are, no latch |
| 3 | The first program run after a robot power-up takes longer than `T_RUN` 3 s; the hand-set run was that slow first run, unnoticed | F1, F2, F5. Not F4 | after an error 11 touch nothing for 20 s and watch `"DB_Robot".Status.xPrgRunning/.xPrgFinished` and `SrlRealRaw[2]` against `State.intRunNr`; a late equality means the start was taken and the first run is slow |
| 4 | The SEW module 1 FB (not in the folder) needs a TRUE → FALSE on its start pin after its first cycle before it passes an edge | F1, F2. Not F4 (the robot reported the edge) | `PO[2]` bit 2 during step 30..40 with `Cmd.Start` TRUE: FALSE = the FB or the AND gate |
| 5 | A condition step 5 does not check: output stage (PI 1.2), STO (PI 1.1), enable state not "position hold control" | weak: PI 1.6 setpoints active (checked, `:199`) already implies enable and no fault (10658-10663), and a hand-set start would not change STO | PI 1.1, PI 1.2 at step 30. Adding both to error 5 costs nothing (proposal C2) |
| 6 | A held `Cmd.Stop` or `Cmd.Pause` from the old screen after a download. 9.4: both must be deactivated, the robot refuses the start without a fault | F1. Not F2 | `"DB_App".Cmd.Stop`, `.Cmd.Pause` at step 30 |
| 7 | A second writer of `Cmd.Start`; retentive `Cmd.Start`; program "not initialized" until a number change | no. A steady second writer is random per move, not once after a download; step 10 writes FALSE anyway (`:260`); "not initialized" would give error 14, not 11 | the cross-reference, as a formality |
| 8 | `T_RUN` 3 s too short | only as hypothesis 3. An ignored edge does not arrive later; a longer wait only delays error 11 | the time from PO 2.2 to PI 2.3 in the FB_StartTrace rows |

Hypotheses 1, 3 and 4 all predict that RESET and a second MOVE, without the hand-set, also work. If a second MOVE is known to fail, only hypothesis 2 survives. That is the cheapest test of all and costs no watch table.

The fix that covers hypothesis 1 whatever the watch shows, two lines in the engine: in step 0 write the program number and start FALSE every cycle, so 20 stands on PO 4 long before the first move and the bus is defined in idle; and count the settle from "program 20 ready", not from the entry of step 20. Section 7 has the code.

What to stop doing: the hand-set `Cmd.Start` in the watch table. It is a second writer of the one tag that starts the robot, OB30 passes it to the robot without any check. With PI 2.6 pending (after every jog) that rising edge starts a repositioning move at motion set 8 outside the engine's checks, timers and program stop (6.6, 4441-4453). And the bit stays TRUE until the next move's step 10 (`:260`), so later jogs run with program start asserted. If a forced edge is needed for a test, first PI 2.6 = FALSE and all gates FALSE, set the bit back within the minute, and write it in `ERROR_LOG.md` as a motion test, not a cure.

### 4.2 Engine 15 at the first START after a jog

What the code does: a jog from "finished" sets PI 2.6 (2127-2129: even when the program has ended, repositioning can be required, e.g. after a jog). Step 10 sees it and goes to 15 with `State.xPrgStop` TRUE (`:261-263`). Step 15 holds the stop and waits for PI 2.2 initialized AND NOT 2.6 AND NOT 2.7 (`:271-272`), 2 s, then error 15.

What the manual says: start, pause and stop are edge-controlled (2082-2083). 9.8 (10834-10850): reset the program start, set program stop, program is initialized, program stopped at standstill, in that order. 9.10 (10905-10909): with repositioning required, either cancel the existing program with a stop and start a new one, or reposition with a start. 2127-2129: repositioning can be suppressed by stopping the program. And one fact that settles the "flag or state" question: the program state is one enum, `E_PROGRAMSTATE` = NotInitialized, Initialized, BeingExecuted, Paused, Finished, BackToPathRequired, BackToPathActive (15110-15125), reported by PI 2 bits 2 to 7 (16607-16612). "Repositioning required" is a state, not a flag beside "finished": the robot cannot show 2.2 and 2.6 together, and after a jog from "finished" bit 5 falls and bit 6 rises. So step 15's exit condition is the manual's own sequence, and my proposal C1 of 2026-11-04 (drop NOT 2.6) changes nothing on a robot that follows its enum. Error 15 therefore means: the stop did not reach the robot, or the start was not reset before it, or the robot does not leave BackToPathRequired on a stop.

The hypotheses, ranked:

| Rank | Hypothesis | Discriminating check |
|---|---|---|
| 1 | the stop never reaches the robot: the OB30 NW2 stop pin in TIA is still `Cmd.Stop` alone, not the OR with `State.xPrgStop`. `CHECKLIST_CUBE_AUTO.md:174` still allows the old wiring while error 15 proves engine v0.5 or later is in the PLC. Fits the RobotMonitor-stop workaround exactly | the pin in TIA; the cross-reference of `State.xPrgStop` must show OB30. During step 15: `State.xPrgStop` TRUE and `PO[2]` bit 4 FALSE → this one |
| 2 | the start was not reset before the stop. A hand-set `Cmd.Start` (the error 11 workaround) stays TRUE through the jog, because step 0 never writes the start (`:165-169`); step 10 then drops start and raises stop in the same telegram (`:260`, `:262`), against the 9.8 order. `ERROR_LOG.md:12-13`: the hand-set day came before the error 15 day | `PO[2]` bit 2 before the START, after a jog: TRUE → this one. Fix: step 0 writes start FALSE (section 7) |
| 3 | the robot keeps BackToPathRequired after a stop from that state, against its own enum | after a jog from finished, engine idle: `"DB_App".Cmd.Stop` TRUE by hand for 1 s, watch PI 2.2, 2.5, 2.6. Bit 6 clears and bit 2 sets → the robot takes a bus stop and only the engine's path is broken (1 or 2). Bits unchanged → this one |
| 4 | `T_STOP` 2 s too short for "stopped at standstill" | unlikely at rest; the same watch shows 2.2 arriving late |

The fix follows the watch:

| The watch shows | Change |
|---|---|
| PO 2.4 stays FALSE in step 15 | the OB30 pin. `stop := "DB_App".Cmd.Stop OR "DB_RobotMove".State.xPrgStop` as `OB30_complete.scl:52` says |
| PO 2.2 was TRUE before the START | step 0 owns the start: `#io_xStart := FALSE` in idle (section 7), and no more hand-set |
| 2.2 comes while 2.6 stays | exit on `xPrgInitialized AND NOT xRepositionActive` (proposal C1); 2.6 is then a lamp |
| 2.5 stays, nothing changes | the stop has no effect from that state. Then the way to "initialized" is the one the diagram gives: change the program number (2094-2095, 2131-2133), e.g. 19 for one robot cycle, then 20 again, and wait for initialized. Needs the backup copy as program 19. Decide after the watch, not before |

The stop as a level held until the exit is correct: edge-controlled means the robot acts on the rising edge, a held level does no harm, and 9.4 only asks for the start to follow the falling edge of stop by at least one cycle, which step 20 guarantees (100 ms after the exit). A fixed pulse would add nothing.

And after an error 15 today: ACK and START repeat 10 → 15 → 15, because the robot is still in BackToPathRequired. The only ways out were RobotMonitor stop or machine off/on. With the cause above removed the loop is gone.

---

## 5. The smooth scan

### 5.1 Where the time goes today (one move per point, block 1)

Per point, the robot stands for the chain of 2.1: 200 ms still + 3 OB1 cycles + 100 ms settle + the robot's start latency + its own deceleration and acceleration at the exact stop. Roughly half a second plus the ramps. For 14 moves about 7 s of standstill, spread over the corners and the turn points. Along one side the robot runs one LIN at constant speed; that is where the sensor looks, and that part is smooth already.

### 5.2 What makes it smoother, in order of safety

| # | Change | Gain | Risk | When |
|---|---|---|---|---|
| 1 | nothing in the timers until the two open errors are closed | – | – | now |
| 2 | path mode, block 5: the seven rectangle segments in one program run (`PROGRAM20.md` section 8). The PLC gaps between side and turn disappear; the turns are exact stops anyway ("Keep" = blending 0, 2155-2157). Program 20 runs 8 times for a cube instead of 14 | about 6 × (settle + still + latency) ≈ 3 to 4 s per cube, and no PLC in the loop along the rectangle | the engine finding 1 and the job finding 25 of section 6 must be fixed first (a path block without a path, Cfg not checked by the job); then `TEST_PROTOCOL.md` part 5 | after 6.1 |
| 3 | `T_STILL` 200 → 100 ms | 100 ms per move | a torn status read is filtered by two OB30 frames already, 100 ms leaves five | after 2 is proven |
| 4 | blending on the sides | none for this scan: a side ends in a turn in place (zero length), there is nothing to blend into, and the corners must be exact for the sensor | – | not for the cube |
| 5 | turn while traveling: drop the turn entries and give every side its end orientation, so RotZ interpolates along the side (the robot does that in a LIN, `PROGRAM20.md` section 3a) | the four turn stops disappear | a different scan geometry: the tool is not square to the side while it travels. Only if the sensor allows it | later, a recipe switch like no-turn |
| 6 | jog as little as possible between scans: every jog after "finished" sets PI 2.6 and costs the stop cycle of step 15 at the next START, and today the error 15 | – | – | now |
| 7 | after a jog wait one second before START: `FB_RobotMode` goes back to automatic 500 ms after the last jog, a START inside that window is error 5 (`ERROR_LOG.md` solved rows) | – | – | now |

### 5.3 What is already smooth: the cycle between the blocks

Walked cycle by cycle in the review and found sound (section 6, "checked"): START → clear → one push per cycle → first engine start in the cycle of the first push; done → pop → next start two cycles later → edge; end of job two cycles after the last done; an engine error reaches the job the next cycle, ACK resets and aborts in one cycle; ABORT or AUTO off: the running move finishes, nothing follows. OB30 interrupting step 10 cannot put a half pose on the robot at a moment where it matters: the gates are FALSE on entry, start is FALSE, the edge comes 100 ms later, and the robot reads a variable only when a command uses it.

Two things break the smooth cycle and are in the findings: switching AUTO or editing the hand pose in the one cycle between step 5 and step 10 (finding 3), and the queue starting on AUTO on without START when a clear pulse was lost (finding 12).

---

## 6. The review

Filled in when the verification returns. The six readings produced 36 findings, merged to 28; each is attacked by three refuters and survives only when two of three cannot refute it.

(pending)

---

## 7. The change set, in the order to type it

(pending, follows section 6)

---

## 8. What I need from the machine

| # | Watch | When | Decides |
|---|---|---|---|
| 1 | `"DB_RobotMove".State.uiMsgIdAtError` | after the next error 11 | 32057 = 125.57 → hypothesis 2 of 4.1 |
| 2 | `"DB_App".ReadyToStart`, `"DB_App".MachineOn`, `"DB_MOVIKIT_Robotics-5AX".PO[2]` bit 2, `"DB_RobotMove".State.intStep` | the first MOVE after a download, steps 30..40 | PO 2.2 stays 0 → hypothesis 1 of 4.1 |
| 3 | PI 2.2, 2.5, 2.6 for 3 s | after a jog from "finished", then a program stop from RobotMonitor | which row of the table in 4.2 |
| 4 | the OB30 NW2 stop pin in TIA | now | hypothesis 2 of 4.2 |
| 5 | the networks that write `"DB_App".ReadyToStart` and `.MachineOn` | now, as a file into `PLC/` | hypothesis 1 of 4.1 |
| 6 | RobotMonitor variable names of REAL[72] and BOOL[71] (PDI67 or PDI66? PDI82.0 or PDI81.0?), and `"DB_MOVIKIT_ROBOTICS_Module2".PI[1]`, `[2]`, `[17]` during one move | any move | finding 5 of section 6, the echo word |
| 7 | the time from PO 2.2 rising to PI 2.3 rising | the FB_StartTrace rows of one good move | whether T_RUN matters at all |
