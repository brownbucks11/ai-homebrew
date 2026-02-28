# CCW Project Analysis — Hannan_BrewControllerV5_1

> Extracted by Claude from the CCW project export on 2026-02-28

## Hardware Configuration

| Component | Part Number | Description |
|-----------|-------------|-------------|
| PLC | 2080-LC20-20QBB | Micro820, 12 DI / 7 DO / 4 AI / 1 AO |
| Serial Module | 2080-SERIALISOL | Isolated RS-232/485 (available, unused) |
| Power Supply | Omron S8VK-S06024 | 24VDC 60W |

## I/O Map

### Analog Inputs
| Tag | Terminal | Signal | Function |
|-----|----------|--------|----------|
| `_IO_EM_AI_00` | AI-00 (Volt0) | 0-10V | Temp Sensor 1 → Temp1degF |
| `_IO_EM_AI_01` | AI-01 (Volt1) | 0-10V | Temp Sensor 2 → Temp2degF |
| `_IO_EM_AI_02` | AI-02 (Volt2) | 0-10V | Temp Sensor 3 → Temp3degF |
| `_IO_EM_AI_03` | AI-03 (Volt3) | 0-10V | Temp Sensor 4 → Temp4degF |

### Discrete Outputs
| Tag | Terminal | Function | Destination |
|-----|----------|----------|-------------|
| `_IO_EM_DO_00` | O-00 | Element 2 PWM | SSR-2 → Kettle |
| `_IO_EM_DO_04` | O-04 | Element 1 PWM | SSR-1 → HLT/Mash |
| `_IO_EM_DO_05` | O-05 | Pump 1 (P1_ON) | Pump contactor |
| `_IO_EM_DO_06` | O-06 | Pump 2 (P2_ON) | Pump contactor |

### Discrete Inputs (DI-00 through DI-11)
Panel pushbuttons, selector switches, E-STOP feedback, key switch.

### Communications
- Ethernet RJ45 → Modbus TCP → Ignition Community Edition
- 2080-SERIALISOL → RS-232/485 (available, unused)

## Program Execution Order

| # | Program | Description |
|---|---------|-------------|
| 1 | TempIn | Raw voltage scaling: 4× AI → °C → °F via SCALER + CtoF |
| 2 | DIO | Discrete I/O mapping |
| 3 | AIO | Analog input scaling with Linear/Poly3 for non-linear sensors |
| 4 | TempCorrection | Selectable temp sources per zone + offset calibration |
| 5 | BrewDay | Boil timer logic (TON with countdown) |
| 6 | E1_Control | Element 1: PID → PWM with auto-tune, ESTOP interlock |
| 7 | E2_Control | Element 2: identical architecture to E1 |
| 8 | MODE_HMI_CTL | Mode HMI interface |
| 9 | E1_HMI_CTL | Element 1 HMI command/status |
| 10 | P2_HMI_CTL | Pump 2 HMI interface |
| 11 | P1_HMI_CTL | Pump 1 HMI interface |
| 12 | E2_HMI_CTL | Element 2 HMI interface |
| 13 | MODE_Control | Master mode: HLT/Mash/Kettle, SP routing, overtemp protection |
| 14 | Pump_CTL | Pump 1 & 2 via RS flip-flops → DO outputs |
| 15 | Timer_Mode | Mash step timer config (DEMUX_4 decode, ×60000 → TIME) |
| 16 | TMR1_PGM | 4-step mash timer with RANGE blocks (±5°F deadband) |

## Custom Function Blocks (39 total)

### Control
- `RA_TEMP_CONTROLLER` — PID temperature controller
- `RA_TEMP_CONTROLLER_PWM` — PWM output driver for SSRs
- `RA_TEMP_CONTROLLER_TC_RTD` — TC/RTD input handler
- `RA_PID_AUTOTUNE` — PID auto-tune routine

### Data Handling
- `MOV_EN` / `MOV_EN4` — Conditional data moves (mode-dependent routing)
- `DEMUX_4` — 4-output demultiplexer
- `Temperature_Select` — 4-input temperature selector
- `VALUE_SEL_4` — 4-input value selector
- `MOVE_REAL` — Real number move

### Timers
- `STEP_TIMER_3` / `STEP_TIMER_4` — Multi-step mash timers with pause
- `TON_PAUSE` / `TON_Pauseable` — Pauseable on-delay timers
- `TMR_HMI_20` — Timer HMI interface

### Setpoint Management
- `SP_RANGE` — Setpoint range limiter with stepped breakpoints (130-190°F)
- `RANGE` — Comparison within tolerance band (±5°F for mash timer)

### Math & Conversion
- `CtoF` — Celsius to Fahrenheit
- `Linear` — Linear scaling
- `Poly3` — 3rd-order polynomial (sensor linearization)

## Modbus TCP Configuration

~100 tags mapped including:
- All PVs, SPs, and output percentages
- Mode control (HLT/Mash/Kettle selection)
- PID tuning parameters
- Timer status and setpoints
- Auto-tune commands and status
- E-STOP and fault status

## Key Design Patterns

1. **ISA-style mode control** — Remote/Local, Auto/Manual with proper HMI command/status separation
2. **Flexible temperature routing** — Any of 4 sensors assignable to any zone
3. **Over-temperature protection** — E1_MAX with TON debounce timer
4. **Setpoint clamping** — SP_RANGE with stepped breakpoints at 7/11/14/19
5. **Pauseable mash timers** — Temperature must be within ±5°F of setpoint for timer to count
