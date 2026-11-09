# HMI – stage 1 screen "Robot move"

2026-10-28. WinCC Unified. One screen, all tags in `DB_RobotMove` and `DB_Robot`. Nothing on this screen writes `DB_App`.

## 1. Layout

```
+----------------------------------------------+----------------------------------------------+
|  TARGET                                      |  ROBOT                                       |
|  X     [ 1000 ] mm     RotZ  [   0 ] deg     |  ready  referenced  automatic  access  fault  |
|  Y     [ 1000 ] mm     RotY  [   0 ] deg     |  comm ok   moving   running   finished        |
|  Z     [  500 ] mm                           |  message id  [ 0 ]                           |
|  Override [ 10 ] %     Block [ 1 ] linear    |  actual pose  X [....] Y [....] Z [....]     |
|                                              |               RotZ [....] RotY [....]        |
|  [ Enable ]  (switch)                        |  remaining segments [ 0 ]                     |
|  [  MOVE  ]  [ RESET ]                       |                                              |
+----------------------------------------------+----------------------------------------------+
|  MOVE STATE                                                                                 |
|  busy  done  error     step [ 40 - wait running ]     error [ 0 - no error ]                |
|  run nr [ 12 ]   block [ 1 ]   pose at end  X [....] Y [....] Z [....] RotZ [....] RotY [....] |
|  blocks in program 20 [ 1 ]   (admin only)                                                  |
+---------------------------------------------------------------------------------------------+
```

## 2. Elements and tags

### Target, left

| Element | Tag | Settings |
|---|---|---|
| I/O field X | `"DB_RobotMove".Man.udtReq.udtPose.rX` | Real, format 0 decimals. HMI tag limits 0 … 4380 |
| I/O field Y | `…udtPose.rY` | 0 … 4230 |
| I/O field Z | `…udtPose.rZ` | −10 … 1908 |
| I/O field RotZ | `…udtPose.rRotZ` | −180 … 180 |
| I/O field RotY | `…udtPose.rRotY` | −10 … 10 |
| I/O field Override | `"DB_RobotMove".Man.udtReq.intOverride` | Int, limits 1 … 100. Start value 10 |
| I/O field Block | `"DB_RobotMove".Man.udtReq.intBlock` | Int, limits 1 … 8, with text list `TL_Block`. Start value 1 |
| Switch Enable | `"DB_RobotMove".Man.xEnable` | switch, on = manual moves allowed |
| Button MOVE | `"DB_RobotMove".Man.xStart` | **momentary**: event Press → SetTagValue 1, event Release → SetTagValue 0 |
| Button RESET | `"DB_RobotMove".Man.xReset` | same, momentary |

The HMI tag limits are the same numbers as `FC_PoseCheck`. The HMI refuses the value before the PLC has to. The PLC check stays, it is the one that counts.

### Robot, right

| Element | Tag |
|---|---|
| lamps | `"DB_Robot".Status.xReady`, `.xReferenced`, `.xAutomatic`, `.xAccessGranted`, `.xFault`, `.xCommOK`, `.xMoving`, `.xPrgRunning`, `.xPrgFinished` |
| output field message id | `"DB_Robot".Status.uiMsgId` |
| output fields actual pose | `"DB_Robot".Status.udtPoseBcs.rX`, `.rY`, `.rZ`, `.rRotZ`, `.rRotY` |
| output field remaining segments | `"DB_Robot".Status.intRemainingSeg` |

### Move state, bottom

| Element | Tag | Settings |
|---|---|---|
| lamps busy, done, error | `"DB_RobotMove".State.xBusy`, `.xDone`, `.xError` | done is a one-cycle pulse, you may not see it. Not a problem |
| output field step | `"DB_RobotMove".State.intStep` | text list `TL_Step` |
| output field error | `"DB_RobotMove".State.intErrId` | text list `TL_Err` |
| output fields run nr, block | `"DB_RobotMove".State.intRunNr`, `.intBlockActive` | |
| output fields pose at end | `"DB_RobotMove".State.udtPoseAtEnd.rX` … `.rRotY` | |
| output field message at error | `"DB_RobotMove".State.uiMsgIdAtError` | |
| I/O field blocks in program 20 | `"DB_RobotMove".Cfg.intBlocksInPrg20` | admin user level only. Start value 1 |

## 3. Text lists

**TL_Step**

| Value | Text |
|---|---|
| 0 | idle |
| 5 | check |
| 10 | write pose |
| 20 | settle |
| 30 | start |
| 40 | wait running |
| 50 | moving |
| 60 | end |
| 90 | error |

**TL_Err**

| Value | Text |
|---|---|
| 0 | no error |
| 1 | block not in program 20 |
| 2 | X Y Z outside limits |
| 3 | RotZ RotY outside limits |
| 4 | circle without radius |
| 5 | robot not ready / referenced / automatic / access |
| 6 | robot moving or program running |
| 7 | communication not ok |
| 8 | override is 0 |
| 10 | wrong block echoed |
| 11 | program did not start |
| 12 | move timeout |
| 13 | run number echo wrong |
| 14 | program 20 not ready on the robot |
| 15 | program stop not taken, repositioning stays |
| 20 | aborted: enable off |
| 21 | aborted: robot fault |
| 22 | aborted: access lost |

**TL_Block** (one line per block that exists in program 20; add a line when a block is added)

| Value | Text |
|---|---|
| 1 | linear |
| 2 | linear slow |
| 3 | circle ccw |
| 4 | circle cw |

## 4. Remove from the existing screens

| Element | Was writing | Now |
|---|---|---|
| Start button | `"DB_App".Cmd.Start` | delete. MOVE on this screen replaces it |
| program number field | `"DB_App".Cmd.ProgramNumber` | delete. Always 20, the PLC writes it |
| override field | `"DB_App".Cmd.Override` | delete. The override field on this screen writes `Man.udtReq.intOverride` |

Keep: reset, simulation, pause, stop, homing, jog. They still write `DB_App.Cmd` and the PLC does not touch them.

The automatic switch (2026-11-02): from step 1a of `AUTOMATIC_TODAY.md`, `FB_RobotMode` is the one writer of `"DB_App".Cmd.Automatic`. Delete the switch and show a lamp on `"DB_Robot".Status.xAutomatic` instead. Until then the switch stays: off for jogging, on for every move.

The old enable switch (2026-11-01): since 2026-10-29 `"DB_App".Cmd.Enable` is written by OB1 network 8 only, the switch of this screen is a requester branch of it through `"DB_RobotMove".Man.xEnable`. An old switch still bound to `"DB_App".Cmd.Enable` is a second writer, `BLOCK_MAP.md` B3. Bind it to `Man.xEnable` or delete it.

## 5. Later, not now

- Teach button "take actual pose into pose n" → `DB_Poses`. Comes with a small FC in the next step, so the HMI only presses a bit.
- Pose list screen for `DB_Poses` 1..50.
- One button per block instead of the block number field.
