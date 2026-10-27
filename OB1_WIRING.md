# OB1 — the wiring, network by network, every pin

Notation is the SCL call: `pin := tag` for an input, `pin => tag` for an output. In LAD the same tag goes on the same pin of the box. Every box has EN unconditional: no contact in front of it, every block runs every cycle.

Two kinds of entries:

- **Exact.** `FC_ScanControl` and `FC_CmdArbiter` are mine, their pin names are in `plc/BGE_Scan.scl`. The tags on them are `DB_App` as `DB_MAP.md` defines it, `DB_Mode` as your V128 framework names it, and the instance DBs.
- **By role.** Your four existing blocks, `FB_Mode`, `FB_MachineReady`, `FC_DriveHome`, `FB_OperBox`. I have not seen their interfaces, so the left column is the role of the pin and the right column is the tag that role must carry. The pin name is whatever your block has.

---

## Network 1 — device module, slot 17

Existing. The MOVIRUN device module FB. Keep as it is.

## Network 2 — modes, your `FC_Mode` / `FB_Mode`

| Pin role | Tag |
| --- | --- |
| mode buttons in | your `udtModeBtn` tags, set by the HMI, cleared by the block |
| mode number out | `"DB_Mode".intMode` |
| hand bit out | `"DB_Mode".sttBit.sttHand.xHand` |
| auto bit out | `"DB_Mode".sttBit.sttAuto.xAuto` |

Nothing else writes `DB_Mode`.

## Network 3 — control ON and safety release

Existing. Produces the Control ON bit and the safety release bit in your existing tags. Only network 4 reads them.

## Network 4 — `FB_MachineReady`, instance `"DB_MachineReady"`

| Pin role | Tag |
| --- | --- |
| control on in | the Control ON bit from network 3 |
| safety release in | the safety release bit from network 3 |
| machine start press in | the MACHINE START tag of your HMI, press, the block clears it |
| comm ok in | `"DB_App".Status.CommOk` |
| enabled in | `"DB_App".Status.Enabled` |
| automatic in | `"DB_App".Status.Automatic` |
| referenced in | `"DB_App".Status.Referenced` |
| safe stop in | `"DB_App".Status.SafeStop` |
| fault in | `"DB_App".Status.Fault` |
| enable request out | `"DB_App".Req.Machine.Enable` |
| reset request out | `"DB_App".Req.Machine.Reset` |
| automatic request out | `"DB_App".Req.Machine.Automatic` |
| ready out | stays in `"DB_MachineReady"`, read by network 7 and the HMI |
| missing number out | stays in `"DB_MachineReady"`, read by the HMI and network 9 |

Change from today: the three request outputs went to `Cmd.Enable`, `Cmd.Reset`, `Cmd.Automatic`. They go to `Req.Machine` now. `Cmd` has one writer, network 8.

## Network 5 — `FC_DriveHome`

| Pin role | Tag |
| --- | --- |
| hand in | `"DB_Mode".sttBit.sttHand.xHand` |
| HOME press in | the HOME tag of your HMI |
| referenced in | `"DB_App".Status.Referenced` |
| home request out | `"DB_App".Req.Hand.Home` |

## Network 6 — `FB_OperBox`, instance `"DB_OperBox"`

| Pin role | Tag |
| --- | --- |
| hand in | `"DB_Mode".sttBit.sttHand.xHand` |
| axis select in | the axis selection of your HMI, 1 X, 2 Y, 3 Z, 4 R1, 5 R2 |
| jog − press in | the JOG − momentary tag of your HMI |
| jog + press in | the JOG + momentary tag of your HMI |
| axis out | `"DB_App".Req.Hand.Axis` |
| jog minus out | `"DB_App".Req.Hand.JogMinus` |
| jog plus out | `"DB_App".Req.Hand.JogPlus` |

## Network 7 — `FC_ScanControl`, exact

```
"FC_ScanControl"(i_xAuto         := "DB_Mode".sttBit.sttAuto.xAuto,
                 i_xMachineReady := "DB_MachineReady".<ready output>,
                 i_xProgFinished := "DB_App".Status.ProgFinished,
                 i_xRobotFault   := "DB_App".Status.Fault,
                 i_xCommOk       := "DB_App".Status.CommOk,
                 i_iReal71       := "DB_App".Status.SrlRealRaw[1],
                 i_iReal73       := "DB_App".Status.SrlRealRaw[3],
                 i_iActX         := "DB_App".Status.ActX,
                 i_iActY         := "DB_App".Status.ActY,
                 i_iActZ         := "DB_App".Status.ActZ,
                 i_iActRotZ      := "DB_App".Status.ActRotZ,
                 i_iActRotY      := "DB_App".Status.ActRotY,
                 i_xMeasDone     := <scanner done input, or TRUE until a scanner exists>,
                 o_xStart        => "DB_App".Req.Auto.Start,
                 o_iProgNo       => "DB_App".Req.Auto.ProgNo,
                 o_xStop         => "DB_App".Req.Auto.Stop,
                 o_xScanOn       => <scanner record output, or open>,
                 o_xMeasTrigger  => <scanner trigger output, or open>,
                 o_iRowTag       => <scanner row tag, or open>,
                 o_iJobStatus    => <open, or an input of FB_Alarms>,
                 o_iJobError     => <open, or an input of FB_Alarms>);
```

