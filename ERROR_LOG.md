# Error log – what the machine showed, what it was, what fixed it

One row per error seen at the machine. Newest on top. The numbers: engine = `"DB_RobotMove".State.intErrId`
(`GOALS.md` 5.5, text list TL_Err), job = `"DB_RobotJob".State.intErrId` (TL_JobErr), engine number inside
the job = `"DB_RobotJob".State.intEngineErr`. Before you read a number, check which tag the field is on:
`HMI/SCREEN_FIXES.md` lists the fields that are on the wrong tag today.

## Open

| Since | Seen | Cause | Status |
|---|---|---|---|
| 2026-11-04 | engine 15 "program stop not taken" at the first START after jogging and teaching | the jog after "finished" set "repositioning required" (manual 6.3). Engine step 15 sent the stop, the robot did not show initialized without the request within 2 s | **open.** First check: OB30 network 2 stop pin must be `"DB_App".Cmd.Stop OR "DB_RobotMove".State.xPrgStop`. If it is `Cmd.Stop` alone the stop never reaches the robot. Workaround used: machine off/on, or a program stop from RobotMonitor, then ACK and START without a jog in between |

## Solved

| Date | Seen | Cause | Fix |
|---|---|---|---|
| 2026-11-09 (open since 11-06) | **first drum start**, job 2, AUTO, START: moves 1 "to start" and 2 "above P0" ran (block 1, run numbers 193 and 194 echoed), then step 90, job error **10**, MoveActive 2, BlockActive 1, moves total 10 (watch table photos). On the screens the same stop: "block prog 20" = 1, Err nr "Block not in the right prog" = engine 1, the robot standing at 2457 / 1454 / 500 = exactly "above P0" for center 2500 / 1700, radius 250, offset 100, safe Z 500 | `"DB_RobotMove".Cfg.intBlocksInPrg20` was **1** in the same watch table. Move 3 "down to P0" is block 2, the engine's first check in step 5 refuses a block above the number: **engine error 1**. The field is retentive but a download of `DB_RobotMove` sets its start value again, and that was 1. Note for the next time: on a refusal in step 5 (engine 1 … 8) the active number is the move before the refused one, here 2 for move 3; `Drum.udtState.intEngineErr` and `"DB_RobotMove".State.intErrId` in the watch table give the engine number directly | the field to **4** in the watch table (`DB_RobotMove` v0.2 start value 4, so a download does not bring 1 back; 7 once blocks 6 and 7 are in), program 20 N80 echo into REAL 71 and N100 deleted, ACK, START. **Ran through at the machine 2026-11-09 (your report; run numbers and the poses after moves 2 and 4 not recorded).** The screen faults seen on 11-06 stay in `HMI/SCREEN_FIXES.md` section 6 |
| 2026-11-05 (open since 11-02) | engine 11 "no program running within 3 s" at the first move after a download or a machine off/on. Went away once after `Cmd.Start` was set TRUE by hand | a second writer of `"DB_App".Cmd.Start`: the OB1 coil rung `MachineOn AND Man.xStart → Cmd.Start` under the `FB_MachineReady` call. It cut the engine's start to one OB1 cycle or none, and its own edge was the one the robot refused with 125.57 on 11-03 | `Cmd.Start` written by `IDB_RobotMove` only (cross-reference as in `CORRECTNESS.md` 4.4, `OB1_FIX_RUNGS.md` B1 and B8), `FB_MachineReady` v0.2 loaded. **Proven at the machine 2026-11-05: the first cube scan in automatic ran through.** If 11 comes back after a download: RESET and a second MOVE without any hand step, note `State.uiMsgIdAtError` (32057 = refused, not lost) and `PO[4]` low byte before the first MOVE. Never set `Cmd.Start` by hand |
| 2026-11-04 | first START in automatic refused at once, queue 0, screen text "engine busy" | job error **4**, a pose outside the limits, MoveActive "turn at corner 4": corner 1 taught at RotZ 0, the four +90 turns reach +270. The screen text list showed the text of 3 under number 4 | teach corner 1 at RotZ −180, or the NO TURN switch `"DB_Recipe".xNoTurn` (cube job v0.6): no turns, 10 moves, corner 1 at any RotZ. Text list TL_JobErr 4 corrected |
| 2026-11-04 | angle 100.00 in the CUBE box | typed by mistake. 100 turns the computed corners 100 degrees away from +X | angle 0 when the cube lies at +X and +Y of corner 1. The angle table is in `CHECKLIST_CUBE_AUTO.md` stage D0 |
| 2026-11-03 | "program finished" shown after a jog from the screen, machine off/on used to clear it | not an error. Finished is the normal state of program 20 between two moves, the next start works from it (manual 6.3, 9.7). The real point after a jog is "repositioning required", see engine 15 above | nothing to clear. Engine v0.5 step 15 handles the repositioning request |
| 2026-11-03 | single move refused, engine 11 | start did not reach the robot | `"DB_App".Cmd.Start` set by hand once. Root cause open, first row above |
| 2026-11-02 | engine 11 right after the robot went to automatic, old automatic switch still on the screen | program state "Initialized", no old path, so not repositioning. A blocked start: ReadyToStart or MachineOn on the OB30 start pin | `FB_RobotMode` became the one writer of `Cmd.Automatic`, the screen switch went. The start suspect stayed, first row above |
| before 11-02, photo | engine 5 "not ready / not referenced / not automatic" right after a jog | the robot is still in manual half a second after the jog, `FB_RobotMode` goes back to automatic 500 ms after standstill | wait a second after the jog, ACK, press again |
| before 11-02, photo | engine 14 "program 20 not ready" | program number on the bus not 20, or the program neither initialized nor finished (RobotMonitor: no program selected, or control type active) | select program 20 on the robot, control type inactive, program ready |

## How to add a row

Date, what the screen showed (numbers, not words), the cause once it is known, the fix. An error that is
not understood goes to "Open" with the next check to do. When the fix is proven at the machine, move the
row to "Solved" and put the fix into `GOALS.md` as a dated change.
