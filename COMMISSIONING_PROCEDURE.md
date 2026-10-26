# BGE Portal — commissioning procedure for the scan blocks

Step by step. Every step has **Do**, **Check** and **If not**. A step passes only when every line of its Check is true. Do not skip a stage and do not run a later stage while an earlier one is open.

The order is the safety: stages A to D move nothing. Stage E is the robot alone, from RobotMonitor. Stage F is the first PLC-driven motion, at 10 % override, one short move. Stage G is the rectangle. Stage H is the fault cases.

Covers: `UDT_*`, `DB_Scan`, `DB_Job`, `DB_SrlLane`, `DB_HMI`, `DB_ScanJob`, `FC_AddRow`, `FC_GenRect`, `FB_SrlMove`, `FB_ScanJob`, `FC_SrlWindow`, `FC_ScanControl`, and program 20 on the robot.

---

## 0. Rules for the whole procedure

- **Nobody inside the fence while drives are enabled.** One person commands at the HMI or the PG, a second person stands at an E-stop with eyes on the gantry. Both agree before every motion step.
- **E-stop test first.** Before stage E, press every E-stop once with drives enabled and confirm the robot drops to safe stop and `FB_MachineReady` reports it. If the STO bit still reads active with the chain closed, stop here. That fault blocks everything after stage D.
- **Override 10 %** from stage E until stage G says otherwise.
- **Test poses are in free air.** Every test pose and the commissioning rectangle lie in a volume with nothing in it, above the part, below the ceiling, inside the fence. Zsafe is above everything in the cell.
- **Write it down.** The record sheet in section 10 has one line per step. A step without a line is not done.
- **Known weaknesses of this version**, and what the procedure does about them:
  1. The move timeout does not know the override. At 10 % override a move takes ten times longer than the speed in `DB_Scan` says. The procedure sets `DB_Scan.Vmove` and `Vscan` to the **effective** speed, programmed speed × override / 100, during every low-override stage.
  2. The first retract to Zsafe is not fence checked. It goes to the actual X and Y from the PI words. Before every Start in stages F to H you read the actual pose and confirm it is the real position, not zero.
  3. After a fault, `DB_ScanJob.o_xFault` and so `o_xStop` stay TRUE until the next Start, Reset or not. In Auto that holds the robot Stop. Expected, not a defect of the test.

---

## 1. Preparation

### 1.1 Watch tables

Create four watch tables. Monitor every tag, modify only the ones marked M.

**WT_HMI** — presses and the rectangle

| Tag | M | Purpose |
| --- | --- | --- |
| `DB_HMI.Check`, `.StartScan`, `.Abort`, `.ResetFault`, `.TestMove`, `.Next` | M | press bits, write TRUE, the PLC clears them |
| `DB_HMI.Pause`, `.StepMode` | M | switches |
| `DB_HMI.TestPose.X .. RotY`, `.TestBlock` | M | test move target |
| `DB_HMI.CheckResult` | | 0 ok, 1 bad size, 2 too long, 3 outside fence |
| `DB_Scan.X0 .. Zsafe` | M | the rectangle |

**WT_Job** — the list and the job

| Tag | M | Purpose |
| --- | --- | --- |
| `DB_Job.Count`, `.Lanes`, `.EstTimeS` | | generator result |
| `DB_Job.RowIdx`, `.Status`, `.Error` | | job state |
| `DB_Job.Rows[1].Pose.X` .. `Rows[12].Block` | | the first 12 rows, X Y Z Block ScanOn Measure |
| `DB_Job.Fence.Xmin .. Zmax` | M | the envelope |
| `DB_ScanJob.iState`, `DB_ScanJob.move.iState` | | the two state machines |
| `DB_ScanJob.o_xBusy`, `.o_xDone`, `.o_xFault`, `.o_iError`, `.o_xStart`, `.o_xScanOn`, `.o_xMeasTrigger` | | job outputs |
| `DB_ScanJob.move.iRun`, `.move.tMove`, `.move.xHaveLast` | | run number, current timeout |

