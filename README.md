# Automated Metal Cutting Line — PLC & HMI Control System

A fully structured industrial automation project built in **Siemens TIA Portal V17**, simulating an automated metal bar cutting and sorting line. The project implements a complete control system using **Ladder Logic (LAD)**, structured **Function Blocks (FB)** with instance **Data Blocks (DB)**, a shared global data model, and a multi-screen **WinCC HMI** for operator visualization and control.

This project was built as a hands-on learning exercise in industrial PLC programming, covering the full scope of a typical machine controls project: sequencing, analog simulation, fault handling, recipe management, alarming, and trending.

---

## 1. Project Overview

### Process Description
The system simulates a steel bar cutting station:

1. A **feed conveyor** advances material toward a shear station.
2. A **simulated encoder** (ramp-based, since PLCSIM has no physical analog input) tracks the fed length.
3. Once the target length is reached, a **clamp cylinder** secures the material.
4. A **shear cylinder** cuts the material, with a **fault timeout** if the cylinder fails to confirm position.
5. The clamp retracts, the cut piece is routed to an **eject station**, where it is sorted into **good** or **scrap** lanes based on length tolerance.
6. The cycle repeats automatically while the machine is in `Running` state.

### Scope Covered
| Area | Implementation |
|---|---|
| Ladder Logic | Interlocks, seal-in circuits, sequencing, math/compare instructions |
| Structured Programming | 7 Function Blocks, 1 Global DB, 7 Instance DBs |
| Analog Simulation | Ramp-based simulated encoder (no physical analog hardware available in PLCSIM) |
| Sequencing | Step-counter state machines (clamp and shear cylinders) |
| Fault Handling | TON-based timeout detection, centralized alarm handling |
| Edge-Triggered Logic | P_TRIG-based part counting to prevent double-counting |
| HMI / SCADA | 5-screen WinCC Basic HMI: Overview, Recipe, Alarms, Trends |
| Recipe Management | Operator-adjustable target length, tolerance, feed speed |
| Alarming | HMI Discrete Alarms via bit-packed word, with live Alarm View |
| Trending | Real-time trend of measured length via HMI Trend View |

---

## 2. Hardware / Software Environment

| Item | Detail |
|---|---|
| Engineering software | TIA Portal V17 |
| PLC (simulated) | SIMATIC S7-1500, CPU 1516-3 PN/DP |
| HMI (simulated) | SIMATIC Basic Panel, KTP900 Basic PN |
| Simulation tool | S7-PLCSIM (standard, non-Advanced) |
| Programming language | Ladder Logic (LAD) |

**Note on simulation scope:** Full live co-simulation between the HMI Runtime and PLCSIM (i.e., clicking HMI buttons and watching PLCSIM respond in real time) requires **S7-PLCSIM Advanced**, which was not available in this environment. As a result:
- The **PLC logic was fully tested and verified** end-to-end using PLCSIM combined with a watch table (forcing physical I/O and observing correct sequencing, fault handling, and counting).
- The **HMI was fully built and its tag bindings verified individually** (correct DB/PLC tag links, correct animations, correct alarm configuration), but live networked HMI-to-PLCSIM operation was not demonstrable in this environment.

---

## 3. I/O List

### Digital Inputs
| Symbol | Address | Description |
|---|---|---|
| I_StartPB | %I0.0 | Start pushbutton |
| I_StopPB | %I0.1 | Stop pushbutton |
| I_EStop | %I0.2 | Emergency stop |
| I_AutoManSwitch | %I0.3 | Auto/Manual mode selector |
| I_ClampRetracted | %I0.4 | Clamp cylinder retracted limit switch |
| I_ClampExtended | %I0.5 | Clamp cylinder extended limit switch |
| I_ShearRetracted | %I0.6 | Shear cylinder retracted limit switch |
| I_ShearExtended | %I0.7 | Shear cylinder extended limit switch |
| I_PartAtEject | %I1.0 | Photoeye at eject station |

