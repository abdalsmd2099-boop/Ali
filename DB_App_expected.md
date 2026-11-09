# DB_App – what the blocks in this folder expect inside it

2026-10-28. Built from OB30 networks 1, 2, 3 (your files) and from `FB_RobotMove`. Your real `DB_App` is bigger. Everything else in it stays as it is; only the elements below are touched by our blocks.

**Type column:** ✔ = certain from your code. ~ = assumed, check against your DB. If an assumed type differs, the `io_` pin of `FB_RobotMove` must get your type.

## 1. The structure

```scl
DATA_BLOCK "DB_App"
   VAR
      // ---- handshake, written by ob30 network 1 ------------------------------
      HS : Bool;                       // ✔ toggles, goes to both module fbs
      HS_Stall : Int;                  // ✔ counts cycles without echo, max 32000
      HS_Alive : Bool;                 // ✔ false when the echo stops for > 200 cycles
      HS_Count : DInt;                 // ✔ counts up to 100000, so dint

      ReadyToStart : Bool;             // ✔ and-ed with Cmd.Start in network 2

      // ---- commands to module 1, network 2 ------------------------------------
      Cmd : STRUCT
         Enable : Bool;                // ✔ hmi
         Reset : Bool;                 // ✔ hmi
         Automatic : Bool;             // ✔ hmi
         Simulation : Bool;            // ✔ hmi
         Override : Int;               // ~ FB_RobotMove writes it now, 1..100
         ProgramNumber : Int;          // ~ FB_RobotMove writes it now, 20
         Start : Bool;                 // ✔ FB_RobotMove writes it now
         Pause : Bool;                 // ✔ hmi
         Stop : Bool;                  // ✔ hmi
         Home_X1, Home_Y1, Home_Z1, Home_R1, Home_R2 : Bool;        // ✔ hmi
         Jog_Pos_X, Jog_Pos_Y, Jog_Pos_Z, Jog_Pos_R1, Jog_Pos_R2 : Bool;   // ✔ hmi
         Jog_Neg_X, Jog_Neg_Y, Jog_Neg_Z, Jog_Neg_R1, Jog_Neg_R2 : Bool;   // ✔ hmi
         Jog_Joint_Base : Bool;        // ✔ hmi
         Jog_Axis : Bool;              // ✔ hmi
      END_STRUCT;

      // ---- homing helper, network 2 ---------------------------------------------
      Home : STRUCT
         active : Bool;                // ✔
         JogP : Array[1..5] of Bool;   // ✔ or-ed into the jog pins
         JogN : Array[1..5] of Bool;   // ✔
      END_STRUCT;

      // ---- the 7 path segments of module 1, network 2 ---------------------------------
      Path : STRUCT
         Seg : Array[1..7] of STRUCT
            MotionSet : Int;           // ~
            Wait : Bool;               // ✔
            PathEnd : Bool;            // ✔
            Blend : Real;              // ~ blending distance, mm
            X, Y, Z, R1, R2 : Real;    // ~ pose
         END_STRUCT;
      END_STRUCT;

      // ---- the srl lane of module 2, network 3 -----------------------------------------
      Srl : STRUCT
         Pose : Array[1..5] of Real;   // ~ pose[51] x y z rotz roty. FB_RobotMove writes it
         Par : Array[1..4] of Real;    // ~ real[51..54]. FB_RobotMove writes it
         Gate : Array[1..8] of Bool;   // ✔ bool[51..58]. FB_RobotMove writes it
         GateEnable : Bool;            // ✔ FB_RobotMove writes it
         In : Array[1..4] of Bool;     // ✔ bool[71..74] from the robot, module 2 fb writes it
         CommOK : Bool;                // ✔ module 2 fb writes it
      END_STRUCT;
   END_VAR
END_DATA_BLOCK
```

## 2. Who writes what after step 3

| Element | Writer before | Writer now |
|---|---|---|
| `Cmd.Start` | HMI | `FB_RobotMove` |
| `Cmd.ProgramNumber` | HMI | `FB_RobotMove` |
| `Cmd.Override` | HMI | `FB_RobotMove` (HMI field writes `DB_RobotMove.Man.udtReq.intOverride`) |
| `Srl.Pose[1..5]`, `Srl.Par[1..4]`, `Srl.Gate[1..8]`, `Srl.GateEnable` | HMI or nothing | `FB_RobotMove` |
| `HS`, `HS_Stall`, `HS_Alive`, `HS_Count` | OB30 network 1 | unchanged |
| `Srl.In[1..4]`, `Srl.CommOK` | module 2 FB | unchanged |
| `Cmd.Enable` | OB1 network 7 and 8 | OB1 network 8 only (2026-10-29): machine on, referenced, and a requester. The robot screen Enable switch `DB_RobotMove.Man.xEnable` is one requester branch |
| everything else | HMI / OB1 | unchanged |

## 3. What our blocks read from it

| Block | Reads |
|---|---|
| `FC_RobotStatusMap` | `HS_Alive`, `Srl.CommOK` |
| `FB_RobotMove` | nothing directly; it gets the `Srl` and `Cmd` elements as InOut pins |

## 4. How to check the four assumed types quickly

Open `DB_App` in TIA, find `Srl.Pose`, `Srl.Par`, `Cmd.Override`, `Cmd.ProgramNumber` and read the Data type column. Real / Real / Int / Int = nothing to do. Anything else: tell me the four types and I change the pins.