**WT_Lane** — the window and the module 2 pins

| Tag | M | Purpose |
| --- | --- | --- |
| `DB_SrlLane.Pose[1..5]`, `.Par[1..4]`, `.Gate[1..8]`, `.Commit` | M | the staging window |
| the module 2 pins the `FC_SrlWindow` in-outs are connected to | | what goes on the bus |
| your `DB_RobotCmd` Auto Start, ProgNo, Stop and the robot pins the arbiter writes | | start request and what reaches the robot |

**WT_Robot** — what the PLC sees from the robot

| Tag | M | Purpose |
| --- | --- | --- |
| your status tags on the `FC_ScanControl` pins: program_finished, fault, comm ok, actual pose words, `SrlRealRaw[1]`, `SrlRealRaw[3]` | | robot feedback |
| your mode tag and `FB_MachineReady` Ready, MissingNo | | preconditions |

### 1.2 The commissioning rectangle

Small, short, in free air. Put the real numbers of your free volume in X0, Y0, and choose Z0 100 mm below Zsafe.

| Parameter | Value | Why |
| --- | --- | --- |
| X0, Y0 | centre of the free volume, whole mm | yours |
| Zsafe | the clear height | yours |
| Z0 | Zsafe − 100 | short vertical moves |
| RotZ, RotY | 0 | |
| Lx | 100 | |
| Ly | 60 | |
| Py | 30 | 3 lanes |
| Px | 0 | lanes, not points |
| Over | 10 | |
| Vmove, Vscan | effective, see each stage | timeout only |

With X0 = 1000, Y0 = 500, Zsafe = 600 the generator must produce exactly this. Replace 1000, 500, 600 with yours and shift every number by the same offset.

| Row | X | Y | Z | Block | ScanOn |
| --- | --- | --- | --- | --- | --- |
| 1 | 990 | 500 | 600 | 1 | 0 |
| 2 | 990 | 500 | 500 | 1 | 0 |
| 3 | 1110 | 500 | 500 | 2 | 1 |
| 4 | 1110 | 530 | 500 | 1 | 0 |
| 5 | 990 | 530 | 500 | 2 | 1 |
| 6 | 990 | 560 | 500 | 1 | 0 |
| 7 | 1110 | 560 | 500 | 2 | 1 |
| 8 | 1110 | 560 | 600 | 1 | 0 |

Count 8, Lanes 3, EstTimeS 16 with Vmove 200 and Vscan 50.

### 1.3 The test DB for the dry run

Stage D walks the state machines by hand, with no robot. For that the inputs of `FC_ScanControl` come from a DB you control. Create it:

```
DATA_BLOCK "DB_Commission"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN
   STRUCT
      xAuto : Bool;
      xMachineReady : Bool;
      xProgFinished : Bool;
      xRobotFault : Bool;
      xCommOk : Bool;
      iReal71 : Int;
      iReal73 : Int;
      iActX : Int;
      iActY : Int;
      iActZ : Int;
      iActRotZ : Int;
      iActRotY : Int;
      xMeasDone : Bool;
   END_STRUCT;
BEGIN
END_DATA_BLOCK
```

In stage D the thirteen input pins of `FC_ScanControl` point at these fields. In stage F they go back to the real tags. Write both wirings down in section 10 before you change anything, and tick the second one off pin by pin.

`FC_SrlWindow` stays on the real module 2 pins in every stage. While the robot is in Standby the arbiter never passes Start, so a window on the bus is harmless. One exception, section 3.

---

## 2. Stage A — static, nothing can move

Robot in Standby, drives may be off.

**A1 Download and compile**
- Do: compile hardware and software, download, CPU to RUN.
- Check: 0 errors, 0 warnings about the new blocks. In the block list: 7 UDTs, `DB_Scan`, `DB_Job`, `DB_SrlLane`, `DB_HMI`, `DB_ScanJob`, `FC_AddRow`, `FC_GenRect`, `FC_SrlWindow`, `FC_ScanControl`, `FB_SrlMove`, `FB_ScanJob`.
- If not: the error names the line. Word versus Int on the module 2 pins is the usual one.

