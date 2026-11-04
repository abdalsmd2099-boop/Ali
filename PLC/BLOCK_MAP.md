# Block map – every block, every pin, every writer

2026-11-01, second edition. One map for both states of the project. Every row says where the item is:

- **TIA** – in the PLC, runs tomorrow for the cube.
- **folder** – only in this folder, written 2026-10-31 and 2026-11-01 for the drum and the hand screen. Part 6 of `AUTOMATIC_TODAY.md`, not needed tomorrow.

Sources: your OB30 (`the actual ob30 of today.txt`), your OB1 status of 2026-10-29, the `.scl` and `.udt` files of this folder.

Part A is the map. Part B is the one-writer rule, the bug, and the cross-reference check in TIA. Part C is the short list of what is folder-only.

---

## A. The map

### A1. The two organization blocks

Network numbers of OB1 as in your status of 2026-10-29. If yours differ, the order is what counts.

```
OB30  every 10 ms, the bus                          OB1  main cycle, the decisions
------------------------------------------------    ------------------------------------------------------------
NW1  handshake                 -> DB_App.HS         NW5   slot 17 MOVIRUN device, own handshake (yours)
NW2  "DB_MOVIKIT_Robotics-5AX" module 1, hw 278     NW7   FB_MachineReady (yours). Its Enable output is not wired
NW3  "DB_MOVIKIT_ROBOTICS_Module2" module 2, hw 272 NW8   Cmd.Enable, the one writer    -> DB_App.Cmd.Enable
NW4  FC_RobotStatusMap         -> DB_Robot.Status   NW9   IDB_Teach                      -> DB_Poses, DB_RobotJob.Teach.xTaught/xRefused/intLastTaught
                                                    NW10  IDB_ScanCubeJob                -> DB_RobotJob.Push, DB_RobotJob.State
                                                    NW11  IDB_MoveQueue                  -> DB_MoveQueue
                                                    NW12  FC_MoveSelect, then IDB_RobotMove, one network
                                                             FC_MoveSelect              -> DB_RobotMove.Req, .xReqStart
                                                             IDB_RobotMove              -> DB_RobotMove.State, DB_App.Srl.*, DB_App.Cmd.Start/ProgramNumber/Override
                                                    NW13  FC_RobotAlarms                 -> DB_RobotJob.aoxAlarm   (if you placed it. Nothing tomorrow needs it)

folder, part 6: between NW10 and NW11 come IDB_ScanDrumJob and FC_JobMux. NW10 then writes DB_RobotJob.Cube, FC_JobMux writes Push and State.
folder, step 1a: IDB_RobotMode (2026-11-02) goes after NW8, before NW9. The one writer of DB_App.Cmd.Automatic.
folder, later:  the hand networks of SCREENS.html section 5 (jog, home) go after NW8, before NW9.
```

The order in OB1 is the data flow of one cycle: teach → job pushes → queue stores → select picks the head → engine moves → alarms. OB30 only moves words to and from the bus and maps the status. OB30 delivers, OB1 decides.

**Three facts of your OB30 that matter tomorrow** (they differ from `OB30_complete.scl`, keep them as they are):

| Where | Your OB30 | Consequence |
|---|---|---|
| NW2 `start` | `Cmd.Start AND ReadyToStart AND MachineOn` | `"DB_App".ReadyToStart` and `"DB_App".MachineOn` must be TRUE, else the start never reaches the robot: engine error 11 |
| NW2 `stop` | `"DB_App".Cmd.Stop` | engine v0.5 (2026-11-03): becomes `"DB_App".Cmd.Stop OR "DB_RobotMove".State.xPrgStop`, the engine's program stop that cancels the repositioning after a jog. `Cmd.Stop` keeps the old screen as its one writer, `State` keeps the engine |
| NW3 `enable` | `"DB_App".Cmd.Enable` (file: TRUE) | with the Enable switch off, module 2 stops talking: comm ok goes off, alarm 29 shows, teaching is refused. Normal. Enable stays on all day |
| NW2 segments | `REAL_TO_INT(...)` on the seven path segments | not used by the engine. Nothing to do |

**FC_MoveSelect:** your OB1 of 2026-10-29 had v0.1 with three pins. That version has no auto branch, START would do nothing. Step 1 of `AUTOMATIC_TODAY.md` makes it v0.2 with seven pins. Do it first.

### A2. UDTs (types)

| UDT | Where | Used by | What |
|---|---|---|---|
| `UDT_Pose` | TIA | everything | X Y Z RotZ RotY, Real |
| `UDT_RobotStatus` | TIA | `DB_Robot.Status` | the PI words of module 1 as named bits and values, plus `xHsAlive`, `xM2CommOK`, `xCommOK` |
| `UDT_MoveReq` v0.2 in TIA, v0.3 in the folder (path mode, part 7) | TIA | `DB_RobotMove.Man.udtReq`, `.Req`, queue entries, `DB_RobotJob.Push.udtMove` | one move: block, pose, pose2, blend, 3 parameters, override, sensor stop, id. v0.3: `aoudtPath[1..7]`, `intPathLen` at the end, 0 = single move |
| `UDT_MoveState` | TIA | `DB_RobotMove.State` | busy, done, error, error number, step, run nr, id, block, pose at end, message id. v0.2 (folder, 2026-11-03): `xPrgStop` at the end, the engine's program stop for OB30 NW2 |
| `UDT_MoveQueue` | TIA | `DB_MoveQueue.Q` | the FIFO: 50 entries, head, tail, count |
| `UDT_JobState` | TIA | `DB_RobotJob.State` | running, done, error, error number, step, moves total / done / active, engine error |
| `UDT_JobOut` | **folder** | `DB_RobotJob.Cube`, `.Drum` after part 6 | one job's pushes and state together: `xPushReq`, `udtPush`, `xClear`, `udtState`. So `FC_JobMux` copies one of them |

