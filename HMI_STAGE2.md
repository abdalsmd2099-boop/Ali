# HMI – stage 2 screens "Poses" and "Scan cube"

2026-10-28. WinCC Unified. Adds to the stage 1 screen. All tags in `DB_Poses`, `DB_RobotJob`, `DB_MoveQueue`, `DB_RobotMove`, `DB_Robot`.

## 1. Screen "Poses"

```
+---------------------------------------------------------------------------------+
|  nr  name        valid    X        Y        Z        RotZ     RotY              |
|   1  home        [o]    [....]   [....]   [....]   [....]   [....]              |
|   2  start       [o]    ...                                                     |
|   3  approach    [o]                                                            |
|   4  corner 1    [o]                                                            |
|   5  corner 2    [o]                                                            |
|   6  corner 3    [o]                                                            |
|   7  corner 4    [o]                                                            |
|   8  free        [o]                                                            |
|   9  free        [o]                                                            |
|  10  free        [o]                                                            |
|                                                                                 |
|  safe Z  [ 1500 ] mm                                                            |
|                                                                                 |
|  pose nr [ 2 ]   [ TEACH actual pose ]   [ CLEAR ]    taught (o)  refused (o)   |
|  last taught [ 2 ]      actual pose  X [....] Y [....] Z [....] RotZ [....] RotY [....] |
|  [ COPY pose nr to target ]   (optional, hmi script)                            |
+---------------------------------------------------------------------------------+
```

| Element | Tag | Settings |
|---|---|---|
| row n, valid lamp | `"DB_Poses".aoxValid[n]` | |
| row n, X … RotY | `"DB_Poses".aoudtP[n].rX` … `.rRotY` | I/O fields, 0 decimals, same limits as the stage 1 target fields. Typing is allowed, teaching is better |
| safe Z | `"DB_Poses".rSafeZ` | I/O field, limits −10 … 1908. Must be above the cube |
| pose nr | `"DB_RobotJob".Teach.intPoseNr` | I/O field, 1 … 50 |
| TEACH | `"DB_RobotJob".Teach.xTeach` | momentary |
| CLEAR | `"DB_RobotJob".Teach.xClear` | momentary. Marks the pose as not taught, numbers stay |
| taught, refused lamps | `"DB_RobotJob".Teach.xTaught`, `.xRefused` | one-cycle pulses, may flash only |
| last taught | `"DB_RobotJob".Teach.intLastTaught` | |
| actual pose | `"DB_Robot".Status.udtPoseBcs.rX` … | output fields |
| COPY to target | script on Press: read `"DB_Poses".aoudtP[n].*` and write `"DB_RobotMove".Man.udtReq.udtPose.*` | optional. Then MOVE on the stage 1 screen goes there |

Teaching is refused when: auto is on, the engine is busy, the robot is not referenced, comm is not OK, or the number is outside 1..50.

**How to teach the cube (poses 4 to 7):** stand at each corner with the tool facing the side that starts at that corner. Corner 1 faces side 1 (corner 1 → corner 2), corner 2 faces side 2, and so on. The job builds the sides from that.

## 2. Screen "Scan cube"

```
+-------------------------------------------+-------------------------------------------+
|  [ AUTO ]  (switch)                        |  JOB                                      |
|  override [ 10 ] %                         |  running  done  error  [ 0 - no error ]   |
|  [  START  ]   [  ABORT  ]                 |  step [ 30 - run ]                        |
|                                            |  moves  [ 5 ] / [ 14 ]   active [ 6 ]     |
|  queue  [ 9 ] waiting   full (o)           |  engine error [ 0 ]                       |
|                                            |                                            |
|  MOVE ENGINE (same as stage 1 screen)      |  ROBOT (same as stage 1 screen)           |
|  busy done error  step  run nr  block       |  lamps, actual pose                       |
+-------------------------------------------+-------------------------------------------+
|  ALARMS  (alarm view, discrete alarms on "DB_RobotJob".aoxAlarm)                        |
+-----------------------------------------------------------------------------------------+
```

