# CubeSat OBC Power Distribution Unit (PDU)

**Design Tool:** KiCad 9.0  
**Verification Status:** ERC Passed — 0 Errors, 0 Warnings ✓  
**License:** MIT (Educational & Research Use)

---

## 1. Project Overview & System Specifications

### System Description

The CubeSat On-Board Computer (OBC) Power Distribution Unit (PDU) is a flight-grade power management system designed for 1U nanosatellite missions. It provides four regulated voltage rails with comprehensive protection, current sensing, and telemetry feedback for safe power distribution to all satellite subsystems.

### Key Specifications

| Parameter | Value |
|-----------|-------|
| **Input Bus** | 2S Li-Ion Battery Pack (`VBAT_RAW`) |
| **Input Voltage Range** | 6.0 V – 8.4 V (nominal 7.4 V) |
| **Overcurrent Protection** | 2 A fast-blow fuse |
| **Output Rails** | 4 (3 switched + 1 always-on) |
| **Total Output Power** | ~3.5 W continuous |
| **Current Sensing** | High-side, 0.1 Ω shunt + INA199 (G = 50 V/V) |
| **Telemetry Interface** | 4 current monitors + 1 voltage monitor to OBC ADC |
| **Control Interface** | 16-pin OBC backplane connector (`J2`) |

### Output Power Rails

| Rail | Voltage | Max Current | Tolerance | Regulator | Switched |
|------|---------|-------------|-----------|-----------|----------|
| `3V3_DIG` | 3.3 V | 500 mA | ±3% | AMS1117-3.3 | Yes |
| `3V3_ANA` | 3.3 V | 200 mA | ±1% | REF5033 | Yes |
| `5V0_IO` | 5.0 V | 300 mA | ±5% | AMS1117-5.0 | Yes |
| `3V3_MEM` | 3.3 V | 100 mA | ±2% | AMS1117-3.3 | No (Always-On) |

---

## 2. Hierarchical Architecture & Signal Flow

