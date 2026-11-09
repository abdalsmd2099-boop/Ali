# Roadmap – finishing stage 2, and what stage 3 is

2026-10-29. Where we stand, what closes stage 2, what stage 3 looks like. Every step points to the file that already describes it. Nothing here changes `FB_RobotMove`, the UDTs, OB30 or the program 20 contract.

## 0. Where we stand today

| Layer | State |
|---|---|
| Bus, status map, handshake | works on the machine |
| `FB_RobotMove` v0.3, program 20 block 1 | the robot starts and runs program 20. Open: it does not arrive at the typed target, traced tomorrow with `TEST_SHEET.md` section C |
| Stage 2 blocks (teach, queue, job, alarms) | imported and wired in OB1. `FC_MoveSelect` still v0.1, so auto cannot reach the engine yet |
| HMI | stage 1 screen and stage 2 screen built, not yet tested |
| Program 20 | has placeholders for blocks 3 and 4 and no run-number echo line. Rebuild from `PROGRAM20.md` section 3 |

## 1. Finish stage 1 first (one machine day)

| # | Do | Where | Done when |
|---|---|---|---|
| 1 | the six corrections | `REVIEW_2026-10-29.md` section 2 | compiles, `Cmd.Enable` has one writer, `FC_MoveSelect` v0.2 |
| 2 | program 20 as a block list, blocks 1 and 2, echo lines, end of program, `Cfg` = 2 | `PROGRAM20.md` section 3 | saved on the robot |
| 3 | pose check | `TEST_SHEET.md` section C | robot arrives at the typed target within 1 mm |
| 4 | tests B4 to B8 | `TEST_SHEET.md` part B | written in the sheet |

Stage 1 is closed when a typed pose is reached, twenty times in a row, with no error.

## 2. What finishes stage 2

Stage 2 is done when **one button scans the cube from taught poses, five times in a row, and every stop ends clean with a numbered alarm.** Four pieces are missing, in this order.

### 2.1 Teach the poses (plan step 4)

The blocks and the screen exist. What is left is doing it on the machine.

| # | Do | Done when |
|---|---|---|
| 1 | jog the robot to the start position with the old jog buttons, pose nr 2, TEACH | row 2 valid, numbers = actual pose |
| 2 | power off and on | row 2 still there (retain) |
| 3 | pose 1 home, pose 3 approach, poses 4 to 7 the corners, tool facing the side that starts at each corner (`HMI_STAGE2.md` section 1) | rows 1 to 7 valid |
| 4 | safe Z above the cube | `rSafeZ` set |
| 5 | check every taught pose: type its numbers on the stage 1 screen, MOVE | the robot stands on it |

Tests C1 to C3 of `STAGE2_README.md`.

One small addition makes step 5 one click: a "GO" button per pose row on the Poses screen. HMI script on press: copy `"DB_Poses".aoudtP[n]` into `"DB_RobotMove".Man.udtReq.udtPose`, set `Man.xStart` 1, release 0. Only works in hand mode with Enable on, the engine refuses otherwise. No PLC change.

### 2.2 Drive in automatic (plan steps 5 to 7)

Everything is in OB1. Needs `FC_MoveSelect` v0.2 (correction 2.3 of the review) and the AUTO switch.

| # | Do | Done when |
|---|---|---|
| 1 | C4, C5: job refuses a missing pose and safe Z = 0 | job error 1 and 2, alarms 23 and 24 |
| 2 | C6: first full scan in simulation (PO 1.9 on) | 14 moves, queue counts down, done |
| 3 | C7: first real scan at override 10 | start → approach → corners and sides → lift → start → home |
| 4 | C8 to C11: abort, auto off, access taken, hand MOVE in auto | every case ends clean, next START runs |
| 5 | C12: five scans in a row | done each time, moves done = total |

### 2.3 Program 20 blocks 2 to 4 (plan step 8)

Block 2 (slow linear) is already in the stage 1 program. Blocks 3 and 4 (circle) are only needed for the drum, so they can move to stage 3. If you want them now: `PROGRAM20.md` section 4, then `Cfg` = 4, test C13.

### 2.4 Faults and recovery (plan step 10)

`FC_RobotAlarms` is in OB1. What is left:

| # | Do | Done when |
|---|---|---|
| 1 | HMI: alarm view on the discrete alarms of `"DB_RobotJob".aoxAlarm`, texts from `HMI_STAGE2.md` section 4 | every engine, job and robot error shows as a line with a number |
| 2 | provoke each case of `STAGE2_PLAN.md` step 10 once: comm loss, robot fault, E-stop, stop in the middle | every one recovers with RESET, ABORT, START |
| 3 | stop in the middle of a side: the next START must begin with the lift move before anything else | no collision path after a stop. If the job does not do it yet, one more move at the front of the list in `FB_ScanCubeJob` |

### 2.5 Screens that are easy to drive

Today there are three screens that grew with the steps: the old robot screen, "Robot move" (stage 1) and "teaching and automatic mode" (stage 2). For the operator, make **one** screen "Robot", and keep the stage 1 screen as "Service, single move" for testing blocks.

