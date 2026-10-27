# BGE Portal — rectangle scan through program 20: how to apply it

One document. What the pieces are, in which order to put them in, how to prove each step, and what is still open.

---

## 1. The idea in ten lines

- The robot holds **one pose slot** (POSE[51]) and **four ways to reach it** (program 20, blocks 1 to 4). Nothing else.
- The PLC holds **the list**. A job is rows in `DB_Job`, generated from the rectangle numbers on the HMI.
- One row = write the window, commit, Start program 20, wait for `program_finished`, check two echoes, next row.
- The Bool gate **chooses** the block. The Start **runs** it. Nothing moves on a gate alone.
- Done is proven by **three signals**: `program_finished` rises, REAL[71] equals the block, REAL[73] equals the run number.
- OB30 **delivers** (handshake, window, module FBs, status). OB1 **decides** (job, move, arbiter). They share DBs only.
- **One writer per tag.** The SRL lane has one writer: `FB_SrlMove`. The robot pins have one writer: the arbiter.
- Every job **starts with a retract** to safe height and **ends with a retract**. A fault leaves the gantry where it is; the next Start retracts first.
- Everything on the bus is **whole millimetres and degrees** (decimal places 0).
- The HMI talks to DBs only. Press bits are cleared by the PLC.

---

## 2. The files

Two files. One for the robot, one for TIA.

| File | What it is | Goes where |
| --- | --- | --- |
| `robot/program20.srl` | the motion library, four gated blocks | RobotMonitor, program 20 |
| `plc/BGE_Scan.scl` | one external source, generates every block below | TIA, external source files |

What `BGE_Scan.scl` generates, in this order:

| Block | What it is |
| --- | --- |
| `UDT_Pose5` | X Y Z RotZ RotY |
| `UDT_Fence` | min and max per axis |
| `UDT_Row` | one row: pose, block, speed, radius, scan on, measure |
| `UDT_Job` | count, row index, status, error, lanes, time, fence, 80 rows |
| `UDT_ScanRect` | the rectangle parameters |
| `UDT_SrlLane` | pose words, parameter words, gates, commit |
| `UDT_HMI` | press bits, switches, test pose, check result |
| `DB_Scan` | the rectangle with standard values, **set retentive** |
| `DB_Job` | the list, with a placeholder fence |
| `DB_SrlLane` | the staging window |
| `DB_HMI` | the HMI tags |
| `FC_AddRow` | appends one row, fence checked |
| `FC_GenRect` | rectangle into rows, lanes, time estimate |
| `FB_SrlMove` | one move: window, commit, start, wait, verify |
| `FB_ScanJob` | walks the rows, retract, scanner, pause, step, fault |
| `DB_ScanJob` | the one instance of FB_ScanJob, holds the FB_SrlMove instance inside |
| `FC_SrlWindow` | **you call this in OB30**, ships the window to the module 2 pins |
| `FC_ScanControl` | **you call this in OB1**, HMI presses, job, outputs for the arbiter and the scanner |

The two FCs have pins. Your own tags go on the pins, in LAD, at the two calls. Nothing inside the source has to be renamed.

---

## 3. Before you start: five facts to confirm

Nothing below works until these are true.

| # | Fact | How to confirm |
| --- | --- | --- |
| 1 | The module 2 FB loaded in TIA is **v0.7** with `PoseEnable` and `WrBool1..8` | open the block in TIA, compare the interface with the source |
| 2 | Slot 19 (module 2) is in the **process image partition of OB30**, or OB30 uses direct I/O | hardware config → module → I/O addresses → process image |
| 3 | Decimal places = 0, RotX off, pose = 5 words, POSE[51] at words 114 to 118, REAL[51..54] at 119 to 122, BOOL[51..58] at 123 | Process Data Monitor in RobotMonitor |
| 4 | REAL[71..86] arrive as `SrlRealRaw[1..16]`, words 65 to 80 | write REAL[71] := 7 in RobotMonitor, read `SrlRealRaw[1]` in a watch table |
| 5 | `program_finished` is PI 2.5 and falls on Start, rises when the program ends | start an empty program, watch the bit |