### System Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CUBEAT OBC PDU                                     │
│                          (KiCad Hierarchical)                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  SHEET 1: 01_Input_Protection                                               │
│  ┌─────────┐    ┌──────────┐    ┌─────────────┐    ┌──────────────┐        │
│  │  J1     │───▶│ F1 (2A)  │───▶│ D1 (TVS)    │───▶│ U1 (LM5050)  │        │
│  │ VBAT_RAW│    │ VBAT_FUSED│    │ VBAT_FUSED  │    │ + Q1 MOSFET  │        │
│  └─────────┘    └──────────┘    └─────────────┘    └──────────────┘        │
│                                          │                │                 │
│                                          ▼                ▼                 │
│                              VBAT_SENSE (to Sheet 6)   VBUS_FILT            │
│                              (22k/10k divider)          (L1 + C1 filter)    │
│                                                         │                   │
│                    ┌────────────────────────────────────┼───────────────┐   │
│                    ▼                                    ▼               ▼   │
│  ┌──────────────────────────┐  ┌──────────────────────────┐  ┌───────────┐ │
│  │ SHEET 2: 3V3_DIG         │  │ SHEET 3: 3V3_ANA         │  │ SHEET 4:  │ │
│  │ AMS1117-3.3 + Q2/Q3      │  │ REF5033 + Q4/Q5          │  │ 5V0_IO    │ │
│  │ ISENSE_DIG_P ──R4──┐     │  │ ISENSE_ANA_P ──R7──┐     │  │ AMS1117   │ │
│  └──────────────────────┼────┘  └──────────────────────┼────┘  │ + Q6/Q7 │ │
│                         │                                │     └─────────┘ │
│                         ▼                                ▼               │ │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │              SHEET 5: 3V3_MEM (Always-On)                             │ │
│  │              AMS1117-3.3 + R11 (0.1Ω shunt)                           │ │
│  │              ISENSE_MEM_P ──R11──▶ 3V3_MEM                            │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                    │
│                    ┌───────────────┴───────────────┐
│                    ▼                               ▼
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐
│  │ SHEET 6: Telemetry Interface │  │ SHEET 7: Control Interface   │
│  │ INA199 x4 (U6-U9)            │  │ J2 16-pin OBC connector      │
│  │ IMON_* outputs to ADC        │  │ EN_* control signals         │
│  │ VMON_VBAT (22k/10k divider)  │  │ R18-R20 fail-safe pull-downs │
│  └──────────────────────────────┘  └──────────────────────────────┘
│                                    │
│                    ┌───────────────┴───────────────┐
│                    ▼                               ▼
│              EN_3V3_DIG ◀───────────────────── GPIO from OBC
│              EN_3V3_ANA ◀───────────────────── GPIO from OBC
│              EN_5V0_IO  ◀───────────────────── GPIO from OBC
└─────────────────────────────────────────────────────────────────────────────┘
```

### Schematic Hierarchy Table

| Sheet File | Title | Hierarchical Inputs | Hierarchical Outputs | Core Function |
|------------|-------|---------------------|----------------------|---------------|
| `CubeSat_PDU.kicad_sch` | Root Sheet | None | None | Top-level hierarchy routing 15 inter-sheet nets |
| `01_Input_Protection.kicad_sch` | Input Protection | `VBAT_RAW`, `GND` | `VBUS_FILT`, `VBAT_SENSE` | Fuse, TVS, ideal diode ORing, EMI filter |
| `02_3V3_DIG_Regulator.kicad_sch` | 3.3V Digital Rail | `VBUS_FILT`, `EN_3V3_DIG`, `GND` | `3V3_DIG`, `ISENSE_DIG_P` | Switched LDO for MCU & digital logic |
| `03_3V3_ANA_Regulator.kicad_sch` | 3.3V Analog Rail | `VBUS_FILT`, `EN_3V3_ANA`, `GND` | `3V3_ANA`, `ISENSE_ANA_P` | Low-noise precision reference for sensors |
| `04_5V0_IO_Regulator.kicad_sch` | 5.0V I/O Rail | `VBUS_FILT`, `EN_5V0_IO`, `GND` | `5V0_IO`, `ISENSE_5V0_P` | Switched 5V for peripherals & RF |
| `05_3V3_MEM_Regulator.kicad_sch` | 3.3V Memory Rail | `VBUS_FILT`, `GND` | `3V3_MEM`, `ISENSE_MEM_P` | Always-on rail for FRAM/EEPROM |
| `06_Telemetry_Interface.kicad_sch` | Telemetry | `ISENSE_*_P`, `VBAT_SENSE`, `3V3_DIG`, `GND` | `IMON_*`, `VMON_VBAT` | Current sense amps + voltage divider |
| `07_Control_Interface.kicad_sch` | Control | `IMON_*`, `VMON_VBAT`, `GND` | `EN_*`, connector pins | OBC backplane + fail-safe control |

---

## 3. Detailed Sheet-by-Sheet Circuit Breakdown

### Root Sheet (`CubeSat_PDU.kicad_sch`)

**Function:** Flat top-level hierarchy routing 15 inter-sheet nets across 7 sub-sheet symbols.

- **Inter-Sheet Nets:** `VBAT_RAW`, `GND`, `VBUS_FILT`, `VBAT_SENSE`, `EN_3V3_DIG`, `EN_3V3_ANA`, `EN_5V0_IO`, `3V3_DIG`, `3V3_ANA`, `5V0_IO`, `3V3_MEM`, `ISENSE_DIG_P`, `ISENSE_ANA_P`, `ISENSE_5V0_P`, `ISENSE_MEM_P`
- **Verification:** Zero dangling labels, zero nested sub-sheets, all hierarchical labels match child sheet definitions.

---

### Sheet 1: `01_Input_Protection.kicad_sch` — Input Protection, Ideal Diode ORing & EMI Filter

**Components:**

| Designator | Value | Footprint | Function |
|------------|-------|-----------|----------|
| `J1` | Conn_01x02 | Terminal_Block_1x02 | Battery input (Pin 1: `VBAT_RAW`, Pin 2: `GND`) |
| `F1` | 2A Fast-Blow | Fuse_Holder_1206 | Overcurrent protection → `VBAT_FUSED` |
| `D1` | SMBJ12A | DO-214AA (SMB) | Unidirectional TVS, clamps at 12V |
| `U1` | LM5050-1 | SOT-23-6 | Ideal diode controller |
| `Q1` | QM6006D / SI2302 | SOT-23 | N-MOSFET for reverse blocking |
| `L1` | 10 µH | L_0805 | EMI filter inductor |
| `C1` | 22 µF | C_0805 | Input filter capacitor |
| `C2` | 100 nF | C_0603 | Soft-start capacitor (`C_SS`) |
| `R1` | 10 Ω | R_0603 | Gate resistor |

**Circuit Operation:**

1. **Input Protection:** `VBAT_RAW` enters through `J1`, passes through 2A fuse `F1` producing `VBAT_FUSED`.
2. **TVS Clamping:** `D1` (SMBJ12A) clamps transients at 12V, protecting downstream circuitry from ESD and load dumps.
3. **Ideal Diode ORing:** `U1` (LM5050-1) drives `Q1` gate to create near-zero voltage drop reverse-blocking circuit:
   - `U1` Pin 1 (`VS`) & Pin 4 (`IN`) connected to `VBAT_FUSED`
   - `U1` Pin 2 (`GND`) & Pin 3 (`OFF`) connected to `GND`
   - `U1` Pin 5 (`GATE`) → `R1` (10 Ω) → `Q1` Gate
   - `C2` (100 nF) from `GATE` to `GND` sets soft-start time (~1 ms)
   - `Q1` Source → `VBAT_FUSED`, Drain → `VBUS_PROT` / `VBAT_SENSE`
   - `U1` Pin 6 (`OUT`) → `VBUS_PROT`

**Why Ideal Diode?** Eliminates 0.3–0.7V Schottky forward drop, reducing thermal dissipation by ~90% at 2A load (P_loss = I² × R_DS(on) ≈ 0.1W vs. 1.4W for Schottky).

4. **EMI Filter:** `L1` (10 µH) + `C1` (22 µF) form LC low-pass filter:
   - Cutoff frequency: $f_c = \frac{1}{2\pi\sqrt{LC}} = \frac{1}{2\pi\sqrt{10\mu H \times 22\mu F}} \approx 10.7\text{ kHz}$
   - Output: `VBUS_FILT` (cleaned bus for regulators)

**Outputs:** `VBUS_FILT` (to Sheets 2–5), `VBAT_SENSE` (to Sheet 6)

---

### Sheet 2: `02_3V3_DIG_Regulator.kicad_sch` — Switched 3.3V Digital Logic Rail

**Components:**

| Designator | Value | Footprint | Function |
|------------|-------|-----------|----------|
| `Q2` | SI2301 | SOT-23 | P-MOSFET high-side load switch |
| `Q3` | Q_NMOS_GDS | SOT-23 | N-MOSFET level shifter |
| `R2` | 100 kΩ | R_0603 | P-MOSFET gate pull-up |
| `R3` | 10 kΩ | R_0603 | N-MOSFET gate resistor |
| `U2` | AMS1117-3.3 | SOT-223 | 3.3V LDO regulator |
| `C2`, `C3` | 10 µF ∥ 100 nF | C_0805, C_0603 | Input decoupling |
| `C4`, `C5` | 10 µF ∥ 100 nF | C_0805, C_0603 | Output decoupling |
| `R4` | 0.1 Ω, 1% | R_1206 | Current shunt |

**Circuit Operation:**

1. **Load Switch:** `Q2` (P-MOSFET) controlled by `EN_3V3_DIG` via `Q3` (N-MOSFET level shifter):
   - `EN_3V3_DIG` = HIGH (3.3V) → `Q3` ON → `Q2` Gate pulled LOW → `Q2` ON → Rail enabled
   - `EN_3V3_DIG` = LOW (0V) → `Q3` OFF → `Q2` Gate pulled HIGH via `R2` → `Q2` OFF → Rail disabled
   - `R2` (100 kΩ) ensures `Q2` stays OFF during OBC reset (GPIO high-impedance)

2. **Regulation:** `U2` (AMS1117-3.3) provides 3.3V output:
   - Input: `VBUS_FILT` → `C2` (10 µF) ∥ `C3` (100 nF) → `U2` Pin 3 (`IN`)
   - Output: `U2` Pin 2 (`OUT`) → `C4` (10 µF) ∥ `C5` (100 nF) → `ISENSE_DIG_P`
   - `U2` Pin 1 (`GND`) → `GND`

3. **Current Sensing:** `R4` (0.1 Ω, 1206) placed between `ISENSE_DIG_P` and `3V3_DIG` output:
   - Voltage drop: $V_{R4} = I_{load} \times 0.1\ \Omega$
   - Differential sense lines routed to Sheet 6 (`INA199`)

**Maximum Load:** 500 mA (AMS1117 absolute max: 1A, derated for thermal)

---

### Sheet 3: `03_3V3_ANA_Regulator.kicad_sch` — Switched Low-Noise 3.3V Analog Rail

**Components:**

| Designator | Value | Footprint | Function |
|------------|-------|-----------|----------|
| `Q4` | SI2301 | SOT-23 | P-MOSFET high-side load switch |
| `Q5` | Q_NMOS_GDS | SOT-23 | N-MOSFET level shifter |
| `R5` | 100 kΩ | R_0603 | P-MOSFET gate pull-up |
| `R6` | 10 kΩ | R_0603 | N-MOSFET gate resistor |
| `U3` | REF5033 | SOIC-8 | Precision voltage reference |
| `C6`, `C7` | 10 µF ∥ 100 nF | C_0805, C_0603 | Input decoupling |
| `C8`, `C9` | 10 µF ∥ 100 nF | C_0805, C_0603 | Output decoupling |
| `C10` | 1 µF | C_0603 | Noise bypass (`Trim/NR` pin) |
| `R7` | 0.1 Ω, 1% | R_1206 | Current shunt |

**Circuit Operation:**

1. **Load Switch:** Identical topology to Sheet 2 (`Q4`, `Q5`, `R5`, `R6`), driven by `EN_3V3_ANA`.

2. **Precision Reference:** `U3` (REF5033) chosen over standard LDO for ultra-low noise:
   - **Why REF5033?**
     - Output noise: 3.3 µVrms (10 Hz – 10 kHz) vs. ~300 µVrms for AMS1117
     - Temperature drift: 8 ppm/°C vs. ~50 ppm/°C for standard LDO
     - PSRR: 85 dB at 1 kHz vs. ~60 dB for AMS1117
   - Pin 2 (`Vin`) → `VBUS_FILT` via `C6` ∥ `C7`
   - Pin 4 (`GND`) → `GND`
   - Pin 3 (`Temp`) → No-Connect (NC flag)
   - Pin 5 (`Trim/NR`) → `C10` (1 µF) → `GND` (noise filtering)
   - Pin 6 (`Vout`) → `C8` ∥ `C9` → `ISENSE_ANA_P`

3. **Current Sensing:** `R7` (0.1 Ω) between `ISENSE_ANA_P` and `3V3_ANA` output.

**Maximum Load:** 200 mA (REF5033 absolute max: 10 mA without heatsink; external pass transistor recommended for >50 mA — future revision)

---

### Sheet 4: `04_5V0_IO_Regulator.kicad_sch` — Switched 5.0V Peripheral & RF I/O Rail

**Components:**

| Designator | Value | Footprint | Function |
|------------|-------|-----------|----------|
| `Q6` | SI2301 | SOT-23 | P-MOSFET high-side load switch |
| `Q7` | Q_NMOS_GDS | SOT-23 | N-MOSFET level shifter |
| `R8` | 100 kΩ | R_0603 | P-MOSFET gate pull-up |
| `R9` | 10 kΩ | R_0603 | N-MOSFET gate resistor |
| `U4` | AMS1117-5.0 | SOT-223 | 5.0V LDO regulator |
| `C11`, `C12` | 10 µF ∥ 100 nF | C_0805, C_0603 | Input decoupling |
| `C13`, `C14` | 10 µF ∥ 100 nF | C_0805, C_0603 | Output decoupling |
| `R10` | 0.1 Ω, 1% | R_1206 | Current shunt |

**Circuit Operation:**

1. **Load Switch:** Identical topology to Sheets 2–3 (`Q6`, `Q7`, `R8`, `R9`), driven by `EN_5V0_IO`.

2. **Regulation:** `U4` (AMS1117-5.0) provides 5.0V output:
   - Input: `VBUS_FILT` → `C11` ∥ `C12` → `U4` Pin 3 (`IN`)
   - Output: `U4` Pin 2 (`OUT`) → `C13` ∥ `C14` → `ISENSE_5V0_P`
   - `U4` Pin 1 (`GND`) → `GND`

3. **Current Sensing:** `R10` (0.1 Ω) between `ISENSE_5V0_P` and `5V0_IO` output.

**Maximum Load:** 300 mA (for RF transceivers, actuators, peripherals)

---

### Sheet 5: `05_3V3_MEM_Regulator.kicad_sch` — Always-On 3.3V Non-Volatile Memory Rail

**Components:**

| Designator | Value | Footprint | Function |
|------------|-------|-----------|----------|
| `U5` | AMS1117-3.3 | SOT-223 | 3.3V LDO regulator (always-on) |
| `C15`, `C16` | 10 µF ∥ 100 nF | C_0805, C_0603 | Input decoupling |
| `C17`, `C18` | 10 µF ∥ 100 nF | C_0805, C_0603 | Output decoupling |
| `R11` | 0.1 Ω, 1% | R_1206 | Current shunt |

**Circuit Operation:**

1. **Always-On Architecture:** No load switch — direct connection from `VBUS_FILT` to `U5` input:
   - **Mission Role:** Retains FRAM/EEPROM/Flash state across OBC resets, brownouts, and safe-mode events
   - **Quiescent Current:** ~5 mA (AMS1117 I_Q) + memory leakage (<100 µA)
   - **Use Cases:** Watchdog timer backup, mission clock, fault log storage, calibration constants

2. **Regulation:** `U5` (AMS1117-3.3) with standard decoupling:
   - Input: `VBUS_FILT` → `C15` ∥ `C16` → `U5` Pin 3 (`IN`)
   - Output: `U5` Pin 2 (`OUT`) → `C17` ∥ `C18` → `ISENSE_MEM_P`
   - `U5` Pin 1 (`GND`) → `GND`

3. **Current Sensing:** `R11` (0.1 Ω) between `ISENSE_MEM_P` and `3V3_MEM` output.

**Maximum Load:** 100 mA (sufficient for FRAM/EEPROM + real-time clock)

---

### Sheet 6: `06_Telemetry_Interface.kicad_sch` — High-Side Current & Bus Voltage Telemetry

**Components:**

| Designator | Value | Footprint | Function |
|------------|-------|-----------|----------|
| `U6`–`U9` | INA199xxDCK | SOT-23-6 | Current sense amplifiers (G = 50 V/V) |
| `C19`–`C22` | 100 nF | C_0603 | INA199 supply bypass |
| `R12`–`R15` | 1 kΩ | R_0603 | RC anti-aliasing filter resistors |
| `C23`–`C26` | 10 nF | C_0603 | RC anti-aliasing filter capacitors |
| `R16` | 22 kΩ | R_0603 | Voltage divider (high-side) |
| `R17` | 10 kΩ | R_0603 | Voltage divider (low-side) |
| `C27` | 100 nF | C_0603 | Voltage divider filter |

**Circuit Operation:**

1. **Current Sense Amplifiers (×4):**
   - `U6` (`IMON_3V3_DIG`): `IN+` (Pin 4) → `ISENSE_DIG_P`, `IN-` (Pin 5) → `3V3_DIG`
   - `U7` (`IMON_3V3_ANA`): `IN+` (Pin 4) → `ISENSE_ANA_P`, `IN-` (Pin 5) → `3V3_ANA`
   - `U8` (`IMON_5V0_IO`): `IN+` (Pin 4) → `ISENSE_5V0_P`, `IN-` (Pin 5) → `5V0_IO`
   - `U9` (`IMON_3V3_MEM`): `IN+` (Pin 4) → `ISENSE_MEM_P`, `IN-` (Pin 5) → `3V3_MEM`
   - All amplifiers powered by `3V3_DIG` with `C19`–`C22` (100 nF) bypass
   - `REF` (Pin 1) & `GND` (Pin 2) → `GND`

2. **Anti-Aliasing Filters (×4):**
   - Each `OUT` (Pin 6) → `R12`–`R15` (1 kΩ) → `C23`–`C26` (10 nF) → `GND`
   - Cutoff frequency: $f_c = \frac{1}{2\pi RC} = \frac{1}{2\pi \times 1\text{ k}\Omega \times 10\text{ nF}} \approx 15.9\text{ kHz}$
   - Outputs: `IMON_3V3_DIG`, `IMON_3V3_ANA`, `IMON_5V0_IO`, `IMON_3V3_MEM` (to OBC ADC)

3. **Battery Voltage Divider:**
   - `VBAT_SENSE` → `R16` (22 kΩ) → `R17` (10 kΩ) → `GND`
   - Divider node → `C27` (100 nF) → `GND` (low-pass filter)
   - Filter cutoff: $f_c = \frac{1}{2\pi R_{eq} C} = \frac{1}{2\pi \times (22k \parallel 10k) \times 100nF} \approx 232\text{ Hz}$
   - Output: `VMON_VBAT` (to OBC ADC)
   - Division ratio: $V_{MON} = V_{BAT} \times \frac{10\text{ k}\Omega}{22\text{ k}\Omega + 10\text{ k}\Omega} = 0.3125 \times V_{BAT}$

---

### Sheet 7: `07_Control_Interface.kicad_sch` — OBC Backplane Connector & Safe-State Control

**Connector:** `J2` (Conn_02x08_Odd_Even, 16-pin double-row female header)

**Pinout Table:**

| Pin | Signal | Type | Description |
|-----|--------|------|-------------|
| 1 | `3V3_DIG` | Power | Digital logic supply (500 mA max) |
| 2 | `GND` | Ground | System ground |
| 3 | `3V3_ANA` | Power | Analog/sensor supply (200 mA max) |
| 4 | `GND` | Ground | System ground |
| 5 | `5V0_IO` | Power | Peripheral/RF supply (300 mA max) |
| 6 | `GND` | Ground | System ground |
| 7 | `3V3_MEM` | Power | Always-on memory supply (100 mA max) |
| 8 | `GND` | Ground | System ground |
| 9 | `IMON_3V3_DIG` | Analog | Current telemetry (0–3.3V = 0–660 mA) |
| 10 | `IMON_3V3_ANA` | Analog | Current telemetry (0–3.3V = 0–660 mA) |
| 11 | `IMON_5V0_IO` | Analog | Current telemetry (0–3.3V = 0–660 mA) |
| 12 | `IMON_3V3_MEM` | Analog | Current telemetry (0–3.3V = 0–660 mA) |
| 13 | `VMON_VBAT` | Analog | Battery voltage telemetry (0–3.3V = 0–10.56V) |
| 14 | `EN_3V3_DIG` | GPIO | Rail enable (HIGH = ON, pulled LOW by R18) |
| 15 | `EN_3V3_ANA` | GPIO | Rail enable (HIGH = ON, pulled LOW by R19) |
| 16 | `EN_5V0_IO` | GPIO | Rail enable (HIGH = ON, pulled LOW by R20) |

**Fail-Safe Boot Pull-Downs:**

- `R18` (100 kΩ): `EN_3V3_DIG` → `GND`
- `R19` (100 kΩ): `EN_3V3_ANA` → `GND`
- `R20` (100 kΩ): `EN_5V0_IO` → `GND`

**Design Rationale:** During OBC reset or boot, GPIO pins are high-impedance. Pull-down resistors ensure all switched rails remain OFF until firmware explicitly enables them, preventing inrush current spikes and undefined states.

---

## 4. Telemetry Transfer Equations & Firmware ADC Conversion Formulas

### Rail Current Measurement

**Transfer Function:**

$$V_{\text{IMON}} = I_{\text{rail}} \cdot R_{\text{shunt}} \cdot G_{\text{INA199}}$$

$$V_{\text{IMON}} = I_{\text{rail}} \cdot 0.1\ \Omega \cdot 50\ \text{V/V} = 5.0 \cdot I_{\text{rail}}$$

**Firmware Conversion (ADC Code → Amperes):**

$$I_{\text{rail}}\ (\text{A}) = \frac{V_{\text{IMON}}}{5.0}$$

**Full-Scale Range:**

- ADC reference: 3.3V
- Maximum measurable current: $I_{max} = \frac{3.3\text{ V}}{5.0\ \text{V/A}} = 660\text{ mA}$
- ADC resolution (12-bit): $\frac{660\text{ mA}}{4096} \approx 0.16\text{ mA/bit}$

**Example Calculation:**
- ADC reads 2.5V on `IMON_3V3_DIG`
- Current: $I = \frac{2.5\text{ V}}{5.0} = 0.5\text{ A} = 500\text{ mA}$

---

### Battery Bus Voltage Measurement

**Transfer Function:**

$$V_{\text{MON}} = V_{\text{BAT}} \cdot \frac{R_{17}}{R_{16} + R_{17}} = V_{\text{BAT}} \cdot \frac{10\text{ k}\Omega}{22\text{ k}\Omega + 10\text{ k}\Omega} = 0.3125 \cdot V_{\text{BAT}}$$

**Firmware Conversion (ADC Code → Volts):**

$$V_{\text{BAT}}\ (\text{V}) = \frac{V_{\text{MON}}}{0.3125} = 3.2 \cdot V_{\text{MON}}$$

**Full-Scale Range:**

- ADC reference: 3.3V
- Maximum measurable voltage: $V_{max} = \frac{3.3\text{ V}}{0.3125} = 10.56\text{ V}$
- ADC resolution (12-bit): $\frac{10.56\text{ V}}{4096} \approx 2.58\text{ mV/bit}$

**Example Calculation:**
- ADC reads 2.5V on `VMON_VBAT`
- Battery voltage: $V_{BAT} = 3.2 \times 2.5\text{ V} = 8.0\text{ V}$ (fully charged 2S Li-Ion)

---

### Combined Telemetry Decoding (C Example)

```c
// ADC to Physical Units Conversion
#define ADC_REF         3.3f
#define ADC_BITS        12
#define ADC_MAX         ((1 << ADC_BITS) - 1)
#define SHUNT_GAIN      5.0f       // 0.1Ω × 50 V/V
#define VBAT_DIV_RATIO  0.3125f    // 10k / (22k + 10k)