### A3. DBs (data) and who writes what

| DB | Where | Retain | Written by | Read by |
|---|---|---|---|---|
| `DB_App` | TIA, yours | – | `HS`, `HS_Stall`, `HS_Alive`, `HS_Count` by OB30 NW1. `Cmd.Enable` by OB1 NW8. `Cmd.Start`, `Cmd.ProgramNumber`, `Cmd.Override`, `Srl.Pose[1..5]`, `Srl.Par[1..4]`, `Srl.Gate[1..8]`, `Srl.GateEnable` by `FB_RobotMove` through InOut pins. `Srl.In[1..4]`, `Srl.CommOK` by the module 2 FB. `Cmd.Reset`, `.Automatic`, `.Simulation`, `.Pause`, `.Stop`, `.Home_*`, `.Jog_*` by the old robot screen. `ReadyToStart`, `MachineOn`, `Home.*` by your logic | OB30 NW2, NW3 |
| `DB_Robot` | TIA | – | `FC_RobotStatusMap`, OB30 NW4. The only writer of `Status` | engine, teach, alarms, HMI |
| `DB_RobotMove` | TIA | `Cfg` only | HMI → `Man.*`, `Cfg.intBlocksInPrg20`. `FC_MoveSelect` → `Req`, `xReqStart`. `FB_RobotMove` → `State` | engine, job, queue, select, alarms, HMI |
| `DB_Poses` | TIA | yes | `FB_Teach` → `aoudtP[n]`, `aoxValid[n]` through InOut pins on TEACH / CLEAR. The Poses screen may type into the same fields. HMI → `rSafeZ` | job, HMI |
| `DB_Recipe` | **folder**, step 1b | yes | HMI → `Cube.rLength`, `.rWidth`, `.rAngle` (2026-11-02), `Drum.rZ`, `.rRadius`, `.rStartAngle` (not used since v0.3), `.rRotZOffset`, `.xClockwise`, `xFromHere` (the switch "scan from here", both jobs), `xPathMode` (v0.2, the switch "path mode", both jobs, part 7). A DB of its own so `DB_Poses` in the PLC is never touched and the taught poses stay | `FB_ScanCubeJob`, `FB_ScanDrumJob`, `FB_CubeButtons` |
| `DB_RobotPath` | **folder**, part 7 | no | `FB_RobotMove` v0.6 → `aoudtP[1..7]`, `aorBlend[1..7]` through InOut pins in step 10, only when the request carries a path | OB30 NW2, the segment pins (`OB30_NW2_pins.md`) |
| `DB_RobotJob` | TIA | – | HMI → `Man.xAuto/xStart/xAbort/intOverride`, `Teach.xTeach/xClear/intPoseNr`. `FB_Teach` → `Teach.xTaught/xRefused/intLastTaught`. `FB_ScanCubeJob` → `Push`, `State`. `FC_RobotAlarms` → `aoxAlarm` | queue, select, alarms, HMI |
| `DB_RobotJob.Man.intJob`, `.Cube`, `.Drum`, `.intJobActive` | **folder** | – | HMI → `Man.intJob`. `FB_ScanCubeJob` → `Cube`. `FB_ScanDrumJob` → `Drum`. `FC_JobMux` → `intJobActive`, and from then on `Push` and `State` | `FC_JobMux`, HMI |
| `DB_MoveQueue` | TIA | – | `FB_MoveQueue` only | `FC_MoveSelect`, job, HMI |
| `DB_RobotHand` | **folder** | – | HMI → `Jog.*`, `Home.xX..xRotY`. The hand rungs → `xRobotManual`, `xHandOK`, `Home.x*Edge`, the five `TP_TIME` | the hand rungs, HMI |
| `IDB_Teach`, `IDB_ScanCubeJob`, `IDB_MoveQueue`, `IDB_RobotMove` | TIA | – | instance DBs of the four FBs, one instance each, one call each | – |
| `IDB_ScanDrumJob` | **folder** | – | instance of `FB_ScanDrumJob` | – |
| `"DB_MOVIKIT_Robotics-5AX"` | TIA, yours | – | the module 1 FB instance. `PI`, `PO` arrays of 64 words | `FC_RobotStatusMap` reads `PI` |
| `"DB_MOVIKIT_ROBOTICS_Module2"` | TIA, yours | – | the module 2 FB instance | engine reads `SrlRealRaw[1]`, `[2]`: the echo REAL[71], REAL[72] |

### A4. FCs (functions, no memory)

| FC | Where | Called in | Inputs from | Outputs to |
|---|---|---|---|---|
| `FC_RobotStatusMap` | TIA | OB30 NW4 | `"DB_MOVIKIT_Robotics-5AX".PI`, `DB_App.HS_Alive`, `DB_App.Srl.CommOK` | `DB_Robot.Status`, every field written every call |
| `FC_PoseCheck` | TIA | inside `FB_RobotMove` step 5 and `FB_ScanCubeJob` step 10. Not in OB1 | one `UDT_Pose` | return value 0 ok, 2 xyz outside, 3 rotation outside. Limits X 0..4380, Y 0..4230, Z −10..1908, RotZ ±180, RotY ±10 |
| `FC_MoveSelect` v0.2 | TIA after step 1 | OB1 NW12, before the engine | hand / auto, `DB_RobotMove.Man`, `DB_MoveQueue.Head`, `.xHeadValid`, `DB_RobotMove.State` | `DB_RobotMove.Req`, `.xReqStart`. Auto wins. In auto the start is a level: head valid, engine not busy, not done, no error. The engine takes the edge |
| `FC_RobotAlarms` | TIA | OB1 NW13 | `DB_RobotMove.State`, `DB_RobotJob.State`, `DB_Robot.Status` | `DB_RobotJob.aoxAlarm[1..32]` |
| `FC_JobMux` | **folder** | OB1, after the two jobs, before the queue | `DB_RobotJob.Man.intJob`, `.Cube`, `.Drum` | `DB_RobotJob.Push`, `.State`, `.intJobActive`. Follows `intJob` only while no job runs |

