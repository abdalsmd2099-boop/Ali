# Next steps – from the single move to the first automatic cube scan

2026-10-30. The single move works, stage 1 is passed. Everything below is already in the PLC and on the screens. One line per step, in order. Tick each one. Stop and tell me at the first thing that does not match.

## A. Three checks before anything (10 minutes)

| # | Check | Where | Must be | ☐ |
|---|---|---|---|---|
| 1 | `FC_MoveSelect` box in OB1 has **seven** inputs (`i_xHand`, `i_xAuto`, `i_udtMan`, `i_xManStart`, `i_udtQueueHead`, `i_xQueueHeadValid`, `i_udtMoveState`) | OB1, the network before `IDB_RobotMove` | seven. If three: regenerate from `FC_MoveSelect.scl`, wire as `OB1_Stage2_call.scl` lines 57–65 | ☐ |
| 2 | after a single move, the error field | stage 1 screen | 0. If 13: program 20 is missing the last line `REAL[72] := REAL[54]`, `PROGRAM20.md` section 3 line 11 | ☐ |
| 3 | `"DB_RobotMove".Cfg.intBlocksInPrg20` | watch table | 2 if program 20 has blocks 1 and 2, else 1 | ☐ |

## B. Teach the seven poses (about one hour)

Enable ON, AUTO OFF. Jog with the old jog buttons. Stage 2 screen "teaching and automatic mode".

| # | Do | Done when | ☐ |
|---|---|---|---|
| 4 | jog to the home position. Pose Nr = 1, TEACH | row 1 valid lamp on, numbers = actual pose | ☐ |
| 5 | jog to the start position above the cube. Pose Nr = 2, TEACH | row 2 valid | ☐ |
| 6 | jog to the approach point, near the first corner, safe height. Pose Nr = 3, TEACH | row 3 valid | ☐ |
| 7 | jog to corner 1, tool facing side 1 (the side from corner 1 to corner 2). Pose Nr = 4, TEACH | row 4 valid | ☐ |
| 8 | corner 2 facing side 2 → 5, corner 3 facing side 3 → 6, corner 4 facing side 4 → 7 | rows 5, 6, 7 valid | ☐ |
| 9 | Safe Z = a height above the cube, higher than any corner Z | field set | ☐ |
| 10 | check one pose: on the stage 1 screen type the numbers of row 4, MOVE | the robot stands on corner 1 | ☐ |
| 11 | power the PLC off and on | rows 1 to 7 still valid | ☐ |

## C. First automatic scan, in simulation (20 minutes)

| # | Do | Done when | ☐ |
|---|---|---|---|
| 12 | old screen: simulation ON (PO 1.9). Stage 2 screen: override 10, AUTO ON, Enable ON | AUTO lamp on | ☐ |
| 13 | press START | jBusy on, Queue Waiting counts 13 → 0, mActive goes 1 … 14, jdone on, JobErr 0 | ☐ |
| 14 | if jErr: read JobErr_id. 1 = a pose not taught, 2 = safe Z, 3 = engine busy or in error (RESET on the stage 1 screen, then START), 4 = a pose outside limits (see mActive), 10 = engine error (see the stage 1 error field) | fixed, START again | ☐ |

## D. First real scan (30 minutes)

| # | Do | Done when | ☐ |
|---|---|---|---|
| 15 | simulation OFF, override 10, START. Hand on the stop | start → approach → corner 1 → side 1 → turn → side 2 → … → corner 1 → lift → start → home. jdone | ☐ |
| 16 | START again, press ABORT (the Stop button of the stage 2 screen) in the middle of side 2 | the running move finishes, queue 0, job idle. START: begins with "to start" | ☐ |
| 17 | five scans in a row | jdone each time, no error | ☐ |

**Step 17 done = stage 2 automatic works.** Tell me, and we do the alarm view and the operator screen (`ROADMAP.md` 2.4 and 2.5), then stage 3.

## E. Not today

Drum, laser sensor, StepChain framework. They are stage 3 and need new blocks. `ROADMAP.md` section 3.
