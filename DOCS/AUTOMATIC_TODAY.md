# Drive in automatic today

2026-10-30. **This is the only file you need today.** Read it from top to bottom, do one step after the other, tick each box. Stop at the first step that does not match and tell me the step number.

The three screens, with the names used below:

| Screen | What is on it |
|---|---|
| **Robot move** (stage 1) | Enable switch, target X Y Z RotZ RotY, block, override, MOVE, RESET, error number |
| **Automatic** (stage 2, "teaching and automatic mode") | AUTO switch, pose nr, TEACH, pose rows 1..10 with valid lamps, safe Z, override, START, ABORT, job lamps and numbers |
| **old robot screen** | jog buttons, simulation, homing, robot reset |

Two rules for the whole day:

1. **Enable** (Robot move screen) stays **ON** all day.
2. **AUTO OFF = hand**: jog, teach, single MOVE. **AUTO ON = automatic**: START, ABORT. A button in the wrong mode does nothing, that is wanted.

---

## Part 0 – pre-flight, before step 1 (added 2026-11-01, 20 minutes)

Everything that must be true before you press anything. Tick each line. The block map of this version is `BLOCK_MAP.md` part A, the one-writer check is part B of the same file.

**0.1 The blocks are in the project.** Project tree → Program blocks and PLC data types:

| Must be there | Must NOT be there yet |
|---|---|
| `UDT_Pose`, `UDT_RobotStatus`, `UDT_MoveReq`, `UDT_MoveState`, `UDT_MoveQueue`, `UDT_JobState` | `UDT_JobOut` |
| `DB_Robot`, `DB_RobotMove`, `DB_Poses`, `DB_RobotJob`, `DB_MoveQueue`, your `DB_App` | |
| `FC_PoseCheck`, `FC_RobotStatusMap`, `FC_MoveSelect`, `FC_RobotAlarms` (may be missing, nothing today needs it) | `FC_JobMux` |
| `FB_Teach` + `IDB_Teach`, `FB_ScanCubeJob` + `IDB_ScanCubeJob`, `FB_MoveQueue` + `IDB_MoveQueue`, `FB_RobotMove` + `IDB_RobotMove`, one instance each | `FB_ScanDrumJob` |

Compile the PLC: 0 errors. ☐

**0.2 OB1 order and the cube wiring.** NW8 enable → teach → cube job → queue → `FC_MoveSelect` then `IDB_RobotMove` in one network → alarms. The cube job outputs go to `"DB_RobotJob".Push.xReq`, `.Push.udtMove`, `.Push.xClear`, `"DB_RobotJob".State`. The queue pin `i_xPop` is `"DB_RobotMove".State.xDone`. ☐

**0.3 One writer, the cross-references.** `BLOCK_MAP.md` B4: right-click the tag → Cross-references → exactly one W or RW, the one named there. For an array the RW stands on the array name (`Srl.Pose`), the elements (`Srl.Pose[1]`) show only R; count both lines together. The bug of 2026-10-30 was a screen writing a bus tag, it must not come back. ☐

**0.4 The old robot screen.** No Start button, no program number field, no override field left on it. Its enable switch is bound to `"DB_RobotMove".Man.xEnable`, not to `"DB_App".Cmd.Enable`. Reset, automatic, simulation, pause, stop, homing, jog stay as they are. ☐

**0.5 The two new screens.** Robot move: the five target fields on `"DB_RobotMove".Man.udtReq.udtPose.rX … rRotY`, override on `.udtReq.intOverride`, block on `.udtReq.intBlock`, MOVE and RESET momentary on `Man.xStart`, `Man.xReset`, Enable switch on `Man.xEnable`. Automatic: AUTO on `"DB_RobotJob".Man.xAuto`, START and ABORT momentary on `Man.xStart`, `Man.xAbort`, override on `Man.intOverride`, pose rows on `"DB_Poses".aoudtP[n]`, safe Z on `"DB_Poses".rSafeZ`, pose nr and TEACH on `"DB_RobotJob".Teach.intPoseNr`, `.xTeach`. Every name in `HMI_STAGE1.md` and `HMI_STAGE2.md`. ☐

**0.6 Watch table, PLC online, Enable switch ON, AUTO OFF, robot referenced and idle:**