**A2 The two calls**
- Do: open OB30 and OB1.
- Check: OB30 network order is handshake, `FC_SrlWindow`, module 1 FB, module 2 FB, status map. All 18 in-outs of `FC_SrlWindow` connected. In OB1, `FC_ScanControl` sits after `FB_MachineReady` and before the arbiter, all 13 inputs and the 3 outputs to the arbiter connected. Cross reference: nothing else writes the module 2 pose, parameter and bool pins.
- If not: a second writer of a pin makes stage C flicker. Remove it, usually the old `FC_SrlGates`.

**A3 Cycle and process image**
- Do: OB30 properties, and the module 2 slot properties.
- Check: OB30 cycle 10 ms. Slot 19 assigned to the process image partition of OB30, or accessed as direct I/O in the module FB.
- If not: the window leaves the PLC at the OB1 cycle, not every 10 ms. Fix before stage C.

**A4 Data at rest**
- Do: open WT_Job and WT_HMI online.
- Check: `DB_Job.Status` 0, `RowIdx` 0, `Count` 0, `Error` 0. All `DB_HMI` press bits FALSE. `DB_Scan` holds the standard values. `DB_Job.Fence` holds your real envelope, each limit a few millimetres inside the robot soft limits, not the placeholders 0 / 4000 / 3000 / 1500.
- If not: a press bit stuck TRUE means the HMI writes it cyclically. It must write once on press.

**A5 Retentive**
- Do: `DB_Scan` properties.
- Check: retentive. Power cycle the CPU, values survive.

---

## 3. Stage B — the generator, no motion

Robot in Standby. The generator writes `DB_Job` only.

Safety note for this and the next stage: a window that reaches the bus is a pose in POSE[51]. If anyone starts program 20 from RobotMonitor with a gate TRUE, the gantry goes there. During stages B to D nobody touches RobotMonitor, and every test pose is in free air anyway.

**B1 The commissioning rectangle**
- Do: enter the rectangle of 1.2 in `DB_Scan`. Modify `DB_HMI.Check := TRUE`.
- Check: `Check` back to FALSE within a cycle. `CheckResult` 0. `Count` 8, `Lanes` 3, `EstTimeS` 16. Rows 1 to 8 exactly as the table in 1.2, including Block and ScanOn. Row 3, 5, 7 have ScanOn TRUE, all others FALSE. Measure FALSE everywhere. `Status` still 0.
- If not: a wrong X on rows 1, 2 means Over is not applied to the approach. A wrong lane count means Py or Ly. Compare with the generator rules in APPLY_GUIDE section 11.

**B2 Point mode**
- Do: `Px := 50`. Check.
- Check: `Count` 11, `Lanes` 3. Row 1 (1000, 500, 600) block 1 Measure FALSE. Row 2 (1000, 500, 500) Measure TRUE. Row 3 (1050, 500, 500) block 2 Measure TRUE, ScanOn FALSE. Row 4 (1100, 500, 500). Row 5 (1100, 530, 500) block 1 Measure TRUE. Row 6 (1050, 530, 500), row 7 (1000, 530, 500). Row 8 (1000, 560, 500) block 1. Row 9 (1050, 560, 500), row 10 (1100, 560, 500). Row 11 (1100, 560, 600) block 1. No Over anywhere, point mode ignores it.
- Then: `Px := 0` again.

**B3 Bad size**
- Do: `Zsafe := 400`, below Z0. Check.
- Check: `CheckResult` 1, `Count` 0, `Lanes` 0, `EstTimeS` 0.
- Do: restore Zsafe. `Lx := 0`. Check. Same result, code 1. Restore Lx.

