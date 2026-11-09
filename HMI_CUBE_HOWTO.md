# The cube box – how to build every button and field in WinCC Unified

2026-11-03. One page, object by object. Nothing on this screen needs a script: every button writes one bit, every lamp reads one bit, and the block `FB_CubeButtons` in OB1 does the thinking. Build the PLC part first (section 2, ten minutes), then the screen (sections 3 to 5).

## 1. The drawing

```
+------------------------------------------------------------------------------------------+
|  CUBE                                                                                    |
|                                                                                          |
|   length  [  600 ] mm     width  [  400 ] mm     angle  [    0 ] deg    safe Z [ 1500 ]  |
|   corners: computed from corner 1        (or: taught, rows 5 to 7)                       |
|                                                                                          |
|   [ TEACH HOME ] (o)      [ TEACH START ] (o)      [ TEACH CORNER 1 ] (o)   ready (o)    |
|   [ GO TO START ]         [ GO TO CORNER 1 ]                                             |
|                                                                                          |
|   pose nr [ 4 ]  [ TEACH ]  [ CLEAR ]   last taught [ 4 ]   taught (o)  refused (o)       |
|                                                                                          |
|   SCAN FROM HERE  (o==)        PATH MODE  (o==)                                          |
+------------------------------------------------------------------------------------------+
|  JOB                                                                                     |
|   [ AUTO ] (==o)   override [ 10 ] %   job [ 1 ]   [ START ]  [ ABORT ]  [ ACK ]         |
|   running (o)  done (o)  error (o)   job error [ 0  no error      ]   moves [ 0 ] / [ 14 ] |
|   active [ 3  to corner 1 ]   engine error [ 0  no error ]   queue [ 0 ]                 |
+------------------------------------------------------------------------------------------+
```

`( )` is a lamp, `[ ]` a field or a button, `(o==)` a switch. The same box drawn as a screen is in `SOLUTION_MAP.html` section 2.

## 2. The PLC part first

**2.1 Two imports.** External source files → add `DB_RobotHand.scl` (if it is in the PLC already from the hand rungs, overwrite: the new fields are at the end) and `FB_CubeButtons.scl` → generate blocks from source.

**2.2 One network in OB1**, after the robot mode network and **before the teach network**. Drag `FB_CubeButtons` in, single instance `IDB_CubeButtons`. The pins:

| Pin | Wire |
|---|---|
| `i_xTeachHome` | `"DB_RobotHand".Btn.xTeachHome` |
| `i_xTeachStart` | `"DB_RobotHand".Btn.xTeachStart` |
| `i_xTeachC1` | `"DB_RobotHand".Btn.xTeachC1` |
| `i_xTeachNr` | `"DB_RobotHand".Btn.xTeachNr` |
| `i_xClearNr` | `"DB_RobotHand".Btn.xClearNr` |
| `i_intPoseNrReq` | `"DB_RobotHand".Btn.intPoseNr` |
| `i_xGoStart` | `"DB_RobotHand".Btn.xGoStart` |
| `i_xGoC1` | `"DB_RobotHand".Btn.xGoC1` |
| `i_xAck` | `"DB_RobotHand".Btn.xAck` |
| `i_aoudtP` | `"DB_Poses".aoudtP` |
| `i_aoxValid` | `"DB_Poses".aoxValid` |
| `i_rLength` | `"DB_Recipe".Cube.rLength` |
| `i_rWidth` | `"DB_Recipe".Cube.rWidth` |
| `i_xEnable` | `"DB_RobotMove".Man.xEnable` |
| `i_xAuto` | `"DB_RobotJob".Man.xAuto` |
| `i_xEngineBusy` | `"DB_RobotMove".State.xBusy` |
| `io_udtMan` | `"DB_RobotMove".Man.udtReq` |
| `io_xManStart` | `"DB_RobotMove".Man.xStart` |
| `io_xReset` | `"DB_RobotMove".Man.xReset` |
| `io_xAbort` | `"DB_RobotJob".Man.xAbort` |
| `o_intPoseNr` | `"DB_RobotJob".Teach.intPoseNr` |
| `o_xTeach` | `"DB_RobotJob".Teach.xTeach` |
| `o_xClear` | `"DB_RobotJob".Teach.xClear` |
| `o_xTeachAllowed` | `"DB_RobotHand".Lamp.xTeachAllowed` |
| `o_xGoAllowed` | `"DB_RobotHand".Lamp.xGoAllowed` |
| `o_xCubeReady` | `"DB_RobotHand".Lamp.xCubeReady` |
| `o_xComputed` | `"DB_RobotHand".Lamp.xComputed` |

