# Checklist – the cube in automatic, and teaching (2026-11-04)

Tick every line. It covers exactly one thing: teach three poses on the screen, type the size, press START, the robot scans the cube one move per point. No drum, no job mux, no path mode is needed for this. Where a block exists in two versions, both are named: the older one works, the newer one is the file in the folder.

Reading order on the machine: section 1 (what is in the project tree), 2 (OB1 pins), 3 (OB30 pins), 4 (values), 5 (robot), 6 (screens), 7 (the five-minute proof).

---

## 1. Blocks in the project tree

### 1.1 Types

| UDT | Must be | File | ☐ |
|---|---|---|---|
| `UDT_Pose` | rX, rY, rZ, rRotZ, rRotY | `UDT_Pose.udt` | ☐ |
| `UDT_RobotStatus` | with `xRepositionRequired`, `xRepositionActive`, `xPrgInitialized`, `xPrgFinished`, `udtPoseBcs` | `UDT_RobotStatus.udt` | ☐ |
| `UDT_MoveReq` | v0.2 is enough. v0.3 (folder) adds `aoudtPath`, `intPathLen` at the end, needed only by engine v0.6 | `UDT_MoveReq.udt` | ☐ |
| `UDT_MoveState` | v0.2 with `xPrgStop` at the end, needed by engine v0.5 and v0.6. v0.1 is enough for engine v0.4 | `UDT_MoveState.udt` | ☐ |
| `UDT_MoveQueue` | | `UDT_MoveQueue.udt` | ☐ |
| `UDT_JobState` | | `UDT_JobState.udt` | ☐ |
| `UDT_JobOut` | only with part 6 (job mux). Not needed for the cube | `UDT_JobOut.udt` | – |

### 1.2 Data blocks

| DB | Must have | Written by | ☐ |
|---|---|---|---|
| `DB_App` | yours: `Cmd.*`, `Srl.Pose[1..5]`, `Srl.Par[1..4]`, `Srl.Gate[1..8]`, `Srl.GateEnable`, `HS*`, `ReadyToStart`, `MachineOn`, `Home.*`, `Path.Seg[n]` | OB30 NW1, OB1 NW8, FB_RobotMode, FB_RobotMove, module 2 FB, your logic, old screen | ☐ |
| `DB_Robot` | `Status` of type `UDT_RobotStatus` | `FC_RobotStatusMap`, OB30 NW4 | ☐ |
| `DB_RobotMove` | `Cfg.intBlocksInPrg20`, `Man` (xEnable, xStart, xReset, udtReq), `Req`, `xReqStart`, `State` | Robot move screen, `FC_MoveSelect`, `FB_RobotMove`, `FB_CubeButtons` (Man.udtReq, xStart, xReset on GO TO and ACK) | ☐ |
| `DB_Poses` | `aoudtP[1..50]`, `aoxValid[1..50]`, `rSafeZ`. Retentive. Rows 1 home, 2 start, 4 corner 1 are the three the cube needs | `FB_Teach`, the Poses screen (fields), safe Z field | ☐ |
| `DB_RobotJob` | `Man` (xAuto, xStart, xAbort, intOverride, intJob), `Teach`, `Push`, `State`, `aoxAlarm`, `Cube`, `Drum`, `intJobActive` | Automatic screen (Man), `FB_CubeButtons` (Teach.intPoseNr, xTeach, xClear), `FB_Teach`, the cube job, `FC_RobotAlarms` | ☐ |
| `DB_MoveQueue` | `Q`, `Head`, `xHeadValid`, `intCount`, `xFull`, `xPushRefused` | `FB_MoveQueue` | ☐ |
| `DB_Recipe` | v0.1: `Cube.rLength`, `.rWidth`, `.rAngle`, `Drum.*`, `xFromHere`. v0.2 adds `xPathMode`, harmless. v0.3 (2026-11-04) adds `xNoTurn`, the test switch without turns. Retentive, re-initialized by the download: write the numbers down first | Automatic screen | ☐ |
| `DB_RobotHand` | `Btn` (xTeachHome, xTeachStart, xTeachC1, xTeachNr, xClearNr, intPoseNr, xGoStart, xGoC1, xAck), `Lamp` (xTeachAllowed, xGoAllowed, xCubeReady, xComputed). `Jog`, `Home`, `xRobotManual`, `xHandOK` are for the hand rungs later | screen (Btn), `FB_CubeButtons` (Lamp) | ☐ |
| `DB_RobotPath` | only with engine v0.6 (part 7). Not needed for the cube | `FB_RobotMove` v0.6 | – |

### 1.3 Functions