### A5. FBs (function blocks, with instance DB)

| FB | Where | Instance | Called in | Inputs from | Outputs to |
|---|---|---|---|---|---|
| `FB_Teach` | TIA | `IDB_Teach` | OB1 NW9 | `DB_RobotJob.Teach`, allowed = Enable and not auto and engine not busy, valid = referenced and comm ok, `DB_Robot.Status.udtPoseBcs` | `DB_Poses.aoudtP[n]`, `.aoxValid[n]` (InOut), `DB_RobotJob.Teach.xTaught/xRefused/intLastTaught` |
| `FB_ScanCubeJob` | v0.1 TIA, v0.5 **folder** (step 1b; v0.5 adds the path mode behind `i_xPathMode`) | `IDB_ScanCubeJob` | OB1 NW10 | `DB_RobotJob.Man`, `DB_Poses`, `DB_RobotMove.State`, `DB_MoveQueue.intCount`. v0.2: plus `DB_Recipe.Cube.*`, corners 2, 3, 4 and the approach computed from corner 1 when length and width > 0. v0.3 scan from here, v0.4 the turns −180 → +180, v0.5 `DB_Recipe.xPathMode` | `DB_RobotJob.Push`, `DB_RobotJob.State`. 13 moves, 14 with home, all block 1; path mode: 8 entries, the rectangle as block 5. After part 6: `DB_RobotJob.Cube` |
| `FB_MoveQueue` | TIA | `IDB_MoveQueue` | OB1 NW11 | `DB_RobotJob.Push`, pop = `DB_RobotMove.State.xDone` | `DB_MoveQueue.Q` (InOut), `.Head`, `.xHeadValid`, `.intCount`, `.xFull`, `.xPushRefused` |
| `FB_RobotMove` v0.3 in TIA, v0.6 in the folder (v0.4 gate with the pose, v0.5 program stop in step 15, v0.6 the path on `DB_RobotPath`, inert without a path) | TIA | `IDB_RobotMove` | OB1 NW12, after `FC_MoveSelect` | `DB_RobotMove.Req`, `.xReqStart`, `.Man.xEnable`, `.Man.xReset`, `.Cfg`, `DB_Robot.Status`, module 2 `SrlRealRaw[1..2]` | `DB_App.Srl.Pose/Par/Gate/GateEnable`, `DB_App.Cmd.Start/ProgramNumber/Override`, `DB_RobotPath.aoudtP/aorBlend` (all InOut), `DB_RobotMove.State` (Output; v0.5: `.xPrgStop` goes to the OB30 NW2 stop pin) |
| `FB_RobotMode` | **folder**, step 1a | `IDB_RobotMode` | OB1 after NW8, before NW9 | the or of all jog and home commands and `DB_App.Home.active`, `DB_RobotJob.Man.xAuto`, `DB_RobotMove.State.xBusy`, `DB_Robot.Status` | `DB_App.Cmd.Automatic` (InOut). Automatic at rest, manual while a jog or home is wanted and nothing of ours moves, back 500 ms after standstill |
| `FB_ScanDrumJob` v0.4 | **folder** | `IDB_ScanDrumJob` | OB1, after the cube job | the same as the cube job, plus `DB_Recipe.Drum.*` and pose 8, `DB_Recipe.xPathMode` | `DB_RobotJob.Drum`. Ten moves, four quarter arcs with blocks 3 and 4; path mode: 7 entries, the circle as block 6 or 7 |
| `FB_MachineReady` | TIA, yours | yours | OB1 NW7 | | its Enable output is no longer wired. NW8 writes `Cmd.Enable` |
| `"MOVIKIT_Robotics"` module 1 and 2 FBs | TIA, yours | the two MOVIKIT DBs | OB30 NW2, NW3 | `DB_App` | the bus |
| `FB_ActRobot` | folder | none | not in OB1. Stage 3 | | |

### A6. Pins of every block: what is read, what is written, what is both

This is what the cross-reference list in TIA shows for each pin: **R** for an Input (the block only reads it), **W** for an Output (the block writes it every call), **RW** for an InOut (the block reads and writes the tag itself, in place). For the one-writer rule, W and RW both count as "writer". R never does.

**`FB_RobotMove`, `IDB_RobotMove`, OB1 NW12** (TIA)

