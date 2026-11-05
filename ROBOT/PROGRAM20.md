# Program 20 – the one robot program, complete

2026-10-28, rewritten 2026-10-30: the complete program with the two linear blocks and the two circle blocks, line by line as you build it in RobotMonitor, and how the drum scan uses the circle blocks. Every fact about the blocks comes from the MOVIKIT Robotics manual 33956901 (05/2026), chapters 6.3.2, 6.16, 11.9, 11.10, 11.12.

The PLC side never changes when a block is added: `Cfg.intBlocksInPrg20` + 1 and one text in the HMI block list.

## 0. Before you build: three checks in MOVISUITE and RobotMonitor

| # | Check | Where | If not |
|---|---|---|---|
| 1 | **Circle is licensed and activated.** Circular interpolation must be activated in the configuration of the software module, and the license "MOVIKIT Robotics add-on Circle" must be on the MOVI-C CONTROLLER (manual 6.16.1). Quick test: open "New Block" in RobotMonitor, `CIRC_VAR` must be in the list | MOVISUITE, software module configuration | build lines 1–10 and 21–22 only (blocks 1 and 2), `Cfg` = 2, tell me |
| 2 | **Two motion sets exist**, one normal and one slow, under "Sets of motion parameters" (manual 6.4). Write their numbers here: normal = ___, slow = ___ | MOVISUITE | create them, slow = about 20 % of normal |
| 3 | **The variables exist**: `POSE[51]`, `REAL[51]` to `REAL[54]`, `REAL[71]`, `REAL[72]`, `BOOL[51]` to `BOOL[58]` are visible in the RobotMonitor variable view and change when the PLC writes them | RobotMonitor, variables | the process data mapping of module 2 in MOVISUITE is wrong, `OB30_NW3_Module2_FB_Movikitrobotics.scl` has the word numbers |

## 1. The contract (GOALS.md 5.1, unchanged)

| Item | Rule |
|---|---|
| Block n | gated by `BOOL[50 + n]`, n = 1..8. One motion command per block |
| Target pose | `POSE[51]`, written by the PLC before start (words 114–118). For a circle: the **end point of the arc** |
| Block parameters | `REAL[51..53]` (words 119–121). Circle: `REAL[51]` = **radius in mm** |
| Run number | `REAL[54]` (word 122), written by the PLC before start |
| Echo | each block writes its number into `REAL[71]`. The last line copies `REAL[54]` into `REAL[72]` |

What the PLC writes before it sets the start bit, per block:

| Block | `POSE[51]` | `REAL[51]` | gate |
|---|---|---|---|
| 1 linear, normal speed | target | – | `BOOL[51]` |
| 2 linear, slow | target | – | `BOOL[52]` |
| 3 circle counter-clockwise | end point of the arc, same Z as now | radius | `BOOL[53]` |
| 4 circle clockwise | end point of the arc, same Z as now | radius | `BOOL[54]` |

`FB_RobotMove` does all of this from one request: `udtPose` → `POSE[51]`, `aorPar[1]` → `REAL[51]`, `intBlock` → the gate.

## 2. How a program is built in RobotMonitor (manual 11.2, 11.12)

There is no text editor. Every line is a block: select the line above, pick a value in the "New Block" list, click [Add], then fill the fields of the new line. `IF` and `END_IF` come as one pair, you drag the start and the end to the right lines (manual 11.12.1). The program must end with an `END_PROG` block (11.12.5). Save with the save icon, else the program is gone at the next power cycle (11.3).

At every program start the robot establishes the **basic state** (manual 6.3.2): base coordinate system, absolute positioning, linear interpolation, blending 0 (exact stop), plane XY for circles, tangential control off, and the target is preset to the current position. So nothing from the last run stays, and every block must set its own motion set.

## 3. The complete program, line by line

22 lines. Build them in this order. `Cfg.intBlocksInPrg20` = 4 when all four blocks are in, 2 when only lines 1–10 and 21–22.