Also: the STO reading active with the chain closed (open point 8 of the status page) must be fixed before any automatic test.

---

## 4. The pins and what to connect

The source never names anything from your project. Everything project specific is a pin on one of the two FCs.

All project tags are in `DB_App`, shaped as `DB_MAP.md` section 2 says, or in the instance DBs named below. One name set, used here, in the commissioning procedure and on the HMI canvas.

### `FC_SrlWindow`, OB30 network 2, 18 in-outs

| Pin | Type | Connect to |
| --- | --- | --- |
| `io_iPose1` | Int | `"DB_MOVIKIT_ROBOTICS_Module2"`, pose value pin 1, X, word 114 |
| `io_iPose2` | Int | pose value pin 2, Y, word 115 |
| `io_iPose3` | Int | pose value pin 3, Z, word 116 |
| `io_iPose4` | Int | pose value pin 4, RotZ, word 117 |
| `io_iPose5` | Int | pose value pin 5, RotY, word 118 |
| `io_iPar1` | Int | parameter value pin 1, REAL[51] radius, word 119 |
| `io_iPar2` | Int | parameter value pin 2, REAL[52] run number, word 120 |
| `io_iPar3` | Int | parameter value pin 3, REAL[53], word 121 |
| `io_iPar4` | Int | parameter value pin 4, REAL[54], word 122 |
| `io_xBool1` | Bool | `"DB_MOVIKIT_ROBOTICS_Module2".WrBool1`, BOOL[51], word 123 bit 0 |
| `io_xBool2` | Bool | `.WrBool2`, BOOL[52] |
| `io_xBool3` | Bool | `.WrBool3`, BOOL[53] |
| `io_xBool4` | Bool | `.WrBool4`, BOOL[54] |
| `io_xBool5` .. `io_xBool8` | Bool | `.WrBool5` .. `.WrBool8`, BOOL[55..58], free |
| `io_xPoseEnable` | Bool | `"DB_MOVIKIT_ROBOTICS_Module2".PoseEnable` |

`WrBool1..8` and `PoseEnable` are the pin names from your status page. The names of the five pose value pins and the four parameter value pins I have not seen: they are the inputs of the v0.7 FB next to `PoseFirstWord := 50` and `ParamFirstWord := 55`. Read them off the FB interface and connect in word order. If they are `Word`, not `Int`, change the nine pin types in the source to `Word` and wrap the nine assignments in `INT_TO_WORD`.

### `FC_ScanControl`, OB1 network 7, 13 inputs and 8 outputs

| Pin | Type | Connect to |
| --- | --- | --- |
| `i_xAuto` | Bool | `"DB_Mode".sttBit.sttAuto.xAuto`, the Auto bit of your mode DB as V128 names it. A drop aborts the job |
| `i_xMachineReady` | Bool | the READY TO START output of `"DB_MachineReady"`, your `FB_MachineReady` instance |
| `i_xProgFinished` | Bool | `"DB_App".Status.ProgFinished` |
| `i_xRobotFault` | Bool | `"DB_App".Status.Fault` |
| `i_xCommOk` | Bool | `"DB_App".Status.CommOk` |
| `i_iReal71` | Int | `"DB_App".Status.SrlRealRaw[1]` |
| `i_iReal73` | Int | `"DB_App".Status.SrlRealRaw[3]` |
| `i_iActX` | Int | `"DB_App".Status.ActX` |
| `i_iActY` | Int | `"DB_App".Status.ActY` |
| `i_iActZ` | Int | `"DB_App".Status.ActZ` |
| `i_iActRotZ` | Int | `"DB_App".Status.ActRotZ` |
| `i_iActRotY` | Int | `"DB_App".Status.ActRotY` |
| `i_xMeasDone` | Bool | the scanner's done input, or the constant TRUE until a scanner exists |
| `o_xStart` | Bool | `"DB_App".Req.Auto.Start` |
| `o_iProgNo` | Int | `"DB_App".Req.Auto.ProgNo` |
| `o_xStop` | Bool | `"DB_App".Req.Auto.Stop` |
| `o_xScanOn` | Bool | the scanner's record output, or open |
| `o_xMeasTrigger` | Bool | the scanner's trigger output, or open |
| `o_iRowTag` | Int | the scanner's row tag, or open |
| `o_iJobStatus` | Int | open, or an input of `FB_Alarms` |
| `o_iJobError` | Int | open, or an input of `FB_Alarms` |