| Pin | Kind | Tag | Inside the block |
|---|---|---|---|
| `i_xStart` | Input, R | `"DB_RobotMove".xReqStart` | edge detected inside |
| `i_xEnable` | Input, R | `"DB_RobotMove".Man.xEnable` | falls while busy = error 20 |
| `i_xReset` | Input, R | `"DB_RobotMove".Man.xReset` | only out of step 90 |
| `i_xSensorStop` | Input, R | `FALSE` | reserved for the laser |
| `i_udtReq` | Input, R | `"DB_RobotMove".Req` | the move: block, pose, parameters, override |
| `i_intBlocksInPrg20` | Input, R | `"DB_RobotMove".Cfg.intBlocksInPrg20` | check of step 5 |
| `i_udtStatus` | Input, R | `"DB_Robot".Status` | all robot conditions |
| `i_intEchoBlock` | Input, R | `"DB_MOVIKIT_ROBOTICS_Module2".SrlRealRaw[1]` | REAL[71], checked in step 60 |
| `i_intEchoRun` | Input, R | `"DB_MOVIKIT_ROBOTICS_Module2".SrlRealRaw[2]` | REAL[72], checked in steps 40 and 60 |
| `io_aorSrlPose` | InOut, RW | `"DB_App".Srl.Pose` (the whole array [1..5]) | written in step 10, never read |
| `io_aorSrlPar` | InOut, RW | `"DB_App".Srl.Par` (the whole array [1..4]) | written in step 10, never read. [4] = run number |
| `io_aoxSrlGate` | InOut, RW | `"DB_App".Srl.Gate` (the whole array [1..8]) | all false in 10, 60, 90; one true in 30 |
| `io_xGateEnable` | InOut, RW | `"DB_App".Srl.GateEnable` | true in step 10 |
| `io_xStart` | InOut, RW | `"DB_App".Cmd.Start` | true in 30, false in 10, 60, 90 |
| `io_intPrgNr` | InOut, RW | `"DB_App".Cmd.ProgramNumber` | 20 in step 10 |
| `io_intOverride` | InOut, RW | `"DB_App".Cmd.Override` | **read** in step 5 (error 8 if 0 and the request has 0), **written** in step 10 when the request has > 0 |
| `io_aoudtPath` | InOut, RW | `"DB_RobotPath".aoudtP` (the whole array [1..7]) | v0.6, part 7: written in step 10 when `Req.intPathLen` > 0, never read. Unused segments get the last pose |
| `io_aorPathBlend` | InOut, RW | `"DB_RobotPath".aorBlend` (the whole array [1..7]) | v0.6, part 7: written in step 10 with the path, `Req.rBlend`, 0 today |
| `o_udtState` | Output, W | `"DB_RobotMove".State` | the report. v0.5: `.xPrgStop` TRUE in step 15 only, OB30 NW2 reads it on the stop pin |

Why the seven bus tags are InOut and not Output: an Output is copied from the instance into the DB at every call, so the DB would carry the block's copy. An InOut is the DB tag itself, the block writes into it only in the steps named and leaves it alone in all others, and `io_intOverride` is also read. So `DB_App` always holds exactly what went to the bus, and the one writer of these seven is this call. The whole arrays are on the pins, that is why the cross-reference shows the RW on `Srl.Pose`, `Srl.Par`, `Srl.Gate`, not on `Srl.Pose[1]` (see B4).

**`FC_MoveSelect` v0.2, OB1 NW12** (TIA after step 1)

| Pin | Kind | Tag |
|---|---|---|
| `i_xHand` | Input, R | `"DB_RobotMove".Man.xEnable AND NOT "DB_RobotJob".Man.xAuto` |
| `i_xAuto` | Input, R | `"DB_RobotMove".Man.xEnable AND "DB_RobotJob".Man.xAuto` |
| `i_udtMan` | Input, R | `"DB_RobotMove".Man.udtReq` |
| `i_xManStart` | Input, R | `"DB_RobotMove".Man.xStart` |
| `i_udtQueueHead` | Input, R | `"DB_MoveQueue".Head` |
| `i_xQueueHeadValid` | Input, R | `"DB_MoveQueue".xHeadValid` |
| `i_udtMoveState` | Input, R | `"DB_RobotMove".State` |
| `o_udtReq` | Output, W | `"DB_RobotMove".Req` |
| `o_xStart` | Output, W | `"DB_RobotMove".xReqStart` |

**`FB_ScanCubeJob`, `IDB_ScanCubeJob`, OB1 NW10** (TIA)

| Pin | Kind | Tag |
|---|---|---|
| `i_xStart` | Input, R | `"DB_RobotJob".Man.xStart` (part 6: `AND intJob = 1`) |
| `i_xAbort` | Input, R | `"DB_RobotJob".Man.xAbort` |
| `i_xAuto` | Input, R | `"DB_RobotJob".Man.xAuto AND "DB_RobotMove".Man.xEnable` |
| `i_intOverride` | Input, R | `"DB_RobotJob".Man.intOverride` |
| `i_aoudtP` | Input, R | `"DB_Poses".aoudtP` |
| `i_aoxValid` | Input, R | `"DB_Poses".aoxValid` |
| `i_rSafeZ` | Input, R | `"DB_Poses".rSafeZ` |
| `i_udtMoveState` | Input, R | `"DB_RobotMove".State` |
| `i_intQueueCount` | Input, R | `"DB_MoveQueue".intCount` |
| `i_rLength` | Input, R | `"DB_Recipe".Cube.rLength` (v0.2, folder) |
| `i_rWidth` | Input, R | `"DB_Recipe".Cube.rWidth` (v0.2, folder) |
| `i_rAngle` | Input, R | `"DB_Recipe".Cube.rAngle` (v0.2, folder) |
| `i_xFromHere` | Input, R | `"DB_Recipe".xFromHere` (v0.3, folder) |
| `i_udtActual` | Input, R | `"DB_Robot".Status.udtPoseBcs` (v0.3, folder) |
| `i_xActualValid` | Input, R | `"DB_Robot".Status.xReferenced AND .xCommOK` (v0.3, folder) |
| `i_xPathMode` | Input, R | `"DB_Recipe".xPathMode` (v0.5, folder, part 7) |
| `o_xPushReq` | Output, W | `"DB_RobotJob".Push.xReq` (part 6: `.Cube.xPushReq`) |
| `o_udtPush` | Output, W | `"DB_RobotJob".Push.udtMove` (part 6: `.Cube.udtPush`) |
| `o_xClear` | Output, W | `"DB_RobotJob".Push.xClear` (part 6: `.Cube.xClear`) |
| `o_udtState` | Output, W | `"DB_RobotJob".State` (part 6: `.Cube.udtState`) |