| FC | Must be | ☐ |
|---|---|---|
| `FC_PoseCheck` | limits X 0..4380, Y 0..4230, Z −10..1908, RotZ ±180, RotY ±10 | ☐ |
| `FC_RobotStatusMap` | PI words → `DB_Robot.Status`, with PI 2.6 and 2.7 | ☐ |
| `FC_MoveSelect` | v0.2, seven inputs: `i_xHand`, `i_xAuto`, `i_udtMan`, `i_xManStart`, `i_udtQueueHead`, `i_xQueueHeadValid`, `i_udtMoveState` | ☐ |
| `FC_RobotAlarms` | engine errors on bits 1–22, job errors on 23–27, robot on 28–32 | ☐ |
| `FC_JobMux` | only with part 6. Not needed for the cube | – |

### 1.4 Function blocks and their instances

| FB | Instance | Version | ☐ |
|---|---|---|---|
| `FB_RobotMode` | `IDB_RobotMode` | v0.1. The one writer of `"DB_App".Cmd.Automatic` | ☐ |
| `FB_CubeButtons` | `IDB_CubeButtons` | v0.1. The one writer of `"DB_RobotJob".Teach.intPoseNr`, `.xTeach`, `.xClear` and of `"DB_RobotHand".Lamp` | ☐ |
| `FB_Teach` | `IDB_Teach` | unchanged | ☐ |
| `FB_ScanCubeJob` | `IDB_ScanCubeJob` | v0.4 (six new pins) works. v0.5 adds `i_xPathMode`, wire it to `"DB_Recipe".xPathMode` and leave the switch off. v0.6 (2026-11-04) adds `i_xNoTurn` on `"DB_Recipe".xNoTurn`: on = no turns, corner 1 at any RotZ | ☐ |
| `FB_MoveQueue` | `IDB_MoveQueue` | unchanged, FIFO of 50 | ☐ |
| `FB_RobotMove` | `IDB_RobotMove` | v0.4 in the PLC works (gate with the pose). v0.6 (folder) = v0.4 + program stop after a jog (v0.5) + path pins (v0.6): needs `UDT_MoveState` v0.2, `UDT_MoveReq` v0.3, `DB_RobotPath`, two OB1 pins, the OB30 stop pin. Recommended, not required for the cube | ☐ |
| `FB_StartTrace` | `IDB_StartTrace` | diagnostic only, optional. Delete when the start is stable | – |
| MOVIKIT module 1 and 2 FBs | the two MOVIKIT DBs | yours | ☐ |

---

## 2. OB1 – the networks after your network 8 (enable), in this order, and their pins

The full text of every call: `OB1_Stage2_call.scl`. In LAD one box per network.

**2.1 Robot mode** `IDB_RobotMode` ☐

| Pin | Tag |
|---|---|
| `i_xJogWanted` | 16 contacts in parallel: `"DB_App".Cmd.Jog_Pos_X`, `.Jog_Neg_X`, `.Jog_Pos_Y`, `.Jog_Neg_Y`, `.Jog_Pos_Z`, `.Jog_Neg_Z`, `.Jog_Pos_R1`, `.Jog_Neg_R1`, `.Jog_Pos_R2`, `.Jog_Neg_R2`, `.Home_X1`, `.Home_Y1`, `.Home_Z1`, `.Home_R1`, `.Home_R2`, `"DB_App".Home.active` |
| `i_xAuto` | `"DB_RobotJob".Man.xAuto` |
| `i_xEngineBusy` | `"DB_RobotMove".State.xBusy` |
| `i_udtStatus` | `"DB_Robot".Status` |
| `io_xAutomatic` | `"DB_App".Cmd.Automatic` |
| `o_xManual` | open |

**2.2 Cube buttons** `IDB_CubeButtons`, before teach ☐

| Pin | Tag |
|---|---|
| `i_xTeachHome`, `i_xTeachStart`, `i_xTeachC1` | `"DB_RobotHand".Btn.xTeachHome`, `.xTeachStart`, `.xTeachC1` |
| `i_xTeachNr`, `i_xClearNr`, `i_intPoseNrReq` | `"DB_RobotHand".Btn.xTeachNr`, `.xClearNr`, `.intPoseNr` |
| `i_xGoStart`, `i_xGoC1`, `i_xAck` | `"DB_RobotHand".Btn.xGoStart`, `.xGoC1`, `.xAck` |
| `i_aoudtP`, `i_aoxValid` | `"DB_Poses".aoudtP`, `.aoxValid` |
| `i_rLength`, `i_rWidth` | `"DB_Recipe".Cube.rLength`, `.rWidth` |
| `i_xEnable`, `i_xAuto`, `i_xEngineBusy` | `"DB_RobotMove".Man.xEnable`, `"DB_RobotJob".Man.xAuto`, `"DB_RobotMove".State.xBusy` |
| `io_udtMan`, `io_xManStart`, `io_xReset` | `"DB_RobotMove".Man.udtReq`, `.Man.xStart`, `.Man.xReset` |
| `io_xAbort` | `"DB_RobotJob".Man.xAbort` |
| `o_intPoseNr`, `o_xTeach`, `o_xClear` | `"DB_RobotJob".Teach.intPoseNr`, `.xTeach`, `.xClear` |
| `o_xTeachAllowed`, `o_xGoAllowed`, `o_xCubeReady`, `o_xComputed` | `"DB_RobotHand".Lamp.xTeachAllowed`, `.xGoAllowed`, `.xCubeReady`, `.xComputed` |

