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
| NO TURN switch (A50), CUBE box next to PATH MODE | `"DB_Recipe".xNoTurn` | the test without turns, cube job v0.6. Until then: modify the bit online |
| lamp "repositioning required" in the engine box | `"DB_Robot".Status.xRepositionRequired` | on after every jog. When it is on, the next START goes through step 15. Error 15 today |
| lamp "program initialized" / "finished" | `"DB_Robot".Status.xPrgInitialized`, `.xPrgFinished` | the two states from which a start is allowed |
| field "engine error" in the JOBS box (A22) | `"DB_RobotJob".State.intEngineErr` | with job error 10 this is the number to read |

## 4. Checks, no change needed if right

| Object | Check |
|---|---|
| "Moving" lamp | was green while the robot stood in the morning, grey in the evening. Bound to `"DB_Robot".Status.xMoving`? If it is green at standstill again, the status map of PI bit "moving" is wrong and every move gives engine error 6 |
| "Scan From Here" and "Path Mode" switches | both off for the teach way. One of them looked on in the morning photo |
| Angle field | 0 for a cube along X and Y. The angle table is in `TEST/CHECKLIST_CUBE_AUTO.md` stage D0. A hint text "0 = +X" next to the field helps |

## 5. The screen as it should be

`SCREENS.html`, Automatic screen A1 … A50, every object with its tag and a note. The boxes: POSES (rows 1 … 50,
teach by number), CUBE (numbers, three teach buttons with lamps, ready, two go-to buttons, enable lamps, the three
switches), JOB (start, abort, ACK, override, lamps, the numbers), engine (busy, done, error, step, error, run nr,
block). The JOB box and the engine box are the same on the Robot move screen.