**B4 Outside the fence**
- Do: `X0 := Fence.Xmax − 50`. Check.
- Check: `CheckResult` 3, `Count` 0. The lane end X0 + Lx + Over is outside.
- Then: restore X0.

**B5 Too long**
- Do: `Py := 1`. Check.
- Check: `CheckResult` 2, `Count` 0. 61 lanes would be 124 rows.
- Then: `Py := 30`.

**B6 Start with a bad list**
- Do: `Zsafe := 400`. Modify `DB_HMI.StartScan := TRUE`.
- Check: `StartScan` cleared. `DB_Job.Status` 4, `Error` 101, `Count` 0. Nothing on `DB_SrlLane` changed. `DB_ScanJob.iState` 0.
- Do: `DB_HMI.ResetFault := TRUE`.
- Check: `Status` 0, `Error` 0.
- Then: restore Zsafe. Run B1 once more and confirm the list is good again.

---

## 4. Stage C — the window to the bus, no motion

Robot in Standby, RobotMonitor open on the Process Data Monitor, read only.

**C1 One word**
- Do: in WT_Lane modify `DB_SrlLane.Pose[1] := 1234` and `DB_SrlLane.Commit := TRUE` in one modify.
- Check: `Commit` FALSE on the next refresh. The module 2 pose pin 1 shows 1234. The Process Data Monitor shows 1234 in robot word 114. The PoseEnable pin TRUE.
- If not: Commit stays TRUE means `FC_SrlWindow` is not called or not reached. Pin 1 changes but word 114 does not means the pin is not the one for word 114, or the process image partition is wrong, A3.

**C2 Every word**
- Do: `Pose[1..5] := 1, 2, 3, 4, 5`, `Par[1..4] := 11, 12, 13, 14`, commit.
- Check: words 114 to 118 read 1 2 3 4 5, words 119 to 122 read 11 12 13 14.

**C3 Every gate**
- Do: `Gate[1] := TRUE`, commit. Check word 123 bit 0. Then `Gate[1] := FALSE`, `Gate[2] := TRUE`, commit, bit 1. Same for 3 and 4.
- Check: exactly one bit set each time, in the right position.
- Then: all gates FALSE, pose and par 0, commit. Word 123 reads 0.

**C4 Nothing starts**
- Check during C1 to C3: the robot Start pin stays FALSE, `program_finished` unchanged, no motion, no robot message.

---

## 5. Stage D — the state machines by hand, no robot

Robot in Standby. `FC_ScanControl` inputs wired to `DB_Commission`. Record the original wiring first.

Set the starting state in `DB_Commission`: `xAuto` TRUE, `xMachineReady` TRUE, `xProgFinished` TRUE, `xRobotFault` FALSE, `xCommOk` TRUE, `iReal71` 0, `iReal73` 0, `iActX` 1000, `iActY` 500, `iActZ` 500, rotations 0, `xMeasDone` TRUE. `DB_Scan` as in 1.2 with Vmove 200, Vscan 50.

How you play the robot: a pass of program 20 is three modifies in this order. First `xProgFinished := FALSE` (the start was taken). Then `iReal71 := <block>` and `iReal73 := <run number from DB_SrlLane.Par[2]>` together. Then `xProgFinished := TRUE`. The REALs must be written before the flag, as the real robot does it. The other order gives error 4, which D7 tests on purpose.

**D1 One test move, with retract**
- Do: `TestPose := (1000, 500, 550, 0, 0)`, `TestBlock := 1`. Modify `DB_HMI.TestMove := TRUE`.
- Check, within a cycle: `TestMove` cleared, `CheckResult` 0, `Count` 1, `Rows[1]` = the test pose, `Status` 1, `DB_ScanJob.iState` 30, `move.iState` 30. `DB_SrlLane.Pose` = 1000 500 **600** 0 0, the retract to Zsafe from the actual position, not the test pose. `Par[2]` 1, `Gate[1]` TRUE, others FALSE, `Commit` FALSE. The module 2 pins show the same. `o_xStart` TRUE. `DB_RobotCmd` Auto Start TRUE. **The robot Start pin FALSE**, the arbiter blocks it in Standby.
- If not: `move.iState` stuck at 20 means Commit never cleared, stage C. Robot Start pin TRUE means the arbiter passes Start outside Auto. Stop and fix that before anything else.