**2.3 Teach** `IDB_Teach` ☐

| Pin | Tag |
|---|---|
| `i_xTeach`, `i_xClearPose`, `i_intPoseNr` | `"DB_RobotJob".Teach.xTeach`, `.xClear`, `.intPoseNr` |
| `i_xAllowed` | `"DB_RobotMove".Man.xEnable` AND NOT `"DB_RobotJob".Man.xAuto` AND NOT `"DB_RobotMove".State.xBusy` |
| `i_xActualValid` | `"DB_Robot".Status.xReferenced` AND `.xCommOK` |
| `i_udtActual` | `"DB_Robot".Status.udtPoseBcs` |
| `io_aoudtP`, `io_aoxValid` | `"DB_Poses".aoudtP`, `.aoxValid` |
| `o_xTaught`, `o_xRefused`, `o_intLastTaught` | `"DB_RobotJob".Teach.xTaught`, `.xRefused`, `.intLastTaught` |

**2.4 Cube job** `IDB_ScanCubeJob` ☐

| Pin | Tag |
|---|---|
| `i_xStart` | `"DB_RobotJob".Man.xStart` (with part 6: AND `"DB_RobotJob".Man.intJob = 1`) |
| `i_xAbort` | `"DB_RobotJob".Man.xAbort` |
| `i_xAuto` | `"DB_RobotJob".Man.xAuto` AND `"DB_RobotMove".Man.xEnable` |
| `i_intOverride` | `"DB_RobotJob".Man.intOverride` |
| `i_aoudtP`, `i_aoxValid`, `i_rSafeZ` | `"DB_Poses".aoudtP`, `.aoxValid`, `.rSafeZ` |
| `i_udtMoveState` | `"DB_RobotMove".State` |
| `i_intQueueCount` | `"DB_MoveQueue".intCount` |
| `i_rLength`, `i_rWidth`, `i_rAngle` | `"DB_Recipe".Cube.rLength`, `.rWidth`, `.rAngle` |
| `i_xFromHere` | `"DB_Recipe".xFromHere` |
| `i_udtActual` | `"DB_Robot".Status.udtPoseBcs` |
| `i_xActualValid` | `"DB_Robot".Status.xReferenced` AND `.xCommOK` |
| `i_xPathMode` | only v0.5: `"DB_Recipe".xPathMode` |
| `i_xNoTurn` | only v0.6 (2026-11-04): `"DB_Recipe".xNoTurn`. On = no turn at the corners, the turn moves are left out |
| `o_xPushReq`, `o_udtPush`, `o_xClear`, `o_udtState` | without part 6: `"DB_RobotJob".Push.xReq`, `.Push.udtMove`, `.Push.xClear`, `"DB_RobotJob".State`. With part 6: `"DB_RobotJob".Cube.xPushReq`, `.udtPush`, `.xClear`, `.udtState` and the `FC_JobMux` network after it |

**2.5 Move queue** `IDB_MoveQueue` ☐

| Pin | Tag |
|---|---|
| `i_xPush`, `i_udtPush`, `i_xClear` | `"DB_RobotJob".Push.xReq`, `.udtMove`, `.xClear` |
| `i_xPop` | `"DB_RobotMove".State.xDone` |
| `io_udtQ` | `"DB_MoveQueue".Q` |
| `o_udtHead`, `o_xHeadValid`, `o_intCount`, `o_xFull`, `o_xPushRefused` | `"DB_MoveQueue".Head`, `.xHeadValid`, `.intCount`, `.xFull`, `.xPushRefused` |

**2.6 Move select** `FC_MoveSelect` ☐

| Pin | Tag |
|---|---|
| `i_xHand` | `"DB_RobotMove".Man.xEnable` AND NOT `"DB_RobotJob".Man.xAuto` |
| `i_xAuto` | `"DB_RobotMove".Man.xEnable` AND `"DB_RobotJob".Man.xAuto` |
| `i_udtMan`, `i_xManStart` | `"DB_RobotMove".Man.udtReq`, `.Man.xStart` |
| `i_udtQueueHead`, `i_xQueueHeadValid` | `"DB_MoveQueue".Head`, `.xHeadValid` |
| `i_udtMoveState` | `"DB_RobotMove".State` |
| `o_udtReq`, `o_xStart` | `"DB_RobotMove".Req`, `.xReqStart` |

**2.7 Engine** `IDB_RobotMove`, same network, after the select ☐