The arbiter, OB1 network 8, copies `"DB_App".Req.Auto.Start`, `.ProgNo`, `.Stop` to `"DB_App".Cmd.Start`, `.ProgNo`, `.Stop` in Auto, and holds `Cmd.Start` FALSE in Hand and Standby. OB30 network 3 sends `"DB_App".Cmd` through the module 1 FB.

---

## 5. Step 1 — the robot: program 20

Open program 20 in RobotMonitor. Keep your four blocks. Add **two lines**: clear REAL[71] at the top, echo the run number at the end.

```
// program 20 - one pass = one motion, the plc picks the block with one bool
REAL[71] := 0                        // nothing ran yet in this pass

IF BOOL[51] THEN                     // straight, fast, between lanes and for approach
    LIN POSE[51]
    REAL[71] := 1
END_IF

IF BOOL[52] THEN                     // straight, slow, the scanner records
    LIN POSE[51]
    REAL[71] := 2
END_IF

IF BOOL[53] THEN                     // arc counter clockwise, radius in real 51
    CIRC POSE[51] REAL[51] CCW
    REAL[71] := 3
END_IF

IF BOOL[54] THEN                     // arc clockwise
    CIRC POSE[51] REAL[51] CW
    REAL[71] := 4
END_IF

REAL[73] := REAL[52]                 // echo the run number, this proves the pass that ran
```

Keyword spelling follows manual 33956901. The structure is what matters: REAL[71] := 0 before the blocks, REAL[73] := REAL[52] after them, nothing after that line.

Also in the robot: blending 0 for these moves, program 20 selectable by number, the speeds of blocks 1 and 2 written down (they go into `DB_Scan.Vmove` and `Vscan`).

**Done when:** with all gates FALSE, a manual Start of program 20 ends with `program_finished` TRUE, REAL[71] = 0 and REAL[73] = whatever REAL[52] holds. With BOOL[51] TRUE and a safe pose in POSE[51], the gantry moves there once and REAL[71] = 1.

---

## 6. Step 2 — the PLC: import and generate

In TIA, in the PLC: *External source files → Add new external file* → pick `plc/BGE_Scan.scl`. Right click the file → *Generate blocks from source*. The blocks of section 2 appear under Program blocks and PLC data types, in dependency order, from one file.

Then:

- `DB_Scan` → properties → retentive.
- `DB_Job.Fence`: put the real envelope in, read from the robot soft limits, a few millimetres inside them.

**Done when:** the project compiles with 0 errors and the two FCs `FC_SrlWindow` and `FC_ScanControl` are in the block list. `FB_SrlMove` has no instance DB of its own; it lives inside `DB_ScanJob`.

If generation stops with an error, it names the line. The two usual causes are a `Word` versus `Int` mismatch on the module 2 pins (section 4) and a block of the same name already in the project.

---

## 7. Step 3 — OB30: the window ship

OB30 keeps its four networks. Insert a new **network 2**, between the handshake toggle and the module 1 FB, with one call:

```
"FC_SrlWindow"   (LAD box, or in SCL: "FC_SrlWindow"(io_iPose1 := ..., ...);)
```

Connect its 18 in-out pins to the pose, parameter, `WrBool1..8` and `PoseEnable` pins of the module 2 FB, as in section 4. In LAD that is dragging the module 2 instance DB fields onto the box.

Remove any other writer of those pins (the old `FC_SrlGates` path). OB30 order afterwards: handshake, `FC_SrlWindow`, module 1 FB, module 2 FB, status map. Network 2 before network 4 is what makes the same 10 ms pass transmit the window.

**Done when:** in a watch table, write `DB_SrlLane.Pose[1] := 1234` and `Commit := TRUE`. Within one OB30 pass Commit is FALSE and the Process Data Monitor shows 1234 in word 114.

