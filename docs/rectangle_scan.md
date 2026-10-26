# Rectangle scan with program 20

The robot holds one pose slot and four ways to reach it. The PLC holds the list.

## Pieces

| Piece | Job | Writes |
| --- | --- | --- |
| `robot/program20.srl` | one pass = one motion, block chosen by BOOL[51..54] | REAL[71] block number, REAL[73] run number echo |
| `plc/DB_Scan` | the rectangle: corner, height, size, pitch, speeds, safe height | by the HMI |
| `plc/FC_GenRect` | rectangle into rows, fence checked, empty on error | `DB_Job` |
| `plc/FB_SrlMove` | one row: window, commit, start, wait, verify | `DB_SrlLane`, start bit |
| `plc/FB_ScanJob` | walks the rows, retract first, scanner on the slow lanes | job status, scanner outputs |
| `plc/OB30_srl_window.scl` | ships the window to module 2 in one pass | module 2 pins |

## One row on the bus

1. `FB_SrlMove` writes POSE[51], REAL[51] radius, REAL[52] run number and one gate, then the commit bit.
2. OB30 copies the whole window into module 2 and clears the commit.
3. Start with program 20. program_finished must fall within 2 s.
4. The robot runs the gated block. Timeout is distance over speed, doubled, plus 3 s.
5. program_finished rises. REAL[71] must equal the block and REAL[73] must equal the run number.
6. Gates off, commit, next row.

## The standard rectangle

Lx 300, Ly 200, Py 50, Px 0 gives 5 lanes and 12 rows. Block 1 is fast LIN, block 2 is slow LIN with the scanner recording.

| Row | X | Y | Z | Block | Scan |
| --- | --- | --- | --- | --- | --- |
| 1 | X0 | Y0 | Zsafe | 1 | |
| 2 | X0 | Y0 | Z0 | 1 | |
| 3 | X0+300 | Y0 | Z0 | 2 | on |
| 4 | X0+300 | Y0+50 | Z0 | 1 | |
| 5 | X0 | Y0+50 | Z0 | 2 | on |
| 6 | X0 | Y0+100 | Z0 | 1 | |
| 7 | X0+300 | Y0+100 | Z0 | 2 | on |
| 8 | X0+300 | Y0+150 | Z0 | 1 | |
| 9 | X0 | Y0+150 | Z0 | 2 | on |
| 10 | X0 | Y0+200 | Z0 | 1 | |
| 11 | X0+300 | Y0+200 | Z0 | 2 | on |
| 12 | X0+300 | Y0+200 | Zsafe | 1 | |

Before row 1 the job moves straight up to Zsafe from the actual position. Px above 0 switches to a stop and one measurement every Px along each lane.

## Error numbers

| Number | Meaning | Where |
| --- | --- | --- |
| 1 | start not taken, program_finished did not fall in 2 s | FB_SrlMove |
| 2 | move timeout | FB_SrlMove |
| 3 | robot fault during the move | FB_SrlMove |
| 4 | wrong block or stale run number | FB_SrlMove |
| 5 | communication lost | FB_SrlMove |
| 6 | aborted | both |
| 7 | block number outside 1..4 | FB_SrlMove |
| 10 | no job loaded | FB_ScanJob |
| 11 | machine not ready | FB_ScanJob |
| 12 | measurement timeout | FB_ScanJob |
| gen 1 | bad rectangle parameters | FC_GenRect |
| gen 2 | more than 80 rows | FC_GenRect |
| gen 3 | a point outside the fence | FC_GenRect |

## Still to confirm on the machine

- `Vscan` and `Vmove` in `DB_Scan` must equal the speeds programmed in blocks 2 and 1. They are only used for the timeout.
- Changing POSE[51] during a running LIN must not move the target. If it does, the lane may only be written while the robot stands.
- A zero-length LIN, the same pose twice, must not fault the robot. The generator never produces one, the retract from actual position can when the gantry already stands at Zsafe.
- The pin names in `OB30_srl_window.scl` are those of the v0.7 module 2 FB. Match them to the loaded block.