**`FB_Teach`, `IDB_Teach`, OB1 NW9** (TIA)

| Pin | Kind | Tag |
|---|---|---|
| `i_xTeach` | Input, R | `"DB_RobotJob".Teach.xTeach` |
| `i_xClearPose` | Input, R | `"DB_RobotJob".Teach.xClear` |
| `i_intPoseNr` | Input, R | `"DB_RobotJob".Teach.intPoseNr` |
| `i_xAllowed` | Input, R | `"DB_RobotMove".Man.xEnable AND NOT "DB_RobotJob".Man.xAuto AND NOT "DB_RobotMove".State.xBusy` |
| `i_xActualValid` | Input, R | `"DB_Robot".Status.xReferenced AND "DB_Robot".Status.xCommOK` |
| `i_udtActual` | Input, R | `"DB_Robot".Status.udtPoseBcs` |
| `io_aoudtP` | InOut, RW | `"DB_Poses".aoudtP` (the whole array). Written only on the TEACH edge, pose n |
| `io_aoxValid` | InOut, RW | `"DB_Poses".aoxValid` (the whole array). Written on TEACH and CLEAR, pose n |
| `o_xTaught` | Output, W | `"DB_RobotJob".Teach.xTaught` |
| `o_xRefused` | Output, W | `"DB_RobotJob".Teach.xRefused` |
| `o_intLastTaught` | Output, W | `"DB_RobotJob".Teach.intLastTaught` |

**`FB_MoveQueue`, `IDB_MoveQueue`, OB1 NW11** (TIA)

| Pin | Kind | Tag |
|---|---|---|
| `i_xPush` | Input, R | `"DB_RobotJob".Push.xReq` |
| `i_udtPush` | Input, R | `"DB_RobotJob".Push.udtMove` |
| `i_xPop` | Input, R | `"DB_RobotMove".State.xDone` |
| `i_xClear` | Input, R | `"DB_RobotJob".Push.xClear` |
| `io_udtQ` | InOut, RW | `"DB_MoveQueue".Q` (the FIFO itself, read and written in place) |
| `o_udtHead` | Output, W | `"DB_MoveQueue".Head` |
| `o_xHeadValid` | Output, W | `"DB_MoveQueue".xHeadValid` |
| `o_intCount` | Output, W | `"DB_MoveQueue".intCount` |
| `o_xFull` | Output, W | `"DB_MoveQueue".xFull` |
| `o_xPushRefused` | Output, W | `"DB_MoveQueue".xPushRefused` |

**`FC_RobotAlarms`, OB1 NW13** (TIA)

| Pin | Kind | Tag |
|---|---|---|
| `i_udtMoveState` | Input, R | `"DB_RobotMove".State` |
| `i_udtJobState` | Input, R | `"DB_RobotJob".State` |
| `i_udtStatus` | Input, R | `"DB_Robot".Status` |
| `o_aoxAlarm` | Output, W | `"DB_RobotJob".aoxAlarm` (the whole array [1..32]) |

**`FC_RobotStatusMap`, OB30 NW4** (TIA)

| Pin | Kind | Tag |
|---|---|---|
| `i_aowPI` | Input, R | `"DB_MOVIKIT_Robotics-5AX".PI` |
| `i_xHsAlive` | Input, R | `"DB_App".HS_Alive` |
| `i_xM2CommOK` | Input, R | `"DB_App".Srl.CommOK` |
| `o_udtStatus` | Output, W | `"DB_Robot".Status` |

**`FC_PoseCheck`** (TIA): one Input `i_udtPose`, return value Int. Called inside `FB_RobotMove` and `FB_ScanCubeJob`, it touches no DB.

**`FB_RobotMode`, `IDB_RobotMode`, OB1 after NW8** (folder, step 1a)

| Pin | Kind | Tag |
|---|---|---|
| `i_xJogWanted` | Input, R | 16 parallel contacts: `"DB_App".Cmd.Jog_Pos_X … Jog_Neg_R2`, `.Home_X1 … Home_R2`, `"DB_App".Home.active` |
| `i_xAuto` | Input, R | `"DB_RobotJob".Man.xAuto` |
| `i_xEngineBusy` | Input, R | `"DB_RobotMove".State.xBusy` |
| `i_udtStatus` | Input, R | `"DB_Robot".Status` (`xPrgRunning`, `xMoving`, `xRefMode`) |
| `io_xAutomatic` | InOut, RW | `"DB_App".Cmd.Automatic`. Written every cycle: NOT manual wanted |
| `o_xManual` | Output, W | open, or a lamp tag |

**`FC_JobMux`, OB1 between the jobs and the queue** (folder)

| Pin | Kind | Tag |
|---|---|---|
| `i_intJob` | Input, R | `"DB_RobotJob".Man.intJob` |
| `i_udtCube` | Input, R | `"DB_RobotJob".Cube` |
| `i_udtDrum` | Input, R | `"DB_RobotJob".Drum` |
| `io_intJobActive` | InOut, RW | `"DB_RobotJob".intJobActive`. Read every cycle, written only while no job runs |
| `o_xPushReq` | Output, W | `"DB_RobotJob".Push.xReq` |
| `o_udtPush` | Output, W | `"DB_RobotJob".Push.udtMove` |
| `o_xClear` | Output, W | `"DB_RobotJob".Push.xClear` |
| `o_udtState` | Output, W | `"DB_RobotJob".State` |