---

## 8. Step 4 — the status map

`FC_RobotStatusMap`, OB30 network 5, is the only writer of `"DB_App".Status`. It must deliver:

| `DB_App.Status` field | Type | Source |
| --- | --- | --- |
| `ProgFinished` | Bool | PI 2.5 `program_finished` |
| `ProgRunning` | Bool | the program running bit of the module 1 FB |
| `SafeStop` | Bool | PI 1.1 `safe_stop` |
| `CommOk` | Bool | communication ok output of the module 1 FB |
| `Enabled`, `Automatic`, `Referenced`, `Fault` | Bool | the matching status bits of the module 1 FB |
| `ErrorNo`, `ProgNo`, `Override` | Int | the matching status words of the module 1 FB |
| `ActX`, `ActY`, `ActZ`, `ActRotZ`, `ActRotY` | Int | the five actual pose words in PI, whole mm and deg, same frame as POSE[51] |
| `SrlRealRaw[1..16]` | Int | module 2 words 65 to 80, REAL[71..86] |

Not mapped to anything the job reads: `motion_done` PI 5.2, and `robot_moving` PI 1.4 until SEW says what it means.

---

## 9. Step 5 — OB1: one call

One new network after `FB_MachineReady` and before the arbiter, with one LAD box:

```
"FC_ScanControl"
```

Connect the pins as in section 4. Inputs come from `DB_Mode`, `DB_MachineReady`, `DB_App.Status` and the scanner. Outputs `o_xStart`, `o_iProgNo` and `o_xStop` go to `DB_App.Req.Auto`, which the arbiter reads. `o_xScanOn`, `o_xMeasTrigger` and `o_iRowTag` go to the scanner, or stay open until it exists. Until a scanner exists, tie `i_xMeasDone` to TRUE.

Inside the call, every cycle: the press bits are read and cleared, CHECK or START builds the list with `FC_GenRect`, TEST MOVE builds a one-row list, `DB_ScanJob` runs, and the outputs are copied. The HMI never needs anything from the pins: it reads and writes `DB_HMI`, `DB_Scan` and `DB_Job` directly.

In the arbiter: in Auto, copy the three requests to the robot pins. In Hand and Standby, Start stays FALSE whatever the job asks. Start is a level, held by `FB_SrlMove` until `program_finished` falls.

### OB1 as a whole

Inputs first, decisions in the middle, the single writers last, alarms at the end. Every FB once, every cycle, no contact in front of any box.

| # | Network | Block | Reads | Writes |
| --- | --- | --- | --- | --- |
| 1 | Device module | MOVIRUN device module FB, slot 17 | I/O | comm status |
| 2 | Modes | your `FC_Mode` / `FB_Mode` | HMI mode buttons | `intMode`, `xHand`, `xAuto` |
| 3 | Control ON, safety release | existing | safety, local or CCR | `xControlOn`, `xSafetyOk` |
| 4 | Machine ready | `FB_MachineReady` | network 3, `DB_App.Status`, MACHINE START | `Ready`, `MissingNo`, `DB_App.Req.Machine` Enable / Reset / Automatic |
| 5 | Referencing | your `FC_DriveHome` | Hand, HOME | home request |
| 6 | Jog | your `FB_OperBox` | Hand, axis, JOG −, JOG + | jog requests |
| 7 | Scan | `FC_ScanControl` | `xAuto`, `Ready`, `DB_App.Status`, scanner done | `DB_App.Req.Auto` Start, ProgNo, Stop, scanner outputs |
| 8 | Arbiter | your `FB_SEW_CmdArbiter` | owner from 2, `DB_App.Req.*` from 4 to 7 | `DB_App.Cmd`, only writer |
| 9 | Alarms | `FB_Alarms` | `DB_Job.Status`, `.Error`, `.RowIdx`, `MissingNo` | alarm bits |
| 10 | Stop record | `FB_StopRecord` | existing | existing |
| 11 | Machine logic | plain LAD | FC_ScanControl outputs, I/O | lamps, horn, scanner I/O |

