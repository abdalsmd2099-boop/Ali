# BGE Portal — PROJECT MAP

One file. Every DB with every variable, every block with where it sits and what goes on its pins. Read this, not the others. The others are detail.

Three kinds of things appear here:

| Mark | Meaning |
| --- | --- |
| **GEN** | generated from `plc/BGE_Scan.scl`. You do not type it, TIA makes it. |
| **YOURS** | exists in your project today. Keep it, change what the table says. |
| **NEW** | does not exist yet and must be created by hand: the `Status`, `Req` and `Cmd` groups inside `DB_App`. |

Only **three** blocks are pinned by hand: `FC_SrlWindow` in OB30, `FC_ScanControl` and `FC_CmdArbiter` in OB1. Every other generated block is called inside those three and needs nothing from you.

---

## 1. Where everything is

| Block | Kind | Mark | Called in | Network | Instance DB | Pinned by hand |
| --- | --- | --- | --- | --- | --- | --- |
| device module FB | FB | YOURS | OB1 | 1 | yours | already |
| `FC_Mode` / `FB_Mode` | FC, FB | YOURS | OB1 | 2 | `DB_Mode` | already |
| control ON network | LAD | YOURS | OB1 | 3 | — | already |
| `FB_MachineReady` | FB | YOURS, change 3 outputs | OB1 | 4 | `DB_MachineReady` | section 4.1 |
| `FC_DriveHome` | FC | YOURS, change 1 output | OB1 | 5 | — | section 4.2 |
| `FB_OperBox` | FB | YOURS, change 3 outputs | OB1 | 6 | `DB_OperBox` | section 4.3 |
| `FC_ScanControl` | FC | GEN | OB1 | 7 | — | **yes, 21 pins, 4.4** |
| `FC_CmdArbiter` | FC | GEN | OB1 | 8 | — | **yes, 25 pins, 4.5** |
| `FB_Alarms` | FB | YOURS | OB1 | 9 | `DB_Alarms` | section 4.6 |
| `FB_StopRecord` | FB | YOURS | OB1 | 10 | `DB_StopRecord` | already |
| handshake toggle | LAD | YOURS | OB30 | 1 | — | already |
| `FC_SrlWindow` | FC | GEN | OB30 | 2 | — | **yes, 18 pins, 3.1** |
| module 1 FB | FB, SEW | YOURS | OB30 | 3 | `DB_MOVIKIT_Robotics-5AX` | section 3.2 |
| module 2 FB v0.7 | FB, SEW | YOURS | OB30 | 4 | `DB_MOVIKIT_ROBOTICS_Module2` | already |
| `FC_RobotStatusMap` | FC | YOURS, writes `DB_App.Status` | OB30 | 5 | — | section 3.3 |
| `FB_ScanJob` | FB | GEN | inside `FC_ScanControl` | — | `DB_ScanJob` | no |
| `FB_SrlMove` | FB | GEN | inside `FB_ScanJob` | — | inside `DB_ScanJob` | no |
| `FC_GenRect` | FC | GEN | inside `FC_ScanControl` | — | — | no |
| `FC_AddRow` | FC | GEN | inside `FC_GenRect` and `FC_ScanControl` | — | — | no |

Removed from the project: `FC_SrlGates`, `DB_App.Srl`, the old `DB_Job`, and the seven-segment path `FC_RC_Mngmt`, `FB_RC_Mechanism`, `FB_RobotOrder` until a job-type selector exists.

---

## 2. The DBs, every variable

### 2.1 `DB_App` — YOURS, with the NEW groups. The robot interface.

Target declaration. Your `DB_App` keeps `HS`. The groups `Status`, `Req` and `Cmd` are added or reshaped to exactly this. `Srl` is deleted.