float adc_to_current(uint16_t adc_code) {
    float v_imon = (adc_code / (float)ADC_MAX) * ADC_REF;
    return v_imon / SHUNT_GAIN;  // Amperes
}

float adc_to_battery_voltage(uint16_t adc_code) {
    float v_mon = (adc_code / (float)ADC_MAX) * ADC_REF;
    return v_mon / VBAT_DIV_RATIO;  // Volts
}

// Usage Example
uint16_t imon_dig = adc_read(ADC_CH_IMON_3V3_DIG);
float current_dig = adc_to_current(imon_dig);  // e.g., 0.350 A

uint16_t vmon_bat = adc_read(ADC_CH_VMON_VBAT);
float vbat = adc_to_battery_voltage(vmon_bat);  // e.g., 7.8 V
```

---

## 5. Complete Bill of Materials (BOM) & PCB Layout Guidelines

### Consolidated Bill of Materials

| Ref | Qty | Value / Part Number | Footprint | Manufacturer | Description |
|-----|-----|---------------------|-----------|--------------|-------------|
| `J1` | 1 | Conn_01x02 | Terminal_Block_1x02 | Phoenix Contact | Battery input connector |
| `J2` | 1 | Conn_02x08_Odd_Even | PinSocket_2x08_P2.54mm | Harwin | 16-pin OBC backplane |
| `F1` | 1 | 2A Fast-Blow | Fuse_Holder_1206 | Littelfuse | Overcurrent protection |
| `D1` | 1 | SMBJ12A | DO-214AA (SMB) | Vishay | TVS diode, 12V clamp |
| `U1` | 1 | LM5050-1 | SOT-23-6 | TI | Ideal diode controller |
| `Q1` | 1 | QM6006D / SI2302 | SOT-23 | Diodes Inc | N-MOSFET, 20V, 6A |
| `Q2`, `Q4`, `Q6` | 3 | SI2301 | SOT-23 | Vishay | P-MOSFET, 20V, 2.5A |
| `Q3`, `Q5`, `Q7` | 3 | Q_NMOS_GDS | SOT-23 | Generic | N-MOSFET level shifter |
| `U2`, `U5` | 2 | AMS1117-3.3 | SOT-223 | Advanced Monolithic | 3.3V LDO, 1A |
| `U3` | 1 | REF5033 | SOIC-8 | TI | Precision voltage reference |
| `U4` | 1 | AMS1117-5.0 | SOT-223 | Advanced Monolithic | 5.0V LDO, 1A |
| `U6`–`U9` | 4 | INA199xxDCK | SOT-23-6 | TI | Current sense amplifier, G=50 |
| `L1` | 1 | 10 µH | L_0805 | Murata | EMI filter inductor |
| `R1` | 1 | 10 Ω | R_0603 | Yageo | Gate resistor |
| `R2`, `R5`, `R8` | 3 | 100 kΩ | R_0603 | Yageo | P-MOSFET gate pull-up |
| `R3`, `R6`, `R9` | 3 | 10 kΩ | R_0603 | Yageo | N-MOSFET gate resistor |
| `R4`, `R7`, `R10`, `R11` | 4 | 0.1 Ω, 1% | R_1206 | Yageo | Current shunt resistors |
| `R12`–`R15` | 4 | 1 kΩ | R_0603 | Yageo | Anti-aliasing filter |
| `R16` | 1 | 22 kΩ | R_0603 | Yageo | Voltage divider (high) |
| `R17` | 1 | 10 kΩ | R_0603 | Yageo | Voltage divider (low) |
| `R18`–`R20` | 3 | 100 kΩ | R_0603 | Yageo | Fail-safe pull-down |
| `C1` | 1 | 22 µF | C_0805 | Murata | Input filter capacitor |
| `C2` | 1 | 100 nF | C_0603 | Murata | Soft-start capacitor |
| `C3`–`C18` | 16 | 10 µF ∥ 100 nF | C_0805, C_0603 | Murata | Regulator decoupling |
| `C19`–`C22` | 4 | 100 nF | C_0603 | Murata | INA199 supply bypass |
| `C23`–`C26` | 4 | 10 nF | C_0603 | Murata | Anti-aliasing filter |
| `C27` | 1 | 100 nF | C_0603 | Murata | Voltage divider filter |

**Total Component Count:** 54 (excluding mounting hardware)

**Estimated Cost (prototype qty 5):** ~$85 USD (~$17/board)

---

### PCB Layout Best Practices

#### 1. Current Sense Routing (Kelvin 4-Wire Differential)

- **Trace Pairing:** Route `ISENSE_*_P` and regulated rail as differential pair to INA199 pins
- **Via Matching:** Use matched via pairs for both sense lines (same length, same layer transitions)
- **Keep-Out Zone:** No high-current traces or switching nodes within 5mm of sense traces
- **Guard Ring:** Surround sense traces with `GND` guard ring on same layer

#### 2. Analog Ground Isolation (`3V3_ANA`)

- **Star Ground Topology:** Connect `3V3_ANA` ground return directly to input capacitor ground point
- **No Shared Returns:** Do not share ground traces with digital or power returns
- **Ground Plane Split:** Maintain continuous ground plane; do not split under analog traces

#### 3. Thermal Management (AMS1117 SOT-223)

- **Copper Pour:** Minimum 2 cm² copper area on top layer connected to `OUT` pin thermal pad
- **Stitching Vias:** 8–12 vias (0.3mm drill, 0.6mm pad) under thermal pad to inner/bottom ground planes
- **Component Spacing:** Minimum 10mm between LDOs for airflow
- **Thermal Shutdown:** Place temperature sensor (if added) near hottest LDO

#### 4. Input Protection Placement

- **Edge Placement:** `J1`, `F1`, `D1` at board edge for shortest transient path
- **TVS Grounding:** `D1` cathode → `GND` via shortest path (direct via to ground plane)
- **Filter Positioning:** `L1` + `C1` immediately after TVS, before any branching

#### 5. Connector Interface

- **Mounting Holes:** 4× M3 holes at corners (mechanical support for backplane connection)
- **Keep-Out:** 5mm keep-out around `J2` for mating connector clearance
- **Silkscreen:** Label all pins on top silkscreen (Pin 1 square pad, odd/even markers)

#### 6. Design Rule Check (DRC) Settings

| Parameter | Value |
|-----------|-------|
| Minimum Track Width | 0.25mm (signal), 0.5mm (power ≥500mA) |
| Minimum Clearance | 0.2mm (signal-signal), 0.3mm (power-signal) |
| Minimum Via Drill | 0.3mm |
| Minimum Via Pad | 0.6mm |
| Annular Ring | 0.15mm |
| Copper-to-Board Edge | 0.5mm |

---

## Verification Milestones

| Milestone | Status | Date | Notes |
|-----------|--------|------|-------|
| Schematic Capture Complete | ✓ | TBD | All 8 sheets finished |
| ERC Passed (0 Errors, 0 Warnings) | ✓ | TBD | All hierarchical labels match |
| Component Footprints Assigned | ☐ | TBD | Footprint wizard validation pending |
| PCB Layout Started | ☐ | TBD | 4-layer stackup defined |
| DRC Passed | ☐ | TBD | Pending layout completion |
| Gerber Generation | ☐ | TBD | Pending DRC sign-off |
| Prototype Assembly | ☐ | TBD | JLCPCB assembly quote requested |
| Bench Validation | ☐ | TBD | 6 test procedures defined |

---

## References

### Datasheets
1. Texas Instruments LM5050-1: https://www.ti.com/product/LM5050-1
2. Texas Instruments INA199: https://www.ti.com/product/INA199
3. Texas Instruments REF5033: https://www.ti.com/product/REF5033
4. Advanced Monolithic AMS1117: https://www.advanced-monolithic.com/ams1117.pdf

### Standards
5. ESA ECSS-E-ST-50-05C: Space Engineering — Electrical Power Systems
6. NASA-STD-4003: Spacecraft Electrical Power System Design

### Application Notes
7. TI AN-1144: Current Sensing for Power Management
8. TI AN-1542: LDO Thermal Design Considerations

---

## Author

**Dhruvrajsinh Vansia**  
MSc Space Sciences and Technologies, University of Bremen  
GitHub: https://github.com/dvvansia  
Location: Bremen, Germany

---

*Part of Satellite Engineering Track portfolio projects for career in spacecraft avionics and power systems.*

**Related Projects:**
- [Buck Converter LTSpice](https://github.com/dvvansia/buck_convertor_LTSpice) — 93% efficiency DC-DC converter
- [ADC Sensor Interface Proteus](https://github.com/dvvansia/adc_sensor_interface_proteus) — Multi-channel telemetry
- [RF Link Budget Calculator](https://github.com/dvvansia/RF_Link_Budget_Calculator) — MATLAB link analysis
- [Satellite Sensor Node KiCAD](https://github.com/dvvansia/Satelliet_Sensor_Node) — KiCAD schematic design