| Pin | Tag |
|---|---|
| `i_xStart` | `"DB_RobotMove".xReqStart` |
| `i_xEnable`, `i_xReset` | `"DB_RobotMove".Man.xEnable`, `.Man.xReset` |
| `i_xSensorStop` | FALSE |
| `i_udtReq` | `"DB_RobotMove".Req` |
| `i_intBlocksInPrg20` | `"DB_RobotMove".Cfg.intBlocksInPrg20` |
| `i_udtStatus` | `"DB_Robot".Status` |
| `i_intEchoBlock`, `i_intEchoRun` | `"DB_MOVIKIT_ROBOTICS_Module2".SrlRealRaw[1]`, `[2]` |
| `io_aorSrlPose`, `io_aorSrlPar`, `io_aoxSrlGate`, `io_xGateEnable` | `"DB_App".Srl.Pose`, `.Srl.Par`, `.Srl.Gate`, `.Srl.GateEnable` (the whole arrays) |
| `io_xStart`, `io_intPrgNr`, `io_intOverride` | `"DB_App".Cmd.Start`, `.Cmd.ProgramNumber`, `.Cmd.Override` |
| `io_aoudtPath`, `io_aorPathBlend` | only v0.6: `"DB_RobotPath".aoudtP`, `.aorBlend` |
| `o_udtState` | `"DB_RobotMove".State` |

**2.8 Alarms** `FC_RobotAlarms` ☐: `i_udtMoveState` ← `"DB_RobotMove".State`, `i_udtJobState` ← `"DB_RobotJob".State`, `i_udtStatus` ← `"DB_Robot".Status`, `o_aoxAlarm` → `"DB_RobotJob".aoxAlarm`.

**2.9 Nothing else writes** `"DB_App".Cmd.Start`, `.Cmd.Automatic`, `"DB_RobotJob".Teach.intPoseNr` / `.xTeach` / `.xClear`, `"DB_RobotMove".State`. Cross-reference of each: one writer. ☐

---

## 3. OB30 (your OB33) – the pins that matter

| Network | Pin | Must be | ☐ |
|---|---|---|---|
| NW1 | handshake | yours, the only writer of `"DB_App".HS` | ☐ |
| NW2 module 1 | `enable` | `"DB_App".Cmd.Enable` (written by your network 8) | ☐ |
| NW2 | `automatic` | `"DB_App".Cmd.Automatic`, written by `FB_RobotMode` only. No screen switch on it | ☐ |
| NW2 | `override`, `prog_no` | `"DB_App".Cmd.Override`, `.Cmd.ProgramNumber` (engine) | ☐ |
| NW2 | `start` | `"DB_App".Cmd.Start AND "DB_App".ReadyToStart AND "DB_App".MachineOn`. The two conditions must be TRUE when MOVE or START is pressed, else engine error 11 | ☐ |
| NW2 | `pause` | `"DB_App".Cmd.Pause`, FALSE all day | ☐ |
| NW2 | `stop` | `"DB_App".Cmd.Stop` with engine v0.4; `("DB_App".Cmd.Stop OR "DB_RobotMove".State.xPrgStop)` with v0.5 and v0.6 | ☐ |
| NW2 | `homing_Start_*`, `jog_*` | as today, old screen or your homing helper | ☐ |
| NW2 | segments `Pose_*_n`, `blending_distance_n` | as today on `"DB_App".Path.Seg[n]`. `DB_RobotPath` only with part 7 | ☐ |
| NW3 module 2 | `enable` | `"DB_App".Cmd.Enable` | ☐ |
| NW3 | `PoseX … PoseRotY` | `"DB_App".Srl.Pose[1..5]` → POSE 51 | ☐ |
| NW3 | `WrBool1..8`, `BoolWriteEnable` | `"DB_App".Srl.Gate[1..8]`, `.Srl.GateEnable` → BOOL 51..58 | ☐ |
| NW3 | `Param1..4` | `"DB_App".Srl.Par[1..4]` → REAL 51..54 | ☐ |
| NW3 | `Communication_OK`, `SrlRealRaw[1..2]` | → `"DB_App".Srl.CommOK`; REAL 71, 72 read by the engine | ☐ |
| NW4 | `FC_RobotStatusMap` | `i_aowPI` ← `"DB_MOVIKIT_Robotics-5AX".PI`, `i_xHsAlive` ← `"DB_App".HS_Alive`, `i_xM2CommOK` ← `"DB_App".Srl.CommOK`, `o_udtStatus` → `"DB_Robot".Status` | ☐ |

---

## 4. Values, one watch table

