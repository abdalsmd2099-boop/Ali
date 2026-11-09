# Gantry_Robot – where things are (2026-11-04)

One project, one version of every file. The folders sort by purpose, not by version.
A change goes into the one file of the block in `PLC/`, behind a switch or as a field at the end.
Every change gets a dated row in `GOALS.md`.

| Folder | What is in it | Start with |
|---|---|---|
| `GOALS.md` (root) | the goals, the design, the contract with program 20, the error numbers, the dated change log. The reference for everything | section 5 and the change log |
| `PLC/` | the blocks as they go into TIA: `UDT_*.udt`, `DB_*.scl`, `FC_*.scl`, `FB_*.scl`, `OB1_Stage2_call.scl` (all nine OB1 networks, pin by pin), `OB30_complete.scl` and `OB30_NW2_pins.md` (the robot call). `BLOCK_MAP.md` says which block writes which tag | `BLOCK_MAP.md` |
| `HMI/` | the screens: `SCREENS.html` (every object with its tag, A1 … A77, M43), `HMI_STAGE1.md` (Robot move box), `HMI_STAGE2.md` (JOB box, DRUM box, LINE box with AREA, text lists), `HMI_CUBE_HOWTO.md` (CUBE box), `SCREEN_FIXES.md` (what is wrong on the real screen and how to fix it) | `SCREEN_FIXES.md` |
| `TEST/` | testing and following the errors: `CHECKLIST_CUBE_AUTO.md` (what must exist, the pins, the build order A–D with D0 = no turn), `CHECKLIST_DRUM_AUTO.md` (the drum: TIA, robot, screen, numbers, test; part 0 = commissioning in order), `CHECKLIST_LINE_AUTO.md` (the line scan, job 3: serpentine, raster, patch, the area by two corners, the whole table, 2026-11-09), `TEST_PROTOCOL.md` (the tests with the expected values), `TEST_SHEET.md`, `NEXT_STEPS.md`, `ERROR_LOG.md` (every error seen at the machine, with date, cause and fix) | `ERROR_LOG.md`, then the checklist |
| `ROBOT/` | the SEW side: `PROGRAM20.md` (the program on the robot, block by block), the MOVIKIT manual as text and pdf, the MovirunSmart monitor doc, the bus map picture | `PROGRAM20.md` |
| `DOCS/` | plans and maps: `AUTOMATIC_TODAY.md` (the day plan in parts 1–7), `ROADMAP.md`, the stage and step readmes, `REVIEW_2026-10-29.md`, `SYSTEM_MAP.html` and `SOLUTION_MAP.html` (the pictures of the whole) | `AUTOMATIC_TODAY.md` |
| `REF/` | material you brought: the StepChain docx files, the OB30 exports, photos and screenshots. Read only | – |

## How a day at the machine goes

1. `TEST/CHECKLIST_CUBE_AUTO.md`: what must be in TIA, the pins, the screen objects. Tick what is there.
2. `TEST/TEST_PROTOCOL.md`: run the test of the day, compare with the expected values.
3. An error: `TEST/ERROR_LOG.md` first. If the number is there, the fix is there. If not, the numbers to read are in `GOALS.md` 5.5 (engine) and `HMI/HMI_STAGE2.md` (job). New errors get a row in the log.
4. A screen problem: `HMI/SCREEN_FIXES.md`, then `HMI/SCREENS.html` for the object and its tag.
5. A change to a block: the file in `PLC/`, the version line in its header, a row in `GOALS.md`, the pin in `PLC/OB1_Stage2_call.scl`, the pin row in the checklist.

## Published pages (same links, the files moved)

| Page | File | Link |
|---|---|---|
| Screens | `HMI/SCREENS.html` | https://claude.ai/artifact/K8eztVxRTwZqFbCw4vBh7p |
| System map | `DOCS/SYSTEM_MAP.html` | https://claude.ai/artifact/AagbAehqpKH8WkBnqbmK3x |
| Gantry full map | `DOCS/SOLUTION_MAP.html` | https://claude.ai/artifact/AQSZvUfrtDkGQrE52S4Lnv |