### Digital Outputs
| Symbol | Address | Description |
|---|---|---|
| Q_FeedMotor | %Q0.0 | Feed conveyor motor run |
| Q_ClampExtend | %Q0.1 | Clamp cylinder extend solenoid |
| Q_ClampRetract | %Q0.2 | Clamp cylinder retract solenoid |
| Q_ShearExtend | %Q0.3 | Shear cylinder extend solenoid |
| Q_ShearRetract | %Q0.4 | Shear cylinder retract solenoid |
| Q_EjectGoodLane | %Q0.5 | Diverter to good-parts lane |
| Q_EjectScrapLane | %Q0.6 | Diverter to scrap lane |
| Q_RunLamp | %Q0.7 | Auto-run indicator lamp |
| Q_FaultLamp | %Q1.0 | Fault/alarm indicator lamp |

---

## 4. Data Model — Global DB (`Recipe_Status`, DB1)

Central shared data structure, accessible by all Function Blocks and the HMI.

```
Recipe_Status
├── Recipe (Struct)
│   ├── TargetLength   : Real   — operator setpoint, cut length in mm
│   ├── Tolerance      : Real   — allowed ± tolerance in mm
│   └── FeedSpeed      : Real   — feed conveyor speed reference (%)
├── Status (Struct)
│   ├── CurrentLength  : Real   — live measured length
│   ├── GoodCount      : Int    — total good pieces cut
│   ├── ScrapCount     : Int    — total scrap pieces cut
│   └── TotalCount     : Int    — total pieces processed
├── Mode (Struct)
│   ├── AutoMode       : Bool
│   ├── ManualMode     : Bool
│   └── Running        : Bool
└── Alarms (Struct)
    ├── EStopTripped   : Bool
    ├── ClampFault     : Bool
    ├── ShearFault     : Bool
    ├── LengthError    : Bool
    ├── AnyFaultActive : Bool   — OR of all above, for interlocking
    └── AlarmByte      : Word   — bit-packed alarm word for HMI Discrete Alarms
```

**Design rationale:** using one shared, structured Global DB (rather than scattered memory bits) gives every FB and the HMI a single, self-documenting source of truth — e.g. `"Recipe_Status".Recipe.TargetLength` is unambiguous, unlike a bare address like `MD100`.

---

## 5. Function Block Architecture

| FB | Instance DB | Purpose |
|---|---|---|
| FB1 — `ModeControl` | ModeControl_DB | Start/Stop seal-in circuit, Auto/Manual mode selection |
| FB2 — `FeedConveyor` | FeedConveyor_DB | Feed motor interlock logic |
| FB3 — `LengthMeasurement` | LengthMeasurement_DB | Simulated encoder ramp, length comparison |
| FB4 — `ClampCylinder` | ClampCylinder_DB | Step-sequencer for clamp extend/retract |
| FB5 — `ShearCylinder` | ShearCylinder_DB | Step-sequencer for shear extend/retract, with TON fault timeout |
| FB6 — `EjectSort` | EjectSort_DB | Tolerance check, good/scrap routing, edge-triggered counting |
| FB7 — `AlarmHandler` | AlarmHandler_DB | Combines individual fault bits into `AnyFaultActive` and `AlarmByte` |

### Design pattern
Every FB follows a consistent principle: **FBs make decisions and expose them via local Output tags or writes to the Global DB; physical I/O wiring only happens in OB1.** This keeps the logic hardware-agnostic — if I/O addressing changes, only OB1 needs editing, not the FB internals.

### Sequencer pattern (FB4, FB5)
Both cylinder FBs use a **Static Int `Step`** variable as a state machine:
- `0` = idle, `1` = extending, `2` = extended/holding, `3` = retracting
- Each step transition is a separate network, checked against sensor confirmation
- FB5 additionally uses a **TON timer** to detect a stuck/failed cylinder — if extension isn't confirmed within the preset time, the system raises `ShearFault` and safely aborts back to `Step 0`, rather than hanging indefinitely

