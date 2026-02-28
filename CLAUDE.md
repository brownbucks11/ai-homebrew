# CLAUDE.md — AI Homebrew Project Context

> This file provides persistent context for Claude Code. It captures the full project state
> from multiple conversation sessions with the project owner (Rob), a controls engineer
> migrating a home brewing controller from Allen-Bradley Micro820 to an AI-writable platform.

## Project Owner

- **Name:** Rob (brownbucks11 on GitHub)
- **Background:** Professional controls/instrumentation engineer (familiar with ABB 800xA, DCS, PLC programming, ISA standards)
- **Goal:** Test whether AI can meaningfully assist in migrating and rewriting real PLC automation code — not toy demos, but production control logic with PID loops, safety interlocks, and proper mode control
- **Blog:** Documenting the journey as a public blog series on GitHub Pages

## Repository Structure

```
ai-homebrew/
├── _config.yml              # Jekyll config for GitHub Pages
├── Gemfile                  # Jekyll dependencies
├── blog/
│   ├── _posts/              # Published blog posts
│   ├── _drafts/             # Work in progress
│   └── index.md             # Blog landing page
├── docs/
│   ├── drawings/            # Draw.io engineering drawings (3-page .drawio file)
│   ├── ccw-analysis/        # Extracted PLC program analysis
│   └── photos/              # Build photos (2019 panel build)
├── baseline/
│   ├── ccw-project/         # CCW project export (reference only — proprietary binary)
│   └── ignition/            # Ignition project backup
├── new-platform/            # Future: migrated controller code
└── assets/                  # Images, CSS for blog
```

## Blog Series Roadmap

| # | Title | Status | Content |
|---|-------|--------|---------|
| 1 | **The Baseline** | Draft in `blog/_drafts/` | Current system documentation |
| 2 | **Can AI Read My PLC Code?** | Planned | Claude parsing CCW FBD project |
| 3 | **Picking the New Platform** | Planned | Platform evaluation and comparison |
| 4 | **The Rewrite** | Planned | AI-assisted code translation |
| 5 | **Commissioning & Cutover** | Planned | Running both systems, comparing |
| 6 | **Lessons Learned** | Planned | What worked, what didn't |

---

## Existing System — Complete Technical Reference

### Hardware

**Controller:** Allen-Bradley Micro820 (2080-LC20-20QBB)
- 12 DI / 7 DO / 4 AI / 1 AO
- Ethernet (Modbus TCP)
- 2080-SERIALISOL serial module (available, unused)

**Power Distribution (DIN Rail 1):**
- Main breaker 65A
- C8 breaker → Omron S8VK-S06024 24VDC 60W power supply
- 2× C32 2-pole breakers (Element 1, Element 2)
- 2× HC1-63 contactors (element power isolation)
- ASI E-Stop safety relay (hardwired, not PLC DI)
- Ground terminal bus

**SSR Assembly (Panel-mounted between Rails 2 & 3):**
- 2× Magnecraft W6240ASX-1 SSRs on heatsinks (240V/40A)
- SSR-1 → Element 1 PWM (controlled by DO-04)
- SSR-2 → Element 2 PWM (controlled by DO-00)

**Control Electronics (DIN Rail 3):**
- Omron S8VK-S06024 24VDC 60W power supply
- Marshalling terminal blocks (green, blue, orange, red — color-coded)
- Allen-Bradley Micro820 PLC
- 2080-SERIALISOL serial module
- 4× voltage divider resistor pairs (NTC thermistor → 0-10V AI scaling)

**Operator Face Panel:**
- E-STOP mushroom button (red, top center) — hardwired through ASI safety relay
- Key switch (far left) — LOC/REM mode select → DI-00
- Green pushbutton — ENABLE → DI-01
- Red pushbutton — STOP → DI-02
- 2× Green illuminated pushbuttons — PMP1 (DI-07), PMP2 (DI-11)
- 2× White pushbuttons — W1TPMP (DI-05), RESET (TBD)
- 1× Blue pushbutton — BOIL (DI-08)
- 3× Amber selector switches — HLT (DI-03/04), 1/2OPMP (DI-06), MODE (DI-09/10)
- Digital temperature display (panel meter, center)

**HMI:** HP Slate 21 Pro touchscreen running Ignition Community Edition

### I/O Map

**Analog Inputs (NTC thermistors via voltage divider networks):**
| Tag | Terminal | Function |
|-----|----------|----------|
| `_IO_EM_AI_00` | AI-00 (Volt0) | Temp Sensor 1 → Temp1degF |
| `_IO_EM_AI_01` | AI-01 (Volt1) | Temp Sensor 2 → Temp2degF |
| `_IO_EM_AI_02` | AI-02 (Volt2) | Temp Sensor 3 → Temp3degF |
| `_IO_EM_AI_03` | AI-03 (Volt3) | Temp Sensor 4 → Temp4degF |