**D2 Play the retract**
- Do: `xProgFinished := FALSE`.
- Check: `move.iState` 40, `o_xStart` FALSE.
- Do: `iReal71 := 1`, `iReal73 := 1`. Then `xProgFinished := TRUE`.
- Check: gates all FALSE for a moment, then the next row: `DB_SrlLane.Pose` = 1000 500 550 0 0, `Par[2]` 2, `Gate[1]` TRUE, `RowIdx` 1, `move.iState` 30, `o_xStart` TRUE. `move.xHaveLast` TRUE.

**D3 Play the test move**
- Do: `xProgFinished := FALSE`. Then `iReal73 := 2`. Then `xProgFinished := TRUE`.
- Check: `Status` 3, `iState` 0, `move.iState` 0, `o_xBusy` FALSE, `o_xFault` FALSE, gates FALSE, `o_xStart` FALSE.

**D4 Retract skipped at safe height**
- Do: `iActZ := 600`. TestMove to (1100, 500, 600).
- Check: the first window is the test pose itself, `Par[2]` 3. No retract row. Play it to the end as in D3 with `iReal73 := 3`.

**D5 Error 1, start not taken**
- Do: TestMove to (1200, 500, 600). Leave `xProgFinished` TRUE.
- Check: after 2 s `move.iState` passes 90, `o_iError` 1, `Status` 4, `Error` 1, `o_xFault` TRUE, gates FALSE, `o_xStart` FALSE, `o_xStop` TRUE at the arbiter.
- Do: `ResetFault`. Check `Status` 0, `Error` 0. `o_xFault` stays TRUE, known, section 0.

**D6 Error 2, move timeout**
- Do: run one good move first so `xHaveLast` is TRUE: TestMove to (1100, 500, 600), play it fully. Then TestMove to (1200, 500, 600). `xProgFinished := FALSE`. Do nothing else.
- Check: `move.tMove` reads 4 s (100 mm at 200 mm/s, times 2, plus 3). After 4 s: `o_iError` 2, `Status` 4. Reset.

**D7 Error 4, wrong block or stale echo**
- Do: TestMove, `xProgFinished := FALSE`, `iReal71 := 2` (wrong, block is 1), `iReal73` correct, `xProgFinished := TRUE`.
- Check: `o_iError` 4 at once. Reset.
- Do again with `iReal71` correct and `iReal73` one less than `Par[2]`.
- Check: error 4 again. This is the stale echo case.

**D8 Error 3, 5, 6, 7, 11**
- Do, each from a fresh TestMove in state 40: `xRobotFault := TRUE` → error 3. `xCommOk := FALSE` → error 5. `DB_HMI.Abort := TRUE` → error 6, Abort cleared by the PLC. `xAuto := FALSE` → error 6. Restore each value, Reset between cases.
- Do: `TestBlock := 5`, TestMove.
- Check: error 7 at once, no window written, `Status` 4. `TestBlock := 1`, Reset.
- Do: `xMachineReady := FALSE`, `StartScan`.
- Check: error 11, nothing written. Restore, Reset.

**D9 The rectangle by hand**
- Do: `iActZ := 500`. `StartScan`.
- Check: `Count` 8, `Status` 1. First window is the retract 1000 500 600, `Par[2]` the next run number.
- Do: play 9 passes, retract plus rows 1 to 8, each with the block of the current row in `iReal71` and the current `Par[2]` in `iReal73`.
- Check at every row: `DB_SrlLane.Pose` equals the row in 1.2, `Gate[block]` TRUE, `RowIdx` counts 1 to 8. On rows 3, 5, 7 `o_xScanOn` TRUE from the moment the window is written until the pass is done, FALSE otherwise. After row 8 `Status` 3.

