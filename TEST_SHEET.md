# Test sheet – stage 1, single movement

2026-10-28. Part A is offline, PLCSIM, no machine. Part B is GOALS.md 5.7 on the machine. Write the result in the last column, keep this file.

---

## A. Dry run in PLCSIM (offline, before the machine)

Proves the state machine of `FB_RobotMove` with the robot status set by hand.

### A.1 Preparation

1. Everything imported, OB30 network 4 and the OB1 network wired, PLC compiled clean.
2. **Deactivate OB30 network 4 for the dry run** (comment out the `FC_RobotStatusMap` call). In PLCSIM there is no bus, so the status map would overwrite the status with zeros every 10 ms. **Put it back before the download to the real PLC.**
3. Start PLCSIM, download, RUN.
4. New watch table `RobotMove_dry_run` with these tags:

| Tag | Set to |
|---|---|
| `"DB_Robot".Status.xCommOK` | TRUE |
| `"DB_Robot".Status.xReady` | TRUE |
| `"DB_Robot".Status.xReferenced` | TRUE |
| `"DB_Robot".Status.xAutomatic` | TRUE |
| `"DB_Robot".Status.xAccessGranted` | TRUE |
| `"DB_Robot".Status.xFault` | FALSE |
| `"DB_Robot".Status.xMoving` | FALSE |
| `"DB_Robot".Status.xPrgRunning` | FALSE |
| `"DB_Robot".Status.xPrgFinished` | FALSE |
| `"DB_Robot".Status.intRemainingSeg` | 0 |
| `"DB_RobotMove".Cfg.intBlocksInPrg20` | 1 |
| `"DB_RobotMove".Man.xEnable` | TRUE |
| `"DB_RobotMove".Man.udtReq.intBlock` | 1 |
| `"DB_RobotMove".Man.udtReq.udtPose.rX` | 1000.0 |
| `"DB_RobotMove".Man.udtReq.udtPose.rY` | 1000.0 |
| `"DB_RobotMove".Man.udtReq.udtPose.rZ` | 500.0 |
| `"DB_RobotMove".Man.udtReq.udtPose.rRotZ` | 0.0 |
| `"DB_RobotMove".Man.udtReq.udtPose.rRotY` | 0.0 |
| `"DB_RobotMove".Man.udtReq.intOverride` | 10 |
| `"DB_RobotMove".Man.xStart` | the button |
| `"DB_RobotMove".Man.xReset` | the button |
| `"DB_RobotMove".State.intStep` | watch |
| `"DB_RobotMove".State.xBusy`, `.xDone`, `.xError`, `.intErrId`, `.intRunNr` | watch |
| `"DB_App".Cmd.ProgramNumber`, `.Cmd.Start`, `.Cmd.Override` | watch |
| `"DB_App".Srl.Pose[1]` … `[5]`, `.Srl.Par[4]`, `.Srl.Gate[1]`, `.Srl.GateEnable` | watch |
| `"DB_MOVIKIT_ROBOTICS_Module2".SrlRealRaw[1]`, `[2]` | set, see A.2 step 6 |

### A.2 Good move

| # | Do | Expect | Result |
|---|---|---|---|
| 1 | modify the status tags and the request as in the table | `State.intStep` = 0, `xBusy` FALSE | |
| 2 | `Man.xStart` TRUE, then FALSE | step goes 5 → 10 → 20 → 30 → 40 and waits there. `xBusy` TRUE | |
| 3 | look at the bus side | `Cmd.ProgramNumber` 20, `Srl.Pose` 1000 / 1000 / 500 / 0 / 0, `Srl.Par[4]` = `State.intRunNr`, `Cmd.Override` 10, `Srl.GateEnable` TRUE, `Srl.Gate[1]` TRUE, `Cmd.Start` TRUE | |
| 4 | `Status.xPrgRunning` TRUE | step 50 | |
| 5 | `Status.xPrgRunning` FALSE, `Status.xPrgFinished` TRUE | after 200 ms step 60, then one of the two lines below | |
| 6a | if you can modify `SrlRealRaw[1]` = 1 and `SrlRealRaw[2]` = `State.intRunNr` before step 5 | `xDone` one cycle, step 0, `Cmd.Start` FALSE, `Srl.Gate[1]` FALSE | |
| 6b | if the module 2 FB overwrites `SrlRealRaw` in PLCSIM and they stay 0 | error 10 at step 60, step 90, `Cmd.Start` FALSE, `Srl.Gate[1]` FALSE. **That is correct**: the echo check works. `Man.xReset` TRUE then FALSE → step 0 | |
| 7 | repeat 2 with `Status.xPrgFinished` still TRUE from before | step 40 does not go to 50 on the old finished bit, it waits for running (rule 5). Set `xPrgFinished` FALSE, `xPrgRunning` TRUE → 50 | |

### A.3 Errors

Each line: set the status back to "good" first, then change the one tag named, press Move, read `State.intErrId`, then Reset.

