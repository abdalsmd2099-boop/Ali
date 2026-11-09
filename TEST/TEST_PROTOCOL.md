# Test protocol – after the download, 2026-11-03

Tick every box in order. Stop at the first box that does not match and send me three numbers: the test number, the job error, the engine error, plus what RobotMonitor shows as program state. Do not skip a part to reach a later one, every part is the base of the next.

Rules for the whole protocol: Enable on all day. Override 10 until part 6 says otherwise. One hand near the emergency stop during every real move. Simulation ON for the first run of every new thing, OFF for the second.

Which parts apply depends on what you downloaded. Part 0 tells you.

---

## Part 0 – what is in the PLC now (5 minutes, nothing moves)

**0.1 Versions.** Program blocks, open each block, read the version in the header comment:

| Block | Version in the PLC | Means |
|---|---|---|
| `FB_RobotMove` | 0.3 = single moves only · 0.4 = path mode possible | |
| `FB_ScanCubeJob` | 0.1 taught corners only · 0.4 dimensions, from here, turns · 0.5 + path mode | |
| `FB_ScanDrumJob` | missing = no drum yet · 0.3 · 0.4 · 0.5 (no turn) | |
| `FC_MoveSelect` | must be 0.2 | 0.1 = START does nothing, step 1 of AUTOMATIC_TODAY.md first |
| `FB_RobotMode` | missing or 0.1 | missing = you switch the robot mode by hand |
| `DB_Recipe` | missing or 0.1 or 0.2 | |
| `DB_RobotPath` | missing or present | present = path mode wired |

Write the row down, it decides which parts below are yours. ☐

**0.2 Compile.** PLC compiled with 0 errors before the download. Warnings are allowed. ☐

**0.3 Watch table**, PLC online, Enable switch ON, AUTO OFF, robot standing:

| Tag | Must be | If not |
|---|---|---|
| `"DB_Poses".aoxValid[1]`, `[2]`, `[4]` | TRUE, and the rows show your poses of 2026-11-02 | the download touched DB_Poses: type the poses back from your photo, or teach again |
| `"DB_RobotMove".Cfg.intBlocksInPrg20` | 2 (blocks 1–2), 4 (with circles), 7 (with path blocks) | a DB re-initialization put it back to 1: type the right number |
| `"DB_Recipe".Cube.rLength`, `.rWidth`, `.rAngle` | your numbers, or 0 0 0 for the taught way | re-initialized: type them |
| `"DB_Recipe".xFromHere`, `.xPathMode` | FALSE, FALSE | both off for the first tests |
| `"DB_RobotJob".Man.intOverride`, `"DB_RobotMove".Man.udtReq.intOverride` | 10, 10 | |
| `"DB_App".ReadyToStart`, `.MachineOn` | TRUE, TRUE | your logic. Without them every start dies with error 11 |
| `"DB_App".Cmd.Automatic` | TRUE | with FB_RobotMode: by itself. Without: old screen switch ON |
| `"DB_Robot".Status.xReady`, `.xReferenced`, `.xAutomatic`, `.xAccessGranted`, `.xCommOK` | TRUE | RobotMonitor: referenced, "Automatic" highlighted, "NO control" |
| `"DB_Robot".Status.xFault` | FALSE | robot reset |
| `"DB_Robot".Status.intPrgNr` | 20 | RobotMonitor: program 20 loaded, state Initialized |
| `"DB_RobotMove".State.intErrId`, `.xBusy` | 0, FALSE | RESET |
| `"DB_MoveQueue".intCount` | 0 | ABORT |
| `"DB_App".Cmd.Stop`, `.Cmd.Pause` | FALSE, FALSE | a held stop or pause blocks every start |

☐

**0.4 Robot.** RobotMonitor program tab: program 20 with blocks 1 and 2 at least, the run echo line `REAL 72 := REAL 54` after the last END_IF, `END_PROG` with "wait for the movement to end" ON. Saved. ☐

**0.5 Screen.** The five target fields on `"DB_RobotMove".Man.udtReq.udtPose.*`, MOVE and RESET momentary, START and ABORT momentary, AUTO switch, the pose rows, safe Z. The old screen has no Start, no program number, no override field. ☐

---

## Part 1 – the single move, the proven path (10 minutes)