Out of OB1: `FC_SrlGates`, replaced by network 7 and OB30's `FC_SrlWindow`. The seven-segment path, `FC_RC_Mngmt`, `FB_RC_Mechanism`, `FB_RobotOrder`, comes out too, or sits behind a job-type selector the arbiter honours. Left in as it is, it and network 7 both ask for Start.

The arbiter by owner:

```
owner 0 Standby : every robot command bit FALSE, Stop TRUE
owner 1 Hand    : Enable / Reset / Automatic from 4, jog and home from 5 and 6, Start FALSE
owner 2 Auto    : Enable / Reset / Automatic from 4, Start / ProgNo / Stop from 7, jog FALSE
```

Exactly three things cross between OB1 and OB30: `DB_App.Status` down, `DB_App.Cmd` up, `DB_SrlLane` up with its commit bit coming back.

**Done when:** the project compiles. In Standby, a Test Move ends at once with error 6 and nothing moved: `i_xAuto` is FALSE, so the job aborts in its first cycle before any window is written.

---

## 10. Step 6 — the HMI

### The scan screen

```
+----------------------------------------------------------------------------+
| BGE Portal      Mode: AUTO      Machine: READY      Robot: comm ok, auto   |
+----------------------------------+-----------------------------------------+
| RECTANGLE  (mm, deg, whole)      | JOB                                     |
|  Corner X0     [ 1000 ]          |  Status      RUNNING                    |
|  Corner Y0     [  500 ]          |  Row         7 / 12      Lane  3 / 5    |
|  Height Z0     [  300 ]          |  Est. time   95 s        Elapsed 41 s   |
|  RotZ          [    0 ]          |  Last error  -                          |
|  RotY          [    0 ]          |                                         |
|  Length Lx     [  300 ]          | POSITION                                |
|  Width  Ly     [  200 ]          |  Target   1300   500   300    0    0    |
|  Pitch  Py     [   50 ]          |  Actual   1187   500   300    0    0    |
|  Point  Px     [    0 ]          |  Scanner  RECORDING                     |
|  Overscan      [   20 ]          |                                         |
|  Safe Z        [  600 ]          | MISSING  -                              |
|  [ CHECK ]   12 rows, 5 lanes, ok|                                         |
+----------------------------------+-----------------------------------------+
| [ MACHINE START ] [ START SCAN ] [ PAUSE ] [ ABORT ] [ RESET ]   [ ALARMS ] |
+----------------------------------------------------------------------------+
```

Rules on the screen:

- Number fields are integers, limits from the fence. Locked while Status is 1 or 2.
- START SCAN enabled only when: mode Auto, Machine READY, Status 0 or 3, last CHECK ok.
- PAUSE is a switch. Every other button is a press: the HMI writes TRUE, the PLC clears it.
- MISSING shows the `MissingNo` text while the machine is not ready.

A commissioning screen holds: `Vscan`, `Vmove`, the fence, the override, test move (pose, block, button), STEP MODE with NEXT, `SrlRealRaw[1]`, `[3]`, `program_finished`, program number.

### The tags

"Press" = HMI writes TRUE, PLC clears.