| Tag | Must be | If not |
|---|---|---|
| `"DB_App".ReadyToStart` | TRUE | your logic. Without it the start never reaches the robot: error 11 |
| `"DB_App".MachineOn` | TRUE | your logic, same |
| `"DB_App".Cmd.Enable` | TRUE | NW8. Off = the Enable switch is off, or the machine is not on, or not referenced |
| `"DB_App".Cmd.Automatic` | TRUE | with `FB_RobotMode` (step 1a) the PLC writes it: TRUE at rest, FALSE only while a jog or home button is held. Without the block: old robot screen, automatic on. Without it error 5 |
| `"DB_App".Cmd.Simulation` | FALSE (TRUE only in part 3) | old robot screen |
| `"DB_Robot".Status.xCommOK`, `.xReady`, `.xReferenced`, `.xAutomatic`, `.xAccessGranted` | TRUE | comm: bus and Enable. access: RobotMonitor gives it back |
| `"DB_Robot".Status.xFault` | FALSE | old robot screen: robot reset |
| `"DB_Robot".Status.intPrgNr` | 20 | RobotMonitor: load program 20 |
| `"DB_RobotMove".Cfg.intBlocksInPrg20` | 1 or 2 | admin field, or type it in the watch table |
| `"DB_RobotMove".Man.udtReq.intOverride` | 10 | override field of the Robot move screen |
| `"DB_RobotJob".Man.intOverride` | 10 | override field of the Automatic screen |
| `"DB_RobotMove".State.intErrId`, `.xBusy` | 0, FALSE | RESET |
| `"DB_MoveQueue".intCount` | 0 | ABORT |
| `"DB_RobotJob".State.xRunning`, `.xError` | FALSE | ABORT |

☐

**0.7 Robot.** Program 20 loaded and initialized, with at least lines 1–5 (block 1) and line 22 `END_PROG` with "wait for movement end" ON, `PROGRAM20.md` section 3. RobotMonitor is not holding the access. ☐

**0.8 One single move.** Robot move screen, AUTO off: type a pose near the robot, block 1, override 10, MOVE. Done lamp, error 0, the robot stands on the typed pose, the typed numbers are still in the fields. This is the proven path, it must work before anything else. ☐

All ticked → step 1.

---

## Part 1 – PLC, once (15 minutes)

**Step 1 – FC_MoveSelect with seven inputs.** Open OB1, the network with the `FC_MoveSelect` box. If it already has seven inputs, go to step 2. If it has three:

1. Project tree → PLC → External source files → Add new external file → `FC_MoveSelect.scl` from the Gantry_Robot folder.
2. Right-click the file → Generate blocks from source → confirm the overwrite of `FC_MoveSelect`.
3. Back in OB1: right-click the `FC_MoveSelect` box → Update block call. Four new pins appear empty.
4. Wire them:

| Pin | Wire |
|---|---|
| `i_xHand` | already `"DB_RobotMove".Man.xEnable`. Add a **normally closed** contact `"DB_RobotJob".Man.xAuto` in series after it |
| `i_xAuto` | contact `"DB_RobotMove".Man.xEnable` in series with a **normally open** contact `"DB_RobotJob".Man.xAuto` |
| `i_udtQueueHead` | `"DB_MoveQueue".Head` |
| `i_xQueueHeadValid` | `"DB_MoveQueue".xHeadValid` |
| `i_udtMoveState` | `"DB_RobotMove".State` |

The other pins stay: `i_udtMan`, `i_xManStart`, `o_udtReq` → `"DB_RobotMove".Req`, `o_xStart` → `"DB_RobotMove".xReqStart`.

5. Compile, download.
6. Robot move screen, AUTO off: one single MOVE. It must work exactly as before. ☐

**Step 1a – FB_RobotMode, the robot's mode by the PLC (added 2026-11-02).** After this you never touch the automatic switch again. Jog = manual, MOVE and START = automatic, the PLC switches: automatic is the rest state, manual while a jog or home button is held and nothing of ours moves, automatic again half a second after the button is released and the robot stands still.

1. External source files → Add new external file → `FB_RobotMode.scl` → Generate blocks from source.
2. OB1: new network **after network 8** (enable), **before the teach network**. Drag `FB_RobotMode` in, single instance `IDB_RobotMode`. Pins:

| Pin | Wire |
|---|---|
| `i_xJogWanted` | 16 contacts in parallel, normally open: `"DB_App".Cmd.Jog_Pos_X`, `.Jog_Neg_X`, `.Jog_Pos_Y`, `.Jog_Neg_Y`, `.Jog_Pos_Z`, `.Jog_Neg_Z`, `.Jog_Pos_R1`, `.Jog_Neg_R1`, `.Jog_Pos_R2`, `.Jog_Neg_R2`, `.Home_X1`, `.Home_Y1`, `.Home_Z1`, `.Home_R1`, `.Home_R2`, `"DB_App".Home.active` |
| `i_xAuto` | `"DB_RobotJob".Man.xAuto` |
| `i_xEngineBusy` | `"DB_RobotMove".State.xBusy` |
| `i_udtStatus` | `"DB_Robot".Status` |
| `io_xAutomatic` | `"DB_App".Cmd.Automatic` |
| `o_xManual` | open |

3. Old screen: delete the automatic switch, or turn it into a lamp on `"DB_Robot".Status.xAutomatic`. Two writers of `Cmd.Automatic` would fight every cycle.
4. Compile, download.
5. Test with RobotMonitor open on the operating mode. Press jog X+: "Manual with high speed" lights, the axis moves. Release: half a second later "Automatic" lights. Robot move screen: MOVE. The robot drives, no error 5. A MOVE inside that half second gives error 5: RESET, MOVE again. ☐

**Step 1b – the cube by its dimensions (added 2026-11-02).** Two imports, three pins, three fields. After it the setter teaches three poses instead of seven, and a moved cube of the same size needs one new teach.

1. External source files → `DB_Recipe.scl` → Generate blocks from source. A new retentive DB with the cube numbers and the drum numbers. `DB_Poses` is not touched, the taught poses stay.
2. External source files → `FB_ScanCubeJob.scl` (v0.2) → Generate blocks from source, overwrite. OB1: right-click the cube job box → Update block call. Three new pins at the end:

| Pin | Wire |
|---|---|
| `i_rLength` | `"DB_Recipe".Cube.rLength` |
| `i_rWidth` | `"DB_Recipe".Cube.rWidth` |
| `i_rAngle` | `"DB_Recipe".Cube.rAngle` |
| `i_xFromHere` | `"DB_Recipe".xFromHere` |
| `i_udtActual` | `"DB_Robot".Status.udtPoseBcs` |
| `i_xActualValid` | contact `"DB_Robot".Status.xReferenced` in series with `"DB_Robot".Status.xCommOK` |

3. Automatic screen: the box "CUBE" of `HMI_STAGE2.md` section 6: three I/O fields, three teach buttons with lamps, two go-to buttons, and the ACK button in the job box. All of it is HMI scripts on tags that exist, no PLC change. The minimum for today is the three fields. They start at 0, so the job behaves exactly as before until you type a length and a width.
4. Compile, download. The instance DB of the cube job is re-initialized, that is fine, it holds no setup values. ☐

**Step 2 – three numbers in a watch table.**

| Tag | Must be |
|---|---|
| `"DB_RobotMove".Cfg.intBlocksInPrg20` | 1 or more. The scan uses block 1 only |
| `"DB_RobotJob".Man.intOverride` | 10. If 0: type 10 in the override field of the Automatic screen |
| `"DB_Poses".rSafeZ` | anything now, set in step 8 |

☐

---

## Part 2 – teach the seven poses (about one hour). After step 1b: part 2b, three poses and the dimensions

Enable ON, AUTO OFF, robot referenced, RobotMonitor not holding the access. The robot's own automatic switch on the old screen (`"DB_App".Cmd.Automatic`): **OFF for jogging**, the robot jogs only in manual. **ON again for step 9**, the engine moves only in automatic (manual 6.1.1). The switch positions of every step are the first table of `SCREENS.html`.

**How to teach one pose:** jog the robot there with the old robot screen → Automatic screen → pose nr = n → TEACH → row n: valid lamp on, numbers = the actual pose, "last taught" = n. If "refused" flashes: AUTO is on, the engine is busy, the robot is not referenced, comm is not OK, or the number is not 1..50.