| Element | Tag | Settings |
|---|---|---|
| AUTO | `"DB_RobotJob".Man.xAuto` | switch. On = the queue drives the engine, the stage 1 MOVE button is dead. Off while a job runs = abort |
| override | `"DB_RobotJob".Man.intOverride` | 1 … 100, start value 10 |
| START | `"DB_RobotJob".Man.xStart` | momentary |
| ABORT | `"DB_RobotJob".Man.xAbort` | momentary. Also clears a job error |
| queue waiting, full | `"DB_MoveQueue".intCount`, `.xFull` | |
| job lamps | `"DB_RobotJob".State.xRunning`, `.xDone`, `.xError` | |
| job error | `"DB_RobotJob".State.intErrId` | text list `TL_JobErr` |
| job step | `"DB_RobotJob".State.intStep` | text list `TL_JobStep` |
| moves done / total, active | `"DB_RobotJob".State.intMovesDone`, `.intMovesTotal`, `.intMoveActive` | text list `TL_Move` on active |
| engine error | `"DB_RobotJob".State.intEngineErr` | text list `TL_Err` from stage 1 |

The stage 1 enable switch `"DB_RobotMove".Man.xEnable` must be on for both hand and auto. Show it on this screen too.

## 3. Text lists

**TL_JobErr**

| Value | Text |
|---|---|
| 0 | no error |
| 1 | a pose of the job is not taught (cube: a corner; drum: 8; line: 2, 9, or 10 with AREA) |
| 2 | safe Z not set |
| 3 | engine busy or in error at start |
| 4 | a pose of the job is outside the limits (see active) |
| 5 | a number of the recipe is not set: drum radius; line length, pitch or count; with AREA corner 2 gives no length or too many lines (2026-10-31, widened 2026-11-09) |
| 6 | scan from here not possible: robot not referenced, or no length and width |
| 9 | scan height not above the face (line job, 2026-11-09) |
| 10 | engine error during the job (see engine error) |

**TL_JobStep**

| Value | Text |
|---|---|
| 0 | idle |
| 10 | build list |
| 20 | push |
| 30 | run |
| 90 | error |

**TL_Move** (the 14 moves of the scan)

| Value | Text |
|---|---|
| 0 | – |
| 1 | to start |
| 2 | approach |
| 3 | to corner 1 |
| 4 | side 1 |
| 5 | turn at corner 2 |
| 6 | side 2 |
| 7 | turn at corner 3 |
| 8 | side 3 |
| 9 | turn at corner 4 |
| 10 | side 4 |
| 11 | turn at corner 1 |
| 12 | lift |
| 13 | to start |
| 14 | home |

## 4. Discrete alarms on `"DB_RobotJob".aoxAlarm`

One discrete alarm per bit. Bits 1 to 22 use the texts of `TL_Err` (stage 1) with the same number. The rest:

| Bit | Class | Text |
|---|---|---|
| 9 | error | job: drum radius not set (2026-10-31) |
| 15 | error | job: scan from here not possible (2026-11-02) |
| 16 | error | engine: path longer than 7 poses (path mode, 6.5) |
| 17 | error | job: scan height not above the face (line job, 2026-11-09) |
| 23 | error | job: a cube pose is not taught |
| 24 | error | job: safe Z not set |
| 25 | error | job: engine busy or in error at start |
| 26 | error | job: a pose is outside the limits |
| 27 | error | job: engine error during the job |
| 28 | error | robot fault, see message ID |
| 29 | error | robot communication lost |
| 30 | warning | RobotMonitor has access, PLC cannot move |
| 31 | error | robot outside the workspace |
| 32 | warning | robot warning, see message ID |

Bits 18 and 19 are not used.

## 5. Scan drum, on the same automatic screen (2026-10-31)

The job selector and the drum settings. Everything else of section 2 (AUTO, override, START, ABORT, lamps, numbers) is shared: `"DB_RobotJob".State` shows whichever job is active.

