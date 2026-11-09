# Checklist – the drum scan in automatic (2026-11-06, part 0 "commissioning in order" 2026-11-09)

Tick every line. It covers one thing: pose 8 above the drum axis, four numbers typed, job 2, START, the robot
drives a circle around the drum in four quarter arcs, one move per point. With NO TURN on (A50, the cube's switch,
drum job v0.5 since 2026-11-06) the tool keeps the RotZ of pose 8 all the way round, nothing turns. With it off the
tool turns toward the drum axis, −180 to +180, the design for the sensor. The proven way of the cube (single moves,
PATH MODE off) with two more blocks of program 20. Nothing of the cube changes: job 1 scans the cube exactly as on
2026-11-05. The circle **without a stop** at the quarters, one line of the robot: part F at the end, PATH MODE on.

Three places, in this order: TIA (A), the robot (B), the screen (C). Then the numbers (D), the start key by key,
and the test (E). About two hours in all, the hand test of the circle is the one that decides.

What the job drives, `FB_ScanDrumJob` v0.5 (`PLC/`), list index = `intMoveActive`. The RotZ column is for NO TURN
off; with NO TURN on every RotZ is the RotZ of pose 8:

| # | Move | Block | Target | `REAL[51]` |
|---|---|---|---|---|
| 1 | to start | 1 | pose 2 | – |
| 2 | above P0 | 1 | P0 with Z = safe Z, RotZ −180 | – |
| 3 | down to P0 | 2 | P0 at the scan height | – |
| 4 | arc to P1 | 3 ccw (4 cw) | P1, RotZ −90 | radius |
| 5 | arc to P2 | 3 (4) | P2, RotZ 0 | radius |
| 6 | arc to P3 | 3 (4) | P3, RotZ +90 | radius |
| 7 | arc to P0 | 3 (4) | P0, RotZ +180 | radius |
| 8 | lift | 1 | P0 with Z = safe Z | – |
| 9 | to start | 1 | pose 2 | – |
| 10 | home | 1 | pose 1, only if taught | – |

P0 = center + radius along the angle −offset. With offset 0 that is the +X side of the drum. Clockwise: the
same points the other way round, RotZ +180 down to −180, block 4. Every pose is checked before the first one
goes out: a point outside the limits gives job error 4 and `intMoveActive` = the bad index.

---

## 0. Commissioning in order (2026-11-09)

The short way through parts A to E for the next day at the machine, with the numbers of 2026-11-06 as the worked
example. Every step says where the long form is. Do them in this order; a step that fails sends you to its part,
not further.

### 0.1 Before anything moves: read these in TIA (watch table "drum", A5)

| # | Row | Must read | If not |
|---|---|---|---|
| 1 | `"DB_RobotMove".Cfg.intBlocksInPrg20` | **4** | modify to 4. This was 1 on 2026-11-06 and stopped move 3. A download of `DB_RobotMove` older than v0.2 brings 1 back |
| 2 | `IDB_ScanDrumJob` block header in TIA | `FB_ScanDrumJob` v0.5 or v0.6 | A2 row 4, import. v0.6 changes nothing for this test, path mode stays off |
| 3 | `FC_JobMux` in OB1, after the two job boxes, before the queue | there, `"DB_RobotJob".State` written by it only | A3 row 3 |
| 4 | `"DB_Poses".aoxValid[2]` | TRUE, the start pose | teach pose 2 (A3 = 2, TEACH) |
| 5 | `"DB_Poses".aoxValid[8]` | TRUE, the center | teach pose 8 above the drum axis (0.5) |
| 6 | `"DB_Poses".rSafeZ` | > 0, and **smaller** than the Z of the drum top with the sensor on (smaller Z is up on this machine, seen 11-05 and 11-06) | type it. 500 was right for the cube; a taller drum needs a smaller number |
| 7 | `"DB_Recipe".Drum.rRadius` | > 0 | 0 gives job error 5 before anything moves |
| 8 | `"DB_Recipe".xPathMode` | FALSE | on needs part F, blocks 6 and 7, `Cfg` 7 |
| 9 | `"DB_Recipe".xNoTurn` | TRUE for the first scan | the turning circle is test E7, later |
| 10 | `"DB_Recipe".xFromHere` | FALSE | on takes the actual pose as the center |

### 0.2 The robot: program 20 as it must stand after the two fixes

Checked again 2026-11-09 against `FB_RobotMove` v0.7, steps 10, 40, 50 and 60. The verdicts of part B stand: N80
writes the echo into the wrong variable, N100 is a copy in the wrong place. After "N80 → REAL 71" and "delete N100"
RobotMonitor renumbers, and the program is these 19 lines. Compare line by line on the robot:

| New line | Block | Fields | What the engine does with it |
|---|---|---|---|
| N10 | `SET_MOTIONSET` | 1 | one set for all blocks; the speed is the override (10 %) |
| N20 | `IF` | var BOOL 51 ("[51] PDO123.0 Extra 1") | gate of block 1. Step 10 opens exactly one gate before the start, step 60 closes all |
| N30 | `LIN_VAR` | POSE 51 ("[51] PDO114-118 Extra 1"), Keep | the move |
| N40 | `CALC_REALVAR` | **REAL 71** ("[71] PDI66 Extra 1") := val 1 | step 60: REAL 71 must equal the block, else error 10 |
| N50 | `END_IF` | | |
| N60 | `IF` | var BOOL 52 | gate of block 2 |
| N70 | `LIN_VAR` | POSE 51, Keep | |
| **N80** | `CALC_REALVAR` | **REAL 71 := val 2** (was REAL 72) | the fixed line. Without the fix REAL 71 keeps 1 from the move before, error 10 at move 3 |
| N90 | `END_IF` | | |
| N100 | `IF` | var BOOL 53 | gate of block 3 (was N110) |
| N110 | `CIRC_VAR` | RadiusEndpos, **CCW**, plane XY, Radius var REAL 51 ("[51] PDO119 Extra 1"), Endpos var POSE 51, Keep | the arc. REAL 52 (90) is written too and not used by this form |
| N120 | `CALC_REALVAR` | REAL 71 := val 3 | (was N130) |
| N130 | `END_IF` | | |
| N140 | `IF` | var BOOL 54 | gate of block 4 |
| N150 | `CIRC_VAR` | the same, **CW** | |
| N160 | `CALC_REALVAR` | REAL 71 := val 4 | (was N170) |
| N170 | `END_IF` | | |
| N180 | `CALC_REALVAR` | REAL 72 ("[72] PDI67 Extra 2") := var REAL 54 ("[54] PDO122 Extra 4") | the run echo. Step 40 accepts "finished" only with this number, step 60 gives error 13 without it |
| N190 | `END_PROG` | wait for the movement to end **ON** | finished = arrived. Correction to B4: the engine also waits for "not moving" and "remaining segments 0" for 200 ms (step 50), so a missing wait does not give error 6 today; keep it ON so that "finished" on the screen means the robot stands |

Then save (B5) and look at the program state (B6).

### 0.3 The cube first

Job 1, NO TURN on, AUTO on, override 10, START: the cube scan runs exactly as on 2026-11-05. That proves the
download, the mux and `Cfg` = 4 in one go (A4 row 4). Only then the drum.

### 0.4 One quarter by hand (E1), with numbers

Enable on, AUTO off. Read the actual pose, say X0 2750, Y0 1700, Z0 500. Robot move screen: X 3050, Y 2000, Z 500,
RotZ and RotY = actual, radius M41 = 300, block 3, override 10, MOVE. The robot **leaves along +X**, bends toward
+Y and stops at 3050 / 2000, a quarter circle around 2750 / 2000. Then X 2750, Y 1700, block 4, MOVE: the same arc
back. This test also shows which way the robot calls CCW: if it leaves along **+Y** instead of +X, tell me, the two
blocks are swapped by one line in the job. Seen from above the arc may look clockwise, because Z grows downward on
this machine; the direction of departure is what counts.

### 0.5 The numbers, with the example of 2026-11-06

| Parameter | Tag | Example | How you get it | Rule |
|---|---|---|---|---|
| home | pose 1 | as taught | optional. Without it the job has 9 moves | |
| start | pose 2 | as taught, at a safe height | A3 = 2, TEACH. Job error 1 without it | |
| **center** | pose 8 | X **2500**, Y **1700**, Z 750 (any), RotZ **0**, RotY as taught | jog the tool above the drum axis, A3 = 8, TEACH. Lamp A57 on | only X, Y, RotZ (with NO TURN) and RotY are used; Z is not |
| safe Z | `"DB_Poses".rSafeZ` | **500** | the Z at which the tool clears the drum top with the sensor on | smaller than the drum top's Z; "above P0" and "lift" drive at it |
| scan height | `Drum.rZ` (A53) | **600** | jog to the drum's side at the height to scan, read actual Z | between safe Z and the floor |
| radius | `Drum.rRadius` (A54) | **250** = drum radius 150 + sensor distance 100, for instance | tape measure. First run: a hand's width more than the drum | > 0, and center ± radius inside 0 … 4380 (X), 0 … 4230 (Y) |
| RotZ offset | `Drum.rRotZOffset` (A55) | **0** | first test 0. With the sensor: D5 | 100 was typed on 11-06, it only moved P0 to the −Y side |
| clockwise | `Drum.xClockwise` (A56) | **off** | | on = block 4, the same points the other way round |
| NO TURN | `xNoTurn` (A50) | **on** | | RotZ stays at the center's 0 everywhere |
| PATH MODE | `xPathMode` (A49) | **off** | | |
| SCAN FROM HERE | `xFromHere` (A38) | **off** | | |
| override | A13 | **10** | | 25 after three good scans (E5) |
| job | A51 | **2** | | A52 shows "scan drum" |
| blocks in program 20 | `Cfg.intBlocksInPrg20` (M40) | **4** | | |

### 0.6 What the robot will do with these numbers: compare move by move on A32 or in the watch table

P_k = center + 250 · (cos a_k, sin a_k), a_k = −offset + k · 90°. With offset 0: P0 2750 / 1700, P1 2500 / 1950,
P2 2250 / 1700, P3 2500 / 1450, all inside the limits. RotZ 0 and RotY of pose 8 on every move.