**Discrete Outputs:**
| Tag | Terminal | Function | Destination |
|-----|----------|----------|-------------|
| `_IO_EM_DO_00` | O-00 | Element 2 PWM | SSR-2 → Kettle |
| `_IO_EM_DO_04` | O-04 | Element 1 PWM | SSR-1 → HLT/Mash |
| `_IO_EM_DO_05` | O-05 | Pump 1 (P1_ON) | Pump contactor |
| `_IO_EM_DO_06` | O-06 | Pump 2 (P2_ON) | Pump contactor |

**Discrete Inputs:** DI-00 through DI-11 — panel pushbuttons, selector switches, E-STOP feedback, key switch (see face panel section above for mapping)

**Communications:** Ethernet RJ45 → Modbus TCP (~100 tags) → Ignition Community Edition

### CCW Program Architecture (16 programs, execution order)

1. **TempIn** — Raw voltage scaling: 4× AI → °C → °F via SCALER + CtoF blocks
2. **DIO** — Discrete I/O mapping (basic, nearly empty)
3. **AIO** — Analog input scaling with Linear/Poly3 for non-linear sensors
4. **TempCorrection** — Selectable temp sources per zone (MashTempSel, HLTTempSel, KettleTempSel pick from 4 sensors) + offset calibration
5. **BrewDay** — Boil timer logic (TON with countdown)
6. **E1_Control** — Element 1: PID → PWM with auto-tune, manual power override, ESTOP interlock, remote/auto mode
7. **E2_Control** — Element 2: identical architecture to E1
8. **MODE_HMI_CTL** — Mode HMI interface logic
9. **E1_HMI_CTL** — Element 1 HMI command/status interface
10. **P2_HMI_CTL** — Pump 2 HMI interface
11. **P1_HMI_CTL** — Pump 1 HMI interface
12. **E2_HMI_CTL** — Element 2 HMI interface
13. **MODE_Control** — Master mode controller: HLT/Mash/Kettle modes, SP routing, E1_MAX overtemp protection with TON delay, SP_RANGE clamping (130-190°F range with step breakpoints at 7/11/14/19)
14. **Pump_CTL** — Pump 1 & 2 start/stop via RS flip-flops → DO outputs
15. **Timer_Mode** — Mash step timer config (DEMUX_4 to decode timer control, clock values × 60000 → TIME conversion)
16. **TMR1_PGM** — 4-step mash timer with RANGE blocks (temp must be within ±5°F of TSP to count) feeding STEP_TIMER_4

### Custom Function Block Library (39 blocks)

**Control:**
- `RA_TEMP_CONTROLLER` — PID temperature controller
- `RA_TEMP_CONTROLLER_PWM` — PWM output driver for SSRs
- `RA_TEMP_CONTROLLER_TC_RTD` — TC/RTD input handler
- `RA_PID_AUTOTUNE` — PID auto-tune routine

**Data Handling:**
- `MOV_EN` / `MOV_EN4` — Conditional data moves (mode-dependent routing)
- `DEMUX_4` — 4-output demultiplexer
- `Temperature_Select` — 4-input temperature selector
- `VALUE_SEL_4` — 4-input value selector
- `MOVE_REAL` — Real number move

**Timers:**
- `STEP_TIMER_3` / `STEP_TIMER_4` — Multi-step mash timers with pause
- `TON_PAUSE` / `TON_Pauseable` — Pauseable on-delay timers
- `TMR_HMI_20` — Timer HMI interface

**Setpoint Management:**
- `SP_RANGE` — Setpoint range limiter with stepped breakpoints
- `RANGE` — Comparison within tolerance band (±5°F for mash timer)

**Math & Conversion:**
- `CtoF` — Celsius to Fahrenheit
- `Linear` — Linear scaling
- `Poly3` — 3rd-order polynomial (sensor linearization)

### Key Design Patterns
- **ISA-style mode control:** Remote/Local and Auto/Manual with proper HMI command/status separation
- **Flexible temperature routing:** Any of 4 sensors assignable to any zone
- **Over-temperature protection:** E1_MAX with TON debounce timer
- **Setpoint clamping:** SP_RANGE with stepped breakpoints at 7/11/14/19
- **Pauseable mash timers:** Temperature must be within ±5°F of setpoint for timer to count
- **3 control zones:** HLT (Hot Liquor Tank), Mash, Kettle
- **Dual PID loops** with PWM output to SSRs
- **ESTOP interlock** hardwired through ASI safety relay