**D10 Pause and step mode**
- Do: `StartScan`, play two rows, then `DB_HMI.Pause := TRUE` before playing the third.
- Check: after the running row is done, `Status` 2, `iState` 20, no new window. `Pause := FALSE`: the next row appears.
- Do: `StepMode := TRUE`, `StartScan`.
- Check: after the retract `Status` 2 and no row until `DB_HMI.Next := TRUE`. Each Next releases one row, then holds again. `Next` cleared by the PLC each cycle. Play to the end, `StepMode := FALSE`.

**D11 Point mode with measurement**
- Do: `Px := 50`, `xMeasDone := FALSE`, `StartScan`. Play the retract and row 1. Play row 2, which has Measure TRUE.
- Check: after row 2 is done `o_xMeasTrigger` TRUE, `iState` 40, no new window. `xMeasDone := TRUE`: trigger FALSE, next row. Leave `xMeasDone` FALSE once for 10 s: error 12. Reset, `Px := 0`, `xMeasDone := TRUE`.

**D12 Real tags back**
- Do: rewire the 13 inputs of `FC_ScanControl` to the real tags, pin by pin against your record, compile, download.
- Check: WT_Robot shows live values on every pin: `program_finished` as the robot reports it, actual pose equal to the RobotMonitor position within 1 mm, comm ok TRUE.

---

## 6. Stage E — the robot alone, from RobotMonitor

Drives enabled, override 10 %, PLC in Standby so it sends no Start. E-stop test done, STO reads correctly.

**E1 Program 20 text**
- Check in RobotMonitor: first line `REAL[71] := 0`. Four gated blocks. Last line `REAL[73] := REAL[52]`. Nothing after it. Blending 0 for these moves. Program number 20.

**E2 Empty pass**
- Do: all BOOL[51..54] FALSE from RobotMonitor, REAL[52] := 77. Start program 20 from RobotMonitor.
- Check: program ends by itself. `program_finished` TRUE in WT_Robot. `SrlRealRaw[1]` 0, `SrlRealRaw[3]` 77. No motion.

**E3 One LIN from RobotMonitor**
- Do: POSE[51] := the actual position with Z + 50 mm, written in RobotMonitor. BOOL[51] := TRUE. Start program 20.
- Check: gantry rises 50 mm and stops. `SrlRealRaw[1]` 1, `SrlRealRaw[3]` 77, `program_finished` TRUE. Actual pose in WT_Robot equals the target within 1 mm.
- Do: BOOL[51] := FALSE, BOOL[52] := TRUE, POSE[51] Z − 50. Start.
- Check: gantry descends at the slow speed, `SrlRealRaw[1]` 2.
- Then: all gates FALSE. Write down the two speeds you measured or read from the program. They go into `DB_Scan` as Vmove and Vscan, scaled by the override, in the next stage.

**E4 program_finished timing**
- Do: start the empty pass again and watch `program_finished` in a trace or the watch table at 10 ms.
- Check: it falls within 2 s of Start and rises when the pass ends. If it does not fall at all, `FB_SrlMove` will always end in error 1 and the taken signal has to change before stage F. Stop and say so.

---

## 7. Stage F — first PLC-driven motion

Drives enabled, override 10 %, robot in Auto, machine READY, nobody inside, the second person at the E-stop. Real tags on all pins, confirmed in D12.

**F0 Effective speeds**
- Do: `DB_Scan.Vmove := programmed fast speed × 0.1`, `DB_Scan.Vscan := programmed slow speed × 0.1`. With 200 and 50 that is 20 and 5.
- Check: `move.tMove` after the first window reads a value that fits a 10 % move. Without this every move longer than a few seconds ends in error 2.