| Move (A58) | Block (A30) | X | Y | Z | REAL 51 | REAL 71 after it |
|---|---|---|---|---|---|---|
| 1 to start | 1 | pose 2 | | | – | 1 |
| 2 above P0 | 1 | **2750** | **1700** | **500** | – | 1 |
| 3 down to P0 | **2** | 2750 | 1700 | **600** | – | **2** |
| 4 arc to P1 | **3** | 2500 | 1950 | 600 | 250 | 3 |
| 5 arc to P2 | 3 | 2250 | 1700 | 600 | 250 | 3 |
| 6 arc to P3 | 3 | 2500 | 1450 | 600 | 250 | 3 |
| 7 arc to P0 | 3 | 2750 | 1700 | 600 | 250 | 3 |
| 8 lift | 1 | 2750 | 1700 | 500 | – | 1 |
| 9 to start | 1 | pose 2 | | | – | 1 |
| 10 home | 1 | pose 1 | | | – | 1 |

With the offset 100 that was typed on 11-06 the same table reads P0 2457 / 1454, P1 2746 / 1657, P2 2543 / 1946,
P3 2254 / 1743: that is why the robot stood at 2457 / 1454 / 500 after move 2. The run number (A30 run nr, REAL 72)
counts up by one per move.

### 0.7 Start, key by key

The long form is "Start: what to press" S1 to S10 below. Short: Enable on, AUTO off → pose 8 taught (0.5) → the
three DRUM numbers → job 2 → switches as in 0.5 → override 10 → AUTO on → **START** once. ABORT (A15) lets the
running move finish and drops the rest.

### 0.8 While it runs

A58 counts "to start", "above P0", "down to P0", "arc to P1" … "arc to P0", "lift", "to start", "home". A30 shows
the block sequence 1 1 2 3 3 3 3 1 1 1. A32 follows the table of 0.6. Job error A21 stays 0. In the watch table
`SrlRealRaw[1]` shows the same block sequence one move later, `intMovesDone` counts to 10.

### 0.9 If it stops: the likely causes in this order

| Where | Job error | Engine error | Cause | Do |
|---|---|---|---|---|
| before the first move | 1 | – | pose 2 or pose 8 not taught | 0.1 rows 4, 5 |
| before the first move | 5 | – | radius 0 | 0.5 |
| before the first move | 4, `intMoveActive` = 2 … 8 | – | a point outside the limits | smaller radius, or move the drum. 0.6 says which point |
| move 3 refused, active shows 2 | 10 | **1** | `Cfg` = 1 | 0.1 row 1. Seen 11-06 |
| move 3 arrives, then error | 10 | **10** | N80 still writes REAL 72 | 0.2, the robot. Not yet seen, the first block-2 move ever |
| move 4 arrives, then error | 10 | 10 | the echo line of block 3 wrong | 0.2 line N120 |
| move 4 goes the wrong way | – | – | CW / CCW the other way round on the robot | 0.4, tell me |
| move 4, robot fault | 10 | 21, A33 = 7e43 | the end point is not on a circle of that radius | cannot come from the job; by hand only. B3 |
| any arc longer than 60 s | 10 | 12 | override too low for the quarter | override 25 |
| first START after a jog | 10 | 15 | repositioning required | machine off and on, as before. Still open in `ERROR_LOG.md` |

Read five numbers before ACK: A21, A26, A58, A30, A33. They go into `TEST/ERROR_LOG.md`.

### 0.10 After it ran through

Tell me: the run numbers of the ten moves, and what A32 showed at the end of move 2 and move 4. Then the open row of
2026-11-06 in `ERROR_LOG.md` moves to Solved and `GOALS.md` gets the row "first drum scan proven". Then E3 (clockwise),
E4 (ABORT), E5 (three in a row, override 25), and later E6 / E7 (the turning circle) and part F (one run).

---

## A. TIA, 30 minutes

### A1 First look: what is in OB1 today

| Look at | Found | ☐ |
|---|---|---|
| after the cube job box: a box `IDB_ScanDrumJob` | yes → A4. no → A2 | ☐ |
| after it, before the queue: a box `FC_JobMux` | yes → A4. no → A2 | ☐ |
| the cube job box, output `o_udtState` | on `"DB_RobotJob".Cube.udtState` = the mux way. On `"DB_RobotJob".State` = the old way, A2 | ☐ |
| `"DB_RobotJob"`: does `Man.intJob`, `Cube`, `Drum`, `intJobActive` exist | yes = v0.1 is in. no → A2 | ☐ |

### A2 Import, only what A1 found missing. External source files → Generate blocks from source → overwrite

| # | File in `PLC/` | Makes | ☐ |
|---|---|---|---|
| 1 | `UDT_JobOut.udt` | the output type of both jobs | ☐ |
| 2 | `DB_RobotJob.scl` v0.1 | `Man.intJob` (start value 1), `Cube`, `Drum`, `intJobActive`. Not retentive, nothing is lost | ☐ |
| 3 | `FC_RobotAlarms.scl` | job error 5 on alarm bit 9, job error 6 on bit 15 | ☐ |
| 4 | `FB_ScanDrumJob.scl` v0.5 | the drum job, with the NO TURN pin (v0.5, 2026-11-06). Instance `IDB_ScanDrumJob`, single, when you place it. If v0.4 is already in: import v0.5 over it, one pin more at the end, the instance keeps its name | ☐ |
| 5 | `FC_JobMux.scl` v0.1 | picks the active job, the one writer of `"DB_RobotJob".Push` and `.State` | ☐ |

