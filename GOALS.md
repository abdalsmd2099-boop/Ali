# Gantry robot – goals and plan

Created 2026-10-27 from the talk of tonight. This file is the reference for the whole project.
Nothing in the code may contradict it. A change is written here first, with a date, in section 9.

Only files inside `C:\Users\PC\Desktop\Gantry_Robot` are used. Nothing from other folders.

---

## 1. What we have (fixed facts, from the files in this folder)

| Item | Value | Source |
|---|---|---|
| PLC | Siemens S7-1511F, TIA Portal V18, HMI WinCC Unified | you |
| Robot controller | SEW UHX45A, MOVIKIT Robotics 10.1.113.200, configured in MOVIRUN smart (no IEC editor) | MOVISUITE summary |
| Kinematic model | CART_GANTRY_LLLRR_M10, 5 joints: X, Y, Z, RotZ (R1), RotY (R2) | MOVISUITE basic settings |
| Fieldbus profile | flexible parameterizable, 2 consistent blocks of 64 words, process data length 123 | MOVISUITE fieldbus config |
| Decimal places on the bus | 0, so one word = whole mm or whole degree | MOVISUITE fieldbus config |
| Module 1 | slot 18, HW identifier 278, `%IW19..146` / `%QW19..146`, robot words 1–64, FB `"DB_MOVIKIT_Robotics-5AX"`. Its instance DB keeps the raw words in `PI` and `PO`, each `Array[1..64] of Word`, index = robot word | OB30 network 2, device view, instance DB |
| Module 2 | slot 19, HW identifier 272, `%IW147..274` / `%QW147..274`, robot words 65–128, FB `"DB_MOVIKIT_ROBOTICS_Module2"` | OB30 network 3, device view |
| MOVIRUN device module | slot 17, 2 words, `%IW275..278` / `%QW275..278`, own handshake, handled in OB1 | you, device view |

### 1.1 Module 1, robot words 1–64

| PO (PLC → robot) | Content | PLC side |
|---|---|---|
| 1–4 | control words: enable, reset, automatic, start, pause, stop, homing, jog, override, program number | `DB_App.Cmd.*`, `DB_App.Home.*` through the FB pins |
| 5–53 | path segments 1–7, 7 words each (control, blending, X, Y, Z, RotZ, RotY) | `DB_App.Path.Seg[1..7]` |
| 54–64 | padding | – |

| PI (robot → PLC) | Content |
|---|---|
| 1 | status: ready, STO, power, **moving (bit 4)**, referenced (bit 5), setpoint active, fault (bit 8), warning (bit 9), **access granted (bit 13)**, handshake echo (bit 15) |
| 2 | automatic (bit 0), program initialized (bit 2), **program running (bit 3)**, paused (bit 4), **program finished (bit 5)**, repositioning required / active (6, 7), axis 1–5 referenced (bits 10–14) |
| 3 | message ID (fault / warning / information) |
| 4 | program number (bits 0–7), jog mode (8), referencing mode (9), program mode (10) |
| 5 | outside workspace (bit 1), motion completed (bit 2), constellation (8–15) |
| 6 | TCP speed, mm/s |
| 7–11 | setpoint pose in base coordinates X, Y, Z, RotZ, RotY |
| 12 | current target pose index (bits 0–7), **remaining segments (bits 8–15)** |
| 13 | remaining distance, mm |
| 14–18 | setpoint pose of joints 1–5 |
| 19–64 | padding |

### 1.2 Module 2, robot words 65–128

| PO (PLC → robot) | Content | PLC side |
|---|---|---|
| 65–113 | path segments 8–14 | switched off, `SegEnable := FALSE` |
| 114–118 | **SRL POSE[51]** X, Y, Z, RotZ, RotY | `DB_App.Srl.Pose[1..5]`, `PoseFirstWord := 50` |
| 119–122 | **SRL REAL[51..54]** setpoints | `DB_App.Srl.Par[1..4]`, `ParamFirstWord := 55` |
| 123 | **SRL BOOL[51..58]**, bits 0–7 | `DB_App.Srl.Gate[1..8]`, `BoolWriteWord := 59`, enable `DB_App.Srl.GateEnable` |

| PI (robot → PLC) | Content | PLC side |
|---|---|---|
| 65–80 | **SRL REAL[71..86]** actual values | `"DB_MOVIKIT_ROBOTICS_Module2".SrlRealRaw[1..16]` |
| 81 | SRL BOOL[71..74] actual values | `DB_App.Srl.In[1..4]`, `BoolWord := 17` |
| 82–128 | padding | – |

### 1.3 Handshakes – three bits, three owners

| Bit | Written by | Echo |
|---|---|---|
| MOVIRUN device module, PO 1.15 | OB1 | PI 1.15 of the device module |
| Robotics block 1, PO 1.15 | OB30 network 1, `DB_App.HS` | PI 1.15, block 1 |
| Robotics block 2, PO 65.15 | module 2 FB, same `DB_App.HS` | none, by design |

### 1.4 Limits we use in the PLC check

| Axis | Min | Max | Note |
|---|---|---|---|
| X | 0 | 4380 mm | |
| Y | 0 | 4230 mm | TCP page says 4230, joint 2 says 4320. We take the smaller one |
| Z | −10 | 1908 mm | |
| RotZ | −180° | +180° | |
| RotY | −10° | +10° | joint 5 limit. TCP page says ±45°, you chose the safer one |

### 1.5 Already proven on the machine

- Handshake toggle in OB30, stable.
- From the PLC and HMI: select program number, override, start, stop, jog, referencing.
- PLC writes SRL REAL[51..54], robot writes SRL REAL[71..86] back.
- Program 20 saved in RobotMonitor runs when the PLC starts it.

---

## 2. Goals (stable)

| # | Goal |
|---|---|
| G1 | **The PLC is the master.** RobotMonitor is used only to write program 20 and for diagnostics. The operator works on the HMI. |
| G2 | **One robot program, number 20.** It holds 2 to 8 blocks. Each block is gated by one SRL Bool and does one motion to a pose variable. A new motion is new data from the PLC (pose, Bool, start), never new SRL code. |
| G3 | **Stage 1 – single movement.** One HMI button, one pose, one move. The PLC writes the pose, opens the gate Bool, starts, waits for finished, closes the gate. |
| G4 | **Stage 2 – automatic.** The same single-movement block is fed from a queue of moves. First job: scan a cube from its four sides. Later: drum. Later: a laser sensor stops the approach at 10 mm and finds the corner. |
| G5 | **Reuse the StepChain framework** (FB_Mode, FB_Jobs, FB_OperBox, FB_SeqMechanism and its FCs) in stage 2. The robot joins the chain like the ram joined it, but through one block that serves many step numbers. |
| G6 | **Plain data, no tricks.** Poses live in a retentive DB, editable on the HMI. The 7 segment poses of module 1 and the 7 of module 2 stay available as a second way to hand poses to the robot. |

### 2.1 Rules that never change

1. One writer per tag. Reading is free.
2. One FB instance, one call.
3. OB30 delivers (handshake, two module FB calls, status map). OB1 decides. Nothing else goes into OB30.
4. Write the whole pose to the bus, wait 50 ms, then open the gate and start. A 5-word pose is not atomic.
5. Never trust "program finished" alone. "Program running" must have been seen first in this move.
6. The PLC checks every pose against the limits in 1.4 before it goes to the bus.
7. Override is never 0. At 0 the robot enables and never moves.
8. Safety stays in the F-program. A chain stop is a normal stop.
9. The HMI talks only to DBs, never to a block pin or to the robot directly.

---

## 3. The scan-cube job as a list of moves

2026-11-02: the corners can also be computed. With `"DB_Recipe".Cube.rLength` and `.rWidth` both greater than 0, `FB_ScanCubeJob` v0.2 builds corners 2, 3, 4 and the approach from corner 1 (pose 4) and the angle of side 1. The list below stays the same, only where corners 2, 3, 4 and the approach come from changes. A moved cube of the same size needs one new teach, corner 1. Length 0 = the taught corners as before. With the switch "scan from here" (`"DB_Recipe".xFromHere`) START takes the actual pose as corner 1 and begins with side 1, nothing is taught or stored; the drum job takes it as the center and begins above P0. Orientation (2026-11-02): in dimension mode the tool turns +90° at every corner, −180 at corner 1 to +180 back at corner 1, no wrap, so corner 1 must stand at RotZ −180; the drum circle starts where the tool stands at −180 and turns 90° per quarter to +180, the start angle is computed from the RotZ offset.

