# Step 3 – FB_RobotMove, DB_RobotMove, FC_MoveSelect, the OB1 call

2026-10-28. Step 3 of GOALS.md section 8. Needs the step 2 blocks in the project first.

## 1. Files and import order

| # | File | Block | Needs |
|---|---|---|---|
| 1 | `DB_RobotMove.scl` | `DB_RobotMove` | UDT_MoveReq, UDT_MoveState |
| 2 | `FC_MoveSelect.scl` | `FC_MoveSelect` | UDT_MoveReq |
| 3 | `FB_RobotMove.scl` | `FB_RobotMove` | UDT_MoveReq, UDT_MoveState, UDT_RobotStatus, FC_PoseCheck |
| – | `OB1_RobotMove_call.scl` | wiring reference only, not for import | – |

`DB_RobotMove.scl` has a `VAR RETAIN` section (Cfg) and a `VAR` section. If TIA rejects the mix, make the whole DB non-retain and tick Retain on `Cfg.intBlocksInPrg20` in the editor.

## 2. First compile – check these types

`FB_RobotMove` writes into `DB_App` through InOut pins. I do not have `DB_App` in the folder, so the pin types are assumptions. On the first compile, compare:

| Pin | Assumed | Your `DB_App` |
|---|---|---|
| `io_aorSrlPose` | `Array[1..5] of Real` | `Srl.Pose` |
| `io_aorSrlPar` | `Array[1..4] of Real` | `Srl.Par` |
| `io_aoxSrlGate` | `Array[1..8] of Bool` | `Srl.Gate` |
| `io_xGateEnable` | `Bool` | `Srl.GateEnable` |
| `io_xStart` | `Bool` | `Cmd.Start` |
| `io_intPrgNr` | `Int` | `Cmd.ProgramNumber` |
| `io_intOverride` | `Int` | `Cmd.Override` |
| `i_intEchoBlock`, `i_intEchoRun` | `Int` | `"DB_MOVIKIT_ROBOTICS_Module2".SrlRealRaw[1]`, `[2]` |

If one differs, change that pin's type in the FB header and the matching lines in step 10. Or put the `DB_App` declaration (Srl and Cmd) in the folder and I align the FB.

## 3. OB1 – one new network, two boxes

OB1 is LAD. Put `FC_MoveSelect` and then `FB_RobotMove` (single instance `IDB_RobotMove`) in one network. The pin wiring is in `OB1_RobotMove_call.scl`. OB30 does not change.

Order matters inside the network: the selector first, the engine second, so the engine sees this cycle's request.

## 4. How a move runs

| Step | Name | What the block does |
|---|---|---|
| 0 | idle | waits for the start edge with enable on and no error |
| 5 | check | block 1..`Cfg.intBlocksInPrg20`, pose inside the limits, comm OK, ready, referenced, automatic, access granted, no fault, not moving, program not running. One cycle, nothing on the bus |
| 10 | write | program number 20, pose → `Srl.Pose`, parameters → `Srl.Par[1..3]`, run number → `Srl.Par[4]`, override if > 0, all gates off, start off |
| 20 | settle | 50 ms, the pose is on the bus |
| 30 | go | gate of the block on, start on |
| 40 | running | waits for "program running". Also accepts "finished" with this run's number already in REAL[72], for a move too short to be seen running. 2 s → error 11 |
| 50 | finished | "program finished" and not moving and remaining segments 0, stable for 200 ms. 60 s → error 12 |
| 60 | end | start off, gates off, pose at the end latched, REAL[71] must equal the block (else 10), REAL[72] must equal the run number (else 13), done pulse |
| 90 | error | start off, gates off, error stays until reset. Message ID and pose at the error are latched |

Abort from steps 10 to 50 when comm is lost (7), the robot faults (21), access is lost to RobotMonitor (22), or enable falls (20).

Start is held from step 30 until step 60. If you see program 20 run a second time while start is still on, tell me: then start is dropped as soon as "running" is seen. That is one line in step 40.

## 5. HMI tags for stage 1

All on `DB_RobotMove`:

| Tag | Element | Meaning |
|---|---|---|
| `Man.xEnable` | switch | manual moves allowed. Off = abort and no start |
| `Man.xStart` | button, set on press / reset on release | move |
| `Man.xReset` | button, same | clear the error |
| `Man.udtReq.intBlock` | number or one button per block | 1 = linear |
| `Man.udtReq.udtPose.rX` … `rRotY` | five input fields | target, whole mm / deg |
| `Man.udtReq.intOverride` | input field | 1..100 %. 0 = keep the last |
| `Cfg.intBlocksInPrg20` | input field, admin | how many blocks program 20 has today |
| `State.xBusy`, `xDone`, `xError`, `intErrId`, `intStep` | displays | the move |
| `State.intRunNr`, `intBlockActive` | displays | which move |
| `State.udtPoseAtEnd` | displays | where it ended |
| `State.uiMsgIdAtError` | display | robot message at the error |
| `"DB_Robot".Status.udtPoseBcs` | displays | actual pose |

The HMI stops writing `DB_App.Cmd.Start`, `Cmd.ProgramNumber` and `Cmd.Override`. The override field writes `Man.udtReq.intOverride`.

## 6. Program 20 for the first test

Block 1 only, so `Cfg.intBlocksInPrg20` = 1:

```
IF  BOOL[51] = TRUE
    LIN_VAR  POSE[51]
    REAL[71] := 1
END_IF
REAL[72] := REAL[54]
```

## 7. Tests

GOALS.md 5.7, in this order: 1 (limits), 2 (simulation), 3 (real, 10 %), 4, 5, 6, 7. Write down the result of each one.

## 8. Next

Stage 2 starts with `FB_MoveQueue` and the stage 2 body of `FC_MoveSelect`. `FB_RobotMove` does not change.