| Tag | Type | HMI does | Use | Level |
| --- | --- | --- | --- | --- |
| DB_Scan.X0, Y0, Z0 | Real | write | corner and scan height | operator |
| DB_Scan.RotZ, RotY | Real | write | tool orientation | operator |
| DB_Scan.Lx, Ly | Real | write | size, above 0 | operator |
| DB_Scan.Py | Real | write | lane pitch, 1 to Ly | operator |
| DB_Scan.Px | Real | write | point pitch, 0 = continuous lanes | operator |
| DB_Scan.Over | Real | write | overscan, 0 or more | operator |
| DB_Scan.Zsafe | Real | write | retract height, above Z0 | operator |
| DB_Scan.Vscan, Vmove | Real | write | must equal blocks 2 and 1 | commissioning |
| DB_HMI.StartScan | Bool | press | builds the list and starts | operator |
| DB_HMI.Check | Bool | press | builds the list, shows rows and lanes | operator |
| DB_HMI.Pause | Bool | switch | holds between rows | operator |
| DB_HMI.Abort | Bool | press | stops, fault 6 | operator |
| DB_HMI.ResetFault | Bool | press | clears Status 4 | operator |
| DB_HMI.StepMode | Bool | switch | one row per NEXT | commissioning |
| DB_HMI.Next | Bool | press | next row in step mode | commissioning |
| DB_HMI.TestMove | Bool | press | one row from TestPose | commissioning |
| DB_HMI.TestPose.X .. RotY | Real | write | target of the test move | commissioning |
| DB_HMI.TestBlock | Int | write | 1 to 4 | commissioning |
| DB_HMI.CheckResult | Int | read | 0 ok, 1 bad size, 2 too long, 3 outside the fence | operator |
| DB_Job.Status | Int | read | 0 idle, 1 running, 2 paused, 3 done, 4 fault | operator |
| DB_Job.RowIdx, Count | Int | read | row n of m | operator |
| DB_Job.Lanes, EstTimeS | Int | read | lanes, estimated time | operator |
| DB_Job.Error | Int | read | error number as text | operator |
| DB_Job.Rows[n] | UDT_Row | read | optional list view after CHECK | commissioning |
| DB_Job.Fence.Xmin .. Zmax | Real | write | envelope inside the robot soft limits | admin |
| DB_MachineReady, the Ready and MissingNo outputs | Bool, Int | read | ready, what is missing | operator |
| DB_Mode.intMode | Int | read | Standby, Hand, Auto | operator |
| DB_App.Status.ActX, ActY, ActZ, ActRotZ, ActRotY | Int | read | actual position | operator |
| DB_App.Status.CommOk, Automatic, Enabled, Referenced, SafeStop | Bool | read | robot lamps | operator |
| DB_App.Status.Fault, ErrorNo | Bool, Int | read | robot fault, text from the manual | operator |
| DB_App.Status.ProgFinished, ProgNo, Override | mixed | read | diagnostics | commissioning |
| DB_App.Status.SrlRealRaw[1], [3] | Int | read | block that ran, run number echo | commissioning |
| DB_SrlLane.Pose[1..5] | Int | read | the target on the bus | operator |
| scanner tags, from the `FC_ScanControl` pins | Bool, Int | read | scanner lamps and the row the data is tagged with | operator |

### Text lists

| List | Values |
| --- | --- |
| Status | 0 idle, 1 running, 2 paused, 3 done, 4 fault |
| CheckResult | 0 ok, 1 bad size, 2 more than 80 rows, 3 point outside the fence |
| Error | see section 13 |
| MissingNo | from `FB_MachineReady`: safety, enable, automatic, referenced, communication, robot fault, RobotMonitor has access |

---

## 11. What happens, end to end

### The standard rectangle

Lx 300, Ly 200, Py 50, Px 0, Over 20 gives 5 lanes and 12 rows. Block 1 is fast LIN, block 2 is slow LIN with the scanner recording. With overscan the slow lanes run from X0−20 to X0+320.

| Row | X | Y | Z | Block | Scan |
| --- | --- | --- | --- | --- | --- |
| 1 | X0−20 | Y0 | Zsafe | 1 | |
| 2 | X0−20 | Y0 | Z0 | 1 | |
| 3 | X0+320 | Y0 | Z0 | 2 | on |
| 4 | X0+320 | Y0+50 | Z0 | 1 | |
| 5 | X0−20 | Y0+50 | Z0 | 2 | on |
| 6 | X0−20 | Y0+100 | Z0 | 1 | |
| 7 | X0+320 | Y0+100 | Z0 | 2 | on |
| 8 | X0+320 | Y0+150 | Z0 | 1 | |
| 9 | X0−20 | Y0+150 | Z0 | 2 | on |
| 10 | X0−20 | Y0+200 | Z0 | 1 | |
| 11 | X0+320 | Y0+200 | Z0 | 2 | on |
| 12 | X0+320 | Y0+200 | Zsafe | 1 | |

Before row 1 the job moves straight up to Zsafe from the actual position, unless it already stands there. Px above 0 switches to a stop and one measurement every Px along each lane, without overscan.

