# HMI: the scan screen

WinCC Unified, one operator screen for the scan, one commissioning screen, the existing alarm view.

## Scan screen

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

Rules on the screen, not in the PLC:

- Every number field is an integer with its limits taken from the fence. Speeds and the fence are on the commissioning screen only.
- The fields are locked while Status is 1 or 2.
- START SCAN is enabled only when mode is Auto, Machine is READY, Status is 0 or 3, and the last CHECK was ok.
- PAUSE is a switch, the other buttons are presses. The PLC clears every press bit once seen, so a stuck button does nothing.
- MISSING shows the MissingNo text of FB_MachineReady while the machine is not ready.

## Commissioning screen

Speeds Vscan and Vmove, the fence, the override, the test move (pose, block, TEST MOVE button), STEP MODE switch with NEXT, and the raw robot words SrlRealRaw[1] and [3], program_finished, program number. Visible at commissioning level only.

## Variables

Direction is seen from the HMI. "Press" means the HMI writes TRUE and the PLC clears it.

| Tag | Type | Direction | Use | Level |
| --- | --- | --- | --- | --- |
| DB_Scan.X0, Y0, Z0 | Real | write | corner and scan height, limits from the fence | operator |
| DB_Scan.RotZ, RotY | Real | write | tool orientation, limits from the axis limits | operator |
| DB_Scan.Lx, Ly | Real | write | rectangle size, greater than 0 | operator |
| DB_Scan.Py | Real | write | lane pitch, 1 to Ly | operator |
| DB_Scan.Px | Real | write | point pitch, 0 = continuous lanes | operator |
| DB_Scan.Over | Real | write | overscan, 0 or more | operator |
| DB_Scan.Zsafe | Real | write | retract height, above Z0 | operator |
| DB_Scan.Vscan, Vmove | Real | write | must equal the speeds in blocks 2 and 1 | commissioning |
| DB_HMI.StartScan | Bool | press | builds the list and starts | operator |
| DB_HMI.Check | Bool | press | builds the list, shows rows and lanes | operator |
| DB_HMI.Pause | Bool | switch | holds between rows | operator |
| DB_HMI.Abort | Bool | press | stops, job fault 6 | operator |
| DB_HMI.ResetFault | Bool | press | clears Status 4 | operator |
| DB_HMI.StepMode | Bool | switch | one row per NEXT | commissioning |
| DB_HMI.Next | Bool | press | next row in step mode | commissioning |
| DB_HMI.TestMove | Bool | press | one row from TestPose | commissioning |
| DB_HMI.TestPose.X .. RotY | Real | write | target of the test move | commissioning |
| DB_HMI.TestBlock | Int | write | 1 to 4 | commissioning |
| DB_HMI.CheckResult | Int | read | 0 ok, 1 bad size, 2 too long, 3 outside the fence, as text | operator |
| DB_Job.Status | Int | read | 0 idle, 1 running, 2 paused, 3 done, 4 fault, as text | operator |
| DB_Job.RowIdx, Count | Int | read | row n of m | operator |
| DB_Job.Lanes | Int | read | lanes of the list | operator |
| DB_Job.EstTimeS | Int | read | estimated time | operator |
| DB_Job.Error | Int | read | error number, as text from the error table | operator |
| DB_Job.Rows[n] | UDT_Row | read | optional list view after CHECK | commissioning |
| DB_Job.Fence.Xmin .. Zmax | Real | write | the envelope, inside the robot soft limits | admin |
| DB_Machine.Ready | Bool | read | READY TO START | operator |
| DB_Machine.MissingNo | Int | read | what is missing, as text | operator |
| DB_Mode.intMode | Int | read | Standby, Hand, Auto, as text | operator |
| DB_Robot.Status.ActPose.X .. RotY | Real | read | actual position, whole mm | operator |
| DB_Robot.Status.CommOk, Automatic, Enabled, Referenced, SafeStop | Bool | read | robot state lamps | operator |
| DB_Robot.Status.Fault, ErrorNo | Bool, Int | read | robot fault and its number, text from the manual | operator |
| DB_Robot.Status.ProgFinished, ProgNo | Bool, Int | read | diagnostics | commissioning |
| DB_Robot.Status.SrlRealRaw[1], [3] | Int | read | block that ran, run number echo | commissioning |
| DB_Robot.Status.Override | Int | read | the override on the bus | commissioning |
| DB_SrlLane.Pose[1..5] | Int | read | the target on the bus | operator |
| DB_Scanner.ScanOn, Trigger | Bool | read | scanner state lamps | operator |
| DB_Scanner.Ready, Error, MeasDone | Bool | read | scanner state lamps | operator |
| DB_Scanner.RowTag | Int | read | row the data is tagged with | commissioning |