| Tag | Must be | ☐ |
|---|---|---|
| `"DB_RobotMove".Cfg.intBlocksInPrg20` | 1 or 2 (the blocks that exist in program 20). The cube uses block 1 only | ☐ |
| `"DB_RobotMove".Man.xEnable` | TRUE (the Enable switch), for hand and for auto | ☐ |
| `"DB_RobotJob".Man.intOverride` | 10 | ☐ |
| `"DB_RobotJob".Man.intJob` | 1 (only matters with part 6) | ☐ |
| `"DB_RobotJob".Man.xAuto` | FALSE for teaching and single moves, TRUE for START | ☐ |
| `"DB_Poses".rSafeZ` | above the cube, inside −10 … 1908 | ☐ |
| `"DB_Poses".aoxValid[1]`, `[2]`, `[4]` | TRUE after teaching home, start, corner 1 (`Lamp.xCubeReady` TRUE) | ☐ |
| `"DB_Recipe".Cube.rLength`, `.rWidth` | the cube, mm. 0 and 0 = the taught rows 5, 6, 7 are used instead | ☐ |
| `"DB_Recipe".Cube.rAngle` | 0 when side 1 runs along +X | ☐ |
| `"DB_Recipe".xFromHere` | FALSE (TRUE only for "scan from here") | ☐ |
| `"DB_Recipe".xPathMode` | FALSE | ☐ |
| `"DB_App".Cmd.Stop`, `.Cmd.Pause` | FALSE | ☐ |
| `"DB_App".ReadyToStart`, `.MachineOn` | TRUE while a move is started | ☐ |
| `"DB_App".Cmd.Enable` | TRUE | ☐ |
| `"DB_Robot".Status.xReady`, `.xReferenced`, `.xAccessGranted`, `.xCommOK`, `.xAutomatic` | TRUE, TRUE, TRUE, TRUE, TRUE at rest | ☐ |
| `"DB_Robot".Status.intPrgNr` | 20 | ☐ |
| `"DB_RobotMove".State.intErrId`, `"DB_RobotJob".State.intErrId` | 0 | ☐ |

---

## 5. Robot side, RobotMonitor

| Check | ☐ |
|---|---|
| Program 20 loaded, blocks 1 (and 2) in the IF form of `PROGRAM20.md` section 3, the echo line `REAL 71 := n` inside each block, `REAL 72 := REAL 54` after the last END_IF, `END_PROG` with "wait for the movement to end" ON | ☐ |
| Variables: POSE 51 ("PDO114-118 Extra 1"), BOOL 51 ("PDO123.0 Extra 1"), REAL 51..54, REAL 71, 72 exist and follow the PLC | ☐ |
| Control by the PLC: "NO control" in RobotMonitor, access granted | ☐ |
| All axes referenced, robot reset, no fault | ☐ |
| Operating mode follows the PLC: "Automatic" at rest, "Manual with high speed" only while a jog button is held | ☐ |
| Program state "Initialized" or "Finished" at rest. "Finished" is normal, nothing to clear. "Repositioning required" after a jog: engine v0.5 clears it itself, with v0.4 press program STOP once | ☐ |
| Simulation (PO 1.9) off for real moves, on for the first dry run | ☐ |

---

## 6. The screens, object by object

### 6.1 Robot move screen (hand, single moves)

| Object | Tag | ☐ |
|---|---|---|
| X, Y, Z, RotZ, RotY fields | `"DB_RobotMove".Man.udtReq.udtPose.rX`, `.rY`, `.rZ`, `.rRotZ`, `.rRotY` | ☐ |
| override, block | `"DB_RobotMove".Man.udtReq.intOverride`, `.intBlock` (text list `TL_Block`) | ☐ |
| Enable switch | `"DB_RobotMove".Man.xEnable` | ☐ |
| MOVE, RESET, momentary | `"DB_RobotMove".Man.xStart`, `.Man.xReset` | ☐ |
| busy, done, error lamps | `"DB_RobotMove".State.xBusy`, `.xDone`, `.xError` | ☐ |
| step, error | `"DB_RobotMove".State.intStep` (`TL_Step`), `.intErrId` (`TL_Err`, with 15) | ☐ |
| run nr, block, pose at end, message at error | `"DB_RobotMove".State.intRunNr`, `.intBlockActive`, `.udtPoseAtEnd.*`, `.uiMsgIdAtError` | ☐ |
| robot lamps | `"DB_Robot".Status.xReady`, `.xReferenced`, `.xAutomatic`, `.xAccessGranted`, `.xFault`, `.xCommOK`, `.xMoving`, `.xPrgRunning`, `.xPrgFinished` | ☐ |
| actual pose, message id, remaining segments | `"DB_Robot".Status.udtPoseBcs.*`, `.uiMsgId`, `.intRemainingSeg` | ☐ |
| blocks in program 20, admin | `"DB_RobotMove".Cfg.intBlocksInPrg20` | ☐ |

### 6.2 Automatic screen, box JOB