| Element | Tag | Settings |
|---|---|---|
| job | `"DB_RobotJob".Man.intJob` | I/O field or two buttons: 1 = scan cube, 2 = scan drum. Taken over when no job runs |
| active job | `"DB_RobotJob".intJobActive` | output, text list `TL_Job`: 1 scan cube, 2 scan drum |
| pose row 8 | `"DB_Poses".aoudtP[8]`, `aoxValid[8]` | name "drum center". Teach it with the tool above the drum axis, any height |
| scan height | `"DB_Recipe".Drum.rZ` | mm, −10 … 1908 |
| radius | `"DB_Recipe".Drum.rRadius` | mm, 0 … 3000. Drum radius plus the sensor distance. 0 = job error 5 |
| start angle | `"DB_Recipe".Drum.rStartAngle` | **not on the screen since 2026-11-02.** The circle starts at the point where the tool, looking at the axis, stands at RotZ −180 (counter-clockwise) or +180 (clockwise). From there it turns 90° per quarter without a wrap and ends at +180 or −180. The job computes that point from the RotZ offset |
| RotZ offset | `"DB_Recipe".Drum.rRotZOffset` | deg, −180 … 180. See below |
| clockwise | `"DB_Recipe".Drum.xClockwise` | switch. Off = block 3 counter-clockwise, on = block 4 clockwise |

**Finding the RotZ offset once.** Jog the tool to the +X side of the drum, at the scan height, and turn RotZ until the sensor looks at the drum axis, that is along −X. Read RotZ on the screen. Offset = RotZ − 180, wrapped into −180 … 180. Type it. From then on the job turns the tool toward the center at every point.

**No turn (2026-11-06, drum job v0.5).** The NO TURN switch of the cube box, `"DB_Recipe".xNoTurn`, holds for the drum too: on = every point of the circle keeps the RotZ of pose 8 (or of the actual pose with SCAN FROM HERE), the tool does not turn toward the axis, −180 and +180 are never driven. The offset still places P0, it no longer sets an orientation. Same 10 moves, same texts. For the first scan, and for a sensor that does not care which side it faces.

**What the job drives** (`TL_MoveDrum`, the text list for `intMoveActive` when the active job is 2):

| Value | Text |
|---|---|
| 1 | to start |
| 2 | above P0 |
| 3 | down to P0 |
| 4 | arc to P1 |
| 5 | arc to P2 |
| 6 | arc to P3 |
| 7 | arc to P0 |
| 8 | lift |
| 9 | to start |
| 10 | home |

**TL_JobErr** gets one more line: 5 = drum radius not set.

Before the first drum scan: `"DB_RobotMove".Cfg.intBlocksInPrg20` = 4 and program 20 with lines 11–20 of `PROGRAM20.md` section 3, else move 4 ends with engine error 1.

## 6. The cube by its dimensions (2026-11-02)

`FB_ScanCubeJob` v0.2 computes corners 2, 3, 4 and the approach from corner 1 when length and width are both greater than 0. The tags live in `DB_Recipe`, a DB of its own, so `DB_Poses` in the PLC is never touched and the taught poses stay. Everything of sections 1 and 2 keeps working: pose nr + TEACH for any row, START, ABORT, the lamps. This section only adds a box that makes the cube a three-button job. No PLC change is needed for the buttons, they are HMI scripts on tags that exist.

### 6.1 Box "CUBE" on the Automatic screen

```
+------------------------------------------------------------------------------+
|  CUBE                                                                        |
|  length [  600 ] mm    width [  400 ] mm    angle [   0 ] deg                |
|  corners: computed from corner 1          safe Z [ 1500 ] mm                 |
|                                                                              |
|  [ TEACH HOME ] (o)    [ TEACH START ] (o)    [ TEACH CORNER 1 ] (o)         |
|  [ GO TO START ]       [ GO TO CORNER 1 ]        hand only, AUTO off         |
|  [ SCAN FROM HERE ] (switch)   on: START takes the actual pose as corner 1   |
+------------------------------------------------------------------------------+
```

