# Checklist – the line scan in automatic (2026-11-09)

Tick every line. It covers one thing: pose 9 at the first point to scan, a handful of numbers typed, job 3, START,
the robot drives n straight lines at one height. Two switches make the pattern: **ZIGZAG** on = serpentine (every
odd line runs backwards), off = raster (every line the same way, the move back is the long diagonal); **INDEX
ALONG** on = the next line lies further along the same line (the patch scan: approach, scan, index, repeat). A
third switch, **LIFT BETWEEN**, puts a lift, a travel and a slow descent between two lines. Every line is one slow
move from a lead-in point to a lead-out point, so the ramps lie outside the scanned patch; a window bit says when
the tool is inside the patch.

Nothing new on the robot: blocks 1 and 2 of program 20, `Cfg.intBlocksInPrg20` stays 4. The engine gets one field,
**min speed** (M43, `"DB_RobotMove".Cfg.rMinSpeed`, engine v0.8): the time limit of a move grows with its length, so
a 4 m line at a low override no longer ends in engine error 12; 0 = exactly as before. Nothing of the cube or the drum
changes: job 1 and job 2 run exactly as before. Block: `PLC/FB_ScanLineJob.scl` **v0.2** (v0.1 the same morning; v0.2
adds the **AREA** switch, two taught corners instead of length and lines, and takes the limit of 50 lines away), with
`PLC/FC_LineMove.scl` and `PLC/UDT_LineGeo.udt`. The design and the discussion in
`check_other_ideas_changes_critiqe/SCAN_TYPES.md` sections 4, 5, 7.

Three places, in this order: TIA (A), the robot (B, nothing to do), the screen (C). Then the numbers (D) with the
worked example, the start key by key, and the test (E). About two hours in all.

What the job drives, `FB_ScanLineJob` v0.2 (`PLC/`). `intMoveActive` (A72) is the **kind** of move, `intLineActive`
(A73) the line k:

| Kind | Move | Block | Target |
|---|---|---|---|
| 1 | to start | 1 | pose 2 |
| 2 | above line 0 | 1 | the lead-in point of line 0 at safe Z, already with the RotZ of line 0 |
| 3 | down onto line 0 | 2 | the same point at the scan height |
| 4 | scan line k | 2 | the lead-out point of line k. **The window bit is on while the tool is inside the patch** |
| 5 | index to next line | 1 | the lead-in point of line k+1 at the scan height (LIFT BETWEEN off) |
| 6 | lift between | 1 | the end of line k at lift Z (LIFT BETWEEN on) |
| 7 | travel | 1 | the lead-in point of line k+1 at lift Z |
| 8 | down onto the line | 2 | the same point at the scan height |
| 9 | lift | 1 | the end of the last line at safe Z |
| 10 | to start | 1 | pose 2 |
| 11 | home | 1 | pose 1, only if taught |

Moves total = 3 + n + (n − 1) with LIFT BETWEEN off, 3 + n + 3 × (n − 1) with it on, + 2 for lift and start, + 1
with home, − 1 with SCAN FROM HERE (no "to start" at the beginning). The job keeps no list (v0.2): it freezes the
geometry at START and `FC_LineMove` computes every move from its number when it is pushed, so there is no limit of
50 lines any more (5000 at most, job error 5 above). Before the first push the job checks the four outer line ends
(line 0 and the last line, lead-in to lead-out) at the scan height, at safe Z and at lift Z with both RotZ values,
plus pose 2 and pose 1: every point of the job lies between those corners, so a point outside the limits gives job
error 4 before anything moves, A72 the kind (4 line, 2 above, 6 lift) and A73 the line (0 or the last). The engine
checks every target again in its step 5. The queue holds 50 entries, so the job pushes while the queue holds fewer
than 8; the engine starts the first move while the job still pushes, as before.

**AREA** (A75, v0.2): the area by two taught corners instead of length and lines. Pose 9 is corner 1 (the start of
line 0, as before), pose 10 the opposite corner. The angle sets the axis of the lines; the extent of corner 2 along
the axis is the length, the extent across it divided by the pitch (only its size counts) gives the lines, and the
lines walk toward corner 2. Corner 2 behind corner 1 along the axis: the axis is turned 180, the lines still run from
corner 1 toward corner 2. A77 shows the computed number after START. With INDEX ALONG the patch length stays A59 and
the patches run from corner 1 up to corner 2. Off = A59 and A61 as typed, exactly v0.1.