`DB_Recipe` v0.3 is in since the cube, it holds `Drum.rZ`, `.rRadius`, `.rStartAngle`, `.rRotZOffset`, `.xClockwise`.
`DB_Poses` row 8 exists since stage 2. Nothing of the engine changes.

### A3 OB1, three networks. Every pin in `PLC/OB1_Stage2_call.scl`

| # | Network | Do | ☐ |
|---|---|---|---|
| 1 | cube job (exists) | `i_xStart` := `"DB_RobotJob".Man.xStart AND ("DB_RobotJob".Man.intJob = 1)`, a compare box in series. The four outputs go to `"DB_RobotJob".Cube.xPushReq`, `.udtPush`, `.xClear`, `.udtState`. Without the compare, START with job 2 starts both jobs | ☐ |
| 2 | **new, after it: drum job** | box `IDB_ScanDrumJob`. `i_xStart` := `Man.xStart AND (Man.intJob = 2)`, `i_xAbort` := `Man.xAbort`, `i_xAuto` := `Man.xAuto AND "DB_RobotMove".Man.xEnable`, `i_intOverride` := `Man.intOverride`, `i_aoudtP` := `"DB_Poses".aoudtP`, `i_aoxValid` := `"DB_Poses".aoxValid`, `i_rSafeZ` := `"DB_Poses".rSafeZ`, `i_rZ` := `"DB_Recipe".Drum.rZ`, `i_rRadius` := `.Drum.rRadius`, `i_rStartAngle` := `.Drum.rStartAngle`, `i_rRotZOffset` := `.Drum.rRotZOffset`, `i_xClockwise` := `.Drum.xClockwise`, `i_udtMoveState` := `"DB_RobotMove".State`, `i_intQueueCount` := `"DB_MoveQueue".intCount`, `i_xFromHere` := `"DB_Recipe".xFromHere`, `i_udtActual` := `"DB_Robot".Status.udtPoseBcs`, `i_xActualValid` := `"DB_Robot".Status.xReferenced AND .xCommOK`, `i_xPathMode` := `"DB_Recipe".xPathMode`, `i_xNoTurn` := `"DB_Recipe".xNoTurn` (v0.5, the same tag as on the cube box), `o_udtOut` => `"DB_RobotJob".Drum` | ☐ |
| 3 | **new, after it, before the queue: job mux** | box `FC_JobMux`. `i_intJob` := `"DB_RobotJob".Man.intJob`, `i_udtCube` := `"DB_RobotJob".Cube`, `i_udtDrum` := `"DB_RobotJob".Drum`, `io_intJobActive` := `"DB_RobotJob".intJobActive`, `o_xPushReq` => `"DB_RobotJob".Push.xReq`, `o_udtPush` => `.Push.udtMove`, `o_xClear` => `.Push.xClear`, `o_udtState` => `"DB_RobotJob".State` | ☐ |
| 4 | queue (exists) | its pins read `"DB_RobotJob".Push.*` as before. If they read the cube box directly, move them to `Push` | ☐ |

Cross-reference after the compile: `"DB_RobotJob".State` written by `FC_JobMux` only. `"DB_RobotJob".Push.*` by `FC_JobMux` only.

### A4 Compile, download, one number

| # | Do | ☐ |
|---|---|---|
| 1 | Write the recipe numbers down (cube length, width, angle, safe Z), as before every download | ☐ |
| 2 | Compile → Software (rebuild all), 0 errors. Download | ☐ |
| 3 | `"DB_RobotMove".Cfg.intBlocksInPrg20` := **4**. Screen field M40 (admin) or the watch table, retain. **A download of `DB_RobotMove` sets it back to the start value**: `DB_RobotMove` v0.2 (2026-11-06) has 4 there, the old file had 1. Check the field after every download: the first drum start of 2026-11-06 stopped at move 3 because it was 1 | ☐ |
| 4 | Job 1, AUTO on, START: **one cube scan runs exactly as on 2026-11-05.** That proves the mux. Only then the drum | ☐ |

### A5 Watch table "drum", the rows to add

```
"DB_RobotJob".Man.intJob                    1 cube, 2 drum. modify 2 if the screen field is not there yet
"DB_RobotJob".intJobActive                  follows intJob while no job runs
"DB_RobotJob".Drum.udtState.intStep         0 10 20 30 90
"DB_RobotJob".Drum.udtState.intErrId        job error of the drum job
"DB_RobotJob".Drum.udtState.intEngineErr    the engine number behind job error 10 (missing on the photos of 2026-11-06)
"DB_RobotJob".Drum.udtState.intMoveActive   list index, table above. On a refusal (engine 1 … 8) it is the move BEFORE the refused one
"DB_RobotJob".Drum.udtState.intMovesTotal   10, 9 without home
"DB_RobotMove".State.intErrId               the engine error itself
"DB_RobotMove".State.uiMsgIdAtError         the robot message id at the error
"DB_Recipe".xNoTurn                         on for the first scan
"DB_Recipe".xPathMode                       off (on only with part F)
"DB_Recipe".Drum.rZ                         scan height
"DB_Recipe".Drum.rRadius                    radius
"DB_Recipe".Drum.rRotZOffset                0 for the first test
"DB_Recipe".Drum.xClockwise                 false
"DB_Poses".aoxValid[8]                      center taught
"DB_RobotMove".Cfg.intBlocksInPrg20         4
"DB_RobotMove".State.intBlockActive         3 or 4 during an arc
"DB_MOVIKIT_ROBOTICS_Module2".SrlRealRaw[1] the echo, 3 or 4 after an arc
```