Today every pose is taught and stored in `DB_Poses`. Later the laser sensor replaces the taught approach and corner poses.

| # | Move | Target | Type | Today | Later with laser |
|---|---|---|---|---|---|
| 1 | check | robot ready, referenced, automatic, no fault, comm OK | check | – | – |
| 2 | reference | homing of all 5 axes if not referenced | homing | `DB_App.Cmd.Home_*` | same |
| 3 | go to start | `P_Start`, at safe height `SafeZ` | LIN | taught | same |
| 4 | approach | `P_Approach`, above / in front of the cube | LIN | taught | LIN towards the cube, stop at 10 mm |
| 5 | first corner | `P_C1` | LIN | taught | search move, sensor finds the corner |
| 6 | side 1 | `P_C2` | LIN | taught | length from the sensor |
| 7 | turn | same X Y Z as `P_C2`, RotZ + 90° | LIN | data | data |
| 8 | side 2 | `P_C3` | LIN | taught | |
| 9 | turn | RotZ + 90° | LIN | data | |
| 10 | side 3 | `P_C4` | LIN | taught | |
| 11 | turn | RotZ + 90° | LIN | data | |
| 12 | side 4 | back to `P_C1` | LIN | taught | |
| 13 | lift | `P_C1` with Z = `SafeZ` | LIN | data | |
| 14 | back to start | `P_Start` | LIN | taught | |
| 15 | home pose | `P_Home` (optional) | LIN | taught | |
| 16 | end | job finished | – | – | – |

Every line from 3 to 15 is **one call of the single-movement block** with a different pose. That is why stage 1 comes first: when one move is solid, the job is only a list.

A "turn" is a LIN with the same X Y Z and a new RotZ. No extra block type needed.

---

## 4. Blocks – how many and what each one does

### 4.1 Stage 1 (single movement)

| Block | Kind | Job | Writes |
|---|---|---|---|
| `UDT_Pose` | UDT | X, Y, Z, RotZ, RotY as Real (mm, deg) | – |
| `UDT_MoveReq` | UDT | one move request: `intBlock` (block of program 20, 1 = linear), `udtPose`, `udtPose2` (reserved), `rBlend`, `aorPar[1..3]` (block parameters, circle radius in 1), `intOverride`, `xStopOnSensor` (reserved), `intId` | – |
| `UDT_MoveState` | UDT | what the move engine reports: busy, done, error, error number, step, run number, active id and block, pose at the end, message id at the error | – |
| `UDT_RobotStatus` | UDT | the PI words of 1.1 as named bits and values, plus `xCommOK` | – |
| `DB_Robot` | DB | `Status : UDT_RobotStatus` | only `FC_RobotStatusMap` writes it |
| `FC_RobotStatusMap` | FC, OB30 network 4 | PI words → `DB_Robot.Status` | `DB_Robot.Status` |
| `DB_Poses` | DB, retentive | taught poses: `P_Home`, `P_Start`, `P_Approach`, `P_C1..P_C4`, `SafeZ`; later more | HMI |
| `FC_PoseCheck` | FC | pose against the limits of 1.4, returns an error number, 0 = ok | nothing |
| `FB_RobotMove` | FB, one instance, OB1 | **the single movement**. Takes one `UDT_MoveReq`, drives program 20, reports busy / done / error | `DB_App.Srl.Pose[1..5]`, `DB_App.Srl.Par[1..4]`, `DB_App.Srl.Gate[1..8]`, `DB_App.Srl.GateEnable`, `DB_App.Cmd.Start`, `DB_App.Cmd.ProgramNumber` |
| `DB_RobotMove` | DB | `Cfg` (`intBlocksInPrg20`: how many blocks exist in program 20 today), `Man` (HMI: enable, start, reset and one `UDT_MoveReq`), `Req` and `xReqStart` (the active request), `State` (`UDT_MoveState`) | HMI writes `Man` and `Cfg`. `FC_MoveSelect` writes `Req` and `xReqStart`. `FB_RobotMove` writes `State` |
| `FC_MoveSelect` | FC, OB1 | the one writer of the active request. Stage 1 body: copy `Man`. Stage 2 adds the queue head in Auto. `FB_RobotMove` never changes for that | `DB_RobotMove.Req`, `xReqStart` |

So stage 1 is **four UDTs, three FCs, one FB, three DBs**. OB30 keeps its four networks. OB1 gets one new network with two boxes: `FC_MoveSelect`, then `FB_RobotMove`.

Because `FB_RobotMove` becomes the only writer of `DB_App.Cmd.Start`, `DB_App.Cmd.ProgramNumber` and `DB_App.Cmd.Override`, the HMI must stop writing these three. The HMI override field writes `DB_RobotMove.Man.udtReq.intOverride` instead. The HMI keeps Enable, Reset, Automatic, Simulation, Pause, Stop, Home and Jog.

### 4.2 Stage 2 (automatic) – outline only

| Block | Job |
|---|---|
| `FB_MoveQueue` | FIFO of `UDT_MoveReq`, 1..50. Push from the job, pop into `FB_RobotMove` when the last move is done |
| `FB_ScanCubeJob` | fills the queue with the list of section 3 from `DB_Poses` |
| `FB_Mode`, `FC_Mode` | from the framework: Standby 0, Hand 10–12, Auto 20–22. Hand = single movement from the HMI. Auto = queue |
| `FB_Jobs` | from the framework: job "scan cube" is offered only when referenced, poses taught, comm OK |
| `FB_SeqMechanism` + FCs + DBs | from the framework, unchanged |
| `FB_ActRobot` | one block for all robot step numbers: turns `intActStepNr` into a `UDT_MoveReq` and calls `FB_RobotMove`. Step numbers: 301–350 LIN to pose n, 351 lift to `SafeZ`, 010–014 checks, 020 question "area clear?" |
| `FC_MoveSelect` | the one writer of `DB_RobotMove.Req`: copies `Man` in Hand mode, the queue head in Auto mode |

### 4.3 Built for change – what a future change touches

Every layer talks to the next one through **one UDT**. A change stays inside one layer. The layers, top to bottom:

```
HMI, mode, jobs            FB_Mode, FB_Jobs, FB_OperBox (framework)
      |  job number, mode bits
job builder / step chain   FB_ScanCubeJob, later FB_ScanDrumJob, or FB_SeqMechanism + FB_ActRobot
      |  UDT_MoveReq, pushed into FB_MoveQueue
move source                FC_MoveSelect: hand = HMI, auto = queue       -> DB_RobotMove.Req
      |  UDT_MoveReq
move engine                FB_RobotMove                                  -> DB_RobotMove.State (UDT_MoveState)
      |  DB_App.Srl, DB_App.Cmd (down)        DB_Robot.Status (up)
bus                        OB30: handshake, module 1 FB, module 2 FB, FC_RobotStatusMap
```

| Change you may want later | What changes | What stays untouched |
|---|---|---|
| New block in program 20 (circle, slow linear, …) | SRL: one new `IF BOOL[50+n]` section. PLC: `DB_RobotMove.Cfg.intBlocksInPrg20` + 1. HMI: one text in the block list | `FB_RobotMove`, UDTs, OB30, bus |
| Circle needs a second pose (via point or center) | MOVISUITE: pose setpoints 1 → 2, so `POSE[52]` exists. OB30 network 3: the three "first word" numbers move. `UDT_MoveReq.udtPose2` is already there | everything above the bus |
| Speed per move | `UDT_MoveReq.intOverride` is already there. `FB_RobotMove` writes `Cmd.Override` when it is > 0 | – |
| Automatic mode | add the layers above the engine: `FB_MoveQueue`, `FC_MoveSelect`, a job builder | `FB_RobotMove`, OB30 |
| Step chain of the framework | `FB_ActRobot` turns `intActStepNr` into a `UDT_MoveReq` and pushes it | `FB_RobotMove`, queue |
| New job (drum, …) | a new job builder FB that pushes `UDT_MoveReq`; poses in `DB_Poses` 8..50 | queue, engine, bus |
| Laser sensor | `UDT_MoveReq.xStopOnSensor` and the pin `i_xSensorStop` are already there; `State.udtPoseAtEnd` keeps the pose at the stop | – |
| More SRL reals or bools on the bus | MOVISUITE count, OB30 network 3 numbers, `DB_App.Srl.Par[..]` grows | the engine reads by index |
| Limits change | `FC_PoseCheck` constants and section 1.4 | – |
| New HMI screen | reads and writes DBs only | – |

Rules that make this work:

