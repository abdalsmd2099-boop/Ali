# OB1 fix list, 2026-11-05 – the two rungs under the FB_MachineReady call

Why: the screenshots in `MachineReady.docx` show two coil rungs that write tags the design gives one writer.
`Cmd.Start` is written by the engine and by a rung from the MOVE button; `Cmd.Automatic` by `FB_RobotMode` and by a rung
from a screen switch. The start rung cuts the engine's start to one OB1 cycle or none. Both go. Half an hour in TIA,
ten minutes on the screen, five minutes of test.

## A. Before you touch anything

| # | Do | ☐ |
|---|---|---|
| A1 | Note the network number of the `FB_MachineReady` call and of the two rungs | ☐ |
| A2 | HMI: note which screen objects are on `"DB_RobotHand".xAutomatic_from_Screen`, `"DB_App".Cmd.Reset`, `"DB_RobotMove".Man.xStart` (tag table → where used) | ☐ |
| A3 | Recipe numbers written down, as before every download | ☐ |

## B. TIA, OB1

| # | Do | How | ☐ |
|---|---|---|---|
| B1 | Delete the start rung | The rung with the contacts `"DB_App".MachineOn`, `"DB_RobotMove".Man.xStart` and the coil `"DB_App".Cmd.Start`. Click the coil → Delete, then the two contacts. If the rung is a network of its own, delete the network | ☐ |
| B2 | Delete the automatic rung | The rung with `"DB_RobotHand".xAutomatic_from_Screen`, NC `"IDB_RobotMode".xManualWanted`, NC `"DB_RobotHand".xHandModeJust` and the coil `"DB_App".Cmd.Automatic`. Same way | ☐ |
| B3 | Wire the robot mode box | Network "Robot Mode", `FB_RobotMode` box, pin `io_xAutomatic`: type `"DB_App".Cmd.Automatic`. The screenshot shows it not on that tag | ☐ |
| B4 | `DB_RobotHand` v0.2 | Open the DB, in `Btn` add the last row `xResetRobot`, Bool, false. Or import `PLC/DB_RobotHand.scl` | ☐ |
| B5 | `FB_MachineReady` v0.2 | External source files → add `PLC/FB_MachineReady.scl` → Generate blocks from source → overwrite. `DB_MachineReady` updates with the compile | ☐ |
| B6 | The call | New pin at the end, `ResetReq` := `"DB_RobotHand".Btn.xResetRobot`. If the box says "interface changed": right-click → Update block call. `Reset` stays on `"DB_App".Cmd.Reset`, `Enable` and `Automatic` stay open | ☐ |
| B7 | Compile → Software (rebuild all) | 0 errors | ☐ |
| B8 | Cross-references, three tags, writers only | `"DB_App".Cmd.Start`: `IDB_RobotMove` only. `"DB_App".Cmd.Automatic`: `IDB_RobotMode` only. `"DB_App".Cmd.Reset`: `DB_MachineReady` only. A screen in the list as a writer → section C | ☐ |
| B9 | Download, CPU restart when asked. Type the recipe numbers again | | ☐ |

## C. HMI

| # | Do | ☐ |
|---|---|---|
| C1 | RESET button of the old robot screen: tag `"DB_RobotHand".Btn.xResetRobot`, set bit while pressed | ☐ |
| C2 | The automatic switch on `xAutomatic_from_Screen`: delete it. A lamp on `"DB_Robot".Status.xAutomatic` shows the mode (the JOB box has it, A48) | ☐ |
| C3 | No object anywhere writes `"DB_App".Cmd.Start` or `.Cmd.Automatic`. The MOVE button stays on `"DB_RobotMove".Man.xStart` | ☐ |
| C4 | Compile and load the screen | ☐ |

## D. Test, five minutes

| # | Do | Done when | ☐ |
|---|---|---|---|
| D1 | Machine on | `ReadyState` 90, `MachineReady` on, `MissingText` "READY TO START" | ☐ |
| D2 | Jog one axis, release | automatic lamp off while jogging, on again about half a second after release | ☐ |
| D3 | Hand MOVE to a taught pose, three times | done, error 0, with no hand step on `Cmd.Start` | ☐ |
| D4 | Machine off and on, then MOVE | the move runs. If error 11: note `State.uiMsgIdAtError`, press RESET, MOVE again without any hand step, note whether it runs. Both go into `TEST/ERROR_LOG.md` | ☐ |
| D5 | Enable switch off while on | `MachineReady` off, `ReadyState` 30, `MissingText` "NOT POWERED". Enable on: 40, then 90 by itself | ☐ |
| D6 | RESET on the screen with a robot fault present | the fault clears, `Cmd.Reset` pulses once in the watch table | ☐ |

Never again: `Cmd.Start` set by hand in a watch table. With "repositioning required" pending the edge drives the robot
outside the engine.