```
DATA_BLOCK "DB_App"
   STRUCT
      HS : Bool;                        // handshake toggle. writer: ob30 network 1

      Status : STRUCT                   // from the robot. writer: fc_robotstatusmap, ob30 network 5
         CommOk : Bool;
         Enabled : Bool;
         Automatic : Bool;
         Referenced : Bool;
         SafeStop : Bool;               // pi 1.1
         Fault : Bool;
         ProgFinished : Bool;           // pi 2.5
         ProgRunning : Bool;
         ProgNo : Int;
         ErrorNo : Int;
         Override : Int;
         ActX : Int;                    // actual pose words, whole mm and deg
         ActY : Int;
         ActZ : Int;
         ActRotZ : Int;
         ActRotY : Int;
         SrlRealRaw : Array[1..16] of Int;   // real 71..86, words 65..80
      END_STRUCT;

      Req : STRUCT                      // requests, one writer per sub-struct
         Machine : STRUCT               // writer: fb_machineready, ob1 network 4
            Enable : Bool;
            Reset : Bool;
            Automatic : Bool;
         END_STRUCT;
         Hand : STRUCT                  // writer: fb_operbox (axis, jog), fc_drivehome (home)
            Axis : Int;                 // 1 x, 2 y, 3 z, 4 r1, 5 r2
            JogMinus : Bool;
            JogPlus : Bool;
            Home : Bool;
         END_STRUCT;
         Auto : STRUCT                  // writer: fc_scancontrol, ob1 network 7
            Start : Bool;
            Stop : Bool;
            ProgNo : Int;
         END_STRUCT;
      END_STRUCT;

      Cmd : STRUCT                      // the pins ob30 sends. writer: fc_cmdarbiter only, ob1 network 8
         Enable : Bool;
         Reset : Bool;
         Automatic : Bool;
         Start : Bool;
         Stop : Bool;
         ProgNo : Int;
         Override : Int;
         Axis : Int;
         JogMinus : Bool;
         JogPlus : Bool;
         Home : Bool;
      END_STRUCT;
   END_STRUCT;
BEGIN
END_DATA_BLOCK
```

Not retentive.

### 2.2 `DB_Scan` — GEN, type `UDT_ScanRect`. The rectangle. **Retentive.** Writer: HMI.

| Variable | Type | Default | Meaning |
| --- | --- | --- | --- |
| `X0` | Real | 1000 | first corner X, mm |
| `Y0` | Real | 500 | first corner Y |
| `Z0` | Real | 300 | scan height |
| `RotZ` | Real | 0 | tool orientation, deg |
| `RotY` | Real | 0 | |
| `Lx` | Real | 300 | length along X |
| `Ly` | Real | 200 | width along Y |
| `Py` | Real | 50 | lane pitch |
| `Px` | Real | 0 | point pitch, 0 = continuous lanes |
| `Over` | Real | 20 | overscan on the slow lanes |
| `Vscan` | Real | 50 | speed of robot block 2, for the timeout only |
| `Vmove` | Real | 200 | speed of robot block 1, for the timeout only |
| `Zsafe` | Real | 600 | retract height |

### 2.3 `DB_Job` — GEN, type `UDT_Job`. The list. Retain **`Fence` only**.

| Variable | Type | Writer | Meaning |
| --- | --- | --- | --- |
| `Count` | Int | `FC_GenRect` | rows in use |
| `RowIdx` | Int | `FB_ScanJob` | row running now |
| `Status` | Int | `FB_ScanJob`, reset in `FC_ScanControl` | 0 idle, 1 running, 2 paused, 3 done, 4 fault |
| `Error` | Int | same | error number, section 6 |
| `Lanes` | Int | `FC_GenRect` | lanes of the list |
| `EstTimeS` | Int | `FC_GenRect` | rough job time, s |
| `Fence.Xmin`, `Xmax`, `Ymin`, `Ymax`, `Zmin`, `Zmax` | Real | HMI level 2 | the envelope, defaults 0 4000 0 3000 0 1500, replace with the real one |
| `Rows[1..80]` | `UDT_Row` | `FC_AddRow` | the rows |
| `Rows[n].Pose.X`, `.Y`, `.Z`, `.RotZ`, `.RotY` | Real | | target |
| `Rows[n].Block` | Int | | 1 lin fast, 2 lin slow, 3 circ ccw, 4 circ cw |
| `Rows[n].Speed` | Real | | for the timeout |
| `Rows[n].Radius` | Real | | circ only |
| `Rows[n].ScanOn` | Bool | | scanner records during this move |
| `Rows[n].Measure` | Bool | | one measurement after this move |

### 2.4 `DB_HMI` — GEN, type `UDT_HMI`. Presses and switches. Not retentive.