**F1 The retract is the first motion**
- Do: jog the gantry in Hand to a known free position 100 mm below Zsafe. Switch to Auto. In WT_Robot read the actual pose and confirm it is that position, not 0 0 0. `TestPose := actual X, actual Y, Zsafe − 50, 0, 0`, `TestBlock := 1`.
- Do: `TestMove`.
- Check: the gantry rises straight to Zsafe and stops. Then it descends 50 mm to the test pose and stops. `Status` 3. `SrlRealRaw[1]` 1, `SrlRealRaw[3]` the run number. Actual pose equals the target within 1 mm after each move.
- If not: it moves anywhere else, E-stop. A wrong first target means the actual pose words are not what the pins read, or not the same frame as POSE[51].

**F2 Slow block**
- Do: same test pose, `TestBlock := 2`.
- Check: the move runs at the slow speed, `SrlRealRaw[1]` 2.

**F3 Same pose twice**
- Do: `TestMove` again without changing the pose. The gantry already stands there.
- Check: either the robot accepts a zero-length LIN and finishes at once, `Status` 3, or it faults, error 3. Write down which. If it faults, every list must avoid a repeated pose and the skip at safe height is not enough.

**F4 Pose change during a move**
- Do: `TestPose` 200 mm away in X at Zsafe. `TestMove`. While it moves, in WT_Lane modify `DB_SrlLane.Pose[1]` to the start X and `Commit := TRUE`.
- Check: the gantry finishes at the original target. If it turns back to the new value, the robot reads POSE[51] live. Then the window may only be written while the robot stands, and `FB_SrlMove` state 10 needs a wait on robot_moving. Stop and say so.

**F5 Abort during a move**
- Do: `TestPose` 300 mm away. `TestMove`. Mid-move `DB_HMI.Abort := TRUE`.
- Check: the gantry stops on its path. `Status` 4, `Error` 6, `o_xStop` TRUE at the arbiter and the robot Stop pin TRUE. Reset. `TestMove` to a free pose: the first motion is the retract to Zsafe, then the pose.

---

## 8. Stage G — the rectangle

Override 10 %, effective speeds in `DB_Scan`, the commissioning rectangle of 1.2 in free air.

**G1 Step mode**
- Do: `StepMode := TRUE`, Check, `StartScan`.
- Check: retract, then one row per Next. Each stop position equals the row table within 1 mm. On rows 3, 5, 7 `o_xScanOn` TRUE while moving, FALSE at the stop. After row 8 `Status` 3 and the gantry stands at Zsafe above the last lane end.

**G2 Continuous at 10 %**
- Do: `StepMode := FALSE`. `StartScan`.
- Check: all 8 rows without a stop longer than the robot's own start and finish time. `Status` 3. `RowIdx` 8. No error.

**G3 Pause**
- Do: `StartScan`, `Pause := TRUE` during row 3.
- Check: row 3 completes, then the gantry waits, `Status` 2. `Pause := FALSE`, it continues with row 4.

**G4 Up the override**
- Do: override 30 %, `DB_Scan.Vmove` and `Vscan` to 30 % of the programmed speeds. `StartScan`.
- Check: clean run. Then 60 % and 60 % of the speeds. Clean run.
- Do not go above 60 % until the override scaling is fixed.

**G5 Twenty runs**
- Do: at 60 %, 20 consecutive `StartScan` with the real scanner connected or `i_xMeasDone` tied TRUE.
- Check: 20 times `Status` 3, no error, `EstTimeS` within 20 % of the measured time when the speeds in `DB_Scan` are the effective ones. Write the time down.

**G6 The real rectangle**
- Do: enter the production rectangle. Check. Walk it once in step mode at 30 %. Then continuous.
- Check: every stop in the expected place, scanner recording on the slow lanes.

---

## 9. Stage H — the fault cases on the real machine

Each from a running rectangle at 30 %.

**H1 E-stop**
- Do: press an E-stop during row 3.
- Check: gantry stops, robot fault, `Status` 4, `Error` 3. Release, Reset, `MACHINE START`, Reset again if needed, `StartScan`. First motion is the retract, then the full rectangle from row 1.

