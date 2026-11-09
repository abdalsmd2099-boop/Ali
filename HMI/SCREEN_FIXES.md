# Screen fixes – what is wrong on the real screen, seen 2026-11-04

From the photos of the "teaching and automatic mode" and "Scan functions" screens. The object numbers
are those of `SCREENS.html` (same folder). Fix them in this order: the number fields first, because a
wrong number costs an hour of searching at the machine.

## 1. Fields on the wrong tag

| Screen object | Shows today | Must be on | Why it matters |
|---|---|---|---|
| JOBS box "Job error" | the engine's number (15, 11) | `"DB_RobotJob".State.intErrId` (A21) | today it said 15 while the job's own error was 10. Put the engine number in a second field "engine error" on `"DB_RobotJob".State.intEngineErr` (A22) |
| JOBS box "JobError" text | "-------" | text list TL_JobErr on `"DB_RobotJob".State.intErrId`, entries 0 … 6 and 10 | 10 "engine error, see engine error" has no entry today |
| Func_Move "Nr block" | 90, 0 | `"DB_RobotMove".State.intBlockActive` (A30) | today it shows the engine step. 90 is step 90, the error step |
| Func_Move "Move Step" | "Default" | text list TL_Step on `"DB_RobotMove".State.intStep` (A28), entries 0 5 10 15 20 30 40 50 60 90 | today it shows block 1 through the step list, so "Default" |
| Func_Move "Err nr" | "no error" while the engine is in error | text list TL_Err on `"DB_RobotMove".State.intErrId` (A29), entries 0 … 16, 20 … 22 | the one number you need first when it stops |
| first screen "Job error" | 0 while the error lamp is on | `"DB_RobotJob".State.intErrId` | same as the first row |

## 2. Text lists

| List | Wrong today | Right |
|---|---|---|
| TL_JobErr | 4 = "error at start, or engine busy" | 3 = engine busy or in error at start. 4 = a pose outside the limits, see MoveActive. 5 = drum radius not set. 6 = scan from here not possible. 10 = engine error, see engine error |
| TL_Step | no entry 0 ("Default") | 0 = idle, 5 = check, 10 = write, 15 = program stop, 20 = settle, 30 = start, 40 = wait running, 50 = wait finished, 60 = end, 90 = error |
| TL_Err | 15, 16 missing? | 15 = program stop not taken, 16 = path longer than 7. Full list in `GOALS.md` 5.5 |

## 3. Objects to add

| Object | Tag | Why |
|---|---|---|
| NO TURN switch (A50), CUBE box next to PATH MODE | `"DB_Recipe".xNoTurn` | the test without turns, cube job v0.6 and drum job v0.5 (2026-11-06, one switch for both). Until then: modify the bit online |
| lamp "repositioning required" in the engine box | `"DB_Robot".Status.xRepositionRequired` | on after every jog. When it is on, the next START goes through step 15. Error 15 today |
| lamp "program initialized" / "finished" | `"DB_Robot".Status.xPrgInitialized`, `.xPrgFinished` | the two states from which a start is allowed |
| field "engine error" in the JOBS box (A22) | `"DB_RobotJob".State.intEngineErr` | with job error 10 this is the number to read |

## 4. Checks, no change needed if right

| Object | Check |
|---|---|
| "Moving" lamp | was green while the robot stood in the morning, grey in the evening. Bound to `"DB_Robot".Status.xMoving`? If it is green at standstill again, the status map of PI bit "moving" is wrong and every move gives engine error 6 |
| "Scan From Here" and "Path Mode" switches | both off for the teach way. One of them looked on in the morning photo |
| Angle field | 0 for a cube along X and Y. The angle table is in `TEST/CHECKLIST_CUBE_AUTO.md` stage D0. A hint text "0 = +X" next to the field helps |

## 5. From the OB1 screenshots of 2026-11-05 (MachineReady.docx)