| Object | Tag | ☐ |
|---|---|---|
| AUTO switch | `"DB_RobotJob".Man.xAuto` | ☐ |
| override | `"DB_RobotJob".Man.intOverride` | ☐ |
| START, ABORT, momentary | `"DB_RobotJob".Man.xStart`, `.Man.xAbort` | ☐ |
| ACK, momentary | `"DB_RobotHand".Btn.xAck` (engine reset and job abort together) | ☐ |
| job field | `"DB_RobotJob".Man.intJob`, only with part 6 | – |
| Enable, shown again | `"DB_RobotMove".Man.xEnable` | ☐ |
| queue waiting, full | `"DB_MoveQueue".intCount`, `.xFull` | ☐ |
| running, done, error lamps | `"DB_RobotJob".State.xRunning`, `.xDone`, `.xError` | ☐ |
| job error, job step | `"DB_RobotJob".State.intErrId` (`TL_JobErr`), `.intStep` (`TL_JobStep`) | ☐ |
| moves done / total, active | `"DB_RobotJob".State.intMovesDone`, `.intMovesTotal`, `.intMoveActive` (`TL_Move`) | ☐ |
| engine error | `"DB_RobotJob".State.intEngineErr` (`TL_Err`) | ☐ |
| mode lamps | "automatic" on `"DB_Robot".Status.xAutomatic`, "manual" on NOT the same. No switch on `"DB_App".Cmd.Automatic` anywhere | ☐ |
| alarm view | discrete alarms on `"DB_RobotJob".aoxAlarm` | ☐ |

### 6.3 Automatic screen, box CUBE

| Object | Tag | ☐ |
|---|---|---|
| length, width, angle | `"DB_Recipe".Cube.rLength`, `.rWidth`, `.rAngle` | ☐ |
| safe Z | `"DB_Poses".rSafeZ` | ☐ |
| text "computed from corner 1" / "taught, rows 5 to 7" | visibility by `"DB_RobotHand".Lamp.xComputed` | ☐ |
| TEACH HOME, TEACH START, TEACH CORNER 1, momentary | `"DB_RobotHand".Btn.xTeachHome`, `.xTeachStart`, `.xTeachC1`, enabled by `"DB_RobotHand".Lamp.xTeachAllowed` | ☐ |
| lamp after each teach button | `"DB_Poses".aoxValid[1]`, `[2]`, `[4]` | ☐ |
| ready lamp | `"DB_RobotHand".Lamp.xCubeReady` | ☐ |
| GO TO START, GO TO CORNER 1, momentary | `"DB_RobotHand".Btn.xGoStart`, `.xGoC1`, enabled by `"DB_RobotHand".Lamp.xGoAllowed` | ☐ |
| SCAN FROM HERE switch | `"DB_Recipe".xFromHere` | ☐ |
| PATH MODE switch | `"DB_Recipe".xPathMode`, only with part 7 | – |
| NO TURN switch (A50) | `"DB_Recipe".xNoTurn`, cube job v0.6. On: the tool keeps the orientation of corner 1, 10 moves instead of 14, corner 1 at any RotZ. Or set the bit online in DB_Recipe | ☐ |

### 6.4 Poses screen, teaching any row

| Object | Tag | ☐ |
|---|---|---|
| row n: valid lamp, X … RotY | `"DB_Poses".aoxValid[n]`, `"DB_Poses".aoudtP[n].rX` … `.rRotY` | ☐ |
| pose nr field | `"DB_RobotHand".Btn.intPoseNr` (**not** `"DB_RobotJob".Teach.intPoseNr` any more) | ☐ |
| TEACH, CLEAR, momentary | `"DB_RobotHand".Btn.xTeachNr`, `.xClearNr` (**not** `Teach.xTeach`, `.xClear`) | ☐ |
| last taught, taught, refused | `"DB_RobotJob".Teach.intLastTaught`, `.xTaught`, `.xRefused` | ☐ |
| actual pose | `"DB_Robot".Status.udtPoseBcs.*` | ☐ |

### 6.5 Old robot screen

| Object | Must be | ☐ |
|---|---|---|
| jog X+ … RotY−, home X … RotY | on `"DB_App".Cmd.Jog_*`, `.Home_*` as today (the hand rungs come later) | ☐ |
| automatic switch | gone, or a lamp on `"DB_Robot".Status.xAutomatic`. A switch on `"DB_App".Cmd.Automatic` fights `FB_RobotMode` every cycle | ☐ |
| simulation, robot reset | `"DB_App".Cmd.Simulation`, `.Cmd.Reset` | ☐ |
| PAUSE, STOP | stay off. No Start, no program number, no override field on this screen | ☐ |

### 6.6 Text lists

| List | Entries | ☐ |
|---|---|---|
| `TL_Step` | 0 idle, 5 check, 10 write pose, 15 program stop, 20 settle, 30 start, 40 wait running, 50 moving, 60 end, 90 error | ☐ |
| `TL_Err` | 0 … 22 as `HMI_STAGE1.md` section 3, with 15 "program stop not taken, repositioning stays" | ☐ |
| `TL_Block` | 1 linear, 2 linear slow, 3 circle ccw, 4 circle cw | ☐ |
| `TL_JobErr` | 1 pose not taught, 2 safe Z, 3 engine busy, 4 outside limits, 5 drum radius, 6 scan from here not possible, 10 engine error | ☐ |
| `TL_JobStep`, `TL_Move` | as `HMI_STAGE2.md` section 3. `TL_Move` 1 to 14: start, approach, corner 1, side 1, turn c2, side 2, turn c3, side 3, turn c4, side 4, turn c1, lift, start, home | ☐ |