**`FB_ScanDrumJob`, `IDB_ScanDrumJob`, OB1 after the cube job** (folder): the nine Inputs of the cube job, plus `i_rZ`, `i_rRadius`, `i_rStartAngle`, `i_rRotZOffset`, `i_xClockwise` (Input, R) from `"DB_Recipe".Drum.*`, `i_xFromHere`, `i_udtActual`, `i_xActualValid` (Input, R, v0.2), `i_xPathMode` (Input, R, v0.4, part 7) from `"DB_Recipe".xPathMode`, and one Output `o_udtOut` (W) to `"DB_RobotJob".Drum`.

### A7. How one cycle runs in automatic (checked against the code)

1. START edge, AUTO on: the job builds the 14 poses, checks every one, pulses `Push.xClear`. The queue (same cycle, after the job) is empty.
2. Next 14 cycles: one push per cycle. After the first push the queue has a head, `FC_MoveSelect` sees head valid and engine idle, `xReqStart` goes high, the engine starts move 1 while the job is still pushing.
3. Move done: the engine pulses `State.xDone` for one cycle. Next cycle: the job counts it, the queue pops the head, `FC_MoveSelect` sees `xDone` still high and waits one cycle. The cycle after: the new head starts. The one-cycle gap gives the engine a clean edge.
4. Queue empty, engine not busy, no done pulse: the job reports done. Lamp done, running off.
5. Engine error during a move: `FC_MoveSelect` stops starting, the job clears the queue, job error 10 with the engine number. **The engine stays in error until RESET on the Robot move screen.** Then ABORT clears the job, START runs again.

---

## B. One tag, one writer

### B1. The table, as it is in TIA

W and RW both count as "writer". A tag may have many R.

| Tag | The one writer | Kind | Where |
|---|---|---|---|
| `DB_App.HS`, `.HS_Stall`, `.HS_Alive`, `.HS_Count` | OB30 NW1 | assignment | your code |
| `DB_App.Cmd.Enable` | OB1 NW8 | coil | your code. Since 2026-10-29 the only writer. The Enable switch of the Robot move screen is a requester branch of it through `DB_RobotMove.Man.xEnable` |
| `DB_App.Cmd.Automatic` | today the old screen switch. From step 1a `FB_RobotMode` | HMI, then InOut RW | one or the other, never both |
| `DB_App.Cmd.Start`, `.ProgramNumber`, `.Override` | `FB_RobotMove` | InOut, RW | OB1 NW12 |
| `DB_App.Srl.Pose`, `.Par`, `.Gate` (whole arrays), `.GateEnable` | `FB_RobotMove` | InOut, RW | OB1 NW12 |
| `DB_App.Srl.In[1..4]`, `.CommOK` | module 2 FB | Output, W | OB30 NW3 |
| `DB_Robot.Status` | `FC_RobotStatusMap` | Output, W | OB30 NW4 |
| `DB_RobotMove.Req`, `.xReqStart` | `FC_MoveSelect` | Output, W | OB1 NW12 |
| `DB_RobotMove.State` | `FB_RobotMove` | Output, W | OB1 NW12 |
| `DB_RobotMove.Man.*`, `.Cfg.*` | the Robot move screen | HMI | |
| `DB_MoveQueue.Q` | `FB_MoveQueue` | InOut, RW | OB1 NW11 |
| `DB_MoveQueue.Head`, `.xHeadValid`, `.intCount`, `.xFull`, `.xPushRefused` | `FB_MoveQueue` | Output, W | OB1 NW11 |
| `DB_RobotJob.Push`, `.State` | `FB_ScanCubeJob` | Output, W | OB1 NW10. After part 6: `FC_JobMux` |
| `DB_RobotJob.aoxAlarm` | `FC_RobotAlarms` | Output, W | OB1 NW13 |
| `DB_RobotJob.Man.*`, `.Teach.xTeach`, `.xClear`, `.intPoseNr` | the Automatic screen | HMI | |
| `DB_RobotJob.Teach.xTaught`, `.xRefused`, `.intLastTaught` | `FB_Teach` | Output, W | OB1 NW9 |
| `DB_Poses.aoudtP`, `.aoxValid` (whole arrays) | `FB_Teach` on TEACH / CLEAR | InOut, RW | OB1 NW9. The Poses screen may type into the same fields, see B3 |
| `DB_Poses.rSafeZ` | the Poses screen | HMI | |
| `DB_Recipe.Cube.*`, `.Drum.*`, `.xFromHere` | the Automatic screen | HMI | from step 1b |
| `DB_Recipe.xPathMode` | the Automatic screen | HMI | from part 7, the path mode switch |
| `DB_RobotPath.aoudtP`, `.aorBlend` (whole arrays) | `FB_RobotMove` v0.6 | InOut, RW | OB1 NW12, from part 7. Read by the OB30 NW2 segment pins. `DB_App.Path.Seg[n].X … R2`, `.Blend` then have no writer and no reader |

If a tag of this table ever gets a second W or RW, that is the bug.

### B2. What a screen may write, and what never

A screen field, button or switch may be bound only to a tag whose writer in B1 is "HMI" or "the … screen":

- `"DB_RobotMove".Man.xEnable`, `.xStart`, `.xReset`, `.udtReq.udtPose.rX … rRotY`, `.udtReq.intBlock`, `.udtReq.intOverride`, later `.udtReq.aorPar[1]`
- `"DB_RobotMove".Cfg.intBlocksInPrg20` (admin)
- `"DB_RobotJob".Man.xAuto`, `.xStart`, `.xAbort`, `.intOverride`, later `.intJob`
- `"DB_RobotJob".Teach.xTeach`, `.xClear`, `.intPoseNr`
- `"DB_Poses".aoudtP[n].*`, `.aoxValid[n]` (typing), `.rSafeZ`
- `"DB_Recipe".Cube.rLength`, `.rWidth`, `.rAngle`, `.Drum.*`, `.xFromHere` (from step 1b)
- `"DB_App".Cmd.Reset`, `.Simulation`, `.Pause`, `.Stop`, `.Home_*`, `.Jog_*` (old robot screen, until the hand rungs come). `.Automatic` only until step 1a, then `FB_RobotMode` writes it and the screen shows a lamp on `"DB_Robot".Status.xAutomatic`
- later `"DB_RobotHand".Jog.*`, `.Home.xX … xRotY`