---

## Platform Evaluation (Completed)

Rob's key requirements for new platform:
- **Economical** (DIY hobby budget)
- **AI-writable code** (Claude/Claude Code can generate and modify)
- **Proper I/O** (not bare microcontrollers with DIY relay boards)
- **Modular architecture** (function blocks, reusable library)
- **Live inspection/forcing** (online monitoring, point override)
- **Decoupled I/O** architecture via Modbus TCP, MQTT, or similar

### Recommended Options

**Budget Tier (~$175-195):**
- Raspberry Pi 4 + Node-RED or OpenPLC (free)
- Waveshare Modbus RTU I/O modules (~$120)
- MQTT → Ignition Maker Edition (free)

**Mid Tier (~$350-600):**
- RevPi Connect 4 ($275) or UniPi Neuron M203 (~$300)
- Advantech ADAM-6000 Modbus TCP I/O (native MQTT)
- Node-RED or CODESYS

**Professional Tier (~$1,500-2,000):**
- Opto 22 groov RIO
- Universal auto-configuring I/O
- Node-RED as first-class programming environment
- Native MQTT/Sparkplug B + Ignition Edge

### HMI Decision
**Ignition Maker Edition** (free for personal use) — full Perspective web HMI, historian, alarming, trending. Connects via MQTT or OPC-UA.

### AI Code Generation Paths
| Target | AI Reliability | Format |
|--------|---------------|--------|
| Node-RED flows | Excellent | JSON — Claude's sweet spot |
| Structured Text (CODESYS/OpenPLC) | Very good | Pascal-like text |
| PLCopen XML FBD (OpenPLC) | Good | ISO standard XML schema |
| CODESYS FBD XML | Fragile | Proprietary, coordinate-heavy |

---

## Engineering Drawings

Located in `docs/drawings/BrewController_Drawings.drawio` — 3 pages:

1. **Interior Panel Layout** (PNL-001 Rev B) — DIN rail component layout at 28 pts/inch scale, uses mxgraph.cabinets shape library, includes I/O assignment table
2. **Operator Face Panel** (PNL-002 Rev B) — Front view with all buttons/switches, DI mapping table
3. **System Architecture** (SYS-001 Rev A) — Full signal flow: field instruments → signal conditioning → PLC → network → HMI with color-coded signal paths

The original Draw.io file from Rob is preserved as `ComponentLayoutRev2_original.html`.

All drawings use Draw.io format and the `mxgraph.cabinets` library for proper electrical symbols.

---

## Conversation History Summary

### Session 1 — Platform Planning
Evaluated controller options from ESP32 through industrial PLCs. Narrowed to decoupled architecture with separate logic solver and remote I/O. Established blog series roadmap.

### Session 2 — CCW Project Analysis
Rob uploaded the complete CCW project export (Hannan_BrewControllerV5_1.zip). Parsed all 16 programs, 39 function blocks, I/O map, Modbus config. Documented the full control architecture.

### Session 3 — Panel Layout & Drawings
Rob uploaded 10 build photos (Feb-Jun 2019). Analyzed operator face panel and interior DIN rail components. Generated SVG panel layout drawing (later migrated to Draw.io format). Created 3-page Draw.io drawing package.

### Session 4 — Repository Setup (Current)
Created GitHub repo structure, Jekyll blog scaffold, initial README, CCW analysis docs, engineering drawings. Generated this CLAUDE.md for continuity.

---

## What's Next

Immediate priorities:
1. **Push initial repo content** (zip ready, needs git push from PC)
2. **Enable GitHub Pages** (Settings → Pages → Source: main branch)
3. **Flesh out Part 1 blog post** with photos, drawings, more technical detail
4. **Begin Part 2** — document how Claude analyzed the CCW project
5. **Finalize platform selection** — Rob hasn't committed to a specific tier yet

Future work:
- AI translation of CCW FBD logic to target platform code
- Generate Node-RED flows or Structured Text from CCW analysis
- Commission new system and compare to baseline
- P&ID drawing (vessels, pumps, instruments, control loops)
- Wiring diagrams and electrical one-line

---

## Working Conventions

- **Drawings:** Use Draw.io format (.drawio), not SVG — Rob prefers editable source
- **Scale:** 28 pts = 1 inch for panel layouts
- **Shape library:** mxgraph.cabinets for DIN rail, terminals, breakers, contactors
- **Blog:** Jekyll on GitHub Pages with minima theme
- **Tone:** Technical but accessible — this is aimed at engineers and advanced hobbyists
- **DI mapping:** Estimated from CCW code analysis — needs verification against physical wiring
- **E-STOP:** Hardwired through ASI safety relay, NOT a PLC discrete input