---

## 7. The five-minute proof, in this order

1. Robot move screen, AUTO off, Enable on, robot in automatic: MOVE a target 200 mm away in X. ☐ Done, error 0. (`TEST_PROTOCOL.md` 1.1)
2. Jog X+ for two seconds, release, MOVE back. ☐ Straight to the target. With engine v0.4 the robot may first drive back to the old point, that is the repositioning of the manual, v0.5 removes it. (2.5)
3. Jog to the park place, TEACH HOME. Jog to the start place, TEACH START. Jog the sensor to corner 1 with RotZ −180, facing side 1, TEACH CORNER 1. ☐ Three lamps green, ready lamp on. (3.x)
4. Type length, width, angle 0, safe Z above the cube. ☐ "computed from corner 1" visible. GO TO CORNER 1 drives there. (4.1)
5. Old screen simulation ON, AUTO on, START. ☐ Moves total 14 (13 without home), done, job error 0, nothing moved. (4.2)
6. Simulation OFF, override 10, AUTO on, START. ☐ The robot drives the 14 moves. ABORT stops after the current move, queue 0. (4.3, 4.4)
7. Five scans in a row. ☐ Then override 25, 50. (4.8, 7)

If a box does not match: step number, job error, engine error. Those three numbers are enough.

---

## 8. Build and test, one thing after the other (2026-11-04)

Four stages, in this order. Each stage ends with a test; do not start the next before the test passes. The screens as pictures with every object and tag: `SCREENS.html`, section 2 (the Automatic screen with its three boxes POSES, CUBE, JOB). A box is only a frame on the screen that groups objects; the cube is the thing the robot scans.

### Stage A – PLC, one compile, one download

| # | Do | Done when | ☐ |
|---|---|---|---|
| A1 | Every block of section 1 is in the project tree, with its instance | nothing red in the tree | ☐ |
| A2 | OB1: the nine networks of section 2 in that order, every pin on its tag | no empty pin | ☐ |
| A3 | OB30: the pins of section 3, above all `automatic`, `start`, `stop` | as in the table | ☐ |
| A4 | Compile the PLC | 0 errors | ☐ |
| A5 | Download. Watch table with the values of section 4 | all as listed | ☐ |
| A6 | Cross-reference of `"DB_App".Cmd.Start`, `.Cmd.Automatic`, `"DB_RobotJob".Teach.xTeach`, `"DB_RobotMove".State` | one writer each | ☐ |
| A7 | Robot move screen: MOVE a target 200 mm away | done, error 0 | ☐ |

A7 fails: `AUTOMATIC_TODAY.md` part 5, the engine error number says where. Stop here until A7 passes.

### Stage B – the screens, box by box, each one tested on the running PLC

| # | Build | Objects | Test | ☐ |
|---|---|---|---|---|
| B1 | HMI tag table: the tags of `HMI_CUBE_HOWTO.md` section 3, with their limits | `Btn_*`, `Lamp_*`, `Cube_*`, `Poses_*`, `Recipe_FromHere` | the tags show values online | ☐ |
| B2 | Robot move screen, complete | M1 … M40 of `SCREENS.html` section 1 | MOVE, RESET work, the lamps follow the robot | ☐ |
| B3 | POSES box | A1 rows 1 … 10, A2 safe Z, A3 pose nr, A4 TEACH, A5 CLEAR, A6 … A9 | pose nr 9, TEACH: row 9 valid lamp on, last taught 9. CLEAR: lamp off, numbers stay | ☐ |
| B4 | CUBE box | A35 … A37 numbers, A45 corners text, A39 … A41 teach buttons with lamps, A42 ready, A43 … A44 go-to buttons, A46 enable lamps, A38 SCAN FROM HERE. A49 PATH MODE not yet | type length 600, width 400: the text "computed from corner 1" appears. AUTO on: the teach buttons grey out | ☐ |
| B5 | JOB box | A11 Enable, A12 AUTO, A13 override, A14 START, A15 ABORT, A47 ACK, A16 … A17 queue, A18 … A26 state, A48 mode lamps, A27 … A33 engine and robot | AUTO on: MOVE on screen 1 is dead. AUTO off: MOVE works. ACK clears an engine error | ☐ |
| B6 | Text lists bound to their fields | TL_Step, TL_Err, TL_Block, TL_JobErr, TL_JobStep, TL_Move, section 6.6 | the step field reads "idle", the error field "no error" | ☐ |
| B7 | Old robot screen | the automatic switch is gone or a lamp; PAUSE and STOP stay off | jog X+ held: "manual" lamp on, released: "automatic" after half a second | ☐ |

### Stage C – teach, one pose after the other (`TEST_PROTOCOL.md` part 3)