| # | Change | Expect `intErrId` | Result |
|---|---|---|---|
| 1 | `udtPose.rX` = 5000.0 | 2, nothing changes on the bus | |
| 2 | `udtPose.rRotY` = 20.0 | 3 | |
| 3 | `intBlock` = 2 (Cfg is 1) | 1 | |
| 4 | `Status.xCommOK` FALSE | 7 | |
| 5 | `Status.xReferenced` FALSE | 5 | |
| 6 | `Status.xAccessGranted` FALSE | 5 | |
| 7 | `Status.xMoving` TRUE | 6 | |
| 8 | good move, at step 40 set `Man.xEnable` FALSE | 20, `Cmd.Start` FALSE, `Srl.Gate[1]` FALSE | |
| 9 | good move, at step 40 set `Status.xAccessGranted` FALSE | 22 | |
| 10 | good move, at step 40 set `Status.xFault` TRUE | 21 | |
| 11 | good move, at step 40 do nothing for 2 s | 11 | |
| 12 | good move, at step 50 do nothing for 60 s | 12 | |
| 13 | `Man.xStart` while `xError` TRUE | nothing happens until Reset | |

### A.4 After the dry run

1. Put OB30 network 4 back.
2. Compile, and check the cross references of `"DB_App".Cmd.Start`, `.Cmd.ProgramNumber`, `.Cmd.Override`, `.Srl.Pose`, `.Srl.Gate`: the only write access must be `FB_RobotMove`. An HMI tag that still writes one of them shows up here.

---

## B. On the machine (GOALS.md 5.7)

Robot powered, referenced, automatic selected on the HMI, override 10 %. Program 20 with block 1 loaded, `Cfg.intBlocksInPrg20` = 1. First in simulation (PO 1.9 on), then real.

Before the first move, note the actual pose from `"DB_Robot".Status.udtPoseBcs` and choose the first target close to it: same Z, X + 100 mm.

| # | Test | Pass when | Result |
|---|---|---|---|
| 0 | status map | `DB_Robot.Status` bits follow the process data monitor: ready, referenced, automatic, access granted, pose BCS = monitor pose, handshake echo toggling | |
| 1 | pose outside the limits, Move | error 2 or 3, nothing changes on the bus | |
| 2 | simulation on, valid pose, Move | steps 10 … 60, done. RobotMonitor shows POSE[51] = typed values, REAL[71] = 1, REAL[72] = run number | |
| 3 | simulation off, same pose, override 10 % | robot arrives, done, `State.udtPoseAtEnd` = target within 1 mm | |
| 4 | Move while busy | ignored, run number does not change | |
| 5 | HMI Stop in the middle of a move | robot stops, error 12 after 60 s or 21 if the robot faults, Reset clears, next move runs | |
| 6 | 20 moves in a row, alternating two poses | every move runs exactly once, run number 1..20 in order, REAL[72] follows | |
| 7 | RobotMonitor takes access during a move | error 22, PLC drops start and gate, resumes after access is back and Reset | |
| 8 | program 20 restarts while start is held? | watch "program running" after done: it must **not** come back. If it does, tell me, start is then dropped at step 50 | |

Write down for every test: date, result, and the error number if any.

---

## C. Trouble: the robot moves, but not to the target (2026-10-29)

The pose travels through six points. Read the same five numbers (X Y Z RotZ RotY) at every point, after one MOVE. The first point where the numbers differ names the layer.

| # | Point | Where to read | Means when it differs from the point before |
|---|---|---|---|
| 1 | typed | HMI target fields | – |
| 2 | request | `"DB_RobotMove".Req.udtPose.rX` … `.rRotY` | the HMI field writes another tag. Open the field, read its tag. Must be `Man.udtReq.udtPose.*` |
| 3 | engine out | `"DB_App".Srl.Pose[1]` … `[5]` | a second writer of `Srl.Pose`. Cross references of `"DB_App".Srl.Pose`: every write access except `FB_RobotMove` (the old arc endpoint network, an HMI field) must go |
| 4 | bus | `"DB_MOVIKIT_ROBOTICS_Module2".PO[50]` … `[54]`, display format Dec signed | module 2 FB: `PoseEnable` TRUE, `PoseFirstWord` 50, no scaling in the FB |
| 5 | robot variable | RobotMonitor, variables, `POSE[51]` | MOVISUITE process data: POSE[51] not at words 114–118, decimal places not 0, or the component order is not X Y Z RotZ RotY |
| 6 | robot arrived | `"DB_RobotMove".State.udtPoseAtEnd` (= BCS pose at end) | program 20: the LIN line does not use `POSE[51]`, or uses it in another coordinate system or tool. Or a reposition move came first |

If point 6 equals point 1 and the place still looks wrong: the number is right, the coordinate system is not. The BCS origin or the tool offset of the robot is not where you think. Check with a known taught pose: type its numbers, MOVE, the robot must stand on it.

Note also `State.intErrId` and `State.intStep` after the move. A wrong place with error 10 or 13 means program 20 ran a different block or an old run.