| Element | Tag or script | Settings |
|---|---|---|
| length | `"DB_Recipe".Cube.rLength` | I/O field, mm, 0 … 4380. Side 1, from corner 1 to corner 2. 0 = the taught corners 5, 6, 7 are used |
| width | `"DB_Recipe".Cube.rWidth` | I/O field, mm, 0 … 4230. Side 2 |
| angle | `"DB_Recipe".Cube.rAngle` | I/O field, deg, −180 … 180. Direction of side 1 in the base system, 0 = +X |
| corners text | two texts on the same spot, visibility by the tag `"DB_RobotHand".Lamp.xComputed`: "computed from corner 1" when 1, "taught, rows 5 to 7" when 0 | no condition to write |
| safe Z | `"DB_Poses".rSafeZ` | as section 1 |
| TEACH HOME | `"DB_RobotHand".Btn.xTeachHome` | momentary button, press 1 release 0. `FB_CubeButtons` does the rest. Enabled by `"DB_RobotHand".Lamp.xTeachAllowed` |
| TEACH START | `"DB_RobotHand".Btn.xTeachStart` | momentary |
| TEACH CORNER 1 | `"DB_RobotHand".Btn.xTeachC1` | momentary. Tool at the corner, facing side 1, RotZ −180 |
| lamp next to each teach button | `"DB_Poses".aoxValid[1]`, `[2]`, `[4]` | green = taught. The three lamps green = the cube can run |
| ready | `"DB_RobotHand".Lamp.xCubeReady` | the three in one |
| GO TO START | `"DB_RobotHand".Btn.xGoStart` | momentary. `FB_CubeButtons` copies row 2 into the target, block 1, and starts. Enabled by `"DB_RobotHand".Lamp.xGoAllowed` |
| GO TO CORNER 1 | `"DB_RobotHand".Btn.xGoC1` | momentary |
| SCAN FROM HERE | `"DB_Recipe".xFromHere` | switch, one for both jobs. On: START takes the actual pose as corner 1 (cube) or as the drum center (drum) and begins with side 1 / above P0. Needs length and width for the cube, and a referenced robot, else job error 6. Off: row 4 / row 8 as taught. Retentive, stays as set |

No scripts (2026-11-03): every button is one bit in `"DB_RobotHand".Btn`, and `FB_CubeButtons` in OB1, before the teach network, does the rest. It is the one writer of `"DB_RobotJob".Teach.intPoseNr`, `.xTeach`, `.xClear` from now on, so the pose nr field, TEACH and CLEAR of section 1 move to `"DB_RobotHand".Btn.intPoseNr`, `.xTeachNr`, `.xClearNr`. How to build every object in WinCC Unified, recipe by recipe: `HMI_CUBE_HOWTO.md`.

### 6.2 Box "JOB", two additions

| Element | Tag or script | Settings |
|---|---|---|
| ACK | `"DB_RobotHand".Btn.xAck` | momentary. `FB_CubeButtons` pulses the engine reset and the job abort together. RESET and ABORT may stay next to it |
| robot mode | lamp "automatic" on `"DB_Robot".Status.xAutomatic`, lamp "manual" on NOT the same | after `FB_RobotMode` the old automatic switch becomes these two lamps |

### 6.3 Which corner is corner 1

With angle 0: the corner with the smallest X and the smallest Y. Side 1 runs from there along +X, side 2 along +Y, the scan goes around the cube counter-clockwise seen from above. The computed corners take Z and RotY of corner 1. RotZ turns +90° at every corner (2026-11-02): corner 1 at −180, corner 2 at −90, corner 3 at 0, corner 4 at +90, back at corner 1 +180, and the lift keeps +180. So **corner 1 is taught, or approached with "scan from here", with RotZ −180.** Any other RotZ makes the last turn exceed +180, and the job stops with error 4 at move 11 before anything moves. Row 4 of the pose table is best named "corner 1 (reference)".

### 6.4 How the operator uses it

**Setup of a cube, once.** TEACH HOME, TEACH START, TEACH CORNER 1 with the tool at the corner. Type length, width, angle, safe Z. Pose 3 is not needed, the approach is corner 1 at safe Z. GO TO CORNER 1 to check. AUTO on, START.

**A moved cube of the same size.** Jog the tool to its corner 1, TEACH CORNER 1, START. The three numbers are retentive and stay. A rotated cube: type the new angle too.