Enable on, AUTO off, simulation off. The robot jogs in manual, the PLC switches the mode by itself.

| # | Do | Done when | ☐ |
|---|---|---|---|
| C1 | Jog to the park place. TEACH HOME | lamp after the button green, row 1 shows the actual pose, last taught 1 | ☐ |
| C2 | Jog above the cube at a free height. TEACH START | row 2 green | ☐ |
| C3 | Jog the sensor to corner 1 of the cube: with angle 0 the corner with the smallest X and the smallest Y, tool facing side 1, **RotZ −180**. TEACH CORNER 1 | row 4 green, ready lamp on | ☐ |
| C4 | Safe Z = the Z of row 2, above the cube, on the free side, never toward the table | safe Z field | ☐ |
| C5 | Length, width in mm, angle 0 | "computed from corner 1" | ☐ |
| C6 | On purpose: AUTO on, TEACH START | refused lamp flashes, row 2 unchanged. AUTO off | ☐ |
| C7 | GO TO START, then GO TO CORNER 1 | the robot drives to each, done, error 0 | ☐ |

C7 fails with error 5: the robot was still in manual, half a second after a jog. RESET, press again.

### Stage D – scan in automatic (`TEST_PROTOCOL.md` part 4)

SCAN FROM HERE off, PATH MODE off or not on the screen.

| # | Do | Done when | ☐ |
|---|---|---|---|
| D1 | Old screen simulation ON. AUTO on, override 10, START | running lamp, queue 13 → 0 (14 with home), moves counting, active text 1 to 14, done lamp, job error 0. Nothing moved. RobotMonitor: POSE 51 changes at every move | ☐ |
| D2 | Simulation OFF. One hand on the emergency stop. START | to start, approach, corner 1, the four sides with a +90 turn at every corner, lift, start, home. Done lamp | ☐ |
| D3 | START, in side 2 press ABORT | the running move finishes, the robot stops, queue 0, no error. START again begins with "to start" | ☐ |
| D4 | Length 9000, START | job error 4 with active = the first bad move, before anything moves. Length back, ACK | ☐ |
| D5 | AUTO off, jog to corner 1 with RotZ −180, SCAN FROM HERE on, AUTO on, START | moves total 11 (10 without home), the first move is side 1. Switch off after | ☐ |
| D6 | Five scans in a row | done lamp each time, no error | ☐ |
| D7 | Override 25, then 50, five scans each | the same | ☐ |

D6 passed = the cube works in automatic. Then the drum (part 6) or the path mode (part 7), not before.

### Stage D0 – the first scan without turns (2026-11-04, NO TURN switch)

For the first test the tool does not turn at the corners. Corner 1 may then stand at any RotZ, the turn moves are left out, 10 moves instead of 14. Needs `DB_Recipe` v0.3, `FB_ScanCubeJob` v0.6 and the OB1 pin `i_xNoTurn := "DB_Recipe".xNoTurn`.

| # | Do | Done when | ☐ |
|---|---|---|---|
| D0.1 | TIA: write the recipe numbers down. `DB_Recipe`: `xNoTurn : Bool` at the end. `FB_ScanCubeJob` v0.6 from the folder. OB1 cube box: the new pin at the end on `"DB_Recipe".xNoTurn`. Compile 0 errors, download. Type the numbers again | `"DB_Recipe".xNoTurn` exists, the cube box has the pin | ☐ |
| D0.2 | `xNoTurn` TRUE: the NO TURN switch (A50), or modify the bit online in `DB_Recipe` | the bit is TRUE and stays after the switch is released (retentive) | ☐ |
| D0.3 | Numbers: length = side 1, width = side 2, angle 0 when the cube lies at +X and +Y of corner 1 (table below), safe Z above the cube on your Z. SCAN FROM HERE off, PATH MODE off | "computed from corner 1" shows | ☐ |
| D0.4 | ACK. Engine: step idle, error 0, busy off. AUTO off. GO TO START, then GO TO CORNER 1 | both done, engine error 0 | ☐ |
| D0.5 | AUTO on, override 10, START | queue 9 → 0, active: to start, approach, corner 1, side 1, side 2, side 3, side 4, lift, start, home. Done lamp, total 10, job error 0 | ☐ |
| D0.6 | Error instead: job error number and MoveActive, engine error number | 4 = a computed corner outside the limits (angle or length wrong), 1 = a pose not taught, 10 = the engine refused, see its number | ☐ |

Angle, direction of side 1 seen from corner 1, side 2 is 90° to the left of it:

| Angle | side 1 goes to | side 2 goes to | corner 1 is the corner with |
|---|---|---|---|
| 0 | +X | +Y | smallest X, smallest Y |
| 90 | +Y | −X | largest X, smallest Y |
| 180 | −X | −Y | largest X, largest Y |
| −90 | −Y | +X | smallest X, largest Y |

Switch NO TURN off again for D2 and teach corner 1 at RotZ −180: the turns of the real scan need it.