The scan drives this way, so you know what you are teaching:

```
start → approach → corner 1 → side 1 → corner 2 (turn) → side 2 → corner 3 (turn)
      → side 3 → corner 4 (turn) → side 4 → corner 1 (turn) → lift → start → home
```

| Step | Pose nr | Where | ☐ |
|---|---|---|---|
| 3 | 1 | **home**: the park position | ☐ |
| 4 | 2 | **start**: above the cube, at a height where the robot can move freely | ☐ |
| 5 | 3 | **approach**: near corner 1, still above the cube | ☐ |
| 6 | 4 | **corner 1**: tool at corner 1, turned toward side 1 (the side from corner 1 to corner 2) | ☐ |
| 7 | 5, 6, 7 | **corner 2** turned toward side 2 (corner 2 → 3), **corner 3** toward side 3 (3 → 4), **corner 4** toward side 4 (4 → 1). Each corner has its own RotZ, on a cube about 90° apart | ☐ |
| 8 | – | **safe Z**: type the Z of pose 2 (start) into the safe Z field. The lift move goes to corner 1 X Y at this height. 2026-11-02: your poses all stand at Z 691 and safe Z was 450, 241 mm away from that plane. Safe Z must be on the free side of the cube, never toward the table | ☐ |

A side is: go to the next corner with the orientation of the corner before. A turn is: stay there, take the own orientation. That is why the orientation of each corner matters.

**Step 9 – check two poses by hand.** Robot move screen, AUTO off. Type the five numbers of row 4 (corner 1), block 1, override 10, MOVE. The robot must stand on corner 1. Same with row 2 (start). If it goes somewhere else, that row was taught at the wrong place: jog there again and TEACH again. ☐

---

## Part 2b – the cube by its dimensions (added 2026-11-02, replaces steps 5 to 7)

After step 1b. Three taught poses and three numbers.

| Step | Pose nr | What | ☐ |
|---|---|---|---|
| 3 | 1 | **home**, as in part 2 | ☐ |
| 4 | 2 | **start**, as in part 2 | ☐ |
| 5b | 4 | **corner 1**, the reference corner. With angle 0 it is the corner with the smallest X and the smallest Y. Tool at the corner, facing side 1, **RotZ −180**. The tool turns +90 at every corner and ends at +180 | ☐ |
| 6b | – | type **length** (side 1, along +X), **width** (side 2, along +Y), **angle** 0 | ☐ |
| 8 | – | **safe Z** = the Z of pose 2 | ☐ |

Then step 9 with row 4 and row 2, and part 3 as before. The 14 move texts stay the same. The approach is corner 1 at safe Z, pose 3 is not used. The turns at the corners are moves of zero length, every corner has the orientation of corner 1.

**The cube moved, same size:** jog to its new corner 1, pose nr 4, TEACH, START. Nothing else. **Another size:** type the new length and width. **Back to the taught corners:** length 0. **Scan from here:** jog to the corner, switch SCAN FROM HERE on, START. Nothing taught, nothing stored, the scan begins with side 1.

---

## Part 3 – first scan without moving (simulation, 15 minutes)

**Step 10.** Old robot screen: simulation ON. Robot move screen: Enable ON. Automatic screen: override 10, AUTO ON. ☐

**Step 11.** Press START. You see, in this order:

- running lamp on, job step "run"
- queue waiting: 13 → 0 (14 → 0 when home is taught)
- moves: 0/14 counting up, active: 1, 2, 3 … 14
- done lamp on, running off, job error 0

☐

**Step 12.** If the error lamp comes: read the job error number, part 5 says what to do. Fix it, press ABORT, press START again. ☐

---

## Part 4 – real scan (30 minutes)

**Step 13.** Simulation OFF. The robot can stand anywhere, the first move is "to start". One hand on the emergency stop. AUTO ON, override 10, START. The robot drives the sequence of part 2. Done lamp at the end. ☐

**Step 14 – ABORT.** START again. In the middle of side 2 press ABORT. The running move finishes, then the robot stops. Queue 0, running off, no error. START again: the robot begins with "to start". ☐

