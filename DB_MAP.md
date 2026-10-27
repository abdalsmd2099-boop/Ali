# BGE Portal — the DBs, who writes them, who reads them

One page. `DB_App` is the robot interface DB the project already has. Everything my documents call `DB_Robot.Status`, `DB_RobotCmd` or `DB_Machine` lives in `DB_App` or in an instance DB, as the mapping at the end says.

## 1. Every DB

| DB | Kind | Holds | Writer | Readers | Retentive |
| --- | --- | --- | --- | --- | --- |
| `DB_App` | global, yours | robot interface: `HS`, `Status`, `Req`, `Cmd` | one writer per group, section 2 | OB1 blocks, OB30, HMI reads | no |
| `DB_MOVIKIT_Robotics-5AX` | instance, SEW module 1 FB | control words out, status words in | OB30 only, pins from `DB_App.Cmd` | `FC_RobotStatusMap` | no |
| `DB_MOVIKIT_ROBOTICS_Module2` | instance, SEW module 2 FB v0.7 | pose, parameter, bool pins, `SrlRealRaw` | `FC_SrlWindow` in OB30 only | `FC_RobotStatusMap` | no |
| `DB_SrlLane` | `UDT_SrlLane` | the staging window and `Commit` | `FB_SrlMove` inside `DB_ScanJob`; `Commit` cleared by `FC_SrlWindow` | `FC_SrlWindow`, HMI reads | no |
| `DB_Scan` | `UDT_ScanRect` | the rectangle | HMI | `FC_ScanControl`, HMI | **yes** |
| `DB_Job` | `UDT_Job` | list, job status, fence | `FC_GenRect`, `FB_ScanJob`, the reset in `FC_ScanControl`; `Fence` by HMI level 2 | HMI, `FB_Alarms` | **Fence only** |
| `DB_HMI` | `UDT_HMI` | scan presses, switches, test pose, check result | HMI sets presses, `FC_ScanControl` clears them and writes `CheckResult` | `FC_ScanControl`, HMI | no |
| `DB_ScanJob` | instance `FB_ScanJob` | job and move state machines, holds the `FB_SrlMove` instance | itself | HMI lamps, `FB_Alarms` read | no |
| `DB_Mode` | yours | `intMode`, mode bits, mode buttons | `FB_Mode`; buttons set by HMI, cleared by `FB_Mode` | arbiter, every OB1 block, HMI | no |
| `DB_MachineReady` | instance `FB_MachineReady` | sequence, `Ready`, `MissingNo` | itself | arbiter, `FC_ScanControl`, HMI | no |
| `DB_OperBox` | instance `FB_OperBox` | axis select, jog | itself; selections by HMI | arbiter via `Req.Hand` | no |
| `DB_Alarms`, `DB_StopRecord` | instances | alarms, stop record | themselves | HMI | no |
| `DB_Scanner` | optional | scanner I/O image, if not direct I/O | OB1 network 11 | `FC_ScanControl` pins | no |
| `DB_Commission` | temporary | fake robot for commissioning stage D | watch table | `FC_ScanControl` pins in stage D only | no, delete after |

## 2. `DB_App`, one writer per group

```
DB_App
  HS            Bool                              writer: OB30 network 1
  Status        STRUCT                            writer: FC_RobotStatusMap, OB30 network 5
     CommOk, Enabled, Automatic, Referenced, SafeStop, Fault : Bool
     ProgFinished, ProgRunning : Bool
     ProgNo, ErrorNo, Override : Int
     ActX, ActY, ActZ, ActRotZ, ActRotY : Int     // whole mm and deg, same frame as pose 51
     SrlRealRaw : Array[1..16] of Int
  Req           STRUCT                            requests, one writer per sub-struct
     Machine    Enable, Reset, Automatic : Bool   writer: FB_MachineReady
     Hand       Axis : Int; JogMinus, JogPlus, Home : Bool   writer: FB_OperBox, FC_DriveHome
     Auto       Start, Stop : Bool; ProgNo : Int  writer: FC_ScanControl
  Cmd           STRUCT                            the pins, writer: the arbiter, reader: OB30 module 1
     Enable, Reset, Automatic, Start, Stop : Bool
     ProgNo, Override, Axis : Int
     JogMinus, JogPlus, Home : Bool
```

Removed from `DB_App`:

- `Srl`, the old lane. `DB_SrlLane` is the lane. Delete `Srl` together with `FC_SrlGates`; two lanes means two writers of the module 2 pins.
- Direct writes into `Cmd` by `FB_MachineReady`. It writes `Req.Machine`; the arbiter copies to `Cmd` in Hand and Auto. The arbiter is the only writer of `Cmd`.
- A copy of `ready` or `MissingNo`. They live in the `FB_MachineReady` instance DB only.

## 3. Two conflicts when generating `BGE_Scan.scl`

- **The old `DB_Job`.** It held the 80 rows, the cube and drum numbers, the fence and the status. The new one has the same name with `UDT_Job`. Generation stops at "block already exists". Save the cube and drum numbers, delete the old DB, generate. Those numbers come back as their own parameter DBs, `DB_Drum` and so on, built like `DB_Scan`.
- **Retentivity of `DB_Job`.** Generated retentive. If `Status` survives a power cycle as 1, nothing can start: Check and Start need Status not 1 or 2, Reset clears only 4. In the DB editor set Retain on `Fence` only. Then a power cycle gives Status 0, and commissioning step H5 passes.

## 4. Rules

1. **One writer per tag.** This page is the law. After every change, cross-reference `DB_App.Cmd` and the module 2 pins: exactly one writer each.
2. **Requests are not commands.** Blocks write `Req.*`. The arbiter writes `Cmd`. OB30 sends `Cmd`. No block writes `Cmd`.
3. **OB30 owns the robot's edge.** It alone writes `HS`, the module instance DB pins and `Status`, and clears `DB_SrlLane.Commit`. OB1 never touches a module instance DB.
4. **The HMI writes parameters and presses only:** `DB_Scan`, `DB_HMI`, `DB_Job.Fence`, the mode buttons. Never `DB_App`, never `DB_SrlLane`, never an instance DB.
5. **Instance DBs are private.** Only their FB writes them. The HMI and alarms may read.
6. **One purpose per DB,** typed by a UDT where the HMI needs stable names. No "misc" DB.
7. **Temporary carries the word.** `DB_Commission` leaves before handover.

## 5. Retentivity

| DB | Retain |
| --- | --- |
| `DB_Scan` | all |
| `DB_Job` | `Fence` only, per tag in the DB editor |
| every other DB on this page | none |

## 6. Name mapping, documents to project

| In APPLY_GUIDE, COMMISSIONING_PROCEDURE and the HMI canvas | In the project |
| --- | --- |
| `DB_Robot.Status.*` | `DB_App.Status.*` |
| `DB_RobotCmd.Auto.*` | `DB_App.Req.Auto.*` |
| `DB_RobotCmd.Out`, "the robot pins" | `DB_App.Cmd` |
| `DB_Machine.Ready`, `.MissingNo` | the `FB_MachineReady` instance outputs |
| `DB_Mode.Auto` | the Auto bit of your mode DB |

The pins of `FC_ScanControl` and `FC_SrlWindow` take these names directly. Nothing in `BGE_Scan.scl` changes.