```
+---------------------------------------------------------------------------------------------+
|  HAND / AUTO (switch)   ENABLE (switch)   ready  referenced  comm  fault   msg [ 0 ]  [RESET] |
+----------------------------------+---------------------------------+------------------------+
|  POSES                           |  JOB  scan cube                 |  ACTUAL POSE           |
|  nr  name      valid  [GO]       |  override [ 10 ] %              |  X Y Z RotZ RotY       |
|   1  home       (o)   [GO]       |  [ START ]   [ ABORT ]          |                        |
|   2  start      (o)   [GO]       |  step  [ run ]                  |  ENGINE                |
|   3  approach   (o)   [GO]       |  move  [ 5 ] / [ 14 ]  side 1   |  busy done error       |
|   4  corner 1   (o)   [GO]       |  queue [ 9 ]                    |  step  error           |
|   5..7 corners  (o)   [GO]       |                                 |                        |
|  pose nr [ 2 ]  [TEACH] [CLEAR]  |                                 |                        |
|  safe Z [ 1500 ]                 |                                 |                        |
+----------------------------------+---------------------------------+------------------------+
|  ALARMS  (last three lines, number and text)                                                |
+---------------------------------------------------------------------------------------------+
```

Rules that make it easy, all done in the HMI with existing tags, no PLC change:

| Rule | How |
|---|---|
| a button that cannot work is grey | GO, TEACH, CLEAR enabled only when `Man.xEnable` AND NOT `Man.xAuto` AND NOT `State.xBusy`. START only when `Man.xAuto` AND `Man.xEnable` AND NOT `JobState.xRunning`. ABORT only when running |
| one mode switch, not two | HAND/AUTO writes `"DB_RobotJob".Man.xAuto`. Hand = off |
| the operator never types a pose | GO and TEACH replace the target fields. Typing stays on the service screen |
| numbers are whole mm and deg | all pose fields 0 decimals |
| the state is one word | job step and engine step through the text lists `TL_JobStep`, `TL_Step`, `TL_Move`, never a bare number |
| an error is one line | alarm view, number and text, from `aoxAlarm`. RESET on the top bar clears the engine error, ABORT clears the job error |
| the old robot screen stays for service | enable, reset, automatic, simulation, pause, stop, homing, jog. Not for the operator |

## 3. Stage 3 – what it looks like

Stage 3 turns the working cube scan into a machine function: more jobs, the sensor, the framework, and production features. Four parts, each a plan of its own when it starts.

### 3.1 The StepChain framework around it (plan step 9)

`FB_Mode`, `FB_Jobs`, `FB_OperBox`, `FB_SeqMechanism` from the drum press, as in `GOALS.md` 4.2. The robot becomes one mechanism of the machine with action numbers 301 to 350 (`FB_ActRobot.scl`, step 9, already written). The A1 Automatic screen then looks like the drum press screen (`image (72).png`): Start, Stop, step by step, tap to step, job selection. `FB_ScanCubeJob` stays as the recipe source or becomes a recipe of action numbers, decided then. Needs the chain blocks (FC40 is still missing from the drum press project).

**Looks like:** the operator selects "scan cube" on A1, presses Start, the chain runs the robot through `FB_ActRobot` → queue → engine. Standby means the robot never moves.

### 3.2 The laser sensor (plan step 11)

Two moves become sensor moves: approach until 10 mm, and the corner search. The request already carries `xStopOnSensor`, the engine already has `i_xSensorStop`. Decision when the sensor is chosen: PLC stop with the pose from PI 7–11, or MOVIKIT touchprobe / MEASURE on the robot. Then the job builds the sides from found corners instead of taught ones.

**Looks like:** the operator teaches only start and a rough approach. The robot finds the cube.

### 3.3 The drum job

A second job block, `FB_ScanDrumJob`, next to the cube job. Same queue, same engine. It uses blocks 3 and 4 of program 20 (circle ccw and cw, radius in `REAL[51]`) and poses for the drum: center, radius, tool offset (the start point is computed, 2026-11-02). The job selection on A1 picks cube or drum.

**Looks like:** nothing in the engine or the bus changes. One new job block, two program 20 blocks, one HMI job entry.

### 3.4 Production features

| Feature | Where |
|---|---|
| pose sets per part (cube sizes, drums) | `DB_Poses` 1..50 split into ranges, or a second DB, the job gets a set number |
| counters and times | scans done, last scan time, per job, in `UDT_JobState` at the end (new fields only) |
| scan results | if the sensor delivers numbers: a results DB per side, trend on the HMI |
| user levels | service screens admin only |
| alarm history | WinCC Unified alarm log on the discrete alarms |

## 4. Order from here

```
tomorrow      stage 1 closed            REVIEW corrections, PROGRAM20 block list, pose check, B4–B8
then          2.1 teach                 C1–C3, GO buttons
then          2.2 automatic             C4–C12
then          2.4 faults                alarm view, provoked cases
then          2.5 operator screen       one screen, grey buttons
stage 2 done  five scans in a row from one button
stage 3       3.1 framework  →  3.2 sensor  →  3.3 drum  →  3.4 production
```

2.3 (circle blocks) goes wherever the drum starts.