### One row, cycle by cycle

1. **OB1.** `FB_ScanJob` loads the row, pulses exec. `FB_SrlMove` writes Pose, Par (radius, run number), one gate, then Commit. Timeout computed from the distance to the last target.
2. **OB30.** Network 2 sees Commit, copies the window to the pins, clears Commit. Network 4 sends it.
3. **OB1.** `FB_SrlMove` sees Commit FALSE, raises Start. The `FC_ScanControl` output and the arbiter pass it to the robot Start pin, program 20.
4. **OB30.** Module 1 sends Start. The robot starts the program, `program_finished` falls.
5. **OB1.** `FB_SrlMove` sees it fall: the start was taken. Start drops. Timer 2 s stops, move timer runs.
6. **Robot.** REAL[71] := 0, the gated block runs, blocking. REAL[71] := n, REAL[73] := run number. Program ends, `program_finished` rises. The echo is on the bus before or with the flag.
7. **OB30.** `SrlRealRaw[1]`, `[3]` and `ProgFinished` arrive.
8. **OB1.** `FB_SrlMove` checks block and run number, clears the gates, commits, pulses done. `FB_ScanJob` drops `ScanOn`, takes the next row or waits for the measurement.

Per row the PLC adds about three OB30 passes and three OB1 cycles, on top of the robot's start and finish latency.

### Who writes what

| Data | Only writer | Where |
| --- | --- | --- |
| DB_Scan | HMI | screen |
| DB_Job rows, count, lanes, time | FC_GenRect, inside FC_ScanControl | OB1 |
| DB_Job RowIdx, Status, Error | FB_ScanJob, and the reset inside FC_ScanControl | OB1 |
| DB_SrlLane | FB_SrlMove inside FB_ScanJob | OB1 |
| Module 2 setpoint pins | FC_SrlWindow | OB30 network 2 |
| DB_App.Req.Auto | the FC_ScanControl outputs | OB1 |
| DB_App.Cmd | arbiter | OB1 |
| Module 1 control words | module 1 FB, from DB_App.Cmd | OB30 |
| DB_App.Status | FC_RobotStatusMap | OB30 |
| DB_HMI press bits | HMI sets, FC_ScanControl clears | both |

### When something goes wrong

- **Pause.** Holds in state 20 after the current row, Status 2. Scanner off between rows. Release resumes.
- **Abort, or leaving Auto.** Both FBs go to 90 in the same cycle. Gates off and committed, Start off, Status 4, Error 6, `o_xFault` → arbiter → `Out.Stop`. Next Start retracts first.
- **Robot fault mid-move.** Error 3, row number stays in `RowIdx`.
- **Timeouts.** Error 1 if `program_finished` never falls, error 2 if it never rises in time.
- **Wrong block or stale run number.** Error 4. Never transient: program 20 ran another block or the two added lines are missing.
- **Communication lost.** Error 5. The robot stops on its own when the handshake dies.

---

## 12. Step 7 — tests, in order

Each test passes before the next. Override 10 % until T8.

| # | Test | Done when |
| --- | --- | --- |
| T1 | CHECK with the standard rectangle | `CheckResult` 0, `Count` 12, `Lanes` 5, rows as in section 11, no motion |
| T2 | CHECK with Zsafe below Z0, then with X0 outside the fence | `CheckResult` 1, then 3, `Count` 0 both times |
| T3 | Test Move in **Standby** to a safe pose | error 6 at once, Status 4, nothing moved, gates in word 123 all 0 |
| T4 | Test Move in Auto, block 1, safe pose | gantry moves once, `SrlRealRaw[1]` = 1, `[3]` = run number, Status 3 |
| T5 | Same Test Move again, same pose | no fault from the zero-length LIN, or you know you need the skip in every case |
| T6 | Test Move block 2 | moves at the slow speed, `SrlRealRaw[1]` = 2 |
| T7 | Force word 114 to another value mid-move in a watch table | target does not change; if it does, write the lane only when the robot stands |
| T8 | Abort mid-move | robot stops, Status 4, Error 6; next Start retracts first |
| T9 | Rectangle in STEP MODE, NEXT per row | 12 rows, each stops, order as section 11 |
| T10 | Rectangle at full speed, scanner recording | 20 clean runs, `EstTimeS` within 20 % of elapsed |
| T11 | RobotMonitor takes access mid-job | error 1 or 2, Status 4; decide fault or hold (section 14) |
| T12 | E-stop mid-job, reset, MACHINE START, START SCAN | retract first, then the full rectangle |

