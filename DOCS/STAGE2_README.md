# Stage 2 – teach, queue, cube job, alarms

2026-10-28. Everything of `STAGE2_PLAN.md` steps 4 to 8 and the alarm part of 10 is written. Import only after stage 1 passed `TEST_SHEET.md` part B. The move engine `FB_RobotMove` is not touched.

## 1. Files and import order

| # | File | Block | Needs |
|---|---|---|---|
| 1 | `UDT_MoveQueue.udt` | `UDT_MoveQueue` | UDT_MoveReq |
| 2 | `UDT_JobState.udt` | `UDT_JobState` | – |
| 3 | `DB_MoveQueue.scl` | `DB_MoveQueue` | UDT_MoveQueue |
| 4 | `DB_RobotJob.scl` | `DB_RobotJob` | UDT_MoveReq, UDT_JobState |
| 5 | `FB_Teach.scl` | `FB_Teach` | UDT_Pose |
| 6 | `FB_MoveQueue.scl` | `FB_MoveQueue` | UDT_MoveQueue |
| 7 | `FC_MoveSelect.scl` | `FC_MoveSelect` **v0.2, replaces v0.1** | UDT_MoveReq, UDT_MoveState |
| 8 | `FB_ScanCubeJob.scl` | `FB_ScanCubeJob` | UDT_Pose, UDT_MoveReq, UDT_JobState, FC_PoseCheck |
| 9 | `FC_RobotAlarms.scl` | `FC_RobotAlarms` | the three state UDTs |
| – | `FB_ActRobot.scl` | **do not import yet.** Step 9, needs the framework | |
| – | `OB1_Stage2_call.scl` | wiring reference | |

After generating `FC_MoveSelect` v0.2, the box in OB1 shows four grey pins: right click → "Update block call", then wire them.

## 2. OB1 after stage 2

Five robot networks, in this order. The order is the data flow of one cycle.

| # | Network | Box | New |
|---|---|---|---|
| 1 | robot teach | `FB_Teach`, instance `IDB_Teach` | yes |
| 2 | robot job | `FB_ScanCubeJob`, instance `IDB_ScanCubeJob` | yes |
| 3 | move queue | `FB_MoveQueue`, instance `IDB_MoveQueue` | yes |
| 4 | robot move | `FC_MoveSelect` v0.2 + `IDB_RobotMove` | four new pins on the selector, engine unchanged |
| 5 | robot alarms | `FC_RobotAlarms` | yes |

Pins in `OB1_Stage2_call.scl`. OB30 does not change.

## 3. How it fits together

```
hmi START ──> FB_ScanCubeJob ──push──> FB_MoveQueue ──head──> FC_MoveSelect ──req──> FB_RobotMove ──> robot
                   ^                        ^                       |                    |
                   │ count                  │ pop = done            │ auto / hand        │ state
                   └────────────────────────┴───────────────────────┴────────────────────┘
```

- The job checks the 7 poses and safe Z, builds 13 or 14 moves, checks every pose, clears the queue, pushes one move per cycle, then waits until the queue is empty and the engine idle.
- The queue pops on the engine's done pulse.
- The selector hands the head to the engine with a level start; the engine takes the edge. One cycle gap after each move keeps the edge clean.
- Abort, auto switch off, or an engine error: the queue is cleared, the running move finishes, nothing follows.

## 4. HMI

`HMI_STAGE2.md`: screen "Poses" (teach), screen "Scan cube" (auto switch, start, abort, state, queue), three text lists, 32 discrete alarms.

## 5. RobotMonitor

Nothing new for the cube job: it uses block 1 only. Blocks 2 to 4 are in `PROGRAM20.md` for step 8, each one is optional.

## 6. Tests, part C (after part B passed)

Order matters. Stop at the first line that fails and tell me its number.

| # | Test | Pass when | Result |
|---|---|---|---|
| C0 | stage 1 still works after the import: hand mode, one move from the stage 1 screen | as in part B test 3 | |
| C1 | teach pose 2 at the real start position (auto off, enable on) | row 2 valid, numbers = actual pose. Power off and on: still there | |
| C2 | teach refused: auto on, press TEACH | refused lamp, nothing changes | |
| C3 | teach poses 3 to 7 at the cube: approach, then the four corners, tool facing the side that starts at each corner. Set safe Z above the cube | rows 3–7 valid | |
| C4 | job with pose 5 cleared (CLEAR on 5), auto on, START | job error 1, alarm 23, nothing moves. Teach 5 again | |
| C5 | job with safe Z = 0, START | job error 2, alarm 24 | |
| C6 | **first full scan in simulation** (PO 1.9 on), auto on, override 10, START | job runs: moves 1 → 13 (14 with home), queue counts down, done. No error | |
| C7 | **first real scan** at override 10 | robot: start → approach → corner 1 → side 1 → turn → … → corner 1 → lift → start (→ home). Done. Moves done = total | |
| C8 | ABORT in the middle of side 2 | the running move finishes, queue 0, job idle. START again: begins with "to start" | |
| C9 | AUTO off in the middle | same as C8 | |
| C10 | RobotMonitor takes access in the middle | engine error 22, job error 10, alarms 22 and 27. Give access back, RESET on the stage 1 screen, ABORT, START: scan runs | |
| C11 | hand MOVE button while auto is on | nothing happens | |
| C12 | five scans in a row | every one done, no error, moves done = total each time | |
| C13 | blocks 2 to 4 in program 20 (optional, step 8): `Cfg` = 4, single move with block 2, 3, 4 from the stage 1 screen | echo = block, circles need a radius (error 4 without) | |

## 7. Not in this package

| Step | Why | What is prepared |
|---|---|---|
| 9, framework | `FB_SeqMechanism`, `FC_SeqMngmt` and the finished-bit wiring are not in the folder | `FB_ActRobot.scl`: one block for step numbers 301–350 and 351, pushes into the same queue. Wire its `o_xStepFinished` where the ram's finished bit goes |
| 10, rest | stop behaviour of program 20 and the lift-after-stop rule need the machine | alarms are done; `FB_StopRecord` and the automatic 351 after a stop come after C8–C10 are seen |
| 11, laser | no sensor chosen | `xStopOnSensor`, `i_xSensorStop`, `udtPoseAtEnd` are in place |
