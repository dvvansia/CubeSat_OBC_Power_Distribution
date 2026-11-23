# CubeSat OBC Power Distribution Unit (PDU)

[![KiCAD](https://img.shields.io/badge/KiCAD-9.0-blue)](https://www.kicad.org)
[![ERC](https://img.shields.io/badge/ERC-Passed-brightgreen)](./Hardware/CubeSat_PDU.kicad_sch/)
[![License](https://img.shields.io/badge/License-MIT-green)](./LICENSE)

**Flight-grade power distribution system for 1U CubeSat On-Board Computer with 4 regulated rails, current telemetry, and comprehensive protection.**

---

## 🚀 Quick Links

| Document | Link |
|----------|------|
| **Full Documentation** | [Documentation/README.md](./Documentation/README.md) |
| **KiCAD Schematic** | [Hardware/CubeSat_PDU.kicad_sch/](./Hardware/CubeSat_PDU.kicad_sch/) |
| **Sheet PDFs** | [Results/](./Results/) |
| **Project Specification** | [Documentation/Project_Specification.md](./Documentation/Project_Specification.md) |

---

## 📋 System Overview

### Input Power
- **Source:** 2S Li-Ion Battery Pack
- **Voltage Range:** 6.0V – 8.4V (nominal 7.4V)
- **Protection:** 2A fast-blow fuse + ideal diode ORing + TVS clamp

### Output Rails

| Rail | Voltage | Max Current | Load Type | Switched |
|------|---------|-------------|-----------|----------|
| `3V3_DIG` | 3.3V | 500 mA | MCU, digital logic | ✅ Yes |
| `3V3_ANA` | 3.3V | 200 mA | Sensors, ADC refs | ✅ Yes |
| `5V0_IO` | 5.0V | 300 mA | Peripherals, RF | ✅ Yes |
| `3V3_MEM` | 3.3V | 100 mA | FRAM/EEPROM | ❌ Always-On |

### Key Features

- ✅ **ERC Passed:** 0 Errors, 0 Warnings
- ✅ **Hierarchical Design:** 8-sheet KiCAD schematic (Root + 7 sub-sheets)
- ✅ **Current Telemetry:** INA199 high-side sensing on all 4 rails (G = 50 V/V)
- ✅ **Voltage Monitoring:** Battery bus divider (0–10.56V measurable)
- ✅ **Fail-Safe Control:** 100kΩ pull-downs ensure rails OFF during OBC reset
- ✅ **Ideal Diode ORing:** LM5050-1 eliminates Schottky voltage drop
- ✅ **Low-Noise Analog:** REF5033 precision reference for sensor/ADC isolation

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    VBAT_RAW (6-8.4V)                            │
│                          │                                      │
│                    [2A Fuse + TVS]                              │
│                          │                                      │
│              ┌───────────┴───────────┐                         │
│              │  LM5050-1 Ideal Diode │                         │
│              │     + N-MOSFET        │                         │
│              └───────────┬───────────┘                         │
│                          │                                      │
│                    VBUS_FILT (filtered)                         │
│                          │                                      │
│    ┌──────────┬──────────┼──────────┬──────────┐               │
│    ▼          ▼          ▼          ▼          ▼               │
│ ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌────────────┐        │
│ │3V3   │  │3V3   │  │5V0   │  │3V3   │  │Telemetry   │        │
│ │DIG   │  │ANA   │  │IO    │  │MEM   │  │Interface   │        │
│ │AMS   │  │REF   │  │AMS   │  │AMS   │  │INA199 x4   │        │
│ │1117  │  │5033  │  │1117  │  │1117  │  │            │        │
│ └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  └─────┬──────┘        │
│    │         │         │         │            │                │
│    └─────────┴─────────┴─────────┴────────────┘                │
│                          │                                      │
│              ┌───────────┴───────────┐                         │
│              │   16-pin OBC Connector│                         │
│              │   J2 (Power + Telem)  │                         │
│              └───────────────────────┘                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📁 Project Structure

```
CubeSat_OBC_Power_Distribution/
├── Documentation/
│   ├── README.md              # Full engineering documentation
│   └── Project_Specification.md
├── Hardware/
│   └── CubeSat_PDU.kicad_sch/
│       ├── CubeSat_PDU.kicad_sch          # Root sheet
│       ├── 01_Input_Protection.kicad_sch
│       ├── 02_3V3_DIG_Regulator.kicad_sch
│       ├── 03_3V3_ANA_Regulator.kicad_sch
│       ├── 04_5V0_IO_Regulator.kicad_sch
│       ├── 05_3V3_MEM_Regulator.kicad_sch
│       ├── 06_Telemetry_Interface.kicad_sch
│       └── 07_Control_Interface.kicad_sch
├── Results/
│   ├── Sheet_01.pdf  # Input Protection
│   ├── Sheet_02.pdf  # 3V3_DIG
│   ├── Sheet_03.pdf  # 3V3_ANA
│   ├── Sheet_04.pdf  # 5V0_IO
│   ├── Sheet_05.pdf  # 3V3_MEM
│   ├── Sheet_06.pdf  # Telemetry
│   ├── Sheet_07.pdf  # Control Interface
│   └── All_Sheets.pdf  # Complete schematic
├── .gitignore
└── README.md  # This file
```

---

## 🔬 Telemetry Equations

### Current Measurement (All Rails)

$$I_{rail}\ (A) = \frac{V_{IMON}}{5.0}$$

- Shunt: 0.1 Ω, Amplifier Gain: 50 V/V
- Full-scale: 0–660 mA @ 3.3V ADC

### Battery Voltage Measurement

$$V_{BAT}\ (V) = 3.2 \times V_{MON}$$

- Divider: 22kΩ / 10kΩ (ratio: 0.3125)
- Full-scale: 0–10.56V @ 3.3V ADC

---

## 🧪 Verification Status

| Milestone | Status |
|-----------|--------|
| Schematic Capture | ✅ Complete |
| ERC (Electrical Rules Check) | ✅ Passed (0 Errors, 0 Warnings) |
| Hierarchical Label Matching | ✅ Verified |
| Component Footprints | ✅ Assigned |
| PDF Exports (All 8 Sheets) | ✅ Generated |
| Documentation | ✅ Complete |
| PCB Layout | ⏳ Pending |
| Gerber Generation | ⏳ Pending |
| Prototype Assembly | ⏳ Pending |
| Bench Validation | ⏳ Pending |

---

## 📦 BOM Summary

| Category | Count | Key Components |
|----------|-------|----------------|
| Connectors | 2 | J1 (battery input), J2 (16-pin OBC) |
| Protection | 2 | 2A Fuse, SMBJ12A TVS |
| Power ICs | 6 | LM5050-1, AMS1117×3, REF5033 |
| Current Sense | 4 | INA199xxDCK (G=50) |
| MOSFETs | 6 | SI2301 (P-MOS) ×3, SI2302 (N-MOS) ×3 |
| Passives | ~40 | Resistors, Capacitors, Inductor |

**Total Components:** 54 (excluding mounting hardware)  
**Estimated Cost:** ~$17/board (prototype qty 5)

---

## 🛠️ How to Use

### 1. Open in KiCAD

```bash
# Clone repository
git clone https://github.com/dvvansia/CubeSat_OBC_Power_Distribution.git

# Open project
cd CubeSat_OBC_Power_Distribution/Hardware/CubeSat_PDU.kicad_sch/
# Launch KiCAD → Open Project → CubeSat_PDU.kicad_sch.kicad_sch
```

### 2. Run ERC

```
Schematic Editor → Tools → Electrical Rules Check
Expected: 0 Errors, 0 Warnings ✓
```

### 3. Review Documentation

See [Documentation/README.md](./Documentation/README.md) for:
- Complete sheet-by-sheet circuit breakdown
- Component designators and values
- Telemetry transfer equations
- Firmware ADC conversion formulas (with C code)
- PCB layout best practices
- Full BOM with footprints

---

## 📄 License

MIT License — Educational and research use.

---

## 👨‍💻 Author

**Dhruvrajsinh Vansia**  
MSc Space Sciences and Technologies, University of Bremen  
GitHub: https://github.com/dvvansia  
Location: Bremen, Germany

---

## 🔗 Related Projects

| Project | Description | Repository |
|---------|-------------|------------|
| Buck Converter LTSpice | 93% efficiency 12V→3.3V DC-DC | [View](https://github.com/dvvansia/buck_convertor_LTSpice) |
| ADC Sensor Interface | Multi-channel telemetry (Proteus) | [View](https://github.com/dvvansia/adc_sensor_interface_proteus) |
| RF Link Budget Calculator | MATLAB Eb/N0, BER analysis | [View](https://github.com/dvvansia/RF_Link_Budget_Calculator) |
| Satellite Sensor Node | KiCAD schematic design | [View](https://github.com/dvvansia/Satelliet_Sensor_Node) |

---

*Part of Satellite Engineering Track portfolio for career in spacecraft avionics and power systems.*