1. A new field goes at the **end** of a UDT. Existing field names never change. Then no call and no HMI tag breaks.
2. Block numbers, action numbers and pose indexes are **never reused** with a new meaning. A retired number stays retired.
3. Bus word numbers live in **one place**: the pins of OB30 network 3 and the module 1 FB. No other block knows a word number.
4. The move engine knows only the **contract of program 20** (section 5.1): gate n, `POSE[51]`, `REAL[51..53]`, `REAL[54]`, echo in `REAL[71]` and `REAL[72]`. It does not know what a block does.
5. Every change: a line in section 9 with a date, and the block `VERSION` goes up.
6. TIA folders as in the drum press: `10000 Framework` (mode, jobs, sequence), `20000 Robot` (OB30 pieces, `DB_Robot`, `FB_RobotMove`, `DB_RobotMove`, `DB_Poses`, `FC_PoseCheck`), `21000 Robot Jobs` (queue, job builders, `FB_ActRobot`).

---

## 5. Stage 1 in detail – the single movement

### 5.1 Program 20 on the robot (we write it together in RobotMonitor)

**The contract** between the PLC and program 20. The move engine knows only this table.

| Item | Rule |
|---|---|
| Block n | gated by `BOOL[50 + n]`, n = 1..8. One motion command per block |
| Target pose | `POSE[51]` for every block |
| Block parameters | `REAL[51..53]`, meaning per block. Circle: `REAL[51]` = radius |
| Run number | `REAL[54]`, written by the PLC before start |
| Echo | the block writes its number into `REAL[71]`. The last program line copies `REAL[54]` into `REAL[72]` |
| Path (engine v0.6, 2026-11-02) | `POSE[1..7]`, the seven standard path segments of module 1, written by the PLC into `DB_RobotPath` before the start when the request carries a path (`intPathLen` > 0). Blocks 5, 6, 7 drive them in one program run. `intPathLen` = 0 is a single move exactly as before |
| Adding a block | one new `IF BOOL[50+n]` section in the SRL, `Cfg.intBlocksInPrg20` + 1 in the PLC, one text on the HMI. Nothing else |

```
// program 20 – gated blocks. every block = one motion to POSE[51]
// nothing moves if all gates are FALSE

IF  BOOL[51] = TRUE                // block 1: linear
    LIN_VAR  POSE[51]              // blending 0
    REAL[71] := 1                  // tell the plc which block ran
END_IF

IF  BOOL[52] = TRUE                // block 2: linear, slower motion set (optional)
    LIN_VAR  POSE[51]
    REAL[71] := 2
END_IF

IF  BOOL[53] = TRUE                // block 3: circle ccw, radius REAL[51]   – later
    CIRC_VAR ...
    REAL[71] := 3
END_IF

IF  BOOL[54] = TRUE                // block 4: circle cw                      – later
    ...
    REAL[71] := 4
END_IF

REAL[72] := REAL[54]               // echo the run number, last line
// program ends here → "program finished" (PI 2.5)
```

Manual facts behind this (chapter 9.7 and 11.9, 11.12): select the program number, write the SRL variables, set start, wait for "program running", then "program finished", reset start. The new variable values take effect when the robot executes the command that uses them. A program number may only be changed while the program is initialized or finished.

### 5.2 Bus words used by one move

| Direction | Word | Meaning | PLC tag |
|---|---|---|---|
| PLC → robot | PO 4 low byte | program number 20 | `DB_App.Cmd.ProgramNumber` |
| PLC → robot | PO 2, start bit | program start, held until finished | `DB_App.Cmd.Start` (FB pin `start`) |
| PLC → robot | 114–118 | POSE[51] X Y Z RotZ RotY, whole mm / deg | `DB_App.Srl.Pose[1..5]` |
| PLC → robot | 119–121 | REAL[51..53] block parameters, from `aorPar[1..3]` | `DB_App.Srl.Par[1..3]` |
| PLC → robot | 122 | REAL[54] run number | `DB_App.Srl.Par[4]` |
| PLC → robot | PO 4 high byte | override, only when `Req.intOverride` > 0 | `DB_App.Cmd.Override` |
| PLC → robot | 123 bits 0–7 | BOOL[51..58] gates | `DB_App.Srl.Gate[1..8]` |
| robot → PLC | PI 2.3 | program running | `DB_Robot.Status.xPrgRunning` |
| robot → PLC | PI 2.5 | program finished | `DB_Robot.Status.xPrgFinished` |
| robot → PLC | PI 1.4 | robot moving | `DB_Robot.Status.xMoving` |
| robot → PLC | PI 12 bits 8–15 | remaining segments | `DB_Robot.Status.intRemainingSeg` |
| robot → PLC | 65 | REAL[71] which block ran | `SrlRealRaw[1]` |
| robot → PLC | 66 | REAL[72] run number echo | `SrlRealRaw[2]` |

### 5.3 `FB_RobotMove` – pins

| Pin | Type | Meaning |
|---|---|---|
| `i_xStart` | Bool | rising edge starts the move |
| `i_xEnable` | Bool | moving allowed. Stage 1: HMI "hand" bit. Stage 2: from `FB_Mode`. Falling edge while busy = abort |
| `i_xReset` | Bool | clears the error |
| `i_xSensorStop` | Bool | reserved for the laser. Used only when `Req.xStopOnSensor` |
| `i_udtReq` | UDT_MoveReq | the move: block, pose, parameters |
| `i_intBlocksInPrg20` | Int | `DB_RobotMove.Cfg.intBlocksInPrg20` |
| `i_udtStatus` | UDT_RobotStatus | `DB_Robot.Status` |
| `i_intEchoBlock` | Int | `SrlRealRaw[1]`, REAL[71] |
| `i_intEchoRun` | Int | `SrlRealRaw[2]`, REAL[72] |
| `io_udtSrl` | DB_App.Srl | pose, par, gates, gate enable |
| `io_xStart` | Bool | `DB_App.Cmd.Start` |
| `io_intPrgNr` | Int | `DB_App.Cmd.ProgramNumber` |
| `io_intOverride` | Int | `DB_App.Cmd.Override`, written only when `Req.intOverride` > 0, clamped to 1..100 |
| `io_aoudtPath`, `io_aorPathBlend` | Array[1..7] of UDT_Pose, Array[1..7] of Real | `DB_RobotPath.aoudtP`, `.aorBlend` (v0.6), written in step 10 when the request carries a path |
| `o_udtState` | UDT_MoveState | busy, done, error, error number, step, run number, pose at the end |

Few pins, two UDTs. A new field in a UDT does not change the call in OB1.

### 5.4 `FB_RobotMove` – the steps

| Step | Name | What happens | Goes on when |
|---|---|---|---|
| 0 | idle | busy FALSE. v0.7: program number 20 and start FALSE written every cycle, so the bus is defined in idle and the first move after a download is not a program change | start edge and enable |
| 5 | check | block 1..`Cfg.intBlocksInPrg20`, `FC_PoseCheck` = 0, comm OK, ready, referenced, automatic, access granted, no fault, setpoints active, STO inactive and output stage on (v0.7), not moving, program not running or paused, override > 0. v0.6: every path pose checked too, error 16 if more than 7. v0.7: a path block 5..7 without a path → error 17; the request is copied into the block, step 10 writes the copy | all true, else error and back to 0 |
| 10 | write | program number 20, pose into `Srl.Pose[1..5]`, `aorPar[1..3]` into `Par[1..3]`, run number + 1 into `Par[4]` (v0.7: never the number the robot still echoes in REAL[72]), override if given, the path into `DB_RobotPath` when `intPathLen` > 0 (v0.6), all gates FALSE then the gate of the chosen block TRUE (v0.4), gate enable TRUE, start FALSE | "repositioning required" (PI 2.6) → 15, else → 20 |
| 15 | program stop (v0.5) | `State.xPrgStop` TRUE, OB30 NW2 puts it on the stop pin. The robot cancels the way back to the old path and shows the program initialized (manual 9.8, 9.10) | initialized AND NOT repositioning required → stop FALSE, → 20. `T_STOP` = 2 s → error 15 |
| 20 | settle | wait `T_SETTLE` = 100 ms (v0.4) with the pose and gate on the bus, counted from the moment the robot shows program 20 ready: initialized or finished, and the control type inactive (v0.7, manual 6.3) | → 30. `T_PRG` = 3 s without program 20 ready → error 14 |
| 30 | go | start TRUE (the gate is open since step 10) | at once |
| 40 | wait running | "program running" must appear. Also accepted: "finished" with this run's number already echoed in REAL[72], for a move too short to be seen running. While the robot repositions (PI 2.7) the timer does not count | seen → 50. `T_RUN` = 3 s without it → error 11 |
| 50 | wait finished | "program finished" AND NOT moving AND remaining segments = 0, stable for `T_STILL` = 200 ms | seen → 60. Program initialized or paused by someone else → error 18 (v0.7). `T_MOVE` = 60 s → error 12 (`T_MOVE_PATH` = 300 s for a path, v0.6) |
| 60 | end | start FALSE, gates FALSE, echo block = block else error 10, echo run = run else error 13, pose at the end latched, done pulse, busy FALSE | → 0 |
| 90 | error | start FALSE, gates FALSE, busy FALSE, error TRUE | reset → 0 |
| 95 | stop after an error (v0.7) | entered instead of 90 when the error comes while the robot runs its program, repositions or moves (errors 7, 11, 12, 20, 21, 22): start FALSE, gates FALSE, `State.xPrgStop` TRUE on the bus, busy stays TRUE, error TRUE at once | robot stands (not running, not repositioning, not moving) or `T_STOP` = 2 s → stop FALSE, → 90 |