**Step 15 – five scans in a row.** Done lamp each time, moves 14/14, no error. ☐

**Step 15 done = automatic works.** Tell me. Then the override goes up step by step (10 → 25 → 50), and we build the alarm view and the one operator screen.

---

## Part 5 – when something is wrong

| You see | Cause | Do |
|---|---|---|
| START does nothing | AUTO off, or Enable off, or the error lamp is on, or a job is still running | switch on what is off. Error lamp: ABORT first, then START |
| job error 1 | taught mode: a pose 2..7 has no valid lamp. Dimension mode (length and width > 0): pose 2 or 4 | teach it |
| job error 2 | safe Z is 0 | step 8 |
| job error 3 | engine busy or in error | Robot move screen: RESET. Then ABORT, START |
| job error 4 | a pose is outside the limits. "active" shows which move (1 start, 2 approach, 3 corner 1, 4 side 1, 5 turn c2, 6 side 2, 7 turn c3, 8 side 3, 9 turn c4, 10 side 4, 11 turn c1, 12 lift, 13 start, 14 home) | teach that pose again |
| job error 6 | scan from here not possible: robot not referenced or comm not ok, or (cube) no length and width | reference, or type length and width, or switch SCAN FROM HERE off |
| job error 10 | the engine stopped with an error, number in "engine error" | see the next table. The engine stays in error until RESET: Robot move screen RESET first, then ABORT, then START |
| queue counts down, robot does not move | simulation still ON | old robot screen: simulation OFF |
| in auto the robot goes to a wrong place, by hand it is right | the taught row is wrong, the job uses exactly the rows | jog there, TEACH again |
| the robot stops in the middle, Enable went off | something in network 8 of OB1 dropped the enable | tell me what else went off (ready, referenced, comm) |

Engine error numbers (field "engine error" on the Automatic screen, same as the error field on the Robot move screen):

| Nr | Meaning | Do |
|---|---|---|
| 5 | robot not ready / referenced / automatic / access / setpoints | **first look at RobotMonitor, operating mode: if "Manual with high speed" is highlighted, the robot is in manual, the engine needs automatic.** `"DB_App".Cmd.Automatic` ON (old screen), RobotMonitor shows "Automatic", then RESET on the Robot move screen. Jogging works only in manual, that is why jog works and MOVE does not (2026-11-02). Otherwise: robot reset, referencing, RobotMonitor "NO control", setpoints active |
| 7 | communication lost | check the bus, then RESET |
| 8 | override 0 | override 10 |
| 10, 13 | program 20 echo lines missing | program 20 must be as `PROGRAM20.md` section 3, lines 4 and 11 |
| 11 | program 20 did not start: the start went out, "program running" did not come within 3 s | 1. RobotMonitor, program state: **"Repositioning required"** after a jog? Engine v0.5 (2026-11-03) pulses the program stop itself in step 15 before every start that finds this bit, so the robot never drives back to the old path; error 15 if the robot does not take it. With an older engine: press program STOP once (old screen, momentary), the state goes to "Initialized", then RESET, MOVE (manual 9.10). "Program finished" is not the problem: it is the normal state between two moves. 2. Watch table: `"DB_App".Cmd.Stop` and `.Cmd.Pause` FALSE (a held stop or pause blocks every start), `"DB_App".ReadyToStart` and `.MachineOn` TRUE while MOVE is pressed, `"DB_App".Cmd.Start` goes TRUE in steps 30 to 50. 3. RobotMonitor message list: a warning at that moment. 4. The decisive one: watch table `"DB_MOVIKIT_Robotics-5AX".PO[2]` in binary. Bit 0 = automatic, bit 2 = program start. Press MOVE: bit 2 must go TRUE while the step shows 30 to 50. If it does not, the AND of your OB30 network 2 (`ReadyToStart`, `MachineOn`) blocked it, the robot never saw a start. If it does and the robot still does not run, the robot refused it: program state and message list say why. Tell me the four results (2026-11-02) |
| 12 | move timeout | the move took too long, override too low or the robot paused |
| 20 | Enable fell during the move | see the last row of the first table |
| 21 | robot fault | old robot screen: robot reset. Message ID on the Robot move screen |
| 22 | RobotMonitor took the access | give the access back to the PLC |

