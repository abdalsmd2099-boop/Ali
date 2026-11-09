# Stage 2 plan – from the single movement to the automatic cube scan

2026-10-28. Starts when stage 1 passes `TEST_SHEET.md` part B. Each step is built, imported, tested, then the next one. `FB_RobotMove` does not change in any of them.

```
step 4   teach + pose list          operator stores poses without typing numbers
step 5   FB_MoveQueue               a list of moves the engine eats one by one
step 6   FC_MoveSelect stage 2      auto = queue, hand = hmi
step 7   FB_ScanCubeJob             fills the queue with the cube scan
step 8   program 20 blocks 2..4     slow linear, circle ccw, circle cw
step 9   StepChain framework        modes, jobs, chain, FB_ActRobot
step 10  faults and recovery        every stop ends clean, every alarm has a number
step 11  laser sensor               approach and corner from the sensor
```

---

## Step 4 – teach and the pose list

**Why first:** the cube job needs 7 poses. Typing 35 numbers is slow and error-prone. Teaching is one button.

| Block | What |
|---|---|
| `FC_Teach` | OB1. Input: `i_xTeach` (HMI button, momentary), `i_intPoseNr` (1..50), `i_xAllowed` (hand mode and not busy), `i_udtActual` (`DB_Robot.Status.udtPoseBcs`). Output: `io_udtPoses` (`DB_Poses`). On the edge: `aoudtP[n] := actual pose`, `aoxValid[n] := TRUE`. Range check on n on its own line. |
| HMI screen "Poses" | table of `DB_Poses.aoudtP[1..10]` with valid flag, pose number field, Teach button, "go to pose n" button that copies `aoudtP[n]` into `Man.udtReq.udtPose` and presses MOVE (HMI script or a second FC input `i_xGoTo`). `rSafeZ` field. |

**Done when:** teach pose 2 at the real start position, power off and on, pose 2 still there and valid. "Go to pose 2" moves the robot there. All 7 cube poses taught and valid.

---

## Step 5 – FB_MoveQueue

**What:** a FIFO of `UDT_MoveReq`, 50 deep. The engine takes the head, the job pushes at the tail.

| Pin | Meaning |
|---|---|
| `i_xPush`, `i_udtPush` | one move in at the tail, edge |
| `i_xPop` | the head is done, drop it. Wired to `DB_RobotMove.State.xDone` |
| `i_xClear` | empty the queue (abort, new job) |
| `o_udtHead` | the move at the head |
| `o_xHeadValid` | TRUE when the queue is not empty |
| `o_intCount`, `o_xFull` | for the HMI and the job builder |
| `io_udtQueue` | `DB_MoveQueue`: `aoudtMove[1..50]`, `intHead`, `intTail`, `intCount` |

Rules: push on a full queue is refused and reported, never overwritten. Head and tail wrap at 50. Index range checked on its own line.

**Done when:** in a watch table: push 3 moves, head = move 1, pop, head = move 2, pop, pop, head invalid. Push 51 → refused at 51. Clear → empty.

---

## Step 6 – FC_MoveSelect, stage 2 body

**What:** the only change in the request path. Hand = HMI, auto = queue.

```
IF i_xAuto THEN
   o_udtReq  := queue head
   o_xStart  := head valid AND NOT State.xBusy AND NOT State.xError
ELSIF i_xHand THEN
   o_udtReq  := Man.udtReq
   o_xStart  := Man.xStart
ELSE
   o_xStart  := FALSE
END_IF
```

`i_xHand` and `i_xAuto` come from an HMI switch pair first, from `FB_Mode` in step 9. The two can never be TRUE at the same time: auto wins, and a mode change while busy aborts through `i_xEnable` of the engine.

**Done when:** push two moves by watch table, switch to auto: the robot runs move 1, then move 2, then stops. Switch to hand during move 1: abort, error 20, queue still holds move 2.

---

## Step 7 – FB_ScanCubeJob

**What:** turns the list of GOALS.md section 3 into queue entries. No motion code, only data.