Abort: enable falls while in 10–50 → error 20; with the robot in motion through step 95 (program stop), else straight to 90. Before v0.7 an abort only dropped start and gates, and in automatic the robot finished the move while the engine said idle. The HMI Stop button stays what it is today (`DB_App.Cmd.Stop`). Program stop by the engine (v0.5): `State.xPrgStop` in step 15, OB30 NW2 `stop := Cmd.Stop OR State.xPrgStop`; the old screen stays the one writer of `Cmd.Stop`, the engine the one writer of `State`.

"Program finished" (PI 2.5) is the normal state of program 20 between two moves: every run of program 20 ends there, and the next start works from it (manual 6.3 diagram, 9.7). The engine's step 20 accepts initialized or finished. Nothing clears it, no machine off/on is needed.

### 5.5 Error numbers

| ErrId | Meaning |
|---|---|
| 1 | block number not 1..8, or block not yet in program 20 (`Cfg.intBlocksInPrg20`) |
| 2 | X, Y or Z outside the limits |
| 3 | RotZ or RotY outside the limits |
| 4 | circle without radius |
| 5 | robot not ready / not referenced / not automatic / no access / fault present / setpoints not active / STO active / output stage off (v0.7) |
| 6 | robot still moving, program still running or paused at start |
| 7 | communication not OK |
| 8 | override is 0 in the request and in `Cmd.Override`, nothing would move |
| 10 | wrong block echoed in REAL[71] |
| 11 | no "program running" within 3 s (`T_RUN`), repositioning time not counted |
| 12 | no "program finished" within 60 s (300 s for a path) |
| 13 | run number echo in REAL[72] is not this run |
| 14 | robot does not show program 20 ready within 3 s after the number was written: initialized or finished, control type inactive (v0.7) |
| 15 | program stop not taken: the robot keeps "repositioning required" for 2 s after the stop (v0.5) |
| 16 | path longer than 7 poses in the request (v0.6, never from the jobs) |
| 17 | path block 5, 6 or 7 requested without a path, `intPathLen` = 0 (v0.7). The robot would drive POSE[1..7] as they stand |
| 18 | program stopped or paused by someone else during the move: old screen STOP, RobotMonitor (v0.7) |
| 20 | aborted, enable fell during the move |
| 21 | aborted, robot fault during the move |
| 22 | aborted, access lost during the move (RobotMonitor took over) |

### 5.6 HMI for stage 1

- Inputs: X, Y, Z, RotZ, RotY (whole numbers), override %, button "Move linear" (= start with block 1), button "Reset".
- Later: one button per block ("Move circular" = block 3 or 4) and a radius field. The block names come from one HMI text list keyed by block number.
- Display: busy, done, error with number, step, run number, actual pose from `DB_Robot.Status`, program running / finished, access granted, remaining segments.
- Teach: button "take actual pose into P_n" writes `DB_Robot.Status` pose into `DB_Poses`. Only in hand mode.

### 5.7 Test plan for stage 1

| # | Test | Pass when |
|---|---|---|
| 1 | pose outside the limits, press Move | error 2 or 3, nothing changes on the bus |
| 2 | simulation on (PO 1.9), valid pose, press Move | running seen, finished seen, done pulse, RobotMonitor shows POSE[51] = typed values, REAL[71] = 1, REAL[72] = run number |
| 3 | real move at override 10 % | robot arrives at the pose, done pulse, no error |
| 4 | press Move while busy | ignored, run number does not change |
| 5 | HMI Stop in the middle of a move | robot stops, FB reports error, Reset clears, next move runs |
| 6 | 20 moves in a row | every move runs exactly once, run number counts 1..20 |
| 7 | RobotMonitor takes access during a move | error 5 or 11, PLC does not fight, resumes after reset |

---

## 6. Stage 2 – automatic (outline, to be detailed after stage 1 passes)

1. `FB_MoveQueue`: FIFO of `UDT_MoveReq`. One head, one tail, count. Push, pop, clear.
2. `FB_ScanCubeJob`: on job start, clears the queue and pushes the list of section 3 from `DB_Poses`.
3. `FC_MoveSelect`: hand mode → `DB_RobotMove.Man`, auto mode → queue head. One writer of `Req`.
4. The StepChain framework around it: modes, jobs, chain, action numbers as in 4.2.
5. Laser sensor: a move "towards the cube until the sensor says 10 mm" needs a stop while moving and the actual pose latched at the stop. Options: PLC stop + read PI 7–11, or MOVIKIT touchprobe / MEASURE (manual chapter 11, "MEASURE"). Decided later, after the sensor is chosen.
6. **Path mode** (2026-11-02, one folder since 2026-11-04): the rectangle or the circle as one program run. The robot has seven poses on the bus that nothing used, the standard path segments of module 1, POSE 1 to 7, sent every cycle by OB30 network 2. The engine (v0.6) writes the corners into them through `DB_RobotPath`, opens gate 55 (56, 57), gives one start, and block 5 (6, 7) of program 20 drives them in one run; the moves around it stay single moves of block 1, so the queue carries a mix. Switch `"DB_Recipe".xPathMode`, both jobs; off = one move per point, exactly the proven way. What it buys: one handshake per rectangle instead of seven, and later continuous motion through the corners with blending (the turn lines go, the sides get `rBlend`, the LIN lines take blending "var"). What it costs: an error or an abort points at the whole rectangle. Not in it on purpose: more than seven poses in one request, block 5 by hand from the Robot move screen (a hand request carries no path). Apply: `AUTOMATIC_TODAY.md` part 7; program: `PROGRAM20.md` section 8; screen: `HMI_STAGE2.md` 6.5; tests: `TEST_PROTOCOL.md` part 5; the whole picture: `SOLUTION_MAP.html`.

---

## 7. Open points

| # | Point | Needed for | How to settle |
|---|---|---|---|
| 1 | OB30 network 4 (`FC_RobotStatusMap`) is not in this folder | same status names in `FB_RobotMove` | put the network text in the folder |
| 2 | Can the SRL write REAL[71] / REAL[72] in program 20 | done check | manual shows `REAL[1] := REAL[2] + REAL[3]`, so an assignment block exists. Test it |
| 3 | Who writes `DB_App.Cmd.Start`, `ProgramNumber` and `Override` today | one writer rule | remove them from the HMI, `FB_RobotMove` takes over. The HMI override field writes `DB_RobotMove.Man.udtReq.intOverride` instead |
| 4 | Motion set / speed of the LIN block | speed of one move | set inside the SRL LIN block, override from the HMI. Second block with a slower motion set if needed |
| 5 | Circle move | stage 1 or later | later |
| 6 | Where the PI words of module 1 are | status map | **settled 2026-10-27:** `"DB_MOVIKIT_Robotics-5AX".PI`, `Array[1..64] of Word`, index = robot word |
| 7 | Laser sensor type and signal | stage 2 approach and corner | choose the sensor |

---

## 8. Order of work

1. Agree this file. ✔ when you say so.
2. UDTs, `DB_Poses`, `FC_PoseCheck`, `UDT_RobotStatus`, `FC_RobotStatusMap` check.
3. `FB_RobotMove` in SCL, with the steps of 5.4.
4. Program 20 in RobotMonitor, blocks 1 and 2.
5. HMI fields for stage 1.
6. Tests of 5.7, in simulation first.
7. Then stage 2.
8. From 2026-10-29: `ROADMAP.md` – what closes stage 2 (teach, automatic, faults, operator screen) and what stage 3 is (framework, sensor, drum, production).

---

## 9. Change log