| Variable | Type | Default | HMI does | PLC does |
| --- | --- | --- | --- | --- |
| `StartScan` | Bool | | press TRUE | clears, builds the list, starts |
| `Check` | Bool | | press TRUE | clears, builds the list only |
| `Pause` | Bool | | switch | holds between rows |
| `Abort` | Bool | | press TRUE | clears, stops the job, error 6 |
| `ResetFault` | Bool | | press TRUE | clears, Status 4 to 0 |
| `StepMode` | Bool | | switch | one row per Next |
| `Next` | Bool | | press TRUE | clears |
| `TestMove` | Bool | | press TRUE | clears, one-row job from TestPose |
| `TestPose.X`, `.Y`, `.Z`, `.RotZ`, `.RotY` | Real | | write | |
| `TestBlock` | Int | 1 | write 1..4 | |
| `CheckResult` | Int | | read | 0 ok, 1 bad size, 2 too long, 3 outside fence |
| `Override` | Int | 10 | write | `FC_CmdArbiter` caps to 1..60 |

### 2.5 `DB_SrlLane` — GEN, type `UDT_SrlLane`. The window to the robot. Not retentive.

| Variable | Type | Writer | Meaning |
| --- | --- | --- | --- |
| `Pose[1..5]` | Int | `FB_SrlMove` | POSE[51] X Y Z RotZ RotY, words 114..118 |
| `Par[1..4]` | Int | `FB_SrlMove` | REAL[51] radius, REAL[52] run number, REAL[53], REAL[54], words 119..122 |
| `Gate[1..8]` | Bool | `FB_SrlMove` | BOOL[51..58], word 123 |
| `Commit` | Bool | set by `FB_SrlMove`, cleared by `FC_SrlWindow` | window ready to ship |

### 2.6 `DB_ScanJob` — GEN, instance of `FB_ScanJob`. Read only for you.

| Variable | Type | Meaning |
| --- | --- | --- |
| `o_xBusy` | Bool | a job runs |
| `o_xDone` | Bool | one cycle at job end |
| `o_xFault` | Bool | job fault, stays until the next start |
| `o_iError` | Int | error number |
| `o_xStart` | Bool | start request of the current move |
| `o_iProgNo` | Int | 20 |
| `o_xScanOn` | Bool | scanner records |
| `o_xMeasTrigger` | Bool | one measurement |
| `iState` | Int | job state 0 10 20 30 40 60 90 |
| `move.iState` | Int | move state 0 10 20 30 40 50 90 |
| `move.iRun` | Int | run number |
| `move.tMove` | Time | current move timeout |

### 2.7 `DB_Mode` — YOURS. Known fields, from your V128 framework.

| Variable | Type | Meaning |
| --- | --- | --- |
| `intMode` | Int | 0 standby, 10..12 hand, 20..22 auto |
| `sttBit.sttHand.xHand` | Bool | hand |
| `sttBit.sttAuto.xAuto` | Bool | auto |
| your `udtModeBtn` tags | Bool | HMI mode buttons |

### 2.8 `DB_MachineReady` — YOURS, instance of `FB_MachineReady`.

Read: the Ready output and the MissingNo output, whatever their names are in your block. Nothing in them is copied anywhere else.

### 2.9 `DB_OperBox`, `DB_Alarms`, `DB_StopRecord` — YOURS, instances. Unchanged.

### 2.10 `DB_MOVIKIT_Robotics-5AX`, `DB_MOVIKIT_ROBOTICS_Module2` — YOURS, SEW instance DBs.

Module 2 pins used: the five pose value pins, the four parameter value pins, `WrBool1..8`, `PoseEnable`, written by `FC_SrlWindow` only. Module 1 pins used: the eleven command inputs, from `DB_App.Cmd`, and its status outputs, read by `FC_RobotStatusMap`.

### 2.11 `DB_Commission` — temporary, commissioning stage D only. Deleted after.

Fields: `xAuto`, `xMachineReady`, `xProgFinished`, `xRobotFault`, `xCommOk` : Bool; `iReal71`, `iReal73`, `iActX`, `iActY`, `iActZ`, `iActRotZ`, `iActRotY` : Int; `xMeasDone` : Bool.

---

## 3. OB30 — the 10 ms cyclic interrupt, five networks, nothing else

Settings: cycle 10 ms. Slots 18 and 19, the two 64-word modules, in the process image partition of OB30, or direct I/O inside the SEW FBs. Order in one pass: 1 handshake, 2 window, 3 module 1, 4 module 2, 5 status map. Network 2 before 3 and 4, so the same pass transmits the window. Network 5 after 3 and 4, so `Status` is from this pass.