| Pin | Meaning |
|---|---|
| `i_xStart` | start the job, edge |
| `i_xAbort` | clear the queue, engine aborts through enable |
| `i_udtPoses` | `DB_Poses` |
| `i_udtMoveState` | `DB_RobotMove.State` |
| `i_intOverride` | speed for the whole job |
| `o_xRunning`, `o_xDone`, `o_xError`, `o_intErrId`, `o_intStep` | job state for the HMI |
| `io_udtQueue` + the push pins of `FB_MoveQueue` | |

Sequence inside: check poses 2..7 valid and `rSafeZ` > 0 (else error, nothing pushed) → clear queue → push 13 moves: start, approach, C1, C2, turn, C3, turn, C4, turn, C1, lift, start, home → wait until the queue is empty and the engine is not busy → done. A "turn" is C_n with RotZ + 90°, computed in the block, wrapped into −180..180.

**Done when:** one button runs the whole cube scan in simulation, then real at 10 %. Abort in the middle: robot stops, queue empty, next start begins from the start pose again. 20 runs without a miss.

---

## Step 8 – program 20, blocks 2 to 4

| Block | SRL | PLC |
|---|---|---|
| 2 | `IF BOOL[52]` linear with the slow motion set, `REAL[71] := 2` | `Cfg.intBlocksInPrg20` = 2, text list line |
| 3 | `IF BOOL[53]` `CIRC_VAR` ccw, radius `REAL[51]`, `REAL[71] := 3` | 3, text list line |
| 4 | `IF BOOL[54]` `CIRC_VAR` cw, `REAL[71] := 4` | 4, text list line |

Decide at this step whether the circle needs `POSE[52]` (via point or center). If yes: MOVISUITE pose setpoints 1 → 2, three numbers in OB30 network 3, `udtPose2` is already in the request.

**Done when:** each block runs from the HMI with the block number, echo = block number, limits respected. Error 4 when a circle has radius 0.

---

## Step 9 – StepChain framework around it

Copy from the drum press, as in GOALS.md 4.2: `FB_Mode`, `FC_Mode`, `FB_Jobs`, `FB_OperBox`, `FB_SeqMechanism` with its FCs and DBs. Fix the two known bugs before copying (`FB_Mode` reset timer, `FB_OperBox` last action). Then:

| New | What |
|---|---|
| `FB_ActRobot` | one block for all robot step numbers. `intActStepNr` 301..350 → push "linear to pose n"; 351 → push "lift to SafeZ"; sets the done bit when `State.xDone` arrives for its id. Uses `UDT_MoveReq.intId` = step number |
| job "scan cube" | a recipe of action numbers: 10, 11, 13, 351, 302, 303, 304, 305, (turn), 306, … 302, 301, 2. Replaces `FB_ScanCubeJob` or keeps it as the recipe source, your choice at that time |
| `FC_MoveSelect` | `i_xHand` / `i_xAuto` from `udtMode.sttBit` |

**Done when:** Start, Stop, step by step and tap to step behave as on the drum press; the robot never moves in Standby; the A1 screen offers "scan cube" only when poses are taught and comm is OK.

---

## Step 10 – faults and recovery

| Case | Must happen |
|---|---|
| comm loss | engine error 7, queue cleared, alarm with step number |
| robot fault | error 21, same |
| E-stop | F-program stops the drives; engine sees fault or access loss, ends clean |
| stop in the middle | after Reset, the next job starts with the lift move (351) before anything else |
| job timeout | per move 60 s, per job a sum |

`FB_Alarms`: one alarm per error number, text from `TL_Err`, plus the robot message ID. **Done when:** every case in the table was provoked once and recovered with Reset and a new start.

---

## Step 11 – laser sensor

Only after the sensor is chosen. Two candidates, decided then:

1. **PLC side:** move type "linear until sensor": `xStopOnSensor` TRUE, `i_xSensorStop` from the sensor; the engine stops the program, latches `udtPoseAtEnd`, reports done. Needs a test of how program 20 reacts to a stop mid-move (manual 9.8: stop → program initialized at standstill).
2. **Robot side:** MOVIKIT touchprobe / MEASURE instruction, the robot latches the pose itself into an SRL pose variable the PLC reads.

**Done when:** approach stops at 10 mm from the cube; the corner search finds the corner within 1 mm, 20 times.

---

## What never changes in all of this

`FB_RobotMove`, the UDTs' existing fields, OB30, the program 20 contract. If one of these has to change, the plan is wrong and we stop and look.