Anything else: tell me the step number, the job error number and the engine error number. Those three numbers are enough.

---

## Part 6 – the drum, after step 15 (added 2026-10-31)

Only after five cube scans in a row. Nothing of this is needed for the cube.

**Step 16 – import, in this order.** External source files, generate blocks from source, overwrite when asked:

| # | File | Makes |
|---|---|---|
| 1 | `UDT_JobOut.udt` | the type |
| 2 | `DB_Recipe.scl` | already in since step 1b, it holds the drum fields too. `DB_Poses` is not touched |
| 3 | `DB_RobotJob.scl` | adds `Man.intJob`, `Cube`, `Drum`, `intJobActive` |
| 4 | `FC_RobotAlarms.scl` | job error 5 on bit 9 |
| 5 | `FB_ScanDrumJob.scl` | the drum job. Single instance `IDB_ScanDrumJob` when you place it |
| 6 | `FC_JobMux.scl` | the job selector |

**Step 17 – OB1.** Three changes, the pins are in `OB1_Stage2_call.scl`:

1. the cube job box: `i_xStart` gets a compare `"DB_RobotJob".Man.intJob == 1` in series after `Man.xStart`; the four outputs move to `"DB_RobotJob".Cube.xPushReq`, `.udtPush`, `.xClear`, `.udtState`
2. a new network after it: `IDB_ScanDrumJob`, start with compare `== 2`, the five drum inputs from `"DB_Recipe".Drum`, output `o_udtOut` to `"DB_RobotJob".Drum`
3. a new network after that, before the queue: `FC_JobMux`, outputs to `"DB_RobotJob".Push.xReq`, `.udtMove`, `.xClear`, `"DB_RobotJob".State`, in/out `intJobActive`

Compile, download. Job = 1: one cube scan must run exactly as before. ☐

**Step 18 – robot.** Program 20 lines 11–20 and `Cfg.intBlocksInPrg20` = 4, `PROGRAM20.md` section 3. The circle needs the add-on Circle license, section 0. ☐

**Step 19 – screen.** Job field, pose row 8 "drum center", scan height, radius, RotZ offset, clockwise, as `HMI_STAGE2.md` section 5. No start angle field, the start point is computed. Radius field on the Robot move screen. ☐

**Step 20 – hand test of the circle.** `PROGRAM20.md` section 4, a quarter circle with block 3 and back with block 4. ☐

**Step 21 – teach the drum.** Pose 8 above the drum axis. Type scan height, radius. Find the RotZ offset once (`HMI_STAGE2.md` section 5). The circle starts where the tool stands at RotZ −180 and turns 90° per quarter to +180. Safe Z above the drum. ☐

**Step 22 – scan drum.** Simulation on, job = 2, START: ten moves, done. Then real at override 10. Then the other direction with clockwise on. ☐

## Part 7 – path mode: the rectangle or the circle as one program run (2026-11-02, one folder since 2026-11-04)

Only after five cube scans in a row the single-move way (part 4) and after part 6 (the drum job exists only with the job mux). Nothing of this is needed for the cube or the drum the proven way: with `"DB_Recipe".xPathMode` off, every block of this part behaves exactly as before, so the blocks can go in early and the switch stays off.

The idea: the robot has seven poses on the bus that nothing uses today, the standard path segments of module 1, POSE 1 to 7, sent every cycle by OB30 network 2. The engine writes the corners of the rectangle (or the four points of the circle) into them, opens gate 55 (56, 57), gives one start, and block 5 (6, 7) of program 20 drives them in one run. The moves around it, to start, approach, lift, park, stay single moves of block 1. One handshake per rectangle instead of seven, and later continuous motion through the corners with blending. The cost: an error or an abort points at the whole rectangle, not at a corner.

**Step 23 – PLC, about one hour.** External source files, generate, overwrite when asked, in this order:

| # | File | Makes |
|---|---|---|
| 1 | `UDT_MoveReq.udt` v0.3 | two fields at the end, `aoudtPath[1..7]`, `intPathLen`. The type grows, TIA recompiles every block that uses it at the next compile. Their instance DBs and `DB_MoveQueue`, `DB_RobotMove`, `DB_RobotJob` are re-initialized at the download: `Cfg.intBlocksInPrg20` goes back to 1, set it to 7 after. `DB_Poses` is not touched |
| 2 | `DB_RobotPath.scl` | new DB, the seven poses on their way to the bus |
| 3 | `DB_Recipe.scl` v0.2 | one field at the end, `xPathMode`. Re-initialized: write the cube and drum numbers down before, type them again after |
| 4 | `FB_RobotMove.scl` v0.6 | the engine with the path. OB1: two new pins at the end of the `IDB_RobotMove` box, `io_aoudtPath := "DB_RobotPath".aoudtP`, `io_aorPathBlend := "DB_RobotPath".aorBlend`. If v0.5 is not in yet, its two pieces come with it: `UDT_MoveState` v0.2 and the OB30 NW2 stop pin `("DB_App".Cmd.Stop OR "DB_RobotMove".State.xPrgStop)` |
| 5 | `FB_ScanCubeJob.scl` v0.5, `FB_ScanDrumJob.scl` v0.6 (2026-11-06: the drum circle as one line of the robot, center + 360°, no `POSE 1..7`; needs the engine v0.7 and blocks 6, 7 of five lines, `CHECKLIST_DRUM_AUTO.md` part F) | OB1: one new pin at the end of each box, `i_xPathMode := "DB_Recipe".xPathMode`. All pins in `OB1_Stage2_call.scl` |
| 6 | OB30 network 2 | the 35 pose pins and the 7 blending pins of the segments read `"DB_RobotPath"` instead of `"DB_App".Path.Seg`, pin by pin in `OB30_NW2_pins.md`, the whole network in `OB30_complete.scl` |

Compile, 0 errors, download. Watch table: `Cfg.intBlocksInPrg20` := 7, the recipe numbers, `xPathMode` FALSE. Single move on the Robot move screen: as before. One cube scan with path mode off: as before, 14 moves. Only then the robot side. ☐

**Step 24 – robot, half an hour.** `PROGRAM20.md` section 8: blocks 5, 6, 7 between the END_IF of block 4 and the run echo line. Save. Variables tab: POSE 1 to 7 exist and show what the PLC writes. ☐

**Step 25 – screen, one hour.** `HMI_STAGE2.md` section 6.5: the PATH MODE switch on `"DB_Recipe".xPathMode`, `TL_Block` 5 to 7, `TL_Move` 15, `TL_MoveDrum` 11, `TL_Err` 16. ☐

**Step 26 – tests.** `TEST_PROTOCOL.md` part 5 (cube) and 6.5 (drum). ☐

One tag, one writer: `"DB_RobotPath".aoudtP`, `.aorBlend` by `IDB_RobotMove` (InOut). `"DB_Recipe".xPathMode` by the Automatic screen. `"DB_App".Path.Seg[n].X … R2`, `.Blend` then have no writer and no reader, leave them in DB_App.

Not in here, on purpose: continuous motion through the corners (the turn lines would go, the sides get a blending value, the laser stage); more than seven poses in one request (module 1 has seven segments); block 5 by hand from the Robot move screen (a hand request carries no path, the block field stays 1 to 4). Blending later: the engine already writes `aorBlend[n]` from the request's `rBlend`, 0 today; when the sides may round the corners, the job sets `rBlend`, the LIN lines of block 5 take blending "var" from the segment variable, and the turn lines are removed.

Confidence, honest: UDT and DBs 95 %; the engine's path part 85 % (the proven steps are untouched); the ordering of singles and the path entry in the jobs 80 %; the segment poses as plain variables 80 % (the R1 / R2 order on the pins is to check once, test 5.2); LIN_VAR on POSE 1 to 7 in one run 75 %, a LIN with only an orientation change inside one run is not tested on this machine.

## Files

Today: this file. The block map of the version in TIA and the one-writer check: `BLOCK_MAP.md`. If a tag name of a screen is missing: `HMI_STAGE1.md`, `HMI_STAGE2.md`. The whole system on one page: `SYSTEM_MAP.html`; with every HMI tag, OB33 and OB1 network by network and the path mode: `SOLUTION_MAP.html`. Part 7 only: `OB30_NW2_pins.md`. Everything else in the folder is for later, you do not need to open it today.