---

## A. TIA, 40 minutes

### A1 First look

| Look at | Found | ☐ |
|---|---|---|
| OB1: the drum job box and the `FC_JobMux` box exist (the drum checklist part A) | yes → A2. no → the drum checklist first | ☐ |
| `"DB_Recipe"`: a struct `Line` at the end, 13 fields, the last one `xArea` | yes → A3. no → A2 row 1 | ☐ |
| `"DB_RobotJob"`: `Line`, `xScanWindow`, `intLineActive`, `intLineCount` at the end | yes → A3. no → A2 row 2 | ☐ |
| `"DB_RobotMove"`: `Cfg.rMinSpeed` after `Cfg.intBlocksInPrg20` | yes → A3. no → A2 row 8 | ☐ |

### A2 Import. External source files → Generate blocks from source → overwrite. In this order

| # | File in `PLC/` | Makes | ☐ |
|---|---|---|---|
| 1 | `DB_Recipe.scl` v0.5 | the struct `Line` at the end, 13 fields (12 of v0.4 and `xArea : Bool`). **A re-import sets every recipe number back to 0** (cube length, width, angle; drum scan height, radius, offset, clockwise; the three switches). Write them down first, or add the 13 rows **by hand** at the end of the DB in the editor, exactly as in the file, and skip the import. By hand is the safer way at the machine | ☐ |
| 2 | `DB_RobotJob.scl` v0.3 | `Line : "UDT_JobOut"`, `xScanWindow : Bool`, `intLineActive : Int`, `intLineCount : Int` at the end. Not retentive, nothing is lost. By hand is fine too: four rows at the end | ☐ |
| 3 | `FC_RobotAlarms.scl` v0.2 | job error 9 on alarm bit 17 | ☐ |
| 4 | `UDT_LineGeo.udt` v0.1 | new type, the frozen geometry of a line scan. Before the next two | ☐ |
| 5 | `FC_LineMove.scl` v0.1 | new function: one move of the scan from its number. Called inside the job only, not in OB1 | ☐ |
| 6 | `FB_ScanLineJob.scl` v0.2 | the line job. Instance `IDB_ScanLineJob`, single, when you place it. If v0.1 was already placed: the import replaces the block, the instance is updated at the compile | ☐ |
| 7 | `FC_JobMux.scl` v0.2 | one new input at the end, `i_udtLine`. After the import the mux box in OB1 shows the new pin in red: right-click → **Update block call** | ☐ |
| 8 | `DB_RobotMove.scl` v0.3 | `Cfg.rMinSpeed : Real` after `Cfg.intBlocksInPrg20`, start value 0. **A re-import sets `intBlocksInPrg20` back to its start value 4 and the new field to 0**, both retentive otherwise. By hand: one row in the editor after `intBlocksInPrg20`, Real, 0.0. Then type 10 (A4 row 2) | ☐ |
| 9 | `FB_RobotMove.scl` v0.8 | the engine, one new input at the end, `i_rMinSpeed`. The engine box in OB1 shows the new pin in red: right-click → **Update block call**, then A3 row 3. Nothing else in the engine changes, the cube and the drum run as before with the field at 0 | ☐ |

`DB_Poses` does not change (rows 9 and 10 are used as they are, the comment in the file is for reading). Nothing of
the queue or the two other jobs changes.

### A3 OB1, three things. Every pin in `PLC/OB1_Stage2_call.scl`

