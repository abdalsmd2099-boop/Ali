# Step 2 – UDTs, DB_Poses, FC_PoseCheck, status map

2026-10-27. Step 2 of GOALS.md section 8. Eight source files, all TIA V18 external sources.

## 1. Files and import order

Import in this order (each one needs the ones above it). TIA: *External source files → Add new external file → right click → Generate blocks from source*.

| # | File | Block | Needs |
|---|---|---|---|
| 1 | `UDT_Pose.udt` | `UDT_Pose` | – |
| 2 | `UDT_MoveReq.udt` | `UDT_MoveReq` v0.2 | UDT_Pose |
| 3 | `UDT_MoveState.udt` | `UDT_MoveState` | UDT_Pose |
| 4 | `UDT_RobotStatus.udt` | `UDT_RobotStatus` | UDT_Pose |
| 5 | `DB_Poses.scl` | `DB_Poses`, retentive | UDT_Pose |
| 6 | `DB_Robot.scl` | `DB_Robot` | UDT_RobotStatus |
| 7 | `FC_PoseCheck.scl` | `FC_PoseCheck` : Int | UDT_Pose |
| 8 | `FC_RobotStatusMap.scl` | `FC_RobotStatusMap` | UDT_RobotStatus |

The two DBs are `.scl` files as well: a `DATA_BLOCK` source generates a global DB the same way an `FB` source generates an FB.

If the compiler does not accept `VAR RETAIN` in `DB_Poses.scl`, delete the word `RETAIN`, generate the block, and tick "Retain" on the three variables in the DB editor. The content is the same.

## 2. What each one is

**UDT_Pose** – X, Y, Z, RotZ, RotY as Real, mm and deg. One type for everything: taught poses, move requests, the actual pose.

**UDT_MoveReq** – one move. Built so that later changes add fields at the end and change nothing else:

| Field | Stage 1 | Later |
|---|---|---|
| `intBlock` | 1 = linear | 2..8 = the other blocks of program 20 |
| `udtPose` | target pose | same |
| `udtPose2` | unused | via point or circle center, once POSE[52] is on the bus |
| `rBlend` | 0 | blending for continuous scans |
| `aorPar[1..3]` | 0 | circle radius in 1, other block parameters |
| `intOverride` | from the HMI field | per move from the job |
| `xStopOnSensor` | FALSE | laser stop |
| `intId` | 0 | list index or step number, for diagnostics |

**UDT_MoveState** – what the move engine reports: busy, done, error, error number, step, run number, active id and block, pose at the end, message id at the error. The HMI and the queue read it.

**UDT_RobotStatus** – every PI bit and value the PLC needs, with a name. Source: manual chapter 13.7 and the process data screenshots in the folder. PI 12 is "target pose index / remaining segments" on this machine because the pose has 5 words (RotY on).

**DB_Poses** – retentive. One array `aoudtP[1..50]`, a `aoxValid[1..50]` flag per pose, and `rSafeZ`. Fixed indexes:

| Index | Pose |
|---|---|
| 1 | P_Home |
| 2 | P_Start |
| 3 | P_Approach |
| 4 | P_C1 |
| 5 | P_C2 |
| 6 | P_C3 |
| 7 | P_C4 |
| 8–50 | free, stage 2 action 301–350 = linear to pose n |

Only the HMI writes it. Stage 2 reads it.

**DB_Robot** – `Status : UDT_RobotStatus`. Only `FC_RobotStatusMap` writes it.

**FC_PoseCheck** – returns 0 ok, 2 X Y Z outside, 3 RotZ RotY outside. Limits are constants inside the block, the ones of GOALS.md 1.4. Call it anywhere a pose is about to be used:

```scl
#tmp_intErr := "FC_PoseCheck"(i_udtPose := "DB_RobotMove".Req.udtPose);
```

**FC_RobotStatusMap** – OB30 network 4. Takes the raw PI array of the module 1 instance DB directly:

```scl
"FC_RobotStatusMap"(i_aowPI     := "DB_MOVIKIT_Robotics-5AX".PI,
                    i_xHsAlive  := "DB_App".HS_Alive,
                    i_xM2CommOK := "DB_App".Srl.CommOK,
                    o_udtStatus => "DB_Robot".Status);
```

`PI` is `Array[1..64] of Word`, index = robot word, so no shifting. Words 19..64 are padding and are not read.

## 3. The status map – settled

The PI words come from `"DB_MOVIKIT_Robotics-5AX".PI` (your screenshot of the instance DB, Static: `PI` and `PO`, each `Array[1..64] of Word`). Device view: slot 17 = `%IW/QW 275..278`, slot 18 = `%IW/QW 19..146`, slot 19 = `%IW/QW 147..274`. All recorded in GOALS.md section 1.

Your old network 4 is replaced by the call above. If it also did something else, tell me before you delete it.

### Bits FB_RobotMove needs from the status map

| Name in `UDT_RobotStatus` | PI bit | Used for |
|---|---|---|
| `xReady` | 1.0 | start check |
| `xReferenced` | 1.5 | start check |
| `xMoving` | 1.4 | start check and "finished" |
| `xFault` | 1.8 | start check, abort |
| `xAccessGranted` | 1.13 | start check, RobotMonitor took over |
| `xAutomatic` | 2.0 | start check |
| `xPrgRunning` | 2.3 | "running seen", rule 5 |
| `xPrgFinished` | 2.5 | "finished" |
| `intRemainingSeg` | 12 bits 8–15 | "finished" needs 0 |
| `xCommOK` | PLC side | start check, abort |
| `udtPoseBcs` | 7–11 | HMI, teach button, pose at the end |
| `uiMsgId` | 3 | HMI, alarm text |

## 4. What the HMI gives up

`FB_RobotMove` becomes the only writer of `DB_App.Cmd.Start`, `DB_App.Cmd.ProgramNumber` and `DB_App.Cmd.Override`. Before the first test of step 3, remove these three from the HMI. The HMI override field writes `DB_RobotMove.Man.udtReq.intOverride` instead. Everything else on the HMI stays.

## 5. Next

Step 3: `FB_RobotMove` in SCL against `DB_Robot.Status`, with the steps of GOALS.md 5.4, plus `DB_RobotMove` (Cfg, Man, Req, State) and the OB1 call.