**A cube of another size.** Type the new length and width. Nothing to teach when corner 1 is the same.

**Back to the taught corners.** Length 0, then rows 5 to 7 must be taught as in section 1.

**Scan from here.** Jog the sensor to the corner of the cube, or above the drum axis, SCAN FROM HERE on, AUTO on, START. Nothing is taught, nothing is stored. A repeat needs the switch again from the same place, or TEACH CORNER 1 / TEACH 8 the normal way. Switch it off to go back to the taught rows.

Later a recipe table fills the three fields by a name.

### 6.5 Path mode: one switch, four text lines (2026-11-02, one folder since 2026-11-04)

Only with part 7 of `AUTOMATIC_TODAY.md`. Off, everything runs exactly as in 6.1 to 6.4.

| Element | Tag | Settings |
|---|---|---|
| PATH MODE | `"DB_Recipe".xPathMode` | switch in the CUBE box next to SCAN FROM HERE, both jobs. Off: one move per point, blocks 1 to 4, the proven way. On: the rectangle is one program run on block 5, the circle one run on block 6 or 7 (since 2026-11-06 one circle line, center + 360°, no stop; the tool keeps the RotZ of pose 8). Needs `Cfg.intBlocksInPrg20` = 7, the blocks in program 20, and for the drum the engine v0.7 |

Text lists, the lines the path mode adds: `TL_Block` 5 "rectangle, 7 lin", 6 "circle ccw, one run", 7 "circle cw, one run" (the block field of the Robot move screen shows them; 6 and 7 can be typed by hand for the full-circle test with the angle field M42); `TL_Move` 15 "rectangle, sides 1 to 4"; `TL_MoveDrum` 11 "circle, one run"; `TL_Err` 16 "path longer than 7 poses"; discrete alarm bit 16, class error, "engine: path longer than 7 poses" (the alarm block needs no change, engine errors land on the bit of their number).

What the operator sees in path mode. Cube: moves total 8 (7 without home, 5 with scan from here), the active text goes "to start", "approach", "to corner 1", then "rectangle, sides 1 to 4" for the whole run, then "turn at corner 1", "lift", "to start", "home". Drum: moves total 7, "circle, one run" during the run, no stop on the circle, the tool keeps the RotZ of pose 8. A fault during the rectangle: engine error with its number, job error 10, the active text stays on "rectangle", ACK, START again. ABORT during the rectangle: the whole rectangle finishes, then the robot stops. Nothing else changes in the hand of the operator: TEACH HOME, TEACH START, TEACH CORNER 1 or SCAN FROM HERE, the three numbers, AUTO, START. The switch is a setter's choice.

## 7. Scan line: serpentine, raster, patch (2026-11-09, `FB_ScanLineJob` v0.1; v0.2 the same day: AREA mode, no line limit, job 3)

One job for three patterns, on the same automatic screen. The job drives n straight lines at one height; every line is one slow move (block 2) from a lead-in point to a lead-out point, the moves between the lines are normal moves (block 1). Two switches make the pattern: ZIGZAG (every odd line runs backwards = serpentine; off = raster, every line the same way, the move back is the long diagonal) and INDEX ALONG (the next line lies further along the same line = the patch scan; off = beside it). LIFT BETWEEN puts a lift, a travel and a slow descent between two lines. Nothing new in program 20, `Cfg.intBlocksInPrg20` stays 4. Everything of section 2 (AUTO, override, START, ABORT, lamps, numbers) is shared, `"DB_RobotJob".State` shows the active job.

v0.2 (the same day, "the whole table"): the job no longer keeps a list of its moves, it computes every move when it pushes it (`FC_LineMove`), so the limit of 50 lines is gone (up to 5000). A fourth switch **AREA** (A75) gives the area by two taught corners, pose 9 and pose 10, instead of length and lines (7.5). And the engine v0.8 gets one admin field on the Robot move screen, **min speed** M43: the time limit of a move grows with its length, so a 4 m line at a low override no longer ends in engine error 12 (7.6).

### 7.1 Box "LINE" on the Automatic screen