---

## B. The robot, RobotMonitor, 10 minutes

Your program 20 on the photos of 2026-11-06, line by line in `ROBOT/PROGRAM20.md` 3a. Blocks 1 to 4 are in and the
circle lines are already in the end-point form (Radius REAL 51, Endpos POSE 51): the change asked yesterday is done.
One field is wrong, one line is one too many, one field is to check.

| # | Line | Is | Must be | ☐ |
|---|---|---|---|---|
| B1 | **N80** | `[72] PDI67 Extra 2 := 2` | **`[71] PDI66 Extra 1 := 2`**. The target variable is REAL **71**, like N40 (`:= 1`), N130 (`:= 3`), N170 (`:= 4`). Open the line, change the target from index 72 to index 71, the value 2 stays. As it is, block 2 leaves REAL 71 on the number of the block that ran before, the engine finds it ≠ 2 at the end of the move: **engine error 10**. The cube never ran block 2, the drum runs it for move 3 "down to P0" | ☐ |
| B2 | **N100** | `[72] PDI67 Extra 2 := [54] PDO122 Extra 4`, between block 2 and block 3 | **delete the line.** N190 is the same line at the right place, the last one before `END_PROG`. The copy does no harm today, but the contract has one run echo at the end and nothing between the gates | ☐ |
| B3 | N120, N160 | `CIRC_CCW` / `CIRC_CW`, Plane XY, Radius := REAL 51, Endpos := POSE 51 | **right as they are.** Radius from `REAL 51`, end point from `POSE 51`: the one and only arc of that radius to that point. Nothing to change | ☐ |
| B4 | N200 `END_PROG` | not readable on the photo | the field "wait for the movement to end" = **ON**, so that "finished" means the robot stands. Checked again 2026-11-09: the engine also waits for "not moving" and "remaining segments 0" for 200 ms before done (step 50), so a missing wait does not give error 6; it only makes "finished" on the screen come early. Keep it ON | ☐ |
| B5 | **save** | | the save icon. Without it the change is gone at the next power cycle (manual 11.3) | ☐ |
| B6 | program state after the save | | initialized or finished, control type inactive, access granted. If the robot shows "repositioning required": machine off and on, as until now (error 15 is still open) | ☐ |

The rest is right: N10 one motion set for all blocks (fine, the override makes the speed), N20 to N50 block 1
(proven), N60 / N70 / N90 block 2, N110 to N140 block 3, N150 to N180 block 4, N190 the run echo. After B1 and B2
the program has 19 lines with `END_PROG`, RobotMonitor renumbers them.

Fault to know on the robot side: message 16#7e43 "CircleCenterCorrectionLargerThanThreshold" = the end point does
not lie on a circle of that radius from the start point. The job's points always do. By hand, type a point that
really lies on the circle (E1 does).

---

## C. The screen, 40 minutes. Object numbers as in `HMI/SCREENS.html`

### C1 Automatic screen, new objects

| Nr | Object | Kind | Tag | Settings | ☐ |
|---|---|---|---|---|---|
| A51 | job | I/O field, Int | `"DB_RobotJob".Man.intJob` | JOB box. 1 … 2, start value 1, text list `TL_Job`. Or two buttons CUBE / DRUM that SetTagValue 1 / 2. Taken over by the mux when no job runs | ☐ |
| A52 | active job | output field, Int | `"DB_RobotJob".intJobActive` | JOB box, text list `TL_Job`. This is the job START will run | ☐ |
| A53 | scan height | I/O field, Real | `"DB_Recipe".Drum.rZ` | DRUM box. mm, −10 … 1908, 0 decimals | ☐ |
| A54 | radius | I/O field, Real | `"DB_Recipe".Drum.rRadius` | DRUM box. mm, 0 … 3000. Drum radius + sensor distance. 0 = job error 5 | ☐ |
| A55 | RotZ offset | I/O field, Real | `"DB_Recipe".Drum.rRotZOffset` | DRUM box. deg, −180 … 180. 0 for the first test, D3 for the sensor | ☐ |
| A56 | CLOCKWISE | switch | `"DB_Recipe".Drum.xClockwise` | DRUM box. Off = block 3 counter-clockwise, on = block 4 clockwise. Retentive | ☐ |
| A57 | center taught | lamp | `"DB_Poses".aoxValid[8]` | DRUM box. Teaching: pose nr 8 in A3, TEACH A4. No new button | ☐ |
| A58 | active, drum | output field, Int | `"DB_RobotJob".State.intMoveActive` | JOB box, next to A25, text list `TL_MoveDrum`. Visibility: `"DB_RobotJob".intJobActive = 2`; A25 gets visibility `= 1`. Until you do that: read the number, the table at the top | ☐ |

No start angle field: the job computes the start point from the offset. SCAN FROM HERE (A38) and PATH MODE (A49)
are shared with the cube, both off for this test. NO TURN (A50) is shared too since drum job v0.5 (2026-11-06):
on = the tool keeps the RotZ of pose 8 on the whole circle. On for the first scan.