Inside, it reads and writes `DB_HMI`, `DB_Scan`, `DB_Job`, `DB_SrlLane` and runs `DB_ScanJob`. None of those appear on pins.

## Network 8 — `FC_CmdArbiter`, exact, the only writer of `DB_App.Cmd`

```
"FC_CmdArbiter"(i_xHand         := "DB_Mode".sttBit.sttHand.xHand,
                i_xAuto         := "DB_Mode".sttBit.sttAuto.xAuto,
                i_xReqEnable    := "DB_App".Req.Machine.Enable,
                i_xReqReset     := "DB_App".Req.Machine.Reset,
                i_xReqAutomatic := "DB_App".Req.Machine.Automatic,
                i_iReqAxis      := "DB_App".Req.Hand.Axis,
                i_xReqJogMinus  := "DB_App".Req.Hand.JogMinus,
                i_xReqJogPlus   := "DB_App".Req.Hand.JogPlus,
                i_xReqHome      := "DB_App".Req.Hand.Home,
                i_xReqStart     := "DB_App".Req.Auto.Start,
                i_iReqProgNo    := "DB_App".Req.Auto.ProgNo,
                i_xReqStop      := "DB_App".Req.Auto.Stop,
                i_iOverride     := "DB_HMI".Override,
                o_xEnable       => "DB_App".Cmd.Enable,
                o_xReset        => "DB_App".Cmd.Reset,
                o_xAutomatic    => "DB_App".Cmd.Automatic,
                o_xStart        => "DB_App".Cmd.Start,
                o_xStop         => "DB_App".Cmd.Stop,
                o_iProgNo       => "DB_App".Cmd.ProgNo,
                o_iOverride     => "DB_App".Cmd.Override,
                o_iAxis         => "DB_App".Cmd.Axis,
                o_xJogMinus     => "DB_App".Cmd.JogMinus,
                o_xJogPlus      => "DB_App".Cmd.JogPlus,
                o_xHome         => "DB_App".Cmd.Home,
                o_iOwner        => <open, or an HMI tag for the owner text>);
```

What it does: owner 2 when Auto alone, 1 when Hand alone, else 0. Enable, Reset, Automatic pass in every owner. Standby: Stop TRUE, everything else FALSE. Hand: Stop FALSE, axis and one jog direction at a time, home. Auto: Start, ProgNo, Stop from the scan. Override capped to 1 to 60, a 0 becomes 10.

## Network 9 — `FB_Alarms`

| Pin role | Tag |
| --- | --- |
| job status in | `"DB_Job".Status` |
| job error in | `"DB_Job".Error` |
| job row in | `"DB_Job".RowIdx` |
| job fault in | `"DB_ScanJob".o_xFault` |
| robot fault in | `"DB_App".Status.Fault` |
| robot error number in | `"DB_App".Status.ErrorNo` |
| missing number in | `"DB_MachineReady".<missing number output>` |
| alarm bits out | your alarm tags for the HMI |

One alarm text per `DB_Job.Error` value, section 13 of the guide, with the row number in it.

## Network 10 — `FB_StopRecord`

Existing.

## Network 11 — machine logic

Lamps, horn, and anything the scanner needs beyond the three pins of network 7. No robot tag is written here.

---

## Who writes `DB_App`, by network

| Group | Written by | Network |
| --- | --- | --- |
| `DB_App.HS` | handshake toggle | OB30, 1 |
| `DB_App.Status.*` | `FC_RobotStatusMap` | OB30, 5 |
| `DB_App.Req.Machine.*` | `FB_MachineReady` | OB1, 4 |
| `DB_App.Req.Hand.Home` | `FC_DriveHome` | OB1, 5 |
| `DB_App.Req.Hand.Axis`, `.JogMinus`, `.JogPlus` | `FB_OperBox` | OB1, 6 |
| `DB_App.Req.Auto.*` | `FC_ScanControl` | OB1, 7 |
| `DB_App.Cmd.*` | `FC_CmdArbiter` | OB1, 8 |

Cross-reference after the change: every row above shows exactly one writer.

## The other end, OB30 network 3

The module 1 FB sends `DB_App.Cmd`. Its SEW pin names are on its interface; the roles are:

| Pin role of the module 1 FB | Tag |
| --- | --- |
| enable | `"DB_App".Cmd.Enable` |
| reset | `"DB_App".Cmd.Reset` |
| automatic mode | `"DB_App".Cmd.Automatic` |
| program start | `"DB_App".Cmd.Start` |
| stop | `"DB_App".Cmd.Stop` |
| program number | `"DB_App".Cmd.ProgNo` |
| override | `"DB_App".Cmd.Override` |
| jog axis, jog −, jog +, home | `"DB_App".Cmd.Axis`, `.JogMinus`, `.JogPlus`, `.Home` |

## What leaves OB1

- `FC_SrlGates` and `DB_App.Srl`.
- `FC_RC_Mngmt`, `FB_RC_Mechanism`, `FB_RobotOrder`, the seven-segment path, until a job-type selector exists. Left in, they are a second writer of Start.
- Any direct write into `DB_App.Cmd` from `FB_MachineReady`, `FB_OperBox` or `FC_DriveHome`.