| Line | New Block | Fields | Means |
|---|---|---|---|
| 1 | `IF` | operand 1: **var**, **BOOL**, index **51** | gate of block 1 |
| 2 | `SET_MOTIONSET` | **VAL**, number of your **normal** set | speed of block 1 |
| 3 | `LIN_VAR` | target pose: **POSE**, index **51**. Blending: **Keep** | linear move to the target |
| 4 | `CALC_REALVAR` | target: **REAL**, index **71**. Operand 1: **val**, **1**. No operator | echo "block 1 ran" |
| 5 | `END_IF` | the end of the IF from line 1, dragged under line 4 | |
| 6 | `IF` | var, BOOL, index **52** | gate of block 2 |
| 7 | `SET_MOTIONSET` | VAL, number of your **slow** set | speed of block 2 |
| 8 | `LIN_VAR` | POSE, index 51. Blending: Keep | the same move, slow |
| 9 | `CALC_REALVAR` | REAL 71 := val **2** | echo block 2 |
| 10 | `END_IF` | under line 9 | |
| 11 | `IF` | var, BOOL, index **53** | gate of block 3 |
| 12 | `SET_MOTIONSET` | VAL, **slow** set | speed of block 3 |
| 13 | `CIRC_VAR` | [1] description: **RadiusEndpos**. [2] direction: **CCW**. [3] plane: **XY**. [4] Radius: **var REAL 51**; Endpos: **var POSE 51**. [5] blending: **Keep** | arc counter-clockwise to the target, with the radius |
| 14 | `CALC_REALVAR` | REAL 71 := val **3** | echo block 3 |
| 15 | `END_IF` | under line 14 | |
| 16 | `IF` | var, BOOL, index **54** | gate of block 4 |
| 17 | `SET_MOTIONSET` | VAL, **slow** set | speed of block 4 |
| 18 | `CIRC_VAR` | same as line 13, direction: **CW** | arc clockwise |
| 19 | `CALC_REALVAR` | REAL 71 := val **4** | echo block 4 |
| 20 | `END_IF` | under line 19 | |
| 21 | `CALC_REALVAR` | target: REAL, index **72**. Operand 1: **var**, **REAL**, index **54** | echo the run number. Always runs, last line |
| 22 | `END_PROG` | **"wait for the movement to end" = ON** | end of program |

The fields of `CIRC_VAR`, as the manual numbers them (11.9.2): [1] which parameters describe the circle, [2] CW or CCW, [3] the plane, [4] the variables for the parameters shown by [1], [5] the blending. With "RadiusEndpos" field [4] shows exactly two parameters: the radius and the end point.

Rules that make the engine work:

- Only `POSE[51]` is ever a target. Nothing else in the program can move the robot.
- Every block sets its own motion set. The basic state does not restore it, and the speed must never depend on the block that ran before.
- The two echo lines of each block are not optional. Without line 4 (9, 14, 19) the engine gives error 10, without line 21 error 13.
- Nothing between the gates. A motion block outside an IF would run on every start.
- Line 22 must wait for the movement. The PLC takes "program finished" as "the robot stands at the target". Without the wait, finished comes while the robot still moves, the engine reports done too early and the next move is refused with error 6.
- If RobotMonitor asks for the IF as a comparison: operand 1 var BOOL 51, operator `=`, operand 2 val TRUE. Same thing.

## 3b. Two lines more (2026-11-05, `DOCS/ENGINE_REVIEW_2026-11-04.md` section 3)

The manual (2570-2576) says that variables and expressions in the SRL program are not synchronous with the movement: a `CALC_REALVAR` is processed when the program pointer reaches it, milliseconds after the start, while the robot still moves. So the echo lines prove that a block was interpreted, not that the robot arrived. Only `END_PROG` with "wait for the movement to end" ties "program finished" to arrival. Two lines make the echoes say more:

| Line | New Block | Fields | Means |
|---|---|---|---|
| 0, before line 1 | `CALC_REALVAR` | target: REAL, index **71**. Operand 1: val, **0** | clears the block echo at every run. Then REAL 71 = n proves block n ran in THIS run, and engine error 10 also catches a run in which no gate was taken |
| 20a, before line 21 | `WAIT_MOTION` | argument **Motion_Done** (manual 14022-14031) | the program pointer waits until the motion is done. Line 21 then writes REAL 72 at standstill, and REAL 72 = run number means "arrived" |

Nothing changes in the PLC for these two lines. The bullet in section 3 about line 22 stays true, the reason is `END_PROG` wait, not the echo line.

## 3a. The program as it stands on your robot (photo of 2026-11-02) and what to check

RobotMonitor names the variables that sit on the bus by their process data words. Your names and ours:

| On the robot | Ours | Meaning |
|---|---|---|
| `[51] PDO114-118 Extra 1` | `POSE[51]` | the target, words 114–118 |
| `[51] PDO123.0 Extra 1` … `[54] PDO123.3 Extra 4` | `BOOL[51]` … `BOOL[54]` | the gates of blocks 1 to 4, word 123 bits 0 to 3 |
| `[51] PDO119 Extra 1`, `[52] PDO120 Extra 2` | `REAL[51]`, `REAL[52]` | radius, arc angle |
| `[71] PDI66 Extra 1` | `REAL[71]` | the block echo |