| Date | Change |
|---|---|
| 2026-11-08 | `HMI/SCREENS_V2.html` (published, link in the README): the screens regrouped by who stands at the panel. RUN (operator: AUTO, Enable, START, ABORT, override, progress, engine and robot lamps), SETUP (cube numbers, three switches, teach, go to, poses), HAND (single move, jog, home, actual pose), DIAG (PI 1 and PI 2 bits, the bus side: Cmd.Start, ReadyToStart, MachineOn, PO 2.2 and 2.4, Cmd.Stop and Pause, gates, REAL 71 and 72, run nr, message id at error, handshake). One status strip on all four: mode, enable, robot state, step, job error, engine error, ACK. Every object with its tag, text list and permission bit, numbers T, R, S, H, D; A1 … A50 and M1 … M40 of `SCREENS.html` stay valid. No PLC change; later MOVE, RESET, ABORT, START as `"DB_RobotHand".Btn` bits with `FB_CubeButtons` v0.3 |
| 2026-11-05 | **`FB_RobotMove` v0.7**, after the review `DOCS/ENGINE_REVIEW_2026-11-04.md` (six readings of the code against the manual, every finding attacked by three refuters). Step 0 writes program number 20 and start FALSE every cycle: the bus is defined in idle, the first move after a download is no longer a program change on PO 4 with the start edge 100 ms behind it (the best fit for the open error 11, review 4.1), and a start set by hand cannot stay on the bus (one of the two fits for error 15, review 4.2). The settle of step 20 counts from "program 20 ready", and ready also wants the control type inactive (PI 4.8/9/10, manual 6.3). Step 10 never sends a run number equal to the echo in REAL[72], so a stale echo after a download cannot pass step 40 (review 3.4, hole 1). Step 5 copies the request into the block and step 10 writes the copy, never the live input (a typed pose or an AUTO switch in that one cycle could reach the bus unchecked). A path block 5..7 without a path is refused, new error 17. Step 5 also wants STO inactive and the output stage on (error 5). Step 50 reports a program stopped or paused by someone else at once, new error 18. New step 95: an error while the robot runs puts the program stop on the bus until the robot stands, busy stays TRUE meanwhile; before, an abort only dropped start and gates and the robot finished the move while the engine said idle. Text list TL_Err needs 17 and 18, TL_Step needs 95. No pin change |
| 2026-11-05 | `FC_MoveSelect` v0.3: new pin `i_xJobRunning` := `"DB_RobotJob".State.xRunning` at the end. In auto the start level also needs the job running, so entries left in the queue cannot drive the robot the moment AUTO and Enable go on. `FB_ScanCubeJob` v0.7: the checks of step 10 keep the first cause and put the row into `intMoveActive`; new pin `i_intBlocksInPrg20` := `"DB_RobotMove".Cfg.intBlocksInPrg20` at the end, path mode with block 5 not in program 20 is refused before the first move, new job error 7; safe Z must lie above corner 1, else error 2; with turns, corner 1 must stand at RotZ −180, new job error 8 with its own text (the 2026-11-04 confusion). `FB_CubeButtons` v0.2: a go to writes id, path length 0, blending 0 and parameters 0 into the hand request, not only pose and block. `OB1_Stage2_call.scl`: the two new pins, and `i_xActualValid` of both jobs also wants NOT moving AND NOT program running, so "scan from here" never takes a pose on the way. Text lists: TL_JobErr 7 and 8. `FC_RobotAlarms` bits for 7 and 8 if wanted. Nothing in TIA yet |
| 2026-11-05 | **Program 20, two lines** (`PROGRAM20.md` section 3b): `REAL 71 := 0` as the first line, so REAL[71] = n proves block n was interpreted in this run and error 10 also catches "no block ran"; `WAIT_MOTION Motion_Done` before `REAL 72 := REAL 54`, so the run echo is written at standstill and means "arrived" (manual 2570-2576: variables are not synchronous with the movement; 14022-14031). My proposal of 2026-11-04 to require the echo inside step 50 was wrong for that reason and is not applied |
| 2026-11-05 | `DOCS/ENGINE_REVIEW_2026-11-04.md`: how the engine works step by step with the robot's bits, the timeline of one move, how a move ends and why "finished" alone proves nothing, the two open errors with ranked hypotheses and the watch-table rows that decide, the smooth scan, the review findings and the change set. Open: the two errors need the watch rows of its section 8 at the machine; the hand-set `Cmd.Start` must stop |
| 2026-11-04 | **folders by purpose.** The one folder is sorted into `PLC/` (every block, one version each, `BLOCK_MAP.md`, the OB1 and OB30 pin files), `HMI/` (`SCREENS.html`, the three HMI docs, new `SCREEN_FIXES.md` with the wrong tags and text lists seen on the real screen today), `TEST/` (checklist, protocol, sheet, next steps, new `ERROR_LOG.md`: every error seen at the machine with date, cause, fix, open or solved), `ROBOT/` (`PROGRAM20.md`, the MOVIKIT manual, the monitor doc, the bus map), `DOCS/` (day plan, roadmap, stage and step readmes, review, the two HTML maps), `REF/` (the user's docx, txt exports, photos). `GOALS.md` stays at the root, new `README.md` at the root says what is where and how a day at the machine goes. No file content changed by the move, the published pages keep their links. The rule "one file per block, one version" is untouched: the folders sort by purpose, never by version |
| 2026-11-04 | **first START in automatic refused with job error 4, not 3.** The screen text said "engine busy", the number was 4 and MoveActive "turn at corner 4": corner 1 taught at RotZ 0, the four +90 turns reach +270, outside ±180, the check before the first move stops the job. Two ways out: teach corner 1 at RotZ −180 (the design), or the new switch **"no turn"**: `DB_Recipe` v0.3 field `xNoTurn` at the end, `FB_ScanCubeJob` v0.6 pin `i_xNoTurn` at the end (OB1 pin `"DB_Recipe".xNoTurn`). On: every corner keeps the orientation of corner 1, the four turn entries 5, 7, 9, 11 are not pushed (10 moves with home, 9 without, 7 or 6 with "scan from here"; path mode: 4 segments), corner 1 may stand at any RotZ. Off = exactly v0.5. Also seen on the screen: `TL_JobErr` 4 carries the text of 3, the "Job error" field of the first screen shows 0 while the error lamp is on (wrong tag, must be `"DB_RobotJob".State.intErrId`), the step text shows "Default" for 0, the angle was 100 instead of 0. Screen object A50 in `SCREENS.html` |
| 2026-11-04 | `SCREENS.html` brought to the current state (same link): the Automatic screen has three boxes POSES, CUBE, JOB; the teach objects A3–A5 on `"DB_RobotHand".Btn`, the CUBE box A35–A46 and A49 (three teach buttons with lamps, ready, two go-to buttons, corners text, enable lamps, the two switches), ACK A47 and the mode lamps A48 in the JOB box, TL_Step 15, TL_Err 15 and 16, TL_JobErr 6. `CHECKLIST_CUBE_AUTO.md` section 8: the build order in four stages, PLC → screens box by box → teach pose by pose → scan in automatic, each with its test |
| 2026-11-04 | `CHECKLIST_CUBE_AUTO.md`: what must be in the project tree (types, DBs, FCs, FBs with versions), every OB1 pin of the nine networks, the OB30 pins that matter, the watch-table values, the robot side, every screen object with its tag (Robot move, JOB box, CUBE box, Poses, old screen), the text lists, and the five-minute proof. For the cube in automatic with teaching from the screen, without drum, job mux or path mode |
| 2026-11-04 | **one folder, one line.** The side folder `robot pose solution` is merged into this folder and deleted: its blocks were supersets of the ones here, and the path mode is a switch, `"DB_Recipe".xPathMode`, off = exactly the proven single moves. One version of each file now: `UDT_MoveReq` v0.3, `DB_Recipe` v0.2, `DB_RobotPath`, `FB_RobotMove` v0.6 (v0.4 gate with the pose, v0.5 program stop, v0.6 path), `FB_ScanCubeJob` v0.5, `FB_ScanDrumJob` v0.4 (and one correction: the lift after the circle keeps the last orientation, no full turn on the way up), `OB1_Stage2_call.scl` with the three path pins, `OB30_complete.scl` with the stop pin and the segment pins on `DB_RobotPath`, `OB30_NW2_pins.md`, `SOLUTION_MAP.html` (same link). Docs folded in: `PROGRAM20.md` section 8 (blocks 5–7), `HMI_STAGE2.md` 6.5 (the switch, the texts), `AUTOMATIC_TODAY.md` part 7 (apply order, confidence), section 5.1 contract line "Path", section 6 item 6, error 16. Rule from today: every change goes into the one file of the block, behind a switch or as a field at the end, never into a side folder. Nothing in TIA changes by this merge |
| 2026-11-03 | **"program finished" after a jog, machine off/on used to clear it.** Not needed: finished is the normal state of program 20 between two moves, the next start works from it (manual 6.3 state diagram, 9.7), the engine's step 20 accepts it. The real point after a jog is "repositioning required" (PI 2.6): a start then first drives the robot back to the point where it left the old path, then runs the block (6.3, 6.6, 9.10), wrong for "scan from here" and an unexpected move. `FB_RobotMove` v0.5 (main) and v0.6 (solution): new step 15 between write and settle, entered only when PI 2.6 is set, pulses a program stop through `"DB_RobotMove".State.xPrgStop` (`UDT_MoveState` v0.2, field at the end) until the robot shows initialized without the request, `T_STOP` 2 s → new error 15. OB30 NW2 stop pin becomes `"DB_App".Cmd.Stop OR "DB_RobotMove".State.xPrgStop`; one writer kept on both tags. `FB_CubeButtons` and `DB_RobotHand` are in TIA (user, today). Tests: `TEST_PROTOCOL.md` 2.5 |
| 2026-11-02 | second fault of the day: engine error 11 after the robot was in automatic. Start sent, no "program running" within 3 s. Suspected: "repositioning required" after the jog (manual 6.3, 6.6, 9.10: a start then first repositions to the old path, a program STOP cancels it), or a held `Cmd.Stop` / `Cmd.Pause`, or `ReadyToStart` / `MachineOn` false. Checks in `AUTOMATIC_TODAY.md` part 5 row 11. If it is repositioning: engine v0.4 would pulse program stop when `xRepositionRequired` is set before a start, and own `Cmd.Stop` (one writer). Not decided, waiting for the machine result. Correction: the error came before `FB_RobotMode`, with the old automatic switch, from program state "Initialized" (no old path), so a blocked start (`ReadyToStart` / `MachineOn` in OB30 NW2) is the first suspect. Decisive check: `"DB_MOVIKIT_Robotics-5AX".PO[2]` bit 2 during steps 30–50 |
| 2026-11-03 | **the single move works again** after `"DB_App".Cmd.Start` was set by hand once in DB_App (engine v0.4, gate with the pose, start pin on `Cmd.Start` alone). Keep in mind: the exact cause is not proven. If the move works only after that hand step, a second writer of `Cmd.Start` is still in OB1 and the fault returns after a download or a power cycle; the cross-reference of `Cmd.Start` must show the engine call as its only writer. `FB_StartTrace` stays available but is not needed while it works |
| 2026-11-03 | `FB_StartTrace` v0.1, diagnostic only: records in its instance DB, during one move, the control words to the robot (enable, handshake, automatic, start, pause, stop, jog word, program, override) and the status words from it (ready, setpoints, fault, warning, access, handshake, automatic, initialized, running, finished, repositioning, message id, program, control type, outside workspace) plus Cmd.Start, ReadyToStart, MachineOn, Gate[1] and the engine error; one row per step change, the first ten cycles of step 40, the first three of step 20; the next MOVE clears it. Called in OB30 network 5 after the status map. To be deleted when the start works. With the start pin on `Cmd.Start` alone the engine still ends 0 → 20 → 40 → 90 error 11 |
| 2026-11-03 | machine day, the start is refused: PI words show automatic, initialized, program 20, ready, referenced, setpoints active, access granted, control type inactive, and the message word PI 3 holds 32057 = SEW error 125.57 "Faulty control: the rising edge of the program start is ignored because the robot is not yet ready for movement", first as a warning (PI 1.9), then stored by the engine in `uiMsgIdAtError`. Also found: program 20 on the robot had been changed to `WHILE BOOL 51 / LIN / WAIT 500 ms / END_WHILE` without the echo lines, which the engine can never finish; the IF form must come back. `FB_RobotMove` v0.4 (main) and v0.5 (solution): the gate of the block goes on the bus in step 10 with the pose, settle 100 ms, step 30 only sets the start; before, gate and start left the PLC in the same cycle. Isolation test proposed: start program 20 from RobotMonitor with BOOL 51 set by hand. Not yet tested |
| 2026-11-03 | **the cube box without HMI scripts.** `FB_CubeButtons` v0.1: every button of the box writes one bit into `"DB_RobotHand".Btn` (teach home / start / corner 1, teach nr, clear nr, pose nr, go to start / corner 1, ack); the block gives FB_Teach the pose number and a one-cycle pulse, copies row 2 or 4 into the hand target with block 1 and a start pulse, pulses engine reset and job abort on ack, and computes four lamp bits in `.Lamp` (teach allowed, go allowed, cube ready, computed). It is the one writer of `Teach.intPoseNr`, `.xTeach`, `.xClear`; the pose nr field, TEACH and CLEAR move to `Btn`. Documented exception: the go to and ack pulses on `Man.xStart`, `Man.xReset`, `Man.xAbort`. OB1 network before teach in both OB1 references. `HMI_CUBE_HOWTO.md`: the drawing, the PLC part, the HMI tags, five WinCC Unified recipes (momentary button, I/O field, lamp, switch, text by visibility, text list), the object table, the tests, one writer. `HMI_STAGE2.md` 6.1/6.2 and `SOLUTION_MAP.html` on the bits. Not in TIA, not tested |
| 2026-11-03 | `TEST_PROTOCOL.md` written for the machine day after the download: part 0 what is in the PLC (versions, compile, watch table of 14 tags, robot program, screen), 1 single move, 2 robot mode by the PLC, 3 teaching, 4 cube one move per point (simulation, real, abort, computed corners, wrong number, scan from here, five in a row), 5 path mode (POSE 1..7 check, simulation, real, abort, five in a row), 6 drum (hand quarter, teach, single, real, path, clockwise), 7 override up. Every box with its expected result and what to send back |
| 2026-11-03 | `robot pose solution/OB30_complete.scl`: the four networks of the cyclic OB written out in full from your copy of 2026-10-29. Every pin you have stays (start with `ReadyToStart AND MachineOn`, module 2 `enable := Cmd.Enable`, `REAL_TO_INT` on the segments, the homing helper on the jog pins). Only the 35 pose pins and the 7 blending pins of the segments read `"DB_RobotPath"`. Segment 1 gets the `Motion_Set_1`, `wait_1`, `path_end_1`, `blending_distance_1` lines your copy did not have, deletable if the FB lacks a pin. Gate comments name blocks 5, 6, 7 |
| 2026-11-02 | `robot pose solution/SOLUTION_MAP.html` published: the solution on one page, the Automatic screen with 46 numbered objects and the tag or script on each, OB33 network 1–4 and OB1 network by network with every pin, the DB / FB / FC / UDT cards with their one writer and where they stand (TIA, main folder, this folder), a data-flow drawing of one cycle and a drawing of the path block |
| 2026-11-02 | **robot pose solution**, folder `robot pose solution`, self-contained. The rectangle or the circle as one program run: the PLC writes up to seven poses into the module 1 standard path segments, `POSE[1..7]`, and block 5 (7 LIN), 6 (4 CIRC ccw) or 7 (4 CIRC cw) of program 20 drives them. Contract 5.1 gets the line "path = POSE[1..7]". Files: `UDT_MoveReq` v0.3 (`aoudtPath[1..7]`, `intPathLen` at the end), `DB_RobotPath` (one writer: the engine), `DB_Recipe` v0.2 (`xPathMode`), `FB_RobotMove` v0.4 (checks every path pose, writes `DB_RobotPath` in step 10, unused segments = last pose, `T_MOVE_PATH` 300 s, error 16 path > 7, radius check on 6 and 7), `FB_ScanCubeJob` v0.5 (3 singles + rectangle id 15 + 4 singles), `FB_ScanDrumJob` v0.4 (3 singles + circle id 11 + 3 singles), `OB1_call.scl`, `OB30_NW2_pins.md` (35 pose pins and the blending pins from `DB_RobotPath`), `PROGRAM20_blocks5to7.md`, `HMI.md` (CUBE box with teach buttons, PATH MODE switch, TL_Block 5–7, TL_Move 15, TL_MoveDrum 11, TL_Err 16), `README.md` (order, tests, confidence). The single-move way stays, `xPathMode` picks. Not in TIA, not tested |
| 2026-11-02 | `PROGRAM20.md` section 3a: the SRL program as photographed on the robot, line by line against section 3, with the robot's own variable names (`PDO114-118 Extra 1` = POSE 51, `PDO123.0 Extra 1` = BOOL 51, `PDO119 Extra 1` = REAL 51, `PDO120 Extra 2` = REAL 52, `PDI66 Extra 1` = REAL 71). Blocks 1 to 4 are on the robot; one motion set at N10 for all; the circle as radius + angle; the run echo line and END_PROG wait not on the photo, to check. Nothing missing for the cube |
| 2026-11-02 | `FB_ScanDrumJob` v0.3: the four arc moves also carry `aorPar[2]` = 90, the arc angle in `REAL[52]`, so the circle line on the robot may be written as radius + end point (`PROGRAM20.md`) or as radius + angle (the N110 seen on the robot today). Both have their data, the hand test decides. Program 19 offered by you: not needed, one program with gated blocks covers everything, the engine is fixed on 20; a backup copy of the working program as 19 in RobotMonitor costs nothing and needs no PLC change |
| 2026-11-02 | **orientation −180 to +180.** `FB_ScanCubeJob` v0.4: in dimension mode RotZ turns `ROTZ_STEP` (+90°) at every corner, corner 1 −180, corner 2 −90, corner 3 0, corner 4 +90, the fourth turn and the lift +180; no wrap on purpose, a corner 1 that is not at −180 ends outside ±180 and stops with job error 4 at move 11 before any move; taught mode unchanged. `FB_ScanDrumJob` v0.3: the circle starts at the angle −offset, where the tool looking at the axis stands at RotZ −180 (ccw) or +180 (cw), and RotZ runs −180 + 90k or 180 − 90k without a wrap; `rStartAngle` is no longer used (pin and DB field kept, reserved). The old bounded wrap could have given a 270° turn at the ±180 boundary. `HMI_STAGE2.md` sections 5 and 6.3, `AUTOMATIC_TODAY.md` 5b, 19, 21, `PROGRAM20.md`, `ROADMAP.md`, `BLOCK_MAP.md`, `SCREENS.html`, `SYSTEM_MAP.html`. Not in TIA, not tested |
| 2026-11-02 | **"scan from here", both ways exist.** `FB_ScanCubeJob` v0.3 and `FB_ScanDrumJob` v0.2: three new inputs `i_xFromHere`, `i_udtActual`, `i_xActualValid`. With the switch on, START takes the actual pose (`"DB_Robot".Status.udtPoseBcs`) as corner 1 or as the drum center, nothing is taught or stored, and the list begins with side 1 (cube, index 4) or above P0 (drum, index 2): `intFirst`, the check and push loops and `intMovesTotal` start there, the TL_Move texts stay valid. New job error 6 when the switch is on but the robot is not referenced / comm not ok, or (cube) length and width are 0; `FC_RobotAlarms` bit 15. The switch is `"DB_Recipe".xFromHere` (retentive, one for both jobs, lives there so `DB_RobotJob` in the PLC is untouched). `OB1_Stage2_call.scl` three pins on each job, `HMI_STAGE2.md` (TL_JobErr 6, bit 15, SCAN FROM HERE in the CUBE box, 6.4), `AUTOMATIC_TODAY.md` (step 1b pins, part 2b, part 5), `BLOCK_MAP.md`, `SCREENS.html` A38. Not yet in TIA, not tested. Engine, queue, bus, program 20 unchanged |
| 2026-11-02 | `HMI_STAGE2.md` section 6 made functional: box "CUBE" with length, width, angle, the computed/taught text, TEACH HOME / START / CORNER 1 as one-button scripts (pose number then teach bit), lamps on `aoxValid[1]`, `[2]`, `[4]`, GO TO START / CORNER 1 (copy pose, block 1, momentary start), ACK (engine reset and job abort together), robot mode lamps on `Status.xAutomatic`. HMI scripts on existing tags, no PLC change |
| 2026-11-02 | **the cube by its dimensions.** `FB_ScanCubeJob` v0.2: three new inputs `i_rLength`, `i_rWidth`, `i_rAngle`; when length and width are > 0 the corners 2, 3, 4 and the approach are computed from corner 1 (pose 4): corner 2 = corner 1 + length along the angle, corner 4 = corner 1 + width turned 90° left, corner 3 = corner 2 + width, approach = corner 1 at safe Z; the computed corners keep Z, RotZ, RotY of corner 1; error 1 then checks poses 2 and 4 only; length 0 = taught corners as v0.1. New `DB_Recipe` (retain): `Cube.rLength/rWidth/rAngle` and the `Drum` struct, moved there from the never-imported `DB_Poses.scl`, so `DB_Poses` in the PLC is never touched by a download and the taught poses stay. `DB_Poses.scl` back to the PLC version. `OB1_Stage2_call.scl`: three pins on the cube job, drum inputs from `DB_Recipe`. `FB_ScanDrumJob.scl` comments, `HMI_STAGE2.md` section 6 (fields, which corner is corner 1, moved cube = teach row 4 only), `AUTOMATIC_TODAY.md` step 1b and part 2b, `BLOCK_MAP.md`, `PROGRAM20.md`, `SCREENS.html` A35–A37, `SYSTEM_MAP.html` tags. Not yet in TIA, not tested. Queue, engine, bus, program 20 unchanged |
| 2026-11-02 | `FB_RobotMode` v0.1 written: the PLC sets the robot's operating mode, the one writer of `"DB_App".Cmd.Automatic` through an InOut pin. Rule: automatic is the rest state; manual when a jog or home command is set, our AUTO is off, the engine is not busy and no program runs; back to automatic 500 ms after the last command fell, robot standing, not referencing. `i_xJogWanted` = the or of the 15 jog and home commands and `"DB_App".Home.active`, 16 parallel contacts in LAD. OB1 network after NW8, before teach (`OB1_Stage2_call.scl`). The old screen automatic switch must go, a lamp on `Status.xAutomatic` instead. Step 1a in `AUTOMATIC_TODAY.md`, rows in `BLOCK_MAP.md`, `HMI_STAGE1.md` section 4, `SCREENS.html` switch table. Not yet in TIA, not tested |
| 2026-11-02 | machine day, first fault: engine error 5 on MOVE and on START while jog works. Photos: RobotMonitor "Manual with high speed" highlighted, ready, referenced, setpoints active, no fault, PLC has access. Cause: the robot is in manual, the engine needs automatic (PI 2.0), jog needs manual. Fix: `"DB_App".Cmd.Automatic` ON for MOVE and START, OFF for jog. `AUTOMATIC_TODAY.md` part 5 row 5 and step 8 updated (safe Z 450 while all poses stand at Z 691). Seen on the robot, for part 6: program 20 line N110 is `CIRC_CCW` with Radius `REAL[51]` and Angle `REAL[52]`, not RadiusEndpos with `POSE[51]`; one `MotionSet := 1` at N10 for all blocks. The drum job or the program line must change before part 6. No block changed |
| 2026-11-01 | `BLOCK_MAP.md` second edition after your check in TIA: every table has a "where" column (TIA / folder), so `UDT_JobOut`, `FC_JobMux`, `FB_ScanDrumJob`, the new DB fields and `DB_RobotHand` stand in the main tables. New A6: the pins of every block with R (Input), W (Output), RW (InOut) and the tag on each, as the cross-reference shows them. Correction in B4: an InOut pin takes the whole array, so the RW stands on `"DB_App".Srl.Pose`, `.Srl.Par`, `.Srl.Gate`, `"DB_Poses".aoudtP`, `.aoxValid`, `"DB_MoveQueue".Q`; the elements show only R. The first edition put the RW on `Srl.Pose[1]`, that was wrong. W and RW both count as writer. No block changed |
| 2026-11-01 | `SCREENS.html` written for the test day: the Robot move screen (M1–M40), the Automatic screen (A1–A34) and the old robot screen (O1–O9) with every object, kind, tag, limit and text list; the switch positions per step (the robot's own automatic switch `Cmd.Automatic` must be OFF for jog and home, ON for MOVE and START, manual 6.1.1); the six text lists; and five ladder rungs for jog X and home X (robot manual → hand ok → jog X+/X− levels → home X as a 500 ms TP pulse on `Cmd.Home_X1`) with the table for Y, Z, RotZ, RotY. New `DB_RobotHand.scl` for the rungs, not in TIA, not needed for the cube. `AUTOMATIC_TODAY.md` part 2 names the automatic switch. Facts from the manual: jog is a level (PO 3), referencing start is an edge (PO 2.10–2.14), both only in manual, no signal set at a fault reset (6.1.4, 6.2, 6.7). No block changed |
| 2026-11-01 | `BLOCK_MAP.md` rewritten in two layers: part A = the version in TIA (OB30 as in your file: NW3 `enable := Cmd.Enable`, NW2 start gated by `ReadyToStart AND MachineOn`; OB1 NW9 teach, NW10 cube job → `Push`/`State`, NW11 queue, NW12 select + engine, NW13 alarms; `FC_MoveSelect` v0.1 in your OB1 of 2026-10-29, step 1 makes it v0.2), part B = one-writer table, the HMI may-write list, the bug (the target fields of 2026-10-30; the four old-screen elements Start / program number / override / enable; the half-done part 6 that makes `Push` double-written; the `DB_Poses` exception) and the cross-reference check, part C = the drum, only in the folder. Audit of every block: no second writer inside the PLC code, the risks are HMI bindings. `AUTOMATIC_TODAY.md` part 0 pre-flight added (blocks, OB1 order, cross-references, old screen, new screens, watch table with `ReadyToStart`, `MachineOn`, `Cmd.Automatic`, robot, one single move); part 5 job error 10 now says RESET before ABORT. `HMI_STAGE1.md` section 4: the old enable switch goes to `Man.xEnable`. No block changed, no design change |
| 2026-10-31 | **drum job and job selector.** New: `UDT_JobOut`, `FB_ScanDrumJob` (center = pose 8, `"DB_Poses".Drum` fields: scan height, radius, start angle, RotZ offset, clockwise; ten moves, four quarter arcs with blocks 3/4), `FC_JobMux` (the one writer of `"DB_RobotJob".Push`, `.State`, `.intJobActive`; follows `Man.intJob` only while no job runs). Changed: `DB_RobotJob` (`Man.intJob`, `Cube`, `Drum`, `intJobActive` at the end), `DB_Poses` (`Drum` struct, pose 8 reserved), `FC_RobotAlarms` (job error 5 → bit 9), `FB_ScanCubeJob` outputs now go to `Cube`, both job starts gated by `Man.intJob` in OB1 (`OB1_Stage2_call.scl`). `HMI_STAGE2.md` section 5, `PROGRAM20.md` section 5, `BLOCK_MAP.md`, `AUTOMATIC_TODAY.md` part 6 updated. `SYSTEM_MAP.html` published: OB30, OB1 and the four flows with every tag. Not yet in TIA, not tested. Engine, bus, queue, contract unchanged |
| 2026-10-30 | `BLOCK_MAP.md` written: every UDT, DB, FC and FB, where each is called in OB30 and OB1, and the one-writer table. No design change |
| 2026-10-30 | `PROGRAM20.md` rewritten complete: 22 lines with blocks 1–4, the circle blocks as `CIRC_VAR` with description RadiusEndpos, plane XY, radius `REAL[51]`, end point `POSE[51]` (manual 6.16, 11.9.2). `END_PROG` must wait for the movement to end. Circle needs the add-on Circle license and activation in MOVISUITE. Section 5 shows the drum scan as four quarter arcs, one queue entry each, for `FB_ScanDrumJob` in stage 3. Contract unchanged |
| 2026-10-30 | the target fields of the stage 1 screen were bound to the old pose tags, the engine overwrote them with the empty request and the robot drove to the origin. Fixed in the HMI: the five target fields point to `"DB_RobotMove".Man.udtReq.udtPose.*`. The single move now reaches the typed pose. `AUTOMATIC_TODAY.md` written: one file, 15 numbered steps from `FC_MoveSelect` v0.2 to five scans in a row, with the error tables. Replaces `NEXT_STEPS.md` for the machine day |
| 2026-10-30 | **stage 1 passed**: the single move works on the machine after the review corrections. `NEXT_STEPS.md` rewritten: three checks, teach the seven poses, first scan in simulation, first real scan, five in a row |
| 2026-10-29 | `ROADMAP.md` written: stage 2 is done when one button scans the cube five times in a row from taught poses with clean stops. Missing: teach on the machine, automatic tests C4–C12, alarm view and provoked faults, one operator screen. Stage 3 = framework, laser sensor, drum job, production features. Circle blocks 3 and 4 move to the drum |
| 2026-10-29 | `PROGRAM20.md` rewritten as a RobotMonitor block list (New Block values, fields, end of program). Stage 1 program = blocks 1 and 2 only, each with its own `SET_MOTIONSET`, `Cfg.intBlocksInPrg20` = 2. The circle blocks 3 and 4 are removed until stage 2 step 8 |
| 2026-10-29 | evening review of the project state, `REVIEW_2026-10-29.md`. Decisions: `Cmd.Enable` has one writer, OB1 network 8, with the robot screen Enable switch as a requester branch; module 2 `enable` stays TRUE; `FC_MoveSelect` must be v0.2 before any stage 2 test. The robot now runs program 20 (error 11 solved), the target pose is traced with `TEST_SHEET.md` section C |
| 2026-10-27 | file created from the talk of tonight |
| 2026-10-27 | agreed. Step 2 files written: `UDT_Pose`, `UDT_MoveReq`, `UDT_RobotStatus`, `DB_Poses`, `DB_Robot`, `FC_PoseCheck`, `FC_RobotStatusMap`. See `STEP2_README.md` |
| 2026-10-27 | `DB_Poses` holds one array `aoudtP[1..50]` with fixed indexes 1 home, 2 start, 3 approach, 4–7 corners, instead of seven named poses. Same content, and stage 2 action 301–350 maps straight to the index |
| 2026-10-27 | `FC_RobotStatusMap` takes the PI words of module 1 as an array input `i_aowPI`, plus `HS_Alive` and module 2 `CommOK` |
| 2026-10-27 | the two DB sources are `.scl` files (`DB_Poses.scl`, `DB_Robot.scl`), because TIA on this PC does not take `.db` as an external source |
| 2026-10-27 | PI source settled: `"DB_MOVIKIT_Robotics-5AX".PI`, `Array[1..64] of Word`. `i_aowPI` is now `Array[1..64]` and takes it directly. Slot addresses added to section 1 |
| 2026-10-29 | `FB_RobotMove` v0.3: step 5 checks the start conditions of manual 9.4 (setpoints active → 5, paused → 6, override 0 → new error 8). Error 11 on the machine still open: ReadyToStart, program 20 initialized and no repositioning were all fine |
| 2026-10-29 | first machine test gave error 11. `FB_RobotMove` v0.2: step 20 also waits for program 20 initialized or finished (new error 14), step 40 does not count while repositioning is active, `T_RUN` 3 s. Checks on the machine: `ReadyToStart`, program number on the robot, repositioning required |
| 2026-10-28 | stage 2 steps 4–8 and the alarms of 10 written: `UDT_MoveQueue`, `UDT_JobState`, `DB_MoveQueue`, `DB_RobotJob`, `FB_Teach`, `FB_MoveQueue`, `FC_MoveSelect` v0.2, `FB_ScanCubeJob`, `FC_RobotAlarms`, `PROGRAM20.md`, `HMI_STAGE2.md`, `STAGE2_README.md` with tests part C. `FB_ActRobot` written for step 9 but not to be imported before the framework is in. Cube sides: each corner is taught with the tool facing the side that starts there; side n = xyz of corner n+1 with the orientation of corner n. `FB_RobotMove` unchanged |
| 2026-10-28 | stage 2 planned in `STAGE2_PLAN.md`: steps 4 teach, 5 queue, 6 select, 7 cube job, 8 blocks 2–4, 9 framework, 10 faults, 11 laser. Starts after stage 1 passes |
| 2026-10-28 | step 3 written: `FB_RobotMove`, `DB_RobotMove`, `FC_MoveSelect`, OB1 wiring reference. See `STEP3_README.md`. Errors 13, 21, 22 added. Step 40 also accepts "finished + run number echoed" for very short moves. `FC_MoveSelect` exists from stage 1 on, so OB1 never changes for stage 2. HMI start and reset are momentary buttons, the FB takes the edge, nobody clears them |
| 2026-10-27 | you asked for a structure that does not need rebuilding when we add circle, automatic, step chain, drum or the laser. Section 4.3 added. `UDT_MoveReq` v0.2: `intType` → `intBlock` (block of program 20), plus `udtPose2`, `aorPar[1..3]`, `intOverride`, `xStopOnSensor`, `intId`. New `UDT_MoveState`. `FB_RobotMove` reports through `o_udtState` instead of six pins. Run number moves to `REAL[54]`, parameters are `REAL[51..53]`. Override moves from the HMI into the move request |
