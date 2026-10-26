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

| File | What it is | Goes where |
| --- | --- | --- |
| `robot/program20.srl` | the motion library, four gated blocks | RobotMonitor, program 20 |
| `plc/UDT_Pose5.udt` | X Y Z RotZ RotY | TIA, PLC data types |
| `plc/UDT_Fence.udt` | min and max per axis | TIA |
| `plc/UDT_Row.udt` | one row: pose, block, speed, radius, scan on, measure | TIA |
| `plc/UDT_Job.udt` | count, row index, status, error, lanes, time, fence, 80 rows | TIA |
| `plc/UDT_ScanRect.udt` | the rectangle parameters | TIA |
| `plc/UDT_SrlLane.udt` | pose words, parameter words, gates, commit | TIA |
| `plc/UDT_HMI.udt` | press bits, switches, test pose, check result | TIA |
| `plc/UDT_Scanner.udt` | scanner interface | TIA |
| `plc/DB_Scan.db` | the rectangle, with standard values, **set retentive** | TIA, global DB |
| `plc/DB_Job.db` | the list, with a placeholder fence | TIA |
| `plc/DB_SrlLane.db` | the staging window | TIA |
| `plc/DB_HMI.db`, `plc/DB_Scanner.db` | HMI and scanner tags | TIA |
| `plc/FC_AddRow.scl` | appends one row, fence checked | TIA |
| `plc/FC_GenRect.scl` | rectangle into rows, lanes, time estimate | TIA |
| `plc/FB_SrlMove.scl` | one move: window, commit, start, wait, verify | TIA, instance inside FB_ScanJob |
| `plc/FB_ScanJob.scl` | walks the rows, retract, scanner, pause, step, fault | TIA, one instance `DB_ScanJob` |
| `plc/OB30_srl_window.scl` | the network that ships the window | OB30, network 2 |
| `plc/OB1_networks.scl` | networks 4 to 6 of OB1 | OB1 |

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

## 4. Names: mine to yours

The PLC sources use placeholder names for things that already exist in your project. Rename in the sources before generating, or map after.

| In the sources | Your project |
| --- | --- |
| `"DB_Mode".Auto` | the Auto bit of your mode DB |
| `"DB_Machine".Ready`, `.MissingNo` | the outputs of `FB_MachineReady` |
| `"DB_Robot".Status.ProgFinished` | PI 2.5 as mapped by `FC_RobotStatusMap` |
| `"DB_Robot".Status.Fault`, `.CommOk` | robot fault bit, communication ok from the module FB |
| `"DB_Robot".Status.ActPose` | the actual pose words from PI, same frame as POSE[51] |
| `"DB_Robot".Status.SrlRealRaw[1..16]` | your `SrlRealRaw` |
| `"DB_RobotCmd".Auto.Start`, `.ProgNo`, `.Stop` | the automatic requests your arbiter reads |
| `"DB_MOVIKIT_ROBOTICS_Module2".Pose`, `.Param`, `.WrBool`, `.PoseEnable` | the pins of the v0.7 module 2 FB |
| `"DB_Scanner"` | the real scanner interface, when it exists |

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

In TIA: *External source files → Add new external file*, select the files, right click → *Generate blocks from source*. Generate in this order, each group compiles before the next:

1. `UDT_Pose5`, `UDT_Fence`, `UDT_Row`, `UDT_Job`, `UDT_ScanRect`, `UDT_SrlLane`, `UDT_HMI`, `UDT_Scanner`
2. `DB_Scan`, `DB_Job`, `DB_SrlLane`, `DB_HMI`, `DB_Scanner`
3. `FC_AddRow`, then `FC_GenRect`
4. `FB_SrlMove`, then `FB_ScanJob`

Then: `DB_Scan` → properties → retentive. Put the real fence into `DB_Job.Fence`, read from the robot soft limits, a few millimetres inside them.

**Done when:** the project compiles with 0 errors. `FB_SrlMove` has no instance DB of its own; it lives inside `DB_ScanJob`.

---

## 7. Step 3 — OB30: the window ship

OB30 keeps its four networks. Insert this as **network 2**, between the handshake toggle and the module 1 FB, so the same pass transmits the window:

```scl
// ob30 network 2: ship the srl window in one pass, only when fb_srlmove has committed it
// this is the only place that writes the pose, parameter and gate pins of module 2
IF "DB_SrlLane".Commit THEN
   "DB_MOVIKIT_ROBOTICS_Module2".Pose := "DB_SrlLane".Pose;        // pose 51, words 114..118
   "DB_MOVIKIT_ROBOTICS_Module2".Param := "DB_SrlLane".Par;        // real 51..54, words 119..122
   "DB_MOVIKIT_ROBOTICS_Module2".WrBool := "DB_SrlLane".Gate;      // bool 51..58, word 123
   "DB_MOVIKIT_ROBOTICS_Module2".PoseEnable := TRUE;
   "DB_SrlLane".Commit := FALSE;
END_IF;
```

Remove any other writer of those pins (the old `FC_SrlGates` path). OB30 order afterwards: handshake, window, module 1 FB, module 2 FB, status map.

**Done when:** in a watch table, write `DB_SrlLane.Pose[1] := 1234` and `Commit := TRUE`. Within one OB30 pass Commit is FALSE and the Process Data Monitor shows 1234 in word 114.

---

## 8. Step 4 — the status map

`FC_RobotStatusMap` must deliver into `DB_Robot.Status`:

| Field | Source |
| --- | --- |
| `ProgFinished` | PI 2.5 `program_finished` |
| `Fault` | the robot fault bit |
| `CommOk` | module FB communication ok |
| `ActPose.X .. RotY` | actual pose words, as Real, whole mm |
| `SrlRealRaw[1..16]` | words 65 to 80 |
| `Referenced`, `Automatic`, `Enabled`, `SafeStop` | for the HMI lamps and `FB_MachineReady` |

Do not map `motion_done` (PI 5.2) to anything the job reads.

---

## 9. Step 5 — OB1: the three networks

Paste after `FB_MachineReady` and before the arbiter. OB1 temps: `xStartJob : Bool`, `rTestSpeed : Real`.

```scl
// 4a reset: only clears a fault, never touches a running job
IF "DB_HMI".ResetFault THEN
   "DB_HMI".ResetFault := FALSE;
   IF "DB_Job".Status = 4 THEN
      "DB_Job".Status := 0;
      "DB_Job".Error := 0;
   END_IF;
END_IF;

// 4b check: builds the list from the screen values, shows rows and lanes, starts nothing
IF "DB_HMI".Check THEN
   "DB_HMI".Check := FALSE;
   IF "DB_Job".Status <> 1 AND "DB_Job".Status <> 2 THEN
      "DB_HMI".CheckResult := "FC_GenRect"(i_scan := "DB_Scan", io_job := "DB_Job");
   END_IF;
END_IF;

// 4c start: the list is built again from what is on the screen at the moment of the press
#xStartJob := FALSE;
IF "DB_HMI".StartScan THEN
   "DB_HMI".StartScan := FALSE;
   IF "DB_Job".Status <> 1 AND "DB_Job".Status <> 2 THEN
      "DB_HMI".CheckResult := "FC_GenRect"(i_scan := "DB_Scan", io_job := "DB_Job");
      IF "DB_HMI".CheckResult = 0 THEN
         #xStartJob := TRUE;                 // one cycle, consumed in network 5
      ELSE
         "DB_Job".Status := 4;
         "DB_Job".Error := 100 + "DB_HMI".CheckResult;   // 101 bad size, 102 too long, 103 outside the fence
      END_IF;
   END_IF;
END_IF;

// 4d test move: one row from the test pose, through the same path as production
IF "DB_HMI".TestMove THEN
   "DB_HMI".TestMove := FALSE;
   IF "DB_Job".Status <> 1 AND "DB_Job".Status <> 2 THEN
      IF "DB_HMI".TestBlock = 2 THEN
         #rTestSpeed := "DB_Scan".Vscan;
      ELSE
         #rTestSpeed := "DB_Scan".Vmove;
      END_IF;
      "DB_Job".Count := 0;
      "DB_HMI".CheckResult := "FC_AddRow"(i_rX := "DB_HMI".TestPose.X,
                                          i_rY := "DB_HMI".TestPose.Y,
                                          i_rZ := "DB_HMI".TestPose.Z,
                                          i_rRotZ := "DB_HMI".TestPose.RotZ,
                                          i_rRotY := "DB_HMI".TestPose.RotY,
                                          i_iBlock := "DB_HMI".TestBlock,
                                          i_rSpeed := #rTestSpeed,
                                          i_xScanOn := FALSE,
                                          i_xMeasure := FALSE,
                                          i_fence := "DB_Job".Fence,
                                          io_rows := "DB_Job".Rows,
                                          io_n := "DB_Job".Count);
      IF "DB_HMI".CheckResult = 0 THEN
         #xStartJob := TRUE;
      END_IF;
   END_IF;
END_IF;

// 5 the job, one instance, it owns the one fb_srlmove instance inside it
"DB_ScanJob"(i_xStart := #xStartJob,
             i_xPause := "DB_HMI".Pause,
             i_xStepMode := "DB_HMI".StepMode,
             i_xNext := "DB_HMI".Next,
             i_xAbort := "DB_HMI".Abort OR NOT "DB_Mode".Auto,
             i_xMachineReady := "DB_Machine".Ready,
             i_actPose := "DB_Robot".Status.ActPose,
             i_rZsafe := "DB_Scan".Zsafe,
             i_rVmove := "DB_Scan".Vmove,
             i_xMeasDone := "DB_Scanner".MeasDone,
             i_xProgFinished := "DB_Robot".Status.ProgFinished,
             i_xRobotFault := "DB_Robot".Status.Fault,
             i_xCommOk := "DB_Robot".Status.CommOk,
             i_iReal71 := "DB_Robot".Status.SrlRealRaw[1],
             i_iReal73 := "DB_Robot".Status.SrlRealRaw[3],
             io_job := "DB_Job",
             io_lane := "DB_SrlLane");
"DB_HMI".Next := FALSE;
"DB_HMI".Abort := FALSE;

// 6 the requests of the automatic side, the arbiter decides whether they reach the robot
"DB_RobotCmd".Auto.Start := "DB_ScanJob".o_xStart;
"DB_RobotCmd".Auto.ProgNo := "DB_ScanJob".o_iProgNo;
"DB_RobotCmd".Auto.Stop := "DB_ScanJob".o_xFault;
"DB_Scanner".ScanOn := "DB_ScanJob".o_xScanOn;
"DB_Scanner".Trigger := "DB_ScanJob".o_xMeasTrigger;
"DB_Scanner".RowTag := "DB_Job".RowIdx;
```