A screen must never write: `"DB_App".Srl.*`, `"DB_App".Cmd.Start`, `.ProgramNumber`, `.Override`, `.Enable`, `"DB_Robot".Status.*`, `"DB_RobotMove".Req`, `.xReqStart`, `.State.*`, `"DB_MoveQueue".*`, `"DB_RobotJob".Push.*`, `.State.*`, `.aoxAlarm`, `.Teach.xTaught`, `.xRefused`, `.intLastTaught`. Output fields and lamps on them are fine, they only read.

### B3. The bug

**1. The one that happened, 2026-10-30.** The five target fields of the Robot move screen were bound to the bus pose `"DB_App".Srl.Pose[1..5]` instead of `"DB_RobotMove".Man.udtReq.udtPose.*`. Two writers of one tag: the screen typed into it, and `FB_RobotMove` step 10 wrote the request into it through the InOut pin. The request came from the fields that nobody wrote, so it was empty: the engine overwrote your numbers with zeros and the robot drove to 0,0,0,0,0. Fixed by binding the fields to `Man.udtReq`. That is the whole family of one-writer bugs on this machine: **a screen element bound to a tag that a block writes.** The block always wins, the screen shows the value for a moment and then it is gone.

**2. The same family, still possible tomorrow.** Four elements of the old robot screen, if they still exist, write tags that now have a block as writer:

| Old element | Writes | Who writes it now | Do |
|---|---|---|---|
| Start button | `"DB_App".Cmd.Start` | `FB_RobotMove`, InOut | delete. MOVE on the Robot move screen replaces it |
| program number field | `"DB_App".Cmd.ProgramNumber` | `FB_RobotMove`, InOut, always 20 | delete |
| override field | `"DB_App".Cmd.Override` | `FB_RobotMove`, InOut, from the request | delete. The override fields of the new screens write `Man.udtReq.intOverride` and `Man.intOverride` |
| enable switch | `"DB_App".Cmd.Enable` | OB1 NW8 | bind it to `"DB_RobotMove".Man.xEnable`, or delete it. `HMI_STAGE1.md` section 4 said "keep" on 2026-10-28, NW8 came on 2026-10-29 |

None of them breaks a move by itself, NW8 and the engine write every cycle or every move. They break the operator's picture: the switch shows a state the PLC does not hold, a typed override vanishes at the next move. Check them with B4.

**3. Waiting in the folder, part 6.** The files of 2026-10-31 move the writer of `"DB_RobotJob".Push` and `.State` from `FB_ScanCubeJob` to `FC_JobMux`. If part 6 is done halfway, for example the six blocks imported and `FC_JobMux` placed but the cube job's outputs still on `Push` and `State`, both write them: the job first, the mux after it in the same cycle. The mux copies `Cube`, which nobody fills, so it overwrites every push with an empty one. START does nothing, no error. The rule for part 6: import the six files, change the three OB1 networks, compile, download, in one go. Never run with one half.

**4. The known exception.** `DB_Poses.aoudtP[n]` has two writers by design: `FB_Teach` through its InOut pin on the TEACH edge, and the Poses screen when you type a number. Both are operator actions, never in the same cycle, and the typed value is what you want. It stays. The rule of thumb: teach, do not type, and never both on the same row in the same minute.

### B4. How to see a second writer in TIA, ten minutes

Program blocks → open the DB → right-click the tag → **Cross-references**. The column **Access** says R, W or RW. For the tags below, exactly one W or RW is allowed, and it must be the one named. Any HMI screen with W on one of them is the bug of B3.

**Arrays:** an InOut pin takes the whole array, so the RW stands on the array name, `"DB_App".Srl.Pose`, `"DB_App".Srl.Par`, `"DB_App".Srl.Gate`, `"DB_Poses".aoudtP`, `"DB_Poses".aoxValid`, `"DB_MoveQueue".Q`. The elements, `Srl.Pose[1]`, `Srl.Par[4]`, `Srl.Gate[1]`, show only the R of the readers, for example OB30 NW3. Array and element are one tag: the RW on the array is the writer of every element. Look at both lines and count them together. (The first edition of this file put the RW on `Srl.Pose[1]`, that was wrong, corrected 2026-11-01.)