| # | Network | Block | Mark | Pins |
| --- | --- | --- | --- | --- |
| 1 | handshake toggle | LAD | YOURS | 3.1 |
| 2 | window | `FC_SrlWindow` | GEN | 3.2, 18 pins |
| 3 | module 1 | SEW FB, `DB_MOVIKIT_Robotics-5AX` | YOURS | 3.3 |
| 4 | module 2 | SEW FB v0.7, `DB_MOVIKIT_ROBOTICS_Module2` | YOURS | 3.4 |
| 5 | status map | `FC_RobotStatusMap` | YOURS, writes `DB_App.Status` | 3.5 |

### 3.1 Network 1, handshake toggle

```
"DB_App".HS := NOT "DB_App".HS;                        // every pass
<your HS_Count tag> := <your HS_Count tag> + 1;        // optional diagnostic counter
```

The only writer of `HS`. The level goes to the robot on the handshake pin of both module FBs. Only block 1 echoes it, PI 1.15. Frozen more than 10 s, the robot latches 0x00008112.

### 3.2 Network 2, `FC_SrlWindow`, exact, 18 in-outs

| Pin | Type | Tag |
| --- | --- | --- |
| `io_iPose1` | Int | `"DB_MOVIKIT_ROBOTICS_Module2"` pose value pin 1, X, word 114 |
| `io_iPose2` | Int | pose value pin 2, Y, word 115 |
| `io_iPose3` | Int | pose value pin 3, Z, word 116 |
| `io_iPose4` | Int | pose value pin 4, RotZ, word 117 |
| `io_iPose5` | Int | pose value pin 5, RotY, word 118 |
| `io_iPar1` | Int | parameter value pin 1, REAL[51] radius, word 119 |
| `io_iPar2` | Int | parameter value pin 2, REAL[52] run number, word 120 |
| `io_iPar3` | Int | parameter value pin 3, REAL[53], word 121 |
| `io_iPar4` | Int | parameter value pin 4, REAL[54], word 122 |
| `io_xBool1` .. `io_xBool4` | Bool | `"DB_MOVIKIT_ROBOTICS_Module2".WrBool1` .. `.WrBool4`, BOOL[51..54] |
| `io_xBool5` .. `io_xBool8` | Bool | `.WrBool5` .. `.WrBool8`, free |
| `io_xPoseEnable` | Bool | `"DB_MOVIKIT_ROBOTICS_Module2".PoseEnable` |

The nine value pin names are on the v0.7 FB interface next to `PoseFirstWord` and `ParamFirstWord`. If they are `Word`, change the nine types in the source to `Word` and wrap with `INT_TO_WORD`. It reads `DB_SrlLane` inside and clears `Commit` after the copy.

### 3.3 Network 3, module 1 FB, by role

Inputs:

| Pin role | Tag or value |
| --- | --- |
| hardware identifier | 278, slot 18, robot words 1..64 |
| handshake | `"DB_App".HS` |
| enable | `"DB_App".Cmd.Enable` |
| reset | `"DB_App".Cmd.Reset` |
| automatic mode | `"DB_App".Cmd.Automatic` |
| program start | `"DB_App".Cmd.Start` |
| stop | `"DB_App".Cmd.Stop` |
| program number | `"DB_App".Cmd.ProgNo` |
| override | `"DB_App".Cmd.Override` |
| jog axis | `"DB_App".Cmd.Axis` |
| jog minus, jog plus | `"DB_App".Cmd.JogMinus`, `.JogPlus` |
| home | `"DB_App".Cmd.Home` |
| the seven path segments, PO 5..53 | not connected, the segment path is out |

Outputs stay in the instance DB, read by network 5: handshake echo PI 1.15, safe stop PI 1.1, robot moving PI 1.4 (not used), program finished PI 2.5, motion done PI 5.2 (never used), the status bits comm ok, enabled, automatic, referenced, fault, program running, the status words error number, program number, override actual, and the five actual pose words.

### 3.4 Network 4, module 2 FB v0.7, by role

Inputs:

| Pin role | Tag or value |
| --- | --- |
| hardware identifier | 272, slot 19, robot words 65..128 |
| handshake | `"DB_App".HS` |
| `PoseFirstWord` | 50, word 114 |
| `ParamFirstWord` | 55, word 119 |
| `BoolWriteWord` | 59, word 123 |
| `PoseEnable`, pose value 1..5, parameter value 1..4, `WrBool1..8` | **leave unconnected at the call.** Network 2 writes them into the instance DB. A tag on one of these pins would copy over that write every pass |
| segments 8..14, PO 65..113 | not connected |

Outputs: `SrlRealRaw[1..16]`, REAL[71..86], words 65..80, read by network 5. Boolean actual values, word 81, index unconfirmed, not used.

### 3.5 Network 5, `FC_RobotStatusMap`, the only writer of `DB_App.Status`

```
"DB_App".Status.ProgFinished := "DB_MOVIKIT_Robotics-5AX".<program finished>;     // pi 2.5
"DB_App".Status.SafeStop     := "DB_MOVIKIT_Robotics-5AX".<safe stop>;            // pi 1.1
"DB_App".Status.ProgRunning  := "DB_MOVIKIT_Robotics-5AX".<program running>;
"DB_App".Status.CommOk       := "DB_MOVIKIT_Robotics-5AX".<comm ok>;
"DB_App".Status.Enabled      := "DB_MOVIKIT_Robotics-5AX".<enabled>;
"DB_App".Status.Automatic    := "DB_MOVIKIT_Robotics-5AX".<automatic>;
"DB_App".Status.Referenced   := "DB_MOVIKIT_Robotics-5AX".<referenced>;
"DB_App".Status.Fault        := "DB_MOVIKIT_Robotics-5AX".<fault>;
"DB_App".Status.ErrorNo      := "DB_MOVIKIT_Robotics-5AX".<error number>;
"DB_App".Status.ProgNo       := "DB_MOVIKIT_Robotics-5AX".<program number actual>;
"DB_App".Status.Override     := "DB_MOVIKIT_Robotics-5AX".<override actual>;
"DB_App".Status.ActX         := "DB_MOVIKIT_Robotics-5AX".<actual pose word 1>;
"DB_App".Status.ActY         := "DB_MOVIKIT_Robotics-5AX".<actual pose word 2>;
"DB_App".Status.ActZ         := "DB_MOVIKIT_Robotics-5AX".<actual pose word 3>;
"DB_App".Status.ActRotZ      := "DB_MOVIKIT_Robotics-5AX".<actual pose word 4>;
"DB_App".Status.ActRotY      := "DB_MOVIKIT_Robotics-5AX".<actual pose word 5>;
"DB_App".Status.SrlRealRaw   := "DB_MOVIKIT_ROBOTICS_Module2".<SrlRealRaw 1..16>;
```

The left side is exact. The right side in angle brackets is the output name of the SEW FB, read it off the instance DB.

Not in OB30: no decision, no HMI, no job, no timer. Nothing writes `DB_App.Cmd` here, nothing writes the module 2 pose pins here except network 2. A sixth network is a mistake.

---

## 4. OB1 — eleven networks

| # | Network | Block | Mark |
| --- | --- | --- | --- |
| 1 | device module | yours | unchanged |
| 2 | modes | `FB_Mode` | unchanged |
| 3 | control ON, safety | LAD | unchanged |
| 4 | machine ready | `FB_MachineReady` | 3 outputs move to `Req.Machine` |
| 5 | referencing | `FC_DriveHome` | 1 output to `Req.Hand.Home` |
| 6 | jog | `FB_OperBox` | 3 outputs to `Req.Hand` |
| 7 | scan | `FC_ScanControl` | GEN, 21 pins |
| 8 | arbiter | `FC_CmdArbiter` | GEN, 25 pins |
| 9 | alarms | `FB_Alarms` | add the job inputs |
| 10 | stop record | `FB_StopRecord` | unchanged |
| 11 | machine logic | LAD | lamps, horn |

Every box EN unconditional. Order is data flow: a request raised in 4 to 7 reaches `Cmd` in 8 in the same cycle.

### 4.1 Network 4, `FB_MachineReady`, by role