### Edge-triggered counting (FB6)
Uses **P_TRIG** instructions (with dedicated Static memory bits) to ensure each physical part is counted exactly once, despite the eject sensor remaining TRUE for multiple scan cycles while a part physically passes.

---

## 6. OB1 (Main Program) — Call Structure

```
Network 1: ModeControl     → reads physical Start/Stop/EStop/AutoMan inputs
Network 2: FeedConveyor    → reads LengthMeasurement_DB.LengthReached (prior scan)
Network 3: LengthMeasurement → reads Q_FeedMotor, ShearCylinder_DB.CutComplete
Network 4: ClampCylinder   → triggered by LengthReached, retracted by CutComplete
Network 5: ShearCylinder   → triggered by ClampCylinder_DB.ClampInPosition
Network 6: EjectSort       → reads I_PartAtEject
Network 7: AlarmHandler    → reads all Alarms struct bits
Network 8: Run/Fault lamp logic
```

**Note on data flow ordering:** several FBs reference other FBs' instance DB outputs *before* those FBs execute later in the same scan — this deliberately uses the *previous scan's* value (a few milliseconds old), which has no meaningful impact on a mechanical process operating on this timescale, and avoids circular-dependency issues.

---

## 7. HMI Screens

| Screen | Contents |
|---|---|
| **Overview** | Live process mimic (4 animated station blocks), Start/Stop buttons, Running/Fault status lamps, navigation to all other screens |
| **Recipe** | Operator-editable I/O fields for Target Length, Tolerance, Feed Speed |
| **Alarms** | Alarm View control, populated via 4 configured Discrete Alarms |
| **Trends** | Trend View charting live `CurrentLength`, 0–600mm scale |

### HMI Alarm Implementation Note
KTP900 Basic Panels cannot trigger Discrete Alarms directly from individual Bool tags — this required packing the 4 individual alarm bits (`EStopTripped`, `ClampFault`, `ShearFault`, `LengthError`) into a single `AlarmByte` (Word) tag via dedicated ladder logic in FB7, with each Discrete Alarm configured against a specific bit position (0–3) of that word. This is a genuine Basic Panel platform constraint, not a design choice.

---

## 8. Testing & Validation

Logic was validated using **S7-PLCSIM** with a watch table (`TestWatch`), forcing physical inputs to simulate sensor confirmations and observing:
- Correct Start/Stop seal-in behavior
- Correct simulated length ramp and target detection
- Correct clamp → shear → eject sequencing, including step counter transitions
- Correct fault timeout behavior (shear fault raised and safely recovered when a sensor confirmation was withheld)
- Correct edge-triggered counting (good/scrap/total counts incrementing exactly once per part)
- Correct continuous-cycle behavior (machine automatically begins the next cycle after completing one, consistent with `Running` remaining TRUE)

---

## 9. Known Limitations / Future Work

- **PLCSIM Advanced not available** in this environment — full live HMI Runtime ↔ PLCSIM co-simulation was not demonstrated; HMI tag bindings and screen logic were verified individually instead.
- **Single-cycle vs. continuous mode** is not currently implemented — the machine always continues cycling while `Running` is TRUE. A "single cycle" mode (auto-clear `Running` after one `CutComplete`) would be a reasonable enhancement.
- **`LengthError` alarm** is defined in the data model but not currently set by any logic — reserved for future use (e.g., detecting an implausible length reading indicating sensor failure).
- **OPC UA / PLC-to-PLC communication** was scoped as an optional stretch goal and not implemented in this version.
- **Fault timer preset** (`T#15S` in the current build) was lengthened from a more realistic `T#3S` to ease manual watch-table testing — should be reverted to a production-realistic value before any real deployment.


## Author's Note

This project was built as a self-directed learning exercise covering the full breadth of a typical PLC/HMI machine controls scope — structured programming (FB/DB), sequencing, fault handling, and SCADA-style visualization — using Siemens TIA Portal V17. It reflects both the working control logic and the real debugging process involved in building it, including platform-specific constraints (e.g., Basic Panel alarm limitations, PLCSIM simulation scope) that mirror real-world tooling considerations in industrial automation work.