### C2 Robot move screen

| Nr | Object | Kind | Tag | Settings | ☐ |
|---|---|---|---|---|---|
| M41 | radius | I/O field, Real | `"DB_RobotMove".Man.udtReq.aorPar[1]` | next to the block field M7. mm, 0 … 5000, 0 decimals. Blocks 3 and 4 refuse 0 with engine error 4. For the hand test E1 | ☐ |
| M7 | block | exists | `"DB_RobotMove".Man.udtReq.intBlock` | 3 and 4 must be typeable, text list `TL_Block` 3 "circle ccw", 4 "circle cw" | ☐ |

### C3 Text lists

| List | Entries | ☐ |
|---|---|---|
| `TL_Job` (new) | 1 scan cube, 2 scan drum | ☐ |
| `TL_MoveDrum` (new) | 0 –, 1 to start, 2 above P0, 3 down to P0, 4 arc to P1, 5 arc to P2, 6 arc to P3, 7 arc to P0, 8 lift, 9 to start, 10 home | ☐ |
| `TL_JobErr` | 5 "drum radius not set" is in the list, check the text is under 5 | ☐ |
| `TL_Block` | 3 circle ccw, 4 circle cw. Part F adds 5 rectangle, 6 circle ccw one run, 7 circle cw one run | ☐ |

### C4 Switch positions for the drum

| Doing | Enable | AUTO | SCAN FROM HERE | PATH MODE | NO TURN | CLOCKWISE | job |
|---|---|---|---|---|---|---|---|
| hand test E1 | on | off | any | off | any | – | – |
| teach pose 8 | on | off | off | off | any | – | – |
| first scan E2 | on | on | off (on only above the axis, D2) | off | **on** | off, then on for E3 | 2 |
| turning scan E7 | on | on | off | off | off | off | 2 |

---

## D. The numbers, at the machine

| # | Number | How | ☐ |
|---|---|---|---|
| D1 | **safe Z** `"DB_Poses".rSafeZ` | the same side of the drum top as the cube's safe Z was of the cube top. The drum is taller than the cube: check that the number clears the drum top with the sensor on. The job drives "above P0" at this Z before it goes down | ☐ |
| D2 | **pose 8, the center** | jog the tool above the drum axis, any height. Poses screen: pose nr 8, TEACH. Lamp A57 on. Only X and Y of it are used (and RotY). Or SCAN FROM HERE on while the tool stands above the axis: nothing stored, the list begins with "above P0" | ☐ |
| D3 | **scan height** `Drum.rZ` | jog the tool to the drum's side at the height you want to scan, read actual Z, type it | ☐ |
| D4 | **radius** `Drum.rRadius` | drum radius + the distance from the drum surface to the tool point, tape measure. First test: a number that keeps the tool clear of the drum by a hand's width | ☐ |
| D5 | **RotZ offset** `Drum.rRotZOffset` | first test: **0**. P0 is then on the +X side of the center and the tool stands at RotZ −180 there. With the sensor: jog to the +X side at the scan height, turn RotZ until the sensor looks at the drum axis (along −X), read RotZ, offset = RotZ − 180 wrapped into −180 … 180 | ☐ |
| D6 | **room for the circle** | center X ± radius inside 0 … 4380, center Y ± radius inside 0 … 4230. Else job error 4 with `intMoveActive` = the first point outside, 4 … 7 | ☐ |
| D7 | **RotZ −180 and +180** | only with NO TURN off (E7): the turning circle begins at RotZ −180 and ends at +180. The cube ran with NO TURN at RotZ 0, so −180 was never driven. Hand test E6 before E7. With NO TURN on the circle keeps the RotZ of pose 8 and this row does not apply | ☐ |
| D8 | pose 2 start, pose 1 home | as for the cube, pose 2 must be taught (job error 1 without it) | ☐ |

---

## Start: what to press, key by key

The objects as in `HMI/SCREENS.html`. Before the first time: parts A to D done, B1 on the robot.

| # | Press / type | Where | See | ☐ |
|---|---|---|---|---|
| S1 | Enable on (`"DB_RobotMove".Man.xEnable`, Robot move screen), AUTO A12 **off** | | A46 "teach allowed" on | ☐ |
| S2 | jog the tool above the drum axis, pose nr **8** in A3, TEACH A4 | POSES box | A57 "center taught" on. Pose 2 (start) taught too, lamp at A40 | ☐ |
| S3 | scan height A53, radius A54, RotZ offset A55 (0 the first time) | DRUM box | the three numbers stand, retentive | ☐ |
| S4 | job **2** in A51 | JOB box | A52 shows "scan drum" (the mux takes it over while no job runs) | ☐ |
| S5 | switches: NO TURN A50 **on**, CLOCKWISE A56 off, SCAN FROM HERE A38 off, PATH MODE A49 off | CUBE and DRUM box | | ☐ |
| S6 | override A13 = **10** | JOB box | | ☐ |
| S7 | AUTO A12 **on** | JOB box | A48 lamp "automatic" within a second, M13 on | ☐ |
| S8 | **START** A14, one press | JOB box | A58 counts "to start", "above P0", "down to P0", "arc to P1" … "arc to P0", "lift", "to start", "home". A21 job error stays 0. The robot stands at pose 2, or home, when it is done | ☐ |
| S9 | it stops with an error | | read A21 job error, A26 engine error, A58 active, A30 block, A33 message id. ACK A47, fix the cause, START again from the start | ☐ |
| S10 | to stop it | | ABORT A15: the running move finishes, nothing follows. AUTO off does the same | ☐ |