In the arbiter: in Auto, copy `Auto.Start`, `Auto.ProgNo` and `Auto.Stop` to the robot pins. In Hand and Standby, Start stays FALSE whatever the job asks. Start is a level, held by `FB_SrlMove` until `program_finished` falls.

**Done when:** the project compiles. In Standby, a Test Move ends with error 1 after 2 s and nothing moved: the arbiter blocked the Start, the timeout caught it.

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
| DB_Machine.Ready, MissingNo | Bool, Int | read | ready, what is missing | operator |
| DB_Mode.intMode | Int | read | Standby, Hand, Auto | operator |
| DB_Robot.Status.ActPose | UDT_Pose5 | read | actual position | operator |
| DB_Robot.Status.CommOk, Automatic, Enabled, Referenced, SafeStop | Bool | read | robot lamps | operator |
| DB_Robot.Status.Fault, ErrorNo | Bool, Int | read | robot fault, text from the manual | operator |
| DB_Robot.Status.ProgFinished, ProgNo, Override | mixed | read | diagnostics | commissioning |
| DB_Robot.Status.SrlRealRaw[1], [3] | Int | read | block that ran, run number echo | commissioning |
| DB_SrlLane.Pose[1..5] | Int | read | the target on the bus | operator |
| DB_Scanner.ScanOn, Trigger, Ready, Error, MeasDone | Bool | read | scanner lamps | operator |
| DB_Scanner.RowTag | Int | read | row the data is tagged with | commissioning |

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
3. **OB1.** `FB_SrlMove` sees Commit FALSE, raises Start. Network 6 and the arbiter pass it to `Out.Start`, program 20.
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
| DB_Job rows, count, lanes, time | FC_GenRect | OB1 network 4 |
| DB_Job RowIdx, Status, Error | FB_ScanJob, and the reset in 4a | OB1 |
| DB_SrlLane | FB_SrlMove inside FB_ScanJob | OB1 network 5 |
| Module 2 setpoint pins | window ship | OB30 network 2 |
| DB_RobotCmd.Auto | network 6 | OB1 |
| DB_RobotCmd.Out | arbiter | OB1 network 7 |
| Module 1 control words | module 1 FB | OB30 |
| DB_Robot.Status | FC_RobotStatusMap | OB30 |
| DB_HMI press bits | HMI sets, OB1 clears | both |

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
| T3 | Test Move in **Standby** to a safe pose | error 1 after 2 s, nothing moved, words 114 to 123 show the pose and gate 1 |
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
