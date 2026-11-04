# Error log – what the machine showed, what it was, what fixed it

One row per error seen at the machine. Newest on top. The numbers: engine = `"DB_RobotMove".State.intErrId`
(`GOALS.md` 5.5, text list TL_Err), job = `"DB_RobotJob".State.intErrId` (TL_JobErr), engine number inside
the job = `"DB_RobotJob".State.intEngineErr`. Before you read a number, check which tag the field is on:
`HMI/SCREEN_FIXES.md` lists the fields that are on the wrong tag today.

## Open

| Since | Seen | Cause | Status |
|---|---|---|---|
| 2026-11-02 | engine 11 "no program running within 3 s" after a download or a machine off/on. Goes away after `"DB_App".Cmd.Start` is set TRUE by hand once in a watch table | not proven. Suspects: a second writer of `Cmd.Start` in OB1 or on a screen, or ReadyToStart / MachineOn FALSE on the OB30 start pin at that moment | **open.** Cross-reference of `Cmd.Start`: the only writer may be `IDB_RobotMove`. During a START watch the three inputs of the OB30 start pin and PO word 2 bit 2 with `State.intStep` 30 … 40. `FB_StartTrace` is in `PLC/` for that |
| 2026-11-04 | engine 15 "program stop not taken" at the first START after jogging and teaching | the jog after "finished" set "repositioning required" (manual 6.3). Engine step 15 sent the stop, the robot did not show initialized without the request within 2 s | **open.** First check: OB30 network 2 stop pin must be `"DB_App".Cmd.Stop OR "DB_RobotMove".State.xPrgStop`. If it is `Cmd.Stop` alone the stop never reaches the robot. Workaround used: machine off/on, or a program stop from RobotMonitor, then ACK and START without a jog in between |

## Solved

| Date | Seen | Cause | Fix |
|---|---|---|---|
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