The teach network itself does not change. From now on `FB_CubeButtons` is the one writer of `"DB_RobotJob".Teach.intPoseNr`, `.xTeach` and `.xClear`, so the three screen objects that wrote them move to `"DB_RobotHand".Btn` (section 5). Compile, download.

**2.3 What the block does**, so you know what to expect: a teach button sets the pose number and gives one pulse to FB_Teach, which checks allowed and referenced itself. A go to button copies row 2 or row 4 into the target of the Robot move screen, sets block 1 and gives one start pulse, only when Enable is on, AUTO off and the engine idle. ACK gives the engine reset and the job abort together for one cycle. The four lamp bits are computed every cycle.

## 3. The HMI tags

HMI → HMI tags → Default tag table. For each row: Add new, name, data type, connection to the PLC, PLC tag by browsing, acquisition cycle 100 ms. The limits go on the tag: select the tag → Properties → Range → upper and lower limit.

| HMI tag | PLC tag | Type | Limits |
|---|---|---|---|
| `Cube_Length` | `"DB_Recipe".Cube.rLength` | Real | 0 … 4380 |
| `Cube_Width` | `"DB_Recipe".Cube.rWidth` | Real | 0 … 4230 |
| `Cube_Angle` | `"DB_Recipe".Cube.rAngle` | Real | −180 … 180 |
| `Poses_SafeZ` | `"DB_Poses".rSafeZ` | Real | −10 … 1908 |
| `Btn_TeachHome` | `"DB_RobotHand".Btn.xTeachHome` | Bool | |
| `Btn_TeachStart` | `"DB_RobotHand".Btn.xTeachStart` | Bool | |
| `Btn_TeachC1` | `"DB_RobotHand".Btn.xTeachC1` | Bool | |
| `Btn_TeachNr` | `"DB_RobotHand".Btn.xTeachNr` | Bool | |
| `Btn_ClearNr` | `"DB_RobotHand".Btn.xClearNr` | Bool | |
| `Btn_PoseNr` | `"DB_RobotHand".Btn.intPoseNr` | Int | 1 … 50 |
| `Btn_GoStart` | `"DB_RobotHand".Btn.xGoStart` | Bool | |
| `Btn_GoC1` | `"DB_RobotHand".Btn.xGoC1` | Bool | |
| `Btn_Ack` | `"DB_RobotHand".Btn.xAck` | Bool | |
| `Lamp_TeachAllowed` | `"DB_RobotHand".Lamp.xTeachAllowed` | Bool | |
| `Lamp_GoAllowed` | `"DB_RobotHand".Lamp.xGoAllowed` | Bool | |
| `Lamp_CubeReady` | `"DB_RobotHand".Lamp.xCubeReady` | Bool | |
| `Lamp_Computed` | `"DB_RobotHand".Lamp.xComputed` | Bool | |
| `Poses_Valid_1`, `_2`, `_4` | `"DB_Poses".aoxValid[1]`, `[2]`, `[4]` | Bool | |
| `Teach_LastTaught` | `"DB_RobotJob".Teach.intLastTaught` | Int | |
| `Teach_Taught`, `Teach_Refused` | `"DB_RobotJob".Teach.xTaught`, `.xRefused` | Bool | |
| `Recipe_FromHere` | `"DB_Recipe".xFromHere` | Bool | |
| `Recipe_PathMode` | `"DB_Recipe".xPathMode` | Bool | only with the path mode, part 7 of `AUTOMATIC_TODAY.md` |