What the photo shows, against section 3:

| Robot line | Is | Section 3 line | Verdict |
|---|---|---|---|
| N10 `MotionSet := 1` | one motion set for the whole program | 2, 7, 12, 17 | works. Block 2 is then not slower than block 1. A slow set inside blocks 2, 3, 4 comes when the sensor needs it |
| N20–N50 | `IF BOOL 51`, `LIN POSE 51`, `REAL 71 := 1`, `END_IF` | 1–5 | block 1, correct, proven on 2026-10-30 |
| N60–N90 | the same with `BOOL 52` and `:= 2` | 6–10 | block 2, correct |
| N100–N130 | `IF BOOL 53`, `CIRC_CCW plane XY, Radius := REAL 51, Angle := REAL 52`, `REAL 71 := 3`, `END_IF` | 11–15 | block 3 written as **radius + angle**, section 3 says radius + end point. Since 2026-11-02 the drum job sends both, the angle 90 in `REAL 52` and the end point in `POSE 51`. The hand test of section 4 decides |
| N140 … | `IF BOOL 54`, then cut off on the photo | 16–20 | block 4 clockwise, check it mirrors block 3 with `:= 4` |
| after the last `END_IF` | not on the photo | 21 | `REAL 72 := REAL 54` must be there. It is, or the single move of 2026-10-30 would have ended with error 13 |
| last line | not on the photo | 22 | `END_PROG` with "wait for the movement to end" ON. Without it the engine reports done while the robot still moves |

So for the cube nothing is missing on the robot: block 1 is all it uses. For the drum, blocks 3 and 4 are there, only the hand test is open. A second program is not needed, every block has its own gate inside program 20.

**RadiusEndpos** (manual 6.16.3): from where the robot stands, an arc of the given radius to the end point. Nothing else is needed, no center, no via point. This is why it fits the contract: one pose and one number.

| Fact | Consequence for us |
|---|---|
| The arc lies in the plane XY. Of the end point only X and Y are used | write `POSE[51].Z` = the current Z. The arc does not climb |
| Positive radius = the shorter of the two possible arcs, negative = the longer | the PLC always writes a positive radius. Error 4 refuses 0 and negative |
| CW / CCW is seen looking at the XY plane against the Z axis (from above when Z points up) | block 3 CCW, block 4 CW. Same end point, the other way round |
| A half circle (end point opposite the start) has two equal arcs | never ask for exactly 180°. A full circle is four quarter arcs, four moves |
| The end point must be reachable with that radius: distance start–end ≤ 2 × radius | the PLC job computes the end points on the circle, so this is always true. By hand: type a point that really lies on the circle |
| RotZ and RotY of `POSE[51]` are the orientation at the end of the arc, interpolated along the arc like in a linear move | the job writes the orientation it wants at the end of each quarter |
| A motion set applies to the arc like to a line | slow set, the sensor needs a steady speed |

**One hand test for block 3 and 4 (test C13).** Robot stands at X0, Y0, Z0, RotZ0, RotY0, read from the actual pose. Type on the Robot move screen: X = X0 + r, Y = Y0 + r, Z = Z0, RotZ = RotZ0, RotY = RotY0, radius = r (r = 300 for the first try), block 3, override 10, MOVE. The robot drives a quarter circle counter-clockwise. Type X0, Y0 again with block 4, MOVE: a quarter circle clockwise back. Error 4 if the radius is 0.

The radius field on the Robot move screen is `"DB_RobotMove".Man.udtReq.aorPar[1]`, 0 decimals, limits 0 … 5000. Add it next to the block field if it is not there yet.

## 5. The drum scan with blocks 3 and 4 (`FB_ScanDrumJob`)

The drum stands on the floor, its axis is vertical. The scan is a circle around the drum at one height, the tool always looking at the drum axis.

```
          P1                   taught: drum center C (X, Y), scan height Z
       .--+--.                 typed:  radius r = drum radius + sensor distance
     /         \
    |     C     |  P0 = start of the arc, angle a0
  P2+    +     +P0             P_k = C + r · (cos a_k, sin a_k),  a_k = a0 + k · 90°
    |           |              RotZ_k = a_k + 180°, wrapped to −180 … 180 (tool looks at C)
     \         /               Z = scan height, RotY = taught
       '--+--'
          P3
```

`FB_ScanDrumJob` (written 2026-10-31) pushes, in this order, every one a normal queue entry:

| # | Move | Block | `POSE[51]` | `REAL[51]` |
|---|---|---|---|---|
| 1 | to start | 1 | pose 2 | – |
| 2 | above P0 | 1 | P0 with Z = safe Z | – |
| 3 | down to P0 | 2 | P0 | – |
| 4 | arc to P1 | 3 (ccw) or 4 (cw) | P1 | r |
| 5 | arc to P2 | 3 or 4 | P2 | r |
| 6 | arc to P3 | 3 or 4 | P3 | r |
| 7 | arc to P0 | 3 or 4 | P0 | r |
| 8 | lift | 1 | P0 with Z = safe Z | – |
| 9 | to start | 1 | pose 2 | – |
| 10 | home | 1 | pose 1, only if taught | – |

The center C is pose 8 (taught above the drum axis), the scan height, the radius, the tool offset and the direction are the fields of `"DB_Recipe".Drum` (the circle starts where the tool stands at RotZ −180 and turns 90° per quarter to +180, no wrap, 2026-11-02) (moved there 2026-11-02 so `DB_Poses` in the PLC stays untouched) (`HMI_STAGE2.md` section 5). Clockwise uses block 4 and the angles go down by 90° instead of up. Since 2026-11-02 every arc move also carries the angle 90 in `REAL[52]`, so a circle line written as radius + angle (your N110 of 2026-11-02, `Radius := REAL[51]`, `Angle := REAL[52]`) has its data too. The end point in `POSE[51]` is still sent. The hand test of section 4 decides which description the robot drives correctly; the end-point description is the unambiguous one. Each quarter is one move, so the robot makes an exact stop at P1, P2, P3. For a laser scan that is fine, the stop is on the circle.

Nothing changed for this in the engine, the bus or the queue. New: the job block, `FC_JobMux` in front of the queue, pose 8, five fields, two text lists.

**Later, if the stop at each quarter disturbs the sensor:** the robot can turn the tool itself along the arc. The command `SET_TANGENTIAL_PARA` (manual 6.15, 11.10.6) aligns the tool X axis to the path tangent in the plane XY, with an angular offset, for every following segment. With an offset of 90° the tool looks at the center all the way round. That would be a block 5: `SET_TANGENTIAL_PARA` on, then `CIRC_VAR` CCW. Not now, it needs the tool coordinate system to be right first.

## 6. Motion sets

Speed, acceleration and jerk of a move come from the motion set selected by `SET_MOTIONSET` in the block. The PLC changes the speed of a single move only through the override (PO 4), which `FB_RobotMove` writes from `intOverride` of the request. Override 10 = 10 % of the set.

## 7. What the PLC checks against this program

| Check | Where | Error |
|---|---|---|
| block exists, 1 … `Cfg.intBlocksInPrg20` | `FB_RobotMove` step 5 | 1 |
| target inside the limits | `FC_PoseCheck` | 2 xyz, 3 rotation |
| radius > 0 for blocks 3 and 4 | `FB_RobotMove` step 5 | 4 |
| `REAL[71]` = block | step 60 | 10 |
| `REAL[72]` = run number | step 60, and step 40 for very short moves | 13 |

PLC side when blocks 3 and 4 go in: `Cfg.intBlocksInPrg20` = 4, text list `TL_Block` lines 3 "circle ccw" and 4 "circle cw", radius field on the Robot move screen. That is all.

## 8. Blocks 5, 6 and 7 – the path mode (2026-11-02, one folder since 2026-11-04)

Only with part 7 of `AUTOMATIC_TODAY.md`. Three blocks more in the same program 20, built in RobotMonitor like the first four, sections 2 and 3. They go **between the END_IF of block 4 and the run echo line** (line 21 of section 3, `REAL 72 := REAL 54`), so the echo and `END_PROG` stay the last two lines.

The variables they use are the seven poses of the standard path segments of module 1: **POSE index 1 to 7** in the variable list. RobotMonitor names them after their process data words, like it named POSE 51 "PDO114-118 Extra 1". Nothing in MOVISUITE changes, the segments exist already, your OB30 network 2 sends them (from `DB_RobotPath` after part 7).

The gates: `BOOL 55` = word 123 bit 4, `BOOL 56` = bit 5, `BOOL 57` = bit 6. RobotMonitor shows them as "PDO123.4 Extra 5" and so on.

### Block 5 – the rectangle, seven linear moves in one run