| Nr | Element | Tag | Settings |
|---|---|---|---|
| A59 | length | `"DB_Recipe".Line.rLength` | I/O field, Real, mm, 0 … 4000, 0 decimals. The patch, the scanned part of every line. 0 = job error 5 |
| A60 | pitch | `"DB_Recipe".Line.rPitch` | I/O field, Real, mm, −4000 … 4000. Start to start between two lines. **The sign is the side:** + = the next line lies to the left of the line direction, − = to the right. 0 = job error 5 |
| A61 | lines | `"DB_Recipe".Line.intCount` | I/O field, Int, 1 … 5000 (50 in v0.1). Outside = job error 5. Not used with AREA on, A77 shows the computed number |
| A62 | lead-in | `"DB_Recipe".Line.rLeadIn` | I/O field, Real, mm, 0 … 500. The line begins this much before the patch and ends this much after it, so the ramps lie outside the patch. 50 for the first tests |
| A63 | angle | `"DB_Recipe".Line.rAngle` | I/O field, Real, deg, −180 … 180. Direction of the lines, 0 = +X, 90 = +Y |
| A64 | scan height | `"DB_Recipe".Line.rZ` | I/O field, Real, mm, −10 … 1908 |
| A65 | face Z | `"DB_Recipe".Line.rFaceZ` | I/O field, Real, mm, 0 … 1908. 0 = not checked. Smaller Z is up: a scan height at or below it is job error 9 |
| A66 | RotZ offset | `"DB_Recipe".Line.rRotZOffset` | I/O field, Real, deg, −180 … 180. The tool's RotZ when it looks along +X, as A55. Used only with NO TURN off |
| A67 | ZIGZAG | `"DB_Recipe".Line.xZigzag` | switch. Off = raster, on = serpentine. Retentive |
| A68 | INDEX ALONG | `"DB_Recipe".Line.xIndexAlong` | switch. Off = the next line beside this one. On = further along the same line, the patch scan (pitch larger than length + 2 × lead-in). Retentive |
| A69 | LIFT BETWEEN | `"DB_Recipe".Line.xLiftBetween` | switch. On = lift, travel, down between two lines. Retentive |
| A70 | lift Z | `"DB_Recipe".Line.rLiftZ` | I/O field, Real, mm, −10 … 1908. The height of the travel with LIFT BETWEEN. 0 = safe Z |
| A71 | start taught | `"DB_Poses".aoxValid[9]` | lamp. Teach: pose nr 9 in A3, TEACH A4, the tool at the first point to scan, any height. Or SCAN FROM HERE (A38) at that point |
| A72 | active, line | `"DB_RobotJob".State.intMoveActive` | output field, Int, text list `TL_MoveLine`, JOB box next to A25 and A58. Visibility `"DB_RobotJob".intJobActive = 3` |
| A73 | line | `"DB_RobotJob".intLineActive` | output field, Int, JOB box next to A72: the line k of the active move, 0 … lines − 1. At job error 4 it is the bad line |
| A74 | scan window | `"DB_RobotJob".xScanWindow` | lamp, JOB box. On while the tool is inside a patch during a line move. This is the bit the sensor reads later |
| A75 | AREA | `"DB_Recipe".Line.xArea` | switch, LINE box (v0.2). On = the area by two corners, pose 9 and pose 10, see 7.5. Length (A59) and lines (A61) are then not used, the sign of the pitch neither. Retentive |
| A76 | corner 2 taught | `"DB_Poses".aoxValid[10]` | lamp, LINE box (v0.2). Teach: pose nr 10 in A3, TEACH A4, the tool over the opposite corner of the area, any height. Only with AREA |
| A77 | lines of the job | `"DB_RobotJob".intLineCount` | output field, Int, LINE box next to A61 (v0.2). What the job built at the last START: A61 as typed, or the computed number with AREA |

A51 (job) gets the range 1 … 3, `TL_Job` the line 3 = scan line. NO TURN (A50), SCAN FROM HERE (A38) and the override are shared with the cube and the drum. PATH MODE (A49) does nothing for this job.

### 7.2 What the job drives (`TL_MoveLine`, the text list for `intMoveActive` when the active job is 3)