| Pin role | Tag |
| --- | --- |
| control on, safety release in | the two bits from network 3 |
| machine start press in | your MACHINE START HMI tag |
| comm ok, enabled, automatic, referenced, safe stop, fault in | `"DB_App".Status.CommOk`, `.Enabled`, `.Automatic`, `.Referenced`, `.SafeStop`, `.Fault` |
| enable, reset, automatic request out | `"DB_App".Req.Machine.Enable`, `.Reset`, `.Automatic` |
| ready, missing number out | stay in `"DB_MachineReady"` |

### 4.2 Network 5, `FC_DriveHome`, by role

| Pin role | Tag |
| --- | --- |
| hand in | `"DB_Mode".sttBit.sttHand.xHand` |
| HOME press in | your HOME HMI tag |
| referenced in | `"DB_App".Status.Referenced` |
| home request out | `"DB_App".Req.Hand.Home` |

### 4.3 Network 6, `FB_OperBox`, by role

| Pin role | Tag |
| --- | --- |
| hand in | `"DB_Mode".sttBit.sttHand.xHand` |
| axis select, jog −, jog + in | your HMI tags |
| axis, jog minus, jog plus out | `"DB_App".Req.Hand.Axis`, `.JogMinus`, `.JogPlus` |

### 4.4 Network 7, `FC_ScanControl`, exact

```
"FC_ScanControl"(i_xAuto         := "DB_Mode".sttBit.sttAuto.xAuto,
                 i_xMachineReady := "DB_MachineReady".<ready output>,
                 i_xProgFinished := "DB_App".Status.ProgFinished,
                 i_xRobotFault   := "DB_App".Status.Fault,
                 i_xCommOk       := "DB_App".Status.CommOk,
                 i_iReal71       := "DB_App".Status.SrlRealRaw[1],
                 i_iReal73       := "DB_App".Status.SrlRealRaw[3],
                 i_iActX         := "DB_App".Status.ActX,
                 i_iActY         := "DB_App".Status.ActY,
                 i_iActZ         := "DB_App".Status.ActZ,
                 i_iActRotZ      := "DB_App".Status.ActRotZ,
                 i_iActRotY      := "DB_App".Status.ActRotY,
                 i_xMeasDone     := <scanner done, or TRUE>,
                 o_xStart        => "DB_App".Req.Auto.Start,
                 o_iProgNo       => "DB_App".Req.Auto.ProgNo,
                 o_xStop         => "DB_App".Req.Auto.Stop,
                 o_xScanOn       => <scanner record output, or open>,
                 o_xMeasTrigger  => <scanner trigger output, or open>,
                 o_iRowTag       => <scanner row tag, or open>,
                 o_iJobStatus    => <open>,
                 o_iJobError     => <open>);
```

### 4.5 Network 8, `FC_CmdArbiter`, exact

```
"FC_CmdArbiter"(i_xHand         := "DB_Mode".sttBit.sttHand.xHand,
                i_xAuto         := "DB_Mode".sttBit.sttAuto.xAuto,
                i_xReqEnable    := "DB_App".Req.Machine.Enable,
                i_xReqReset     := "DB_App".Req.Machine.Reset,
                i_xReqAutomatic := "DB_App".Req.Machine.Automatic,
                i_iReqAxis      := "DB_App".Req.Hand.Axis,
                i_xReqJogMinus  := "DB_App".Req.Hand.JogMinus,
                i_xReqJogPlus   := "DB_App".Req.Hand.JogPlus,
                i_xReqHome      := "DB_App".Req.Hand.Home,
                i_xReqStart     := "DB_App".Req.Auto.Start,
                i_iReqProgNo    := "DB_App".Req.Auto.ProgNo,
                i_xReqStop      := "DB_App".Req.Auto.Stop,
                i_iOverride     := "DB_HMI".Override,
                o_xEnable       => "DB_App".Cmd.Enable,
                o_xReset        => "DB_App".Cmd.Reset,
                o_xAutomatic    => "DB_App".Cmd.Automatic,
                o_xStart        => "DB_App".Cmd.Start,
                o_xStop         => "DB_App".Cmd.Stop,
                o_iProgNo       => "DB_App".Cmd.ProgNo,
                o_iOverride     => "DB_App".Cmd.Override,
                o_iAxis         => "DB_App".Cmd.Axis,
                o_xJogMinus     => "DB_App".Cmd.JogMinus,
                o_xJogPlus      => "DB_App".Cmd.JogPlus,
                o_xHome         => "DB_App".Cmd.Home,
                o_iOwner        => <open, or an HMI tag>);
```