**H2 Leaving Auto**
- Do: switch the mode to Hand during a move.
- Check: `Status` 4, `Error` 6, robot stopped by the arbiter.

**H3 RobotMonitor takes access**
- Do: connect RobotMonitor and take access during a move.
- Check: the running move finishes or stops as the robot decides, the next Start is not taken, error 1 after 2 s, `Status` 4. Release access, Reset, StartScan, retract first. Decide now whether this stays a fault or becomes a hold, APPLY_GUIDE section 14.

**H4 Scanner not answering, point mode only**
- Do: `Px := 50`, disconnect the scanner's done signal.
- Check: error 12 after 10 s at the first measured row. Reconnect, `Px := 0`.

**H5 Power cycle**
- Do: switch the PLC off and on with a job finished.
- Check: `DB_Scan` values survive, `DB_Job.Status` 0, no press bit TRUE, `DB_SrlLane` zeros, no motion at power up.

---

## 10. Record sheet

One line per step. Observed is what you saw, not what was expected.

| Step | Expected | Observed | Pass | Date | Initials |
| --- | --- | --- | --- | --- | --- |
| A1 | 0 errors, 18 blocks | | | | |
| A2 | two calls, all pins, one writer | | | | |
| A3 | 10 ms, PIP of OB30 | | | | |
| A4 | zeros, real fence | | | | |
| A5 | DB_Scan retentive | | | | |
| B1 | 8 rows as table, 3 lanes, 16 s | | | | |
| B2 | 11 rows point mode | | | | |
| B3 | code 1, count 0 | | | | |
| B4 | code 3 | | | | |
| B5 | code 2 | | | | |
| B6 | Status 4, Error 101, Reset clears | | | | |
| C1 | 1234 in word 114 | | | | |
| C2 | all 9 words | | | | |
| C3 | gate bits 0 to 3 | | | | |
| C4 | no Start, no motion | | | | |
| D1 | retract window, Start blocked | | | | |
| D2 | taken, next row | | | | |
| D3 | Status 3 | | | | |
| D4 | retract skipped | | | | |
| D5 | error 1 after 2 s | | | | |
| D6 | error 2 after 4 s | | | | |
| D7 | error 4 both ways | | | | |
| D8 | errors 3 5 6 7 11 | | | | |
| D9 | 8 rows, ScanOn on 3 5 7 | | | | |
| D10 | pause, step | | | | |
| D11 | trigger, error 12 | | | | |
| D12 | real tags back on all 13 pins | | | | |
| E1 | program 20 text | | | | |
| E2 | empty pass, 0 and 77 | | | | |
| E3 | +50 fast, −50 slow | | | | |
| E4 | finished falls within 2 s | | | | |
| F0 | effective speeds | | | | |
| F1 | retract then pose, ±1 mm | | | | |
| F2 | slow block | | | | |
| F3 | zero-length result: | | | | |
| F4 | target unchanged mid-move | | | | |
| F5 | abort stops, retract first after | | | | |
| G1 | 8 stops as table | | | | |
| G2 | continuous 10 % | | | | |
| G3 | pause | | | | |
| G4 | 30 %, 60 % | | | | |
| G5 | 20 runs, time | | | | |
| G6 | production rectangle | | | | |
| H1 | E-stop, recovery | | | | |
| H2 | leaving Auto | | | | |
| H3 | access lost | | | | |
| H4 | error 12 | | | | |
| H5 | power cycle | | | | |

---

## 11. After commissioning

- `DB_Scan.Vmove` and `Vscan` back to the programmed speeds, and the override they belong to written next to them on the commissioning screen. Until the override input exists, the two must change together.
- Remove `DB_Commission` or leave it, but nothing may point at it.
- Keep the record sheet with the project.
- Open points that this procedure found go into APPLY_GUIDE section 14: the zero-length answer from F3, the live-pose answer from F4, the fault-or-hold decision from H3, and whether program_finished behaves as E4 needs.