| # | Network | Do | ☐ |
|---|---|---|---|
| 1 | **new, after the drum job, before the mux: line job** | box `IDB_ScanLineJob`. `i_xStart` := `"DB_RobotJob".Man.xStart AND ("DB_RobotJob".Man.intJob = 3)` (compare box in series, as on the other two), `i_xAbort` := `Man.xAbort`, `i_xAuto` := `Man.xAuto AND "DB_RobotMove".Man.xEnable`, `i_intOverride` := `Man.intOverride`, `i_aoudtP` := `"DB_Poses".aoudtP`, `i_aoxValid` := `"DB_Poses".aoxValid`, `i_rSafeZ` := `"DB_Poses".rSafeZ`, then the twelve `"DB_Recipe".Line.*` pins in the order of the box: `i_rLength`, `i_rPitch`, `i_intCount`, `i_rLeadIn`, `i_rAngle`, `i_rZ`, `i_rFaceZ`, `i_rRotZOffset`, `i_xZigzag`, `i_xIndexAlong`, `i_xLiftBetween`, `i_rLiftZ`; `i_udtMoveState` := `"DB_RobotMove".State`, `i_intQueueCount` := `"DB_MoveQueue".intCount`, `i_xFromHere` := `"DB_Recipe".xFromHere`, `i_udtActual` := `"DB_Robot".Status.udtPoseBcs`, `i_xActualValid` := `"DB_Robot".Status.xReferenced AND .xCommOK`, `i_xNoTurn` := `"DB_Recipe".xNoTurn`, **`i_xArea` := `"DB_Recipe".Line.xArea`** (v0.2, the last input). Outputs: `o_udtOut` => `"DB_RobotJob".Line`, `o_xScanWindow` => `"DB_RobotJob".xScanWindow`, `o_intLineActive` => `"DB_RobotJob".intLineActive`, **`o_intLineCount` => `"DB_RobotJob".intLineCount`** (v0.2, the last output) | ☐ |
| 2 | mux (exists) | the new pin `i_udtLine` := `"DB_RobotJob".Line`. The other pins stay | ☐ |
| 3 | engine (exists) | the new pin **`i_rMinSpeed` := `"DB_RobotMove".Cfg.rMinSpeed`**, the last input of `IDB_RobotMove`. The other pins stay | ☐ |

Cross-reference after the compile: `"DB_RobotJob".Line`, `.xScanWindow`, `.intLineActive`, `.intLineCount` written by
the `IDB_ScanLineJob` call only. `"DB_RobotJob".Push.*` and `.State` by `FC_JobMux` only, as before.

### A4 Compile, download, two scans as before

| # | Do | ☐ |
|---|---|---|
| 1 | Compile → Software (rebuild all), 0 errors. Download | ☐ |
| 2 | `"DB_RobotMove".Cfg.intBlocksInPrg20` still **4** (M40 or the watch table). `"DB_RobotMove".Cfg.rMinSpeed` = **10** (M43 or the watch table; 0 keeps the old 60 s and a long line would end in engine error 12). If `DB_Recipe` was re-imported: type the cube and drum numbers and the switches again | ☐ |
| 3 | Job 1, AUTO on, START: **one cube scan exactly as on 2026-11-05.** Job 2, START: **one drum scan exactly as on 2026-11-09.** That proves the mux v0.2, the engine v0.8 with the field at 10, and that nothing else moved. Only then the line | ☐ |

### A5 Watch table "line", the rows to add

```
"DB_RobotJob".Man.intJob                    3 for the line. modify 3 if the screen field is not there yet
"DB_RobotJob".intJobActive                  follows intJob while no job runs
"DB_RobotJob".Line.udtState.intStep         0 10 20 30 90. 20 = still pushing, 30 = all pushed
"DB_RobotJob".Line.udtState.intErrId        job error of the line job
"DB_RobotJob".Line.udtState.intEngineErr    the engine number behind job error 10
"DB_RobotJob".Line.udtState.intMoveActive   the kind of move, table at the top
"DB_RobotJob".intLineActive                 the line k of the active move
"DB_RobotJob".Line.udtState.intMovesTotal   11 for the example of part D
"DB_RobotJob".Line.udtState.intMovesDone
"DB_RobotJob".xScanWindow                   on inside a patch, during a line move
"DB_MoveQueue".intCount                     stays below 8
"DB_RobotMove".State.intErrId               the engine error itself
"DB_RobotMove".State.uiMsgIdAtError         the robot message id at the error
"DB_RobotMove".State.intBlockActive         2 during a line and a descent, else 1
"DB_Recipe".Line.rLength                    300 in the example
"DB_Recipe".Line.rPitch                     100
"DB_Recipe".Line.intCount                   3
"DB_Recipe".Line.rLeadIn                    50
"DB_Recipe".Line.rAngle                     0
"DB_Recipe".Line.rZ                         600
"DB_Recipe".Line.rFaceZ                     0 = not checked
"DB_Recipe".Line.xZigzag                    true for the example
"DB_Recipe".Line.xIndexAlong                false
"DB_Recipe".Line.xLiftBetween               false
"DB_Recipe".Line.xArea                      false for E1 .. E8, true for E9 and E10
"DB_RobotJob".intLineCount                  the lines the job built, 3 for the example
"DB_RobotMove".Cfg.rMinSpeed                10
"DB_Recipe".xNoTurn                         true for the first run
"DB_Recipe".xFromHere                       false
"DB_Poses".aoxValid[9]                      start taught
"DB_Poses".aoxValid[10]                     corner 2 taught, E9 and E10 only
"DB_Poses".rSafeZ                           500 in the example
```