The names are a suggestion. Keep whatever naming you use, the PLC tag is what counts.

## 4. Five recipes, one per kind of object

**Recipe A – momentary button.** Toolbox → Basic objects → Button. Draw it. Properties → General → Text: the label. Properties → Events → **Press**: add the system function `SetTagValue`, tag = the button's tag, value = 1. Events → **Release**: `SetTagValue`, the same tag, value = 0. That is the whole button. To grey it out when not allowed: Properties → Miscellaneous → the property that allows operating it (V18 calls it "Operator control allowed") → in its Dynamization column choose **Tag** → the lamp tag, for example `Lamp_TeachAllowed`. If you do not find that property, dynamize **Visibility** with the same tag instead: the button disappears when not allowed.

**Recipe B – I/O field.** Toolbox → Basic objects → I/O field. Properties → General → Process → Tag: the tag. Mode: Input/output. Format: the display pattern, `9999` for whole numbers, `999.9` for one decimal. The limits come from the tag (section 3), the field refuses a value outside them.

**Recipe C – lamp.** Toolbox → Basic objects → Circle. Properties → Appearance → Background color → Dynamization column → **Tag** → the Bool tag. Two entries appear: value 0 = grey, value 1 = green. Pick the colors.

**Recipe D – switch.** Toolbox → Elements → Switch. Properties → General → Process → Tag: the Bool tag. Texts for on and off. If there is no switch in your toolbox: a Button with Events → Click → `InvertBit`, tag = the Bool tag, and a lamp next to it with recipe C on the same tag.

**Recipe E – text that appears and disappears.** Toolbox → Basic objects → Text. Properties → Visibility → Dynamization column → **Tag** → `Lamp_Computed`, visible when 1. A second Text on the same spot with the other wording, visible when 0: in its dynamization set the visibility for value 0.

**Recipe F – output field with a text list.** Toolbox → Basic objects → I/O field. Mode: Output. Properties → General → Process → Tag. Then "Text list" (resource list): the list of section 6. The field then shows "3 to corner 1" instead of 3.

## 5. The objects, one line each

| Object | Recipe | Tag | Extra |
|---|---|---|---|
| length | B | `Cube_Length` | pattern 9999, unit mm |
| width | B | `Cube_Width` | pattern 9999 |
| angle | B | `Cube_Angle` | pattern 999, unit deg |
| safe Z | B | `Poses_SafeZ` | pattern 9999 |
| corners text | E | `Lamp_Computed` | text 1 "computed from corner 1", text 0 "taught, rows 5 to 7" |
| TEACH HOME | A | `Btn_TeachHome` | enable by `Lamp_TeachAllowed` |
| TEACH START | A | `Btn_TeachStart` | enable by `Lamp_TeachAllowed` |
| TEACH CORNER 1 | A | `Btn_TeachC1` | enable by `Lamp_TeachAllowed`. Tool at the corner, facing side 1, RotZ −180 |
| lamp after each teach button | C | `Poses_Valid_1`, `_2`, `_4` | green = taught |
| ready | C | `Lamp_CubeReady` | the three in one |
| GO TO START | A | `Btn_GoStart` | enable by `Lamp_GoAllowed` |
| GO TO CORNER 1 | A | `Btn_GoC1` | enable by `Lamp_GoAllowed` |
| pose nr | B | `Btn_PoseNr` | **moves here from `"DB_RobotJob".Teach.intPoseNr`** |
| TEACH | A | `Btn_TeachNr` | **moves here from `"DB_RobotJob".Teach.xTeach`** |
| CLEAR | A | `Btn_ClearNr` | **moves here from `"DB_RobotJob".Teach.xClear`** |
| last taught | F or B output | `Teach_LastTaught` | |
| taught, refused | C | `Teach_Taught`, `Teach_Refused` | one-cycle pulses, may only flash |
| SCAN FROM HERE | D | `Recipe_FromHere` | |
| PATH MODE | D | `Recipe_PathMode` | only with the path mode, part 7 of `AUTOMATIC_TODAY.md` |
| ACK | A | `Btn_Ack` | in the JOB box, next to START and ABORT |