A second scan: S8 again. The numbers and the switches stay (retentive), pose 8 stays until you teach it again.
With SCAN FROM HERE on instead of S2: jog above the axis, START; nothing is stored, the list begins with "above P0".
Nothing in TIA is pressed, nothing on the robot: program 20 is selected once and stays.

---

## E. The test, in this order

| # | Do | Done when | ☐ |
|---|---|---|---|
| E1 | **one quarter by hand.** Enable on, AUTO off. Robot stands at X0, Y0, Z0 (read actual, M21 … M25). Robot move screen: X = X0 + 300, Y = Y0 + 300, Z = Z0, RotZ and RotY = actual, radius M41 = 300, block 3, override 10, MOVE | the robot leaves along +X, bends toward +Y and ends at X0 + 300, Y0 + 300, a quarter circle counter-clockwise around the point X0, Y0 + 300. Done, error 0, block 3 in A30. Then X0, Y0, block 4, MOVE: the same arc back, clockwise. Error 4 = radius 0 typed. Engine error 1 = `intBlocksInPrg20` under 3. Engine error 10 = the echo line of the block is wrong (N130 / N170). A different end point or a robot fault 7e43 = the circle line is not the end-point form, B3 | ☐ |
| E2 | **the first scan, NO TURN on.** The Start section S1 to S8: job 2, NO TURN on, CLOCKWISE off, PATH MODE off, SCAN FROM HERE off, AUTO on, override 10, START | moves total 10 (9 without home), A58 counts through "to start" … "home", job error 0. RotZ (A32) stays at the value of pose 8 the whole time. Move 3 "down to P0" is the first block-2 move ever: engine error 10 right there = B1 not done on the robot | ☐ |
| E3 | **clockwise.** CLOCKWISE on, START | the same circle the other way round, block 4 in A30 | ☐ |
| E4 | **ABORT during an arc** | the running arc finishes on the circle, queue 0, job idle | ☐ |
| E5 | three scans in a row, then override 25 | | ☐ |
| E6 | **RotZ −180 and +180 by hand**, only before E7. AUTO off. Actual X Y Z, RotZ −180, block 1, MOVE. Then RotZ +180, MOVE. Then back to 0 | both arrive, error 0. If the robot refuses −180 or +180 (robot fault, engine error 21): tell me the message id, the turning circle then needs another start orientation, one line in the job | ☐ |
| E7 | **the turning circle, NO TURN off.** CLOCKWISE off, START | the same 10 moves, the tool at RotZ −180 at P0, −90 at P1, 0 at P2, +90 at P3, +180 back at P0, looking at the axis all the way with the right offset (D5). This is the scan for the sensor | ☐ |

The numbers to read when it stops: job error (A21), engine error (A26 or A29), active (A58), block (A30), and the
robot message id (A33). Those five go into `TEST/ERROR_LOG.md`.

| Job error | Means | Fix |
|---|---|---|
| 1 | pose 2 not taught, or pose 8 not taught without SCAN FROM HERE | teach it, A57 / A40 lamps |
| 2 | safe Z 0 | D1 |
| 4 | a point outside the limits, `intMoveActive` says which | D6, move the drum or shrink the radius |
| 5 | radius 0 | D4 |
| 6 | SCAN FROM HERE on, robot not referenced or comm not ok | reference, or the switch off and pose 8 |
| 10, engine 1 | a block above `intBlocksInPrg20`. **Seen 2026-11-06 at move 3** (block 2) with the field at 1 after a download: the active number showed 2, the move before, moves 1 and 2 had run | A4.3, the field to 4 (7 with part F), ACK, START |
| 10, engine 4 | radius 0 reached the engine | cannot come from the job, only by hand |
| 10, engine 10 at move 3 "down to P0" | block 2 ran, REAL 71 is not 2: N80 writes the echo into REAL 72 | B1 on the robot |
| 10, engine 10 after an arc | REAL 71 after the arc is not 3 or 4 | the echo line of the block, N130 / N170 |
| 10, engine 12 | an arc took longer than 60 s | override higher, or a slow motion set in the robot |
| 10, engine 21 | robot fault during the arc | A33 message id. 7e43 = end point not on the circle, B3 |

---

## F. The circle in one run, without a stop (2026-11-06)

In parts A to E the circle is four moves: the robot makes an exact stop at P1, P2 and P3, program 20 finishes and
starts four times, like the cube after every side. For a laser scan those stops sit in the middle of the surface.
The way without them: the whole circle is **one line** of program 20, given by its **center and the angle 360°**
(manual 6.16.3 "CenterAngle"). The PLC writes the center into `POSE 51` and 360 into `REAL 52`; the robot drives
from P0 once around the center and comes back to P0, no stop, one program start, "finished" once. On the screen
that is the PATH MODE switch (A49) with job 2. The cube is not touched by it: its path mode, block 5, is part 7 of
`AUTOMATIC_TODAY.md` and stays where it is.