**1.1** Robot move screen, AUTO off, robot in automatic. Type a target 200 mm away from the actual pose in X, same Y, Z, RotZ, RotY. Block 1, override 10, MOVE. ☐ The robot drives there, done lamp, error 0, the typed numbers stay in the fields.

**1.2** The same back to the actual pose of before. ☐

**Program finished** lamp after a move, and still on after a jog: normal. It is the state of program 20 between two moves, the next MOVE starts from it (manual 6.3, 9.7). Nothing to clear, no machine off/on.

**1.3** A target with the actual RotZ ± 90. ☐ The tool turns, no error 3.

**1.4** Error on purpose: X = 5000, MOVE. ☐ Error 2 before any move, RESET clears it.

Part 1 fails with error 11: part 5 of AUTOMATIC_TODAY.md, row 11, the four checks, the start bit on PO word 2 bit 2 decides.

---

## Part 2 – the robot mode by the PLC (only with FB_RobotMode, 5 minutes)

**2.1** RobotMonitor open on the operating mode. Press jog X+ on the old screen and hold. ☐ "Manual with high speed" lights within a second, the axis moves.

**2.2** Release. ☐ Half a second later "Automatic" lights, the robot stands.

**2.3** MOVE a target without touching anything else. ☐ Drives, no error 5.

**2.4** MOVE while holding a jog button. ☐ Nothing happens or error 5, never a move. RESET.

**2.5** (engine v0.5, the program stop) After 2.3 the robot stands at a target. Jog X+ for two seconds, release, wait for "Automatic". RobotMonitor program state shows "Repositioning required". MOVE a target 200 mm away. ☐ The robot goes straight to the new target, no detour back to the point it left. Watch table during MOVE: `"DB_RobotMove".State.intStep` shows 15 for a moment and `.State.xPrgStop` pulses, RobotMonitor shows "Initialized" before the robot moves. Error 15 instead: the robot did not take the program stop within 2 s, tell me the message id and the program state. Without engine v0.5 the robot first drives back to the old point and then to the target: that is the repositioning of manual 6.6, not a fault.

Without FB_RobotMode: the old switch does this by hand, manual for jog, automatic for every move. Skip to part 3.

---

## Part 3 – teaching (10 minutes)

**3.1** Jog to the park position. Pose nr 1, TEACH, or TEACH HOME. ☐ Row 1 valid, numbers = actual pose, last taught 1.

**3.2** Jog above the cube at a free height. Pose nr 2, TEACH, or TEACH START. ☐ Row 2 valid.

**3.3** Jog the tool to corner 1 of the cube: with angle 0 the corner with the smallest X and the smallest Y, tool facing side 1. If the cube job is v0.4 or newer: **RotZ −180** there. Pose nr 4, TEACH, or TEACH CORNER 1. ☐ Row 4 valid.

**3.4** Safe Z = the Z of row 2. ☐ It must be on the free side of the cube, never toward the table. Yesterday it was 450 while the poses stood at 691.

**3.5** With cube job v0.1 only, or when you want the taught way: rows 3, 5, 6, 7 as part 2 of AUTOMATIC_TODAY.md, each corner facing its side. ☐

**3.6** Teach refused on purpose: AUTO on, TEACH. ☐ Refused lamp flashes, nothing stored. AUTO off again.

**3.7** Check by hand: GO TO CORNER 1, or type row 4 into the target and MOVE. ☐ The robot stands on the corner.

---

## Part 4 – the cube, one move per point (30 minutes)

Path mode OFF, scan from here OFF. Length and width 0 for the taught way, or your numbers for the computed way.

**4.1 Simulation.** Old screen simulation ON. AUTO ON, START. ☐ running lamp, queue 13 → 0 (14 with home), moves counting, active text 1 to 14, done lamp, job error 0. RobotMonitor: POSE 51 changes at every move.

**4.2 Real.** Simulation OFF. START. ☐ to start, approach, corner 1, four sides with the turns, lift, start, home. With v0.4 or newer the tool turns +90 at every corner and ends at +180.

**4.3 ABORT.** START, in side 2 press ABORT. ☐ The running move finishes, the robot stops, queue 0, no error. START: begins again with "to start".

**4.4 Computed corners, if not yet.** Length, width, angle 0 typed, row 4 at RotZ −180. Simulation ON, START. ☐ Same 14 moves, corners from the numbers. Then real.