---

## 13. Error numbers

| Number | Meaning | Source |
| --- | --- | --- |
| 1 | start not taken, `program_finished` did not fall in 2 s | FB_SrlMove |
| 2 | move timeout | FB_SrlMove |
| 3 | robot fault during the move | FB_SrlMove |
| 4 | wrong block or stale run number | FB_SrlMove |
| 5 | communication lost | FB_SrlMove |
| 6 | aborted | both |
| 7 | block number outside 1 to 4 | FB_SrlMove |
| 10 | no job loaded | FB_ScanJob |
| 11 | machine not ready | FB_ScanJob |
| 12 | measurement timeout | FB_ScanJob |
| 101 | bad rectangle parameters | FC_GenRect via OB1 |
| 102 | more than 80 rows | FC_GenRect via OB1 |
| 103 | a point outside the fence | FC_GenRect via OB1 |

---

## 14. Still open

Ordered by how much it can hurt later.

1. **Position registration of the scan data.** The PLC pose is whole millimetres at 10 ms, too coarse to stamp data with. Either the scanner runs time based at constant speed (that is what overscan is for) or it needs an encoder signal from the drive. Ask the scanner vendor before the mechanics are fixed.
2. **Scan speed is not on the HMI.** Blocks 1 and 2 carry their speed inside program 20. `Vscan` only sizes the timeout. If the operator must choose a speed: program 20 reads it from REAL[53], or gates 5 to 8 become more LIN speeds.
3. **Override during a job.** A changed override changes the data pitch and makes the timeouts wrong. Freeze it while Status is 1 or 2, and scale the row speed by override in `FB_SrlMove`. Not implemented. The 60 % cap is a scaling bug; find the factor.
4. **Lane direction.** Along X or along Y, serpentine or one direction, RotZ turned on return lanes: three parameters once the scanner is known.
5. **Tool frame.** POSE[51] must be the scanner TCP, set in MOVIRUN, and `ActPose` must be the same frame. Test: after a move, actual equals target within 1 mm.
6. **RobotMonitor taking access.** Today: fault after 2 s. If the PLC should step back and resume, `FB_ScanJob` needs a hold state that stops the timers.
7. **Fence and Zsafe.** Fence from the robot soft limits. Zsafe clears the tallest thing in the cell. Z0 above the part is the operator's responsibility unless something measures the part.
8. **Referenced before Start.** `MissingNo` must contain "not referenced".
9. **Program 20 preconditions.** Program number 20 selected and echoed, blending 0, the zero-length LIN answer (T5).
10. **Robot error text.** `ErrorNo` needs a text list from the manual, including 32073.
11. **Signalling.** Lamp or horn while Auto is active. Machine directive, not yet in any document.

### Questions for SEW that decide some of the above

1. SRCI support on this controller and firmware: it would replace the whole transport.
2. Supported pattern for 100 to 500 PLC-generated points with a stop at each: gated program with one Start per point, 7-segment windows, or a queue.
3. A two-word option per pose element or per REAL, so a 4380 mm axis gets 0.1 mm.
4. `program_finished` timing around Start, its behaviour on a program error, and the error bit.
5. `motion_done` (PI 5.2) semantics; `robot_moving` (PI 1.4) velocity or program based; an in-position bit.
6. Start edge or level, minimum pulse, latency Start to motion and path end to `program_finished`.
7. CIRC with radius and direction: plane, CW definition, arcs over 180°, radius below half the chord, Reset mid-arc.
8. Word and index of SRL Bool actual values (word 81?).
9. A PI bit for "fieldbus has no access".
10. Override word scaling.