What it needs: `FB_ScanDrumJob` v0.6, `FB_RobotMove` v0.7 (one line: blocks 6 and 7 get the 300 s of a path
instead of the 60 s of a single move), blocks 6 and 7 in program 20 (five lines each), `Cfg.intBlocksInPrg20` = 7,
three texts, one field. The line has no end pose, so the tool keeps the orientation it has at P0 for the whole
circle: in this mode the job sends the RotZ of pose 8 everywhere, NO TURN on or off. A tool that turns along the
circle comes later with the tangential control (`PROGRAM20.md` section 5).

### F1 TIA

| # | Do | ☐ |
|---|---|---|
| 1 | `PLC/FB_RobotMove.scl` v0.7 over the engine in TIA. If your engine is older than v0.6: `DB_RobotPath.scl` first, then the two pins at the end of the engine box, `io_aoudtPath := "DB_RobotPath".aoudtP` and `io_aorPathBlend := "DB_RobotPath".aorBlend` (`OB1_Stage2_call.scl`). Nothing else on the box changes | ☐ |
| 2 | `PLC/FB_ScanDrumJob.scl` v0.6 over v0.5. No new pin | ☐ |
| 3 | compile 0 errors, download. Type the recipe numbers again if `DB_Recipe` was re-initialized | ☐ |
| 4 | `"DB_RobotMove".Cfg.intBlocksInPrg20` := **7** | ☐ |
| 5 | PATH MODE off: one cube scan and one drum scan the single-move way, both exactly as before | ☐ |

### F2 The robot: blocks 6 and 7 between N180 (`END_IF` of block 4) and N190 (the run echo)

| # | New Block | Fields | ☐ |
|---|---|---|---|
| 6.1 | `IF` | operand 1: var **BOOL 56**, shown as "[56] PDO123.5 Extra 6" | ☐ |
| 6.2 | `SET_ABS_REL_POS` | [1] **CircAuxPos**, [2] Val, [3] **ABSOLUTE**. The center in POSE 51 is an absolute point. The basic state sets this back to relative at every start (manual 11.10.4), so the line stays inside the block | ☐ |
| 6.3 | `CIRC_VAR` | [1] **CenterAngle**, [2] **CCW**, [3] plane **XY**, [4] Center: var **POSE 51** ("[51] PDO114-118 Extra 1"), Angle: var **REAL 52** ("[52] PDO120 Extra 2"), [5] blending **Keep** | ☐ |
| 6.4 | `CALC_REALVAR` | REAL 71 := val **6** | ☐ |
| 6.5 | `END_IF` | under 6.4 | ☐ |
| 7.1 … 7.5 | the same five lines | gate **BOOL 57** ("[57] PDO123.6 Extra 7"), direction **CW**, echo **7** | ☐ |
| | N190 `REAL 72 := REAL 54` and N200 `END_PROG` stay the last two lines. Save | | ☐ |

### F3 The screen

| Nr | Object | Tag | Settings | ☐ |
|---|---|---|---|---|
| A49 | PATH MODE | `"DB_Recipe".xPathMode` | exists in the CUBE box. On = the drum circle in one run. For the cube leave it off until part 7 | ☐ |
| M42 | angle | `"DB_RobotMove".Man.udtReq.aorPar[2]` | Robot move screen, next to M41. deg, 0 … 360, 0 decimals. Only the hand test F4.1 needs it; the job writes its own 360 | ☐ |
| `TL_Block` | | | 5 "rectangle, 7 lin", 6 "circle ccw, one run", 7 "circle cw, one run" | ☐ |
| `TL_MoveDrum` | | | 11 "circle, one run" | ☐ |

### F4 The test

| # | Do | Done when | ☐ |
|---|---|---|---|
| F4.1 | **the full circle by hand.** Enable on, AUTO off. Robot at X0, Y0, X0 at least 300, any height, read actual. Robot move screen: X = X0 − 300, Y = Y0, Z and the rotations = actual, radius M41 = 300, angle M42 = **360**, block **6**, override 10, MOVE. The target field carries the center in this block, not an end point | the robot drives a full circle of radius 300 counter-clockwise around the point 300 mm in −X of it, leaves toward +Y, ends where it started, no stop. Engine error 1 = `Cfg` under 6. Engine error 4 = radius 0. Engine error 10 = line 6.4. Engine error 12 = longer than 300 s. A robot fault = the message id in A33: if the robot refuses the angle 360, tell me the id, two lines of 180 are the fallback, one constant in the job | ☐ |
| F4.2 | block **7**, the same | the same circle clockwise, leaves toward −Y | ☐ |
| F4.3 | **the scan in one run.** PATH MODE on, job 2, AUTO on, override 10, START | moves total **7** (6 without home): "to start", "above P0", "down to P0", **"circle, one run"**, "lift", "to start", "home". No stop on the circle. RotZ (A32) stays at the value of pose 8 | ☐ |
| F4.4 | CLOCKWISE on, START | the same, block 7 in A30 | ☐ |
| F4.5 | ABORT during the circle | the whole circle finishes, then the robot stops, queue 0 | ☐ |
| F4.6 | PATH MODE off, START | the four-arc scan of part E runs as before | ☐ |