**4.5 Wrong number on purpose.** Length 9000, START. ☐ Job error 4 with active = the first bad move, before anything moves. Length back.

**4.6 Scan from here.** AUTO off. Jog to corner 1, RotZ −180. SCAN FROM HERE on, AUTO on, START. ☐ Moves total 11 (10 without home), the first move is side 1, no "to start" first. Switch off after.

**4.7 Five in a row**, the way you will use. ☐ Done lamp each time, no error.

**Part 4 done = the cube works.** Everything below is the next step, do it only after 4.7.

---

## Part 5 – path mode (only after part 7 of AUTOMATIC_TODAY.md: engine v0.6, DB_RobotPath, blocks 5–7, 30 minutes)

**5.1 Before.** `Cfg.intBlocksInPrg20` = 7. Program 20 has block 5 between block 4 and the run echo line, `PROGRAM20.md` section 8. OB30 network 2 pose pins on `"DB_RobotPath"`. `TL_Block` 5 to 7, `TL_Move` 15 on the screen. ☐

**5.2 The path reaches the robot, nothing moves.** Simulation ON. PATH MODE on, AUTO on, START. During the run, RobotMonitor variable tab: POSE 1 to 7 show the seven corners and turns of the rectangle, POSE 1 = corner 2 with the RotZ of corner 1. ☐ If POSE 1 shows the RotY of the job in its RotZ field, R1 and R2 are swapped on the OB30 pins: swap them, part 5 again.

**5.3 Simulation, whole run.** ☐ Moves total 8 (7 without home), active text "to start", "approach", "to corner 1", then "rectangle, sides 1 to 4" for the whole run, then "turn at corner 1", "lift", "to start", "home". Done, job error 0. REAL 71 = 5 after the rectangle.

**5.4 Real.** Simulation OFF, override 10, START. ☐ Three single moves, then the rectangle in one run with a stop and a turn at every corner, then the four single moves. The robot is where part 4 left it after each move.

**5.5 ABORT during the rectangle.** ☐ The whole rectangle finishes, then the robot stops, queue 0, no error.

**5.6 Scan from here in path mode.** ☐ Moves total 5 (4 without home): rectangle, turn, lift, start, home.

**5.7 Five in a row in path mode.** ☐

If 5.3 ends with engine error 11: the gate 55 did not reach the robot, or block 5 is not in the program. Error 12: the run took more than 300 s, override too low or the robot paused. Error 10: the echo line of block 5 says the wrong number. Error 16: never from the job, only from a request with more than seven poses.

---

## Part 6 – the drum (only after part 6 of AUTOMATIC_TODAY.md, 45 minutes)

**6.1 Hand test of one quarter.** `PROGRAM20.md` section 4: X0 + r, Y0 + r, block 3, radius r = 300, MOVE. ☐ A quarter circle counter-clockwise. Block 4 back. If the robot drives a wrong arc or none: the circle line on the robot is radius + angle, not radius + end point; the hand test with REAL 52 = 90 decides, tell me.

**6.2 Teach the drum.** Pose 8 above the drum axis, or SCAN FROM HERE above it. Scan height, radius, RotZ offset typed. Safe Z above the drum. Job = 2. ☐

**6.3 Simulation, one move per point.** PATH MODE off, START. ☐ Ten moves, four arcs, done.

**6.4 Real.** ☐ The tool looks at the axis all the way round, RotZ −180 to +180.

**6.5 Path mode, the circle in one run (2026-11-06).** Engine v0.7, drum job v0.6, blocks 6 and 7 in the program (five lines each, center + 360°), `Cfg` = 7. Hand test of block 6 first: `CHECKLIST_DRUM_AUTO.md` F4.1, a full circle of radius 300 around a center 300 mm in −X, angle 360, back to the start point without a stop. Then PATH MODE on, START. ☐ Seven moves, "circle, one run" during the run, no stop on the circle, RotZ stays at the value of pose 8.

**6.6 Clockwise.** Clockwise on, 6.3 to 6.5 again. ☐

---

## Part 7 – after the tests

- Override up step by step: 25, 50. Five scans at each. ☐
- Tell me the three numbers of every box that did not match, and which version row of part 0 you have. One line per box is enough.
- What is not in this protocol and stays open: the alarm view, the one operator screen, the laser, blending through the corners.