Owner 2 when Auto alone, 1 when Hand alone, else 0. Enable, Reset, Automatic pass always. Standby: Stop TRUE, all else FALSE. Hand: jog and home. Auto: Start, ProgNo, Stop. Override capped 1..60, 0 becomes 10.

### 4.6 Network 9, `FB_Alarms`, add these inputs

`"DB_Job".Status`, `.Error`, `.RowIdx`, `"DB_ScanJob".o_xFault`, `"DB_App".Status.Fault`, `.ErrorNo`, the MissingNo output of `"DB_MachineReady"`.

---

## 5. Who writes what — the only rule that matters

| Tag group | Only writer | Where |
| --- | --- | --- |
| `DB_App.HS` | handshake toggle | OB30 n1 |
| `DB_App.Status.*` | `FC_RobotStatusMap` | OB30 n5 |
| `DB_App.Req.Machine.*` | `FB_MachineReady` | OB1 n4 |
| `DB_App.Req.Hand.Home` | `FC_DriveHome` | OB1 n5 |
| `DB_App.Req.Hand.Axis`, `.JogMinus`, `.JogPlus` | `FB_OperBox` | OB1 n6 |
| `DB_App.Req.Auto.*` | `FC_ScanControl` | OB1 n7 |
| `DB_App.Cmd.*` | `FC_CmdArbiter` | OB1 n8 |
| module 2 pose, parameter, bool pins | `FC_SrlWindow` | OB30 n2 |
| `DB_SrlLane.*` | `FB_SrlMove`, `Commit` cleared by `FC_SrlWindow` | OB1, OB30 |
| `DB_Job.*` | `FC_GenRect`, `FB_ScanJob`, reset in `FC_ScanControl`; `Fence` by HMI | OB1 |
| `DB_Scan.*`, `DB_HMI` presses and `Override` | HMI; presses cleared by `FC_ScanControl` | — |
| `DB_Mode.*` | `FB_Mode` | OB1 n2 |

---

## 6. Error numbers

| Number | Meaning | From |
| --- | --- | --- |
| 1 | start not taken, `program_finished` did not fall in 2 s | `FB_SrlMove` |
| 2 | move timeout | `FB_SrlMove` |
| 3 | robot fault during the move | `FB_SrlMove` |
| 4 | wrong block or stale run number | `FB_SrlMove` |
| 5 | communication lost | `FB_SrlMove` |
| 6 | aborted, or left Auto | both |
| 7 | block number outside 1..4 | `FB_SrlMove` |
| 10 | no job loaded | `FB_ScanJob` |
| 11 | machine not ready | `FB_ScanJob` |
| 12 | measurement timeout | `FB_ScanJob` |
| 101 | bad rectangle parameters | `FC_GenRect` |
| 102 | more than 80 rows | `FC_GenRect` |
| 103 | a point outside the fence | `FC_GenRect` |

---

## 7. Order of work in TIA

1. `DB_App`: add `Status`, `Req`, `Cmd` as in 2.1. Delete `Srl`.
2. `FC_RobotStatusMap`: write into `DB_App.Status`, table 3.3.
3. `FB_MachineReady`: its three request outputs to `Req.Machine`. `FB_OperBox`: three outputs to `Req.Hand`. `FC_DriveHome`: one output to `Req.Hand.Home`.
4. Delete `FC_SrlGates`. Save the cube and drum numbers from the old `DB_Job`, delete it. Take the seven-segment path out of OB1.
5. External source files, add `plc/BGE_Scan.scl`, Generate blocks from source. Nineteen blocks.
6. `DB_Scan`: retentive. `DB_Job`: Retain on `Fence` only. `DB_Job.Fence`: the real envelope.
7. OB30 network 2: `FC_SrlWindow`, table 3.1. Module 1 FB inputs from `DB_App.Cmd`, table 3.2.
8. OB1 network 7: `FC_ScanControl`, 4.4. Network 8: `FC_CmdArbiter`, 4.5. Network 9: the alarm inputs, 4.6.
9. Compile. Cross-reference `DB_App.Cmd` and the module 2 pins: one writer each.
10. Robot: program 20 with `REAL[71] := 0` first and `REAL[73] := REAL[52]` last.
11. Then `COMMISSIONING_PROCEDURE.md`, stage A.