| Screen object | Today | Must be |
|---|---|---|
| RESET button of the old robot screen | writes `"DB_App".Cmd.Reset` | writes `"DB_RobotHand".Btn.xResetRobot`, momentary. `FB_MachineReady` v0.2 passes it to `Cmd.Reset` and is the one writer |
| automatic switch `"DB_RobotHand".xAutomatic_from_Screen` | a rung in OB1 makes `Cmd.Automatic` from it | delete the switch and the rung. `FB_RobotMode` writes `Cmd.Automatic`, a lamp on `"DB_Robot".Status.xAutomatic` shows it. With the switch off every move ends in engine error 5 |
| `"DB_RobotHand".xHandModeJust` | part of that rung | goes with the rung |
| MOVE button `"DB_RobotMove".Man.xStart` | also copied into `Cmd.Start` by a rung | the rung goes. The button stays on `Man.xStart`, the engine takes the edge through `FC_MoveSelect` |
| machine ready fields | `"DB_App".MissingNo`, `.MissingText`, `.ReadyState`, `.ReadyError`, `.MachineReady` | as they are. `MissingText` is the one line the operator needs when nothing starts |

## 6. From the photos of the first drum start, 2026-11-06 (three screens)

Right now: the "Err nr" field of Func_Move shows the engine text ("Block not in the right prog" = engine error 1),
so the row of section 1 for it is done. The DRUM box exists with "Drum move" on `TL_MoveDrum` ("Above P0" = 2),
the NO TURN switch is there ("No Rotate in RZ", on), the job field, the active job, the drum numbers.

| Screen object | Shows today | Must be | Why it matters |
|---|---|---|---|
| "block prog 20" on the first screen (M40) | **1** | **4** today, 7 with blocks 6 and 7. Type it there, or modify `"DB_RobotMove".Cfg.intBlocksInPrg20` in the watch table | the drum stopped at move 3 (block 2) with engine error 1 because of this number. A download of `DB_RobotMove` sets the start value again: v0.2 in `PLC/` has 4 |
| JOBS box "JobError" text (Scan functions screen) | "drum radius not set" while the number is 10 | text list `TL_JobErr`: 5 = "drum radius not set", **10 = "engine error, see Seq_Err"** | the text of 5 sits under 10, the operator reads a radius problem that does not exist |
| "Job error" number on the teaching screen | 1 while the value is 10 | `"DB_RobotJob".State.intErrId`, field wide enough for two digits | shows the first digit only, or sits on another tag |
| "Center taught" lamp in the DRUM box (A57) | off while row 8 "Drum C" is valid in the POSE table | `"DB_Poses".aoxValid[8]` | the lamp that says the drum can start |
| JOBS box "MoveActive" text during a drum job | "approch", the cube text for 2 | leave it, it is `TL_Move` for the cube. The drum text is "Drum move" in the DRUM box (A58). Or give the two fields a visibility on `"DB_RobotJob".intJobActive` = 1 / = 2 | two texts for one number confuse |
| Func_Move "Nr block" | 90 | still the engine step, row of section 1 | 90 is step 90, the error step, not a block |
| "Center taught", "Teach Home", "Teach Start", "Teach Corner1" lamps on the Scan functions screen | all off while the poses are valid | the lamps of the teaching screen are right; copy their tags | |

What the three screens proved: the robot stood at X 2457, Y 1454, Z 500 after move 2. That is exactly "above P0"
for center 2500 / 1700, radius 250, offset 100 and safe Z 500: the drum job computes the right point. RotZ 0 =
the RotZ of pose 8, NO TURN on.

## 7. The screen as it should be

`SCREENS.html`, Automatic screen A1 … A50, every object with its tag and a note. The boxes: POSES (rows 1 … 50,
teach by number), CUBE (numbers, three teach buttons with lamps, ready, two go-to buttons, enable lamps, the three
switches), JOB (start, abort, ACK, override, lamps, the numbers), engine (busy, done, error, step, error, run nr,
block). The JOB box and the engine box are the same on the Robot move screen.