Everything else of the JOB box, AUTO, override, job, START, ABORT, the lamps and numbers, is already there from `HMI_STAGE2.md` section 2, unchanged.

The three objects marked **moves here** are the only change to what you have: open each, Properties → General → Process → Tag, pick the new tag. That is what keeps one writer: the PLC block now writes the teach tags, the screen writes the button bits.

## 6. Text lists

HMI → Text and graphic lists → Text lists → add a list, then its entries with value and text. Assign the list to an output field with recipe F.

| List | Entries |
|---|---|
| `TL_JobErr` | 0 no error · 1 a pose is not taught · 2 safe Z not set · 3 engine busy or in error · 4 a pose outside the limits (see active) · 5 drum radius not set · 6 scan from here not possible · 10 engine error (see engine error) |
| `TL_Move` | 0 – · 1 to start · 2 approach · 3 to corner 1 · 4 side 1 · 5 turn at corner 2 · 6 side 2 · 7 turn at corner 3 · 8 side 3 · 9 turn at corner 4 · 10 side 4 · 11 turn at corner 1 · 12 lift · 13 to start · 14 home · 15 rectangle, sides 1 to 4 |
| `TL_Err` | the 17 lines of `HMI_STAGE1.md` section 3, plus 16 path longer than 7 poses |
| `TL_Block` | 1 linear · 2 linear slow · 3 circle ccw · 4 circle cw · 5 rectangle, 7 lin · 6 circle ccw, 4 arcs · 7 circle cw, 4 arcs |

## 7. Test each object, two minutes each

| Press | Watch in a TIA watch table | Must |
|---|---|---|
| TEACH HOME, robot standing, Enable on, AUTO off | `"DB_RobotJob".Teach.intPoseNr`, `"DB_Poses".aoxValid[1]`, `.intLastTaught` | 1, TRUE, 1. Row 1 shows the actual pose |
| TEACH HOME with AUTO on | `"DB_RobotJob".Teach.xRefused` | flashes, nothing stored. The button is grey if you dynamized it |
| GO TO START | `"DB_RobotMove".Man.udtReq.udtPose.rX`, `.Man.xStart`, `.State.xBusy` | the X of row 2, a one-cycle start, busy, the robot drives there |
| ACK after an error | `"DB_RobotMove".State.xError`, `"DB_RobotJob".State.xError` | both off |
| type length 600, width 400 | `"DB_RobotHand".Lamp.xComputed` | TRUE, the text changes |
| type length 0 | the same | FALSE, "taught, rows 5 to 7" |
| pose nr 7, TEACH | `"DB_Poses".aoxValid[7]` | TRUE. The generic way still works |

## 8. One writer

| Tag | The one writer |
|---|---|
| `"DB_RobotHand".Btn.*` | the screen |
| `"DB_RobotHand".Lamp.*` | `FB_CubeButtons` |
| `"DB_RobotJob".Teach.intPoseNr`, `.xTeach`, `.xClear` | `FB_CubeButtons`, from today. The screen no longer writes them |

Documented exception, the same class as typing a pose: on a go to edge `FB_CubeButtons` writes the target fields and pulses `Man.xStart`, on ACK it pulses `Man.xReset` and `Man.xAbort`. The screen's MOVE, RESET and ABORT write the same bits on operator action. Both are operator actions, never in the same cycle on purpose.