| Line | New Block | Fields | Means |
|---|---|---|---|
| 5.1 | `IF` | operand 1: var, **BOOL**, index **55** | gate of block 5 |
| 5.2 | `SET_MOTIONSET` | VAL, number of your **normal** set | speed of the rectangle |
| 5.3 | `LIN_VAR` | target: **POSE 1**. Blending: **Keep** | side 1, to corner 2 |
| 5.4 | `LIN_VAR` | POSE 2, Keep | turn at corner 2 (same point, new orientation) |
| 5.5 | `LIN_VAR` | POSE 3, Keep | side 2, to corner 3 |
| 5.6 | `LIN_VAR` | POSE 4, Keep | turn at corner 3 |
| 5.7 | `LIN_VAR` | POSE 5, Keep | side 3, to corner 4 |
| 5.8 | `LIN_VAR` | POSE 6, Keep | turn at corner 4 |
| 5.9 | `LIN_VAR` | POSE 7, Keep | side 4, back to corner 1 |
| 5.10 | `CALC_REALVAR` | REAL 71 := val **5** | echo block 5 |
| 5.11 | `END_IF` | under line 5.10 | |

"Keep" means the blending of the basic state, 0, an exact stop at every line. That is wanted now: the turn lines have no translation, only the orientation changes, and a stop at each corner is where the turn happens.

The PLC fills POSE 1 to 7 with list moves 4 to 10 of the cube job, side 1 to side 4 with the turns between them. The three moves before (to start, approach, to corner 1) and the four after (turn at corner 1, lift, start, home) stay single moves with block 1, so program 20 runs 8 times for a cube instead of 14.

### Block 6 – the drum circle counter-clockwise, four quarters in one run

| Line | New Block | Fields | Means |
|---|---|---|---|
| 6.1 | `IF` | var, BOOL, index **56** | gate of block 6 |
| 6.2 | `SET_MOTIONSET` | VAL, **slow** set (or the normal set if you have only one) | speed of the circle |
| 6.3 | `CIRC_VAR` | [1] **RadiusEndpos**, [2] **CCW**, [3] plane **XY**, [4] Radius: var **REAL 51**, Endpos: var **POSE 1**, [5] blending **Keep** | quarter to P1 |
| 6.4 | `CIRC_VAR` | the same, Endpos: **POSE 2** | quarter to P2 |
| 6.5 | `CIRC_VAR` | the same, Endpos: **POSE 3** | quarter to P3 |
| 6.6 | `CIRC_VAR` | the same, Endpos: **POSE 4** | quarter back to P0 |
| 6.7 | `CALC_REALVAR` | REAL 71 := val **6** | echo block 6 |
| 6.8 | `END_IF` | under line 6.7 | |

The PLC fills POSE 1 to 4 with P1, P2, P3, P0 and REAL 51 with the radius. POSE 5 to 7 hold a copy of P0 and are not used by this block. The moves before (to start, above P0, down to P0) and after (lift, start, home) stay single moves, program 20 runs 7 times for a drum instead of 10.

**If your circle lines are written as radius + angle**, like your N110 of 2026-11-02: the four lines are then `CIRC_VAR`, [1] RadiusAngle, Radius: var REAL 51, Angle: var **REAL 52**, and the PLC sends 90 in REAL 52. The end points in POSE 1 to 4 are then not used by the robot. The hand test of section 4 decides which description drives the right arc, the end-point form is the unambiguous one.

### Block 7 – the drum circle clockwise

The same eight lines as block 6 with gate **BOOL 57**, direction **CW**, echo **7**. The PLC sends the four end points in the clockwise order.

### After the three blocks

| Line | New Block | Fields |
|---|---|---|
| 21 | `CALC_REALVAR` | REAL 72 := var REAL 54, the run echo, unchanged |
| 22 | `END_PROG` | wait for the movement to end = ON, unchanged |

### PLC side

| Setting | Value |
|---|---|
| `"DB_RobotMove".Cfg.intBlocksInPrg20` | **7** |
| text list `TL_Block` | 5 "rectangle, 7 lin", 6 "circle ccw, 4 arcs", 7 "circle cw, 4 arcs" |
| `"DB_Recipe".xPathMode` | the switch on the Automatic screen. Off = blocks 1 to 4 one move per point, on = blocks 5 to 7 |

### What the engine checks with these blocks

| Check | Error |
|---|---|
| block 1 … 7 | 1 |
| the target and every one of the path poses inside the limits | 2, 3 |
| radius > 0 for blocks 3, 4, 6, 7 | 4 |
| path longer than 7 poses | 16 |
| the whole run finished within 300 s | 12 |
| `REAL 71` = 5, 6 or 7 at the end | 10 |