---

## B. The robot, nothing to do

Program 20 stays as it ran the drum on 2026-11-09: block 1 for the normal moves, block 2 for the slow ones, N80 writes
the echo of block 2 into REAL 71 (the drum's "down to P0" proved it), `Cfg.intBlocksInPrg20` = 4. The line job uses
nothing else. The speed of a line is the slow motion set of block 2 times the override; with the override at 10 the
ramps are below a millimetre and the lead-in of 50 is far more than enough (`SCAN_TYPES.md` 3.2).

---

## C. The screen, 40 minutes. Object numbers as in `HMI/SCREENS.html`, texts in `HMI/HMI_STAGE2.md` section 7

### C1 Automatic screen, the LINE box (new, next to the DRUM box)

| Nr | Object | Kind | Tag | Settings | ☐ |
|---|---|---|---|---|---|
| A59 | length | I/O field, Real | `"DB_Recipe".Line.rLength` | mm, 0 … 4000, 0 decimals. The patch | ☐ |
| A60 | pitch | I/O field, Real | `"DB_Recipe".Line.rPitch` | mm, −4000 … 4000. + = the next line to the left of the line direction, − = to the right | ☐ |
| A61 | lines | I/O field, Int | `"DB_Recipe".Line.intCount` | 1 … 50 | ☐ |
| A62 | lead-in | I/O field, Real | `"DB_Recipe".Line.rLeadIn` | mm, 0 … 500 | ☐ |
| A63 | angle | I/O field, Real | `"DB_Recipe".Line.rAngle` | deg, −180 … 180. 0 = +X | ☐ |
| A64 | scan height | I/O field, Real | `"DB_Recipe".Line.rZ` | mm, −10 … 1908 | ☐ |
| A65 | face Z | I/O field, Real | `"DB_Recipe".Line.rFaceZ` | mm, 0 … 1908. 0 = not checked | ☐ |
| A66 | RotZ offset | I/O field, Real | `"DB_Recipe".Line.rRotZOffset` | deg, −180 … 180. 0 the first time | ☐ |
| A67 | ZIGZAG | switch | `"DB_Recipe".Line.xZigzag` | retentive | ☐ |
| A68 | INDEX ALONG | switch | `"DB_Recipe".Line.xIndexAlong` | retentive | ☐ |
| A69 | LIFT BETWEEN | switch | `"DB_Recipe".Line.xLiftBetween` | retentive | ☐ |
| A70 | lift Z | I/O field, Real | `"DB_Recipe".Line.rLiftZ` | mm, 0 = safe Z | ☐ |
| A71 | start taught | lamp | `"DB_Poses".aoxValid[9]` | teaching: pose nr 9 in A3, TEACH A4. No new button | ☐ |
| A75 | AREA | switch | `"DB_Recipe".Line.xArea` | retentive. On = the area by pose 9 and pose 10 (v0.2) | ☐ |
| A76 | corner 2 taught | lamp | `"DB_Poses".aoxValid[10]` | teaching: pose nr 10 in A3, TEACH A4. Only with AREA | ☐ |
| A77 | lines of the job | output field, Int | `"DB_RobotJob".intLineCount` | next to A61. What the last START built | ☐ |

### C1a Robot move screen, one admin field

| Nr | Object | Kind | Tag | Settings | ☐ |
|---|---|---|---|---|---|
| M43 | min speed | I/O field, Real | `"DB_RobotMove".Cfg.rMinSpeed` | mm/s, 0 … 1000, admin user level, next to M40. 10 | ☐ |

### C2 JOB box, three additions

| Nr | Object | Kind | Tag | Settings | ☐ |
|---|---|---|---|---|---|
| A51 | job | exists | `"DB_RobotJob".Man.intJob` | range 1 … 3, text list `TL_Job` with 3 "scan line". Or a third button LINE that SetTagValue 3 | ☐ |
| A72 | active, line | output field, Int | `"DB_RobotJob".State.intMoveActive` | next to A25 and A58, text list `TL_MoveLine`. Visibility `"DB_RobotJob".intJobActive = 3` | ☐ |
| A73 | line | output field, Int | `"DB_RobotJob".intLineActive` | next to A72. Which line the active move belongs to | ☐ |
| A74 | scan window | lamp | `"DB_RobotJob".xScanWindow` | the bit for the sensor. Blinks on and off once per line | ☐ |

### C3 Text lists

| List | Entries | ☐ |
|---|---|---|
| `TL_Job` | 3 scan line | ☐ |
| `TL_MoveLine` (new) | 0 –, 1 to start, 2 above line 0, 3 down onto line 0, 4 scan line, 5 index to next line, 6 lift between, 7 travel, 8 down onto the line, 9 lift, 10 to start, 11 home | ☐ |
| `TL_JobErr` | 5 "a number of the recipe is not set" (was "drum radius not set"), **9 "scan height not above the face"** new | ☐ |
| discrete alarms | bit 17 "job: scan height not above the face", class error | ☐ |

### C4 Switch positions for the line

| Doing | Enable | AUTO | SCAN FROM HERE | NO TURN | ZIGZAG | INDEX ALONG | LIFT BETWEEN | job |
|---|---|---|---|---|---|---|---|---|
| teach pose 9 | on | off | off | any | – | – | – | – |
| first scan E1, serpentine | on | on | off | **on** | on | off | off | 3 |
| raster E2 | on | on | off | on | off | off | off | 3 |
| patch E3 | on | on | off | on | off | on | off | 3 |
| lift between E4 | on | on | off | on | on | off | on | 3 |
| turning lines E8, only after the drum's E6 | on | on | off | **off** | on or off | off | off | 3 |
| area E9, E10 | on | on | off | on | on | off | off | 3 |

AREA (A75) is **off** in every row but the last; E9 and E10 switch it on. PATH MODE (A49) does nothing for this job.
CLOCKWISE (A56) belongs to the drum.

---

## D. The numbers, with a worked example

| # | Number | How | Example | ☐ |
|---|---|---|---|---|
| D1 | **safe Z** `"DB_Poses".rSafeZ` | as for the cube and the drum, above everything that stands on the table | 500 | ☐ |
| D2 | **pose 9, the start** | jog the tool to the first point that is to be scanned, any height, the sensor pointing as it should (RotZ, RotY are taken from it). Poses screen: pose nr 9, TEACH. Lamp A71 on. Or SCAN FROM HERE on while the tool stands there | X 2500, Y 1500, RotZ 0, RotY 0 | ☐ |
| D3 | **scan height** A64 | jog to the height you want to scan at, read actual Z, type it | 600 | ☐ |
| D4 | **length** A59 | the part of the line that is to be scanned | 300 | ☐ |
| D5 | **pitch** A60 | start to start. Positive = the lines walk to the left of the line direction (angle 0: toward +Y), negative = to the right. Stitched coverage: a little less than the length; spot checks: more | 100 | ☐ |
| D6 | **lines** A61 | how many. The job refuses more than 5000 (50 in v0.1). Not used with AREA on, A77 then shows the computed number | 3 | ☐ |
| D7 | **lead-in** A62 | 50 the first time. Later from the speed trace: the distance the tool needs to reach its plateau, plus 10 mm (`SCAN_TYPES.md` 3.2, test T1) | 50 | ☐ |
| D8 | **angle** A63 | the direction of the lines. 0 = +X | 0 | ☐ |
| D9 | **face Z** A65 | 0 the first time (not checked). Later the Z of the face: the job then refuses a scan height at or below it | 0 | ☐ |
| D10 | **RotZ offset** A66 | 0. Only matters with NO TURN off (E8) | 0 | ☐ |
| D11 | **switches** | ZIGZAG on, INDEX ALONG off, LIFT BETWEEN off, lift Z 0, NO TURN on, SCAN FROM HERE off | | ☐ |
| D12 | **room** | every line from X − lead-in to X + length + lead-in and every Y of the lines inside the limits (X 0 … 4380, Y 0 … 4230). Else job error 4 with A72 = 4 and A73 = 0 or the last line | 2450 … 2850, 1500 … 1700 | ☐ |
| D13 | **AREA, pose 10** (E9, E10 only) | AREA A75 on. Jog the tool over the opposite corner of the area, any height, pose nr 10, TEACH. Lamp A76 on. The job then computes length and lines from pose 9 and pose 10; A59 and A61 are not used, the sign of A60 neither | E9: X 2800, Y 1700. E10: pose 9 at 100 / 100, pose 10 at 4280 / 4130 | ☐ |
| D14 | **min speed** M43 | 10 mm/s. The engine allows 60 s plus the way of the move divided by it: a 4180 mm line gets 60 + 418 s | 10 | ☐ |

**What the robot will do with the example** (serpentine, 3 lines, NO TURN on, pose 2 and home as taught), move by
move on A32 or in the watch table. Every point Z 600 except where written, RotZ 0 everywhere:

| # | Kind (A72) | Line (A73) | Target X / Y / Z | Block | Window |
|---|---|---|---|---|---|
| 1 | 1 to start | 0 | pose 2 | 1 | off |
| 2 | 2 above line 0 | 0 | 2450 / 1500 / **500** | 1 | off |
| 3 | 3 down onto line 0 | 0 | 2450 / 1500 / 600 | 2 | off |
| 4 | 4 scan line | 0 | 2850 / 1500 | 2 | **on from X 2500 to 2800** |
| 5 | 5 index | 1 | 2850 / 1600 | 1 | off |
| 6 | 4 scan line | 1 | 2450 / 1600 (backwards) | 2 | **on from X 2800 to 2500** |
| 7 | 5 index | 2 | 2450 / 1700 | 1 | off |
| 8 | 4 scan line | 2 | 2850 / 1700 | 2 | **on from X 2500 to 2800** |
| 9 | 9 lift | 2 | 2850 / 1700 / **500** | 1 | off |
| 10 | 10 to start | 0 | pose 2 | 1 | off |
| 11 | 11 home | 0 | pose 1 | 1 | off |

Moves total 11 (10 without home). The block field of the engine shows 1 1 2 2 1 2 1 2 1 1 1, the echo REAL 71 the
same. A74 blinks on three times, once per line.

**The same numbers with the other switches:**

| Switch change | What changes | Moves total |
|---|---|---|
| ZIGZAG off (raster) | line 1 runs 2450 → 2850 like line 0; the index moves are the diagonals 2850 / 1500 → 2450 / 1600 and 2850 / 1600 → 2450 / 1700 at normal speed; the lift at 2850 / 1700 | 11 |
| INDEX ALONG on, pitch 500 (patch) | patches at X 2500 … 2800, 3000 … 3300, 3500 … 3800, all at Y 1500; lines 2450 → 2850, 2950 → 3350, 3450 → 3850; the index moves 100 mm forward; the lift at 3850 / 1500 | 11 |
| LIFT BETWEEN on, lift Z 0 | between two lines: lift to Z 500 at the line end, travel at Z 500 to the next lead-in point, slow descent to 600 | 15 |
| SCAN FROM HERE on | no "to start" first, the list begins with "above line 0" over the actual X Y | 10 |
| AREA on, pose 10 at 2800 / 1700, pitch 100 or −100 | **exactly the 11 moves of the table above**: length 300 and 3 lines come from the two corners, A77 shows 3. Pose 10 at 2200 / 1300 instead: the lines run from 2550 to 2150 (axis turned 180) at Y 1500, 1400, 1300 | 11 |

---

## Start: what to press, key by key

The objects as in `HMI/SCREENS.html`. Before the first time: parts A, C and D done, the cube and the drum run once
after the download (A4.3).

| # | Press / type | Where | See | ☐ |
|---|---|---|---|---|
| S1 | Enable on, AUTO A12 **off** | | A46 "teach allowed" on | ☐ |
| S2 | jog the tool to the first point to scan, pose nr **9** in A3, TEACH A4 | POSES box | A71 "start taught" on. Pose 2 taught too, lamp at A40 | ☐ |
| S3 | the numbers of part D in the LINE box | LINE box | they stand, retentive | ☐ |
| S4 | job **3** in A51 | JOB box | A52 shows "scan line" | ☐ |
| S5 | switches: NO TURN A50 **on**, ZIGZAG A67 on, INDEX ALONG A68 off, LIFT BETWEEN A69 off, SCAN FROM HERE A38 off | CUBE and LINE box | | ☐ |
| S6 | override A13 = **10** | JOB box | | ☐ |
| S7 | AUTO A12 **on** | JOB box | A48 "automatic" within a second | ☐ |
| S8 | **START** A14, one press | JOB box | A24 moves total 11, A72 goes "to start", "above line 0", "down onto line 0", "scan line", "index", "scan line", "index", "scan line", "lift", "to start", "home"; A73 counts 0, 1, 2; A74 blinks once per line. A21 job error stays 0 | ☐ |
| S9 | it stops with an error | | read A21 job error, A26 engine error, A72 kind, A73 line, A30 block, A33 message id. ACK A47, fix the cause, START again from the start | ☐ |
| S10 | to stop it | | ABORT A15: the running move finishes, nothing follows. AUTO off does the same | ☐ |

---

## E. The test, in this order

| # | Do | Done when | ☐ |
|---|---|---|---|
| E1 | **the first scan, the example of part D.** Serpentine, 3 lines, NO TURN on | 11 moves, the points of the table, RotZ stays at the value of pose 9, A74 on three times. Compare X and Y on A32 after moves 3, 4, 6 and 8 | ☐ |
| E2 | **raster.** ZIGZAG off, START | the same lines all from 2450 to 2850, two diagonal moves back, 11 moves | ☐ |
| E3 | **patch.** ZIGZAG off, INDEX ALONG on, pitch 500, START | three patches along one line at Y 1500, X 2500 … 2800, 3000 … 3300, 3500 … 3800, the index moves 100 mm forward, 11 moves | ☐ |
| E4 | **lift between.** INDEX ALONG off, ZIGZAG on, LIFT BETWEEN on, lift Z 0, START | 15 moves, Z goes to 500 between the lines and comes down slowly onto each line | ☐ |
| E5 | **ABORT during a line** | the running line finishes to its lead-out point, queue 0, job idle, A74 off | ☐ |
| E6 | **SCAN FROM HERE.** AUTO off, jog to 2500 / 1500 at any height, SCAN FROM HERE on, AUTO on, START | 10 moves, the first is "above line 0" over the actual X Y, then the same as E1. Nothing stored in row 9 | ☐ |
| E7 | **the window bit against the trace** (for the sensor, later). TIA trace of `"DB_RobotJob".xScanWindow` with `"DB_Robot".Status.udtPoseBcs.rX` during E1 | the bit rises at X 2500 and falls at X 2800 (and the other way round on line 1), a bus cycle late, which the lead-in covers. The speed `"DB_Robot".Status.intSpeed` is on its plateau while the bit is on | ☐ |
| E8 | **turning lines, NO TURN off.** Only after RotZ −180 and +180 by hand (drum checklist E6). RotZ offset 0, raster first: ZIGZAG off, START. Then ZIGZAG on, START | raster: RotZ 0 on every line (angle 0 + offset 0, the same as pose 9 at RotZ 0, so nothing visible changes). Serpentine: RotZ 0 on lines 0 and 2, **+180 on line 1**, the half turn happens inside the index moves. If the robot refuses +180 (engine 21, message id in A33): tell me, the job then turns the other way, one line | ☐ |
| E9 | **AREA, the example.** Can run right after E1. AUTO off, jog to 2800 / 1700, pose nr 10, TEACH (A76 on). AREA A75 on, ZIGZAG on, NO TURN on, AUTO on, START | A77 = 3, A24 = 11, **the same 11 points as E1**. Then pose 10 at 2200 / 1300, START: lines from 2550 to 2150 at Y 1500, 1400, 1300, A77 = 3. Then pose 10 at 2500 / 1700 (straight across), START: job error 5, nothing moves | ☐ |
| E10 | **AREA, the whole table.** M43 = 10 first. Safe Z and the scan height above everything on the table, face Z 0. Pose 9 at 100 / 100, pose 10 at 4280 / 4130, pitch 20, lead-in 50, angle 0, AREA on, ZIGZAG on, NO TURN on, override 100, START. About 80 minutes; ABORT ends it after the running line | A77 = 202, A24 = 407 with home, line 0 from X 50 to 4330 at Y 100, the last at Y 4120, Z the scan height everywhere, no engine error 12 on the long lines. A74 on 202 times. Compare the first three points on A32 | ☐ |

The numbers to read when it stops: job error (A21), engine error (A26 or A29), kind (A72), line (A73), block (A30),
robot message id (A33). Those six go into `TEST/ERROR_LOG.md`.

| Job error | Means | Fix |
|---|---|---|
| 1 | pose 2 not taught, pose 9 not taught without SCAN FROM HERE, or pose 10 not taught with AREA on | teach it, A40 / A71 / A76 lamps |
| 2 | safe Z 0 | D1 |
| 3 | engine busy or in error at START | ACK, then START |
| 4 | a point outside the limits, A72 the kind (4 line, 2 above, 6 lift, 1 / 10 start, 11 home), A73 the line (0 or the last) | D12: move the start, shorten the lines, fewer lines, or the pitch the other way (sign). With AREA: corner 2 nearer, or a smaller lead-in |
| 5 | length 0, pitch 0, lines outside 1 … 5000, or a negative lead-in. With AREA: corner 2 straight across from corner 1 (no length), or more than 5000 lines at this pitch | D4 … D7, D13 |
| 6 | SCAN FROM HERE on, robot not referenced or comm not ok | reference, or the switch off and pose 9 |
| 9 | face Z set and the scan height at or below it | D3, D9. Smaller Z is up: the scan height must be the smaller number |
| 10, engine 1 | a block above `intBlocksInPrg20` | cannot happen with 4; the job uses 1 and 2 |
| 10, engine 2 | X Y Z outside at the engine | cannot come from the job, every point was checked; by hand only |
| 10, engine 10 at a slow move | block 2 ran, REAL 71 is not 2 | N80 of program 20, the drum checklist B1. The drum proved it on 11-09 |
| 10, engine 12 | a move took longer than its time limit | M43 min speed is 0: the limit is the old 60 s, and a long line at override 10 (4000 mm at 20 mm/s is 200 s) does not fit. M43 = 10, then the limit is 60 s plus the way / 10 mm/s. Or a higher override |
| 10, engine 21 | robot fault during a move | A33 message id. With NO TURN off: the +180 of E8 |
| queue count climbs to 50, `xPushRefused` | cannot happen: the job pushes only while the count is below 8 | tell me if you see it |

---

## F. Later, not today

- The lead-in from the speed trace (test T1 of `SCAN_TYPES.md` section 6): one slow move of 300 mm at override 100,
  trace speed against X, the distance to the plateau plus 10 mm goes into A62.
- The sensor: who reads `"DB_RobotJob".xScanWindow` (a trigger output, a measuring card). The bit exists, the reader
  is open.
- `tDwell` and `xNoLift` of the cube job (`SCAN_TYPES.md` 7.1, 7.2): two fields at the end of `DB_Recipe`, a few lines
  in `FB_ScanCubeJob`. Not built.
- A side face (the approach horizontal instead of Z): the same job with the approach vector turned. Not built.
