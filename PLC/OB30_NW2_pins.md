# OB30 network 2 – the segment pins take the path from DB_RobotPath

2026-11-02, path mode, part 7 of `AUTOMATIC_TODAY.md`. OB30 network 2 is your call of the module 1 FB. Today its seven standard path segments are fed from `"DB_App".Path.Seg[n]`, which nothing of ours writes. With the path mode the engine (v0.6) writes the seven path poses into `"DB_RobotPath"`, and the segment pins read them from there. Only the tags on the pins change, nothing else in the network.

Your network already converts with `REAL_TO_INT(...)` on every pose pin. Keep that: the segments carry whole mm and deg, like POSE 51.

## The 35 pose pins

| Pin | Was | Becomes |
|---|---|---|
| `Pose_X_1` | `REAL_TO_INT("DB_App".Path.Seg[1].X)` | `REAL_TO_INT("DB_RobotPath".aoudtP[1].rX)` |
| `Pose_Y_1` | `… Seg[1].Y` | `REAL_TO_INT("DB_RobotPath".aoudtP[1].rY)` |
| `Pose_Z_1` | `… Seg[1].Z` | `REAL_TO_INT("DB_RobotPath".aoudtP[1].rZ)` |
| `Pose_R1_1` | `… Seg[1].R1` | `REAL_TO_INT("DB_RobotPath".aoudtP[1].rRotZ)` |
| `Pose_R2_1` | `… Seg[1].R2` | `REAL_TO_INT("DB_RobotPath".aoudtP[1].rRotY)` |
| `Pose_X_2` … `Pose_R2_2` | `Seg[2]` | `aoudtP[2]`, same five fields |
| `Pose_X_3` … `Pose_R2_3` | `Seg[3]` | `aoudtP[3]` |
| `Pose_X_4` … `Pose_R2_4` | `Seg[4]` | `aoudtP[4]` |
| `Pose_X_5` … `Pose_R2_5` | `Seg[5]` | `aoudtP[5]` |
| `Pose_X_6` … `Pose_R2_6` | `Seg[6]` | `aoudtP[6]` |
| `Pose_X_7` … `Pose_R2_7` | `Seg[7]` | `aoudtP[7]` |

R1 is the rotation about Z, R2 the rotation about Y, the same order as the jog pins `J4R1`, `J5R2` and as the status map. Check it once: after the first path move, POSE 1 in the RobotMonitor variable list must show the RotZ of corner 2 in its RotZ field.

## The blending pins

| Pin | Was | Becomes |
|---|---|---|
| `blending_distance_2` … `blending_distance_7` | `REAL_TO_INT("DB_App".Path.Seg[n].Blend)` | `REAL_TO_INT("DB_RobotPath".aorBlend[n])` |
| `blending_distance_1` | not in your network of 2026-10-29 | add it the same way if the FB has the pin |

The engine writes 0 into every `aorBlend[n]` today, exact stops. The blending becomes a number when the laser wants continuous motion through the corners, see README section 11.

## The pins that stay as they are

`Motion_Set_n`, `wait_n`, `path_end_n` keep `"DB_App".Path.Seg[n].MotionSet`, `.Wait`, `.PathEnd`. They belong to the standard program of RobotMonitor, which uses the segments through wait and end signals. Our blocks 5, 6 and 7 use the segment poses as plain variables with their own LIN and CIRC lines and their own SET_MOTIONSET, so these three pins have no effect. They are FALSE and 0 today, and the single move works with them, which shows they do not block a program start.

## One writer

`"DB_RobotPath".aoudtP` and `.aorBlend`: written by `IDB_RobotMove` through its InOut pins, read by OB30 network 2. `"DB_App".Path.Seg[n].X … R2` and `.Blend` are not used any more by anything. Leave them in DB_App.