The number is the kind of move, A73 says which line. Moves total = 3 + n + (n − 1) (or 3 × (n − 1) with LIFT BETWEEN) + 2, plus 1 with home, minus 1 with SCAN FROM HERE.

| Value | Text | Block |
|---|---|---|
| 0 | – | |
| 1 | to start | 1 |
| 2 | above line 0 | 1 |
| 3 | down onto line 0 | 2 |
| 4 | scan line | 2 |
| 5 | index to next line | 1 |
| 6 | lift between | 1 |
| 7 | travel | 1 |
| 8 | down onto the line | 2 |
| 9 | lift | 1 |
| 10 | to start | 1 |
| 11 | home | 1 |

### 7.3 RotZ (SCAN_TYPES.md 7.5 in the critique folder)

NO TURN on: every point keeps the RotZ of pose 9 (or of the actual pose with SCAN FROM HERE). The proven way, the first run goes like this. NO TURN off: the tool looks along the line, RotZ = angle + RotZ offset; in the serpentine the odd lines are turned 180 the way that stays inside −180 … 180, and the turn rides on the index move, no extra move. The turn itself is not proven on this machine yet (the cube and the drum ran with NO TURN on): the hand test E6 of the drum checklist comes first.

### 7.4 Job errors of this job

`TL_JobErr`: 1 start (pose 2), line start (pose 9) or, with AREA, corner 2 (pose 10) not taught; 2 safe Z not set; 3 engine busy or in error; 4 a pose outside the limits, A72 the kind, A73 the line (v0.2: the check runs on the four outer line ends at every height the job uses, so A73 is 0 or the last line); 5 length 0, pitch 0, lines outside 1 … 5000 or lead-in negative, with AREA also: corner 2 gives no length along the angle, or more than 5000 lines at this pitch; 6 scan from here not possible; **9 scan height not above the face** (alarm bit 17); 10 engine error, see A26.

### 7.5 AREA: the area by two corners (v0.2, 2026-11-09)

Switch A75 on. Pose 9 is corner 1 of the area, as before the start of line 0. Pose 10 is the opposite corner. The angle (A63) sets the axis of the lines. The job splits the way from corner 1 to corner 2 into the part along the axis and the part across it: the part along is the length of every line, the part across divided by the pitch (A60, only its size counts) is the number of lines minus one. The lines walk toward corner 2, so the sign of the pitch is not needed. If corner 2 lies behind corner 1 along the axis, the axis is turned by 180 and the lines run from corner 1 toward corner 2 anyway. The last line lies at most one pitch before corner 2; a corner 2 that lies straight across from corner 1 gives no length and job error 5. A77 shows the computed number of lines after START.

With INDEX ALONG the length stays the typed patch length (A59) and the patches lie along the axis from corner 1 up to corner 2, one pitch apart. With SCAN FROM HERE the actual pose is corner 1 and pose 10 corner 2. ZIGZAG, LIFT BETWEEN, NO TURN and the lead-in work as without AREA.

The whole table at a pitch of 20: corner 1 at 100 / 100, corner 2 at 4280 / 4130, angle 0, lead-in 50, AREA on. That is 202 lines of 4180 mm, 407 moves with home, about 930 m of travel. At 200 mm/s and override 100 close to 80 minutes plus 202 stops. Set the engine's min speed (7.6) first, else the first line ends in engine error 12.

### 7.6 The engine's min speed, Robot move screen (v0.8, 2026-11-09)

| Nr | Element | Tag | Settings |
|---|---|---|---|
| M43 | min speed | `"DB_RobotMove".Cfg.rMinSpeed` | I/O field, Real, mm/s, 0 … 1000, admin user level, next to M40. The engine allows 60 s (300 s for a path or a circle) **plus the way of the move divided by this speed**. 0 = the fixed limits as before. 10 is a safe number: a 4380 mm line then has 60 + 438 s. Retentive, but a download of `DB_RobotMove` sets it back to 0 |

Checklist with the worked example and the expected points: `TEST/CHECKLIST_LINE_AUTO.md`.