| Tag | The one W or RW | The R you will also see |
|---|---|---|
| `"DB_App".Cmd.Enable` | OB1 NW8, coil, W | OB30 NW2 and NW3 (your NW3 uses it as `enable`), HMI lamps |
| `"DB_App".Cmd.Start` | `IDB_RobotMove` call, OB1 NW12, RW | OB30 NW2 |
| `"DB_App".Cmd.ProgramNumber` | `IDB_RobotMove` call, RW | OB30 NW2 |
| `"DB_App".Cmd.Override` | `IDB_RobotMove` call, RW | OB30 NW2 |
| `"DB_App".Srl.Pose` (array) | `IDB_RobotMove` call, RW | elements [1..5]: OB30 NW3 `PoseX … PoseRotY` |
| `"DB_App".Srl.Par` (array) | `IDB_RobotMove` call, RW | elements [1..4]: OB30 NW3 `Param1 … Param4` |
| `"DB_App".Srl.Gate` (array) | `IDB_RobotMove` call, RW | elements [1..8]: OB30 NW3 `WrBool1 … WrBool8` |
| `"DB_App".Srl.GateEnable` | `IDB_RobotMove` call, RW | OB30 NW3 `BoolWriteEnable` |
| `"DB_Robot".Status` | `FC_RobotStatusMap`, OB30 NW4, W | OB1 NW9, NW12, NW13, HMI lamps and fields |
| `"DB_RobotMove".Req` | `FC_MoveSelect`, OB1 NW12, W | `IDB_RobotMove` call |
| `"DB_RobotMove".State` | `IDB_RobotMove` call, W | NW9, NW10, NW11, NW12 (`FC_MoveSelect`), NW13, HMI, OB30 NW2 (`.xPrgStop` on the stop pin, v0.5) |
| `"DB_RobotJob".Push` | `IDB_ScanCubeJob` call, W | `IDB_MoveQueue` call |
| `"DB_MoveQueue".Q` | `IDB_MoveQueue` call, RW | none |
| `"DB_MoveQueue".Head` | `IDB_MoveQueue` call, W | `FC_MoveSelect` |
| `"DB_Poses".aoudtP` (array) | `IDB_Teach` call, RW | `IDB_ScanCubeJob` call, HMI pose fields (the exception of B3.4) |

Then the HMI once: HMI → HMI tags → default tag table → sort by PLC tag. Every tag that an I/O field, button or switch writes must be in the "may write" list of B2.

---

## C. Only in the folder, not in TIA

The rows marked **folder** in A2 to A6, in one list. Nothing of this runs tomorrow.

| Item | File | Comes in |
|---|---|---|
| `UDT_JobOut` | `UDT_JobOut.udt` | part 6, import 1 |
| `DB_Recipe`: cube length, width, angle and the drum settings | `DB_Recipe.scl` | step 1b, 2026-11-02 |
| `FB_ScanCubeJob` v0.5, the cube by its dimensions and "scan from here", six new pins (plus `i_xPathMode`, inert until part 7) | `FB_ScanCubeJob.scl` | step 1b, 2026-11-02 |
| `DB_RobotJob.Man.intJob`, `.Cube`, `.Drum`, `.intJobActive` | `DB_RobotJob.scl` | part 6, import 3 |
| `FC_RobotAlarms` job error 5 → bit 9 | `FC_RobotAlarms.scl` | part 6, import 4 |
| `FB_ScanDrumJob` v0.4 + `IDB_ScanDrumJob`, with "scan from here" (plus `i_xPathMode`, inert until part 7) | `FB_ScanDrumJob.scl` | part 6, import 5 |
| `FC_JobMux` | `FC_JobMux.scl` | part 6, import 6 |
| OB1: cube job gated by `intJob = 1` and outputs to `Cube`, drum network, mux network | `OB1_Stage2_call.scl` | part 6, step 17 |
| program 20 blocks 3 and 4, `Cfg.intBlocksInPrg20` = 4, Circle license | `PROGRAM20.md` | part 6, step 18 |
| `FB_RobotMode` + `IDB_RobotMode`, the robot mode by the PLC | `FB_RobotMode.scl`, `OB1_Stage2_call.scl` | step 1a of `AUTOMATIC_TODAY.md`, 2026-11-02 |
| `DB_RobotHand` and the five hand rungs | `DB_RobotHand.scl`, `SCREENS.html` section 5 | later, the manual screen |
| `FB_CubeButtons` + `IDB_CubeButtons`, the cube box buttons as plain bits, `DB_RobotHand.Btn` and `.Lamp` | `FB_CubeButtons.scl`, `DB_RobotHand.scl`, `HMI_CUBE_HOWTO.md` | with the CUBE box, 2026-11-03. The one writer of `Teach.intPoseNr`, `.xTeach`, `.xClear` from then on |
| **path mode** (was the side folder "robot pose solution", merged into this folder 2026-11-04): the rectangle or the circle as one program run on `POSE[1..7]` (module 1 segments). `UDT_MoveReq` v0.3, `DB_RobotPath`, `DB_Recipe` v0.2 `xPathMode`, `FB_RobotMove` v0.6, cube job v0.5, drum job v0.4, program 20 blocks 5–7 (`PROGRAM20.md` section 8), OB30 NW2 segment pins (`OB30_NW2_pins.md`, `OB30_complete.scl`), screen (`HMI_STAGE2.md` 6.5), the map `SOLUTION_MAP.html` | `AUTOMATIC_TODAY.md` part 7 | after five cube scans and part 6 |
| `OB30_complete.scl` NW3 `enable := TRUE` | | never: your OB30 has `Cmd.Enable`, keep yours |

After part 6 the writer of `"DB_RobotJob".Push` and `.State` is `FC_JobMux`, `FB_ScanCubeJob` writes `.Cube`, `FB_ScanDrumJob` writes `.Drum`. Everything else in B1 stays.

## D. Files that are not blocks

`GOALS.md` the reference. `AUTOMATIC_TODAY.md` the machine day, part 0 is the pre-flight. `SCREENS.html` the three screens with every object, and the ladder for jog and home. `PROGRAM20.md` the robot program. `HMI_STAGE1.md`, `HMI_STAGE2.md` the screens as text. `SYSTEM_MAP.html` the whole system on one page. `SOLUTION_MAP.html` the full map with every HMI tag, OB33 and OB1 network by network, every block, and the path mode. `OB30_NW2_pins.md` the segment pins of the path mode, pin by pin. `TEST_PROTOCOL.md` the machine tests. `ROADMAP.md`, `STAGE2_PLAN.md`, `STAGE2_README.md`, `TEST_SHEET.md`, `REVIEW_2026-10-29.md`, `DB_App_expected.md` background. `MOVIKIT_33956901.txt` the manual as text.
