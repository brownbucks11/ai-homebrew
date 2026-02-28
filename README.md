# AI-Assisted Home Brewing Controller Migration

> Migrating a fully operational DIY home brewing controller from an Allen-Bradley Micro820 PLC to a modern, AI-writable platform — documented as a blog series.

## What Is This?

This is a real-world industrial controls project, built by a controls engineer, now being used as a testbed to explore a question:

**Can AI meaningfully assist in migrating and rewriting PLC automation code?**

The existing system is a 3-zone electric brewery controller (HLT, Mash, Kettle) with dual PID loops, auto-tune, 4-step mash timers, PWM element control via SSRs, and a full Ignition SCADA HMI — all running on an Allen-Bradley Micro820 programmed in Function Block Diagram (FBD) using Connected Components Workbench (CCW).

The goal is to migrate this to a platform where AI can directly write, inspect, and modify the control logic — while maintaining the same level of process control quality.

## Blog Series

Follow the journey on the [project blog](https://brownbucks11.github.io/ai-homebrew/):

| # | Post | Status |
|---|------|--------|
| 1 | **The Baseline** — What we're starting with | 🔧 Draft |
| 2 | **Can AI Read My PLC Code?** — Feeding CCW to Claude | 📋 Planned |
| 3 | **Picking the New Platform** — Evaluating replacements | 📋 Planned |
| 4 | **The Rewrite** — AI-assisted code translation | 📋 Planned |
| 5 | **Commissioning & Cutover** — Making it work for real | 📋 Planned |
| 6 | **Lessons Learned** — What worked, what didn't | 📋 Planned |

## System Overview

### Existing Hardware
- **Controller:** Allen-Bradley Micro820 (2080-LC20-20QBB)
- **I/O:** 12 DI, 7 DO, 4 AI (NTC thermistors via voltage dividers), 1 AO
- **Power:** 2× Magnecraft W6240ASX-1 SSRs (240V/40A) for element PWM
- **Safety:** ASI E-Stop relay, HC1-63 contactors, C32 breakers
- **HMI:** HP Slate 21 Pro running Ignition Community Edition
- **Comms:** Modbus TCP (~100 tags)

### Software Architecture
- **16 programs** in FBD (Function Block Diagram)
- **39 custom function blocks** including PID auto-tune, step timers, mode control
- **ISA-style control modes:** Remote/Local, Auto/Manual
- **Dual PID loops** with PWM output to SSRs
- **4-step mash timer** with programmable temp/time and pause capability

### Replacement Platform Candidates
| Tier | Platform | Estimated Cost |
|------|----------|---------------|
| Budget | Raspberry Pi 4 + Node-RED/OpenPLC + Waveshare Modbus I/O | ~$175-195 |
| Mid | RevPi Connect 4 or UniPi Neuron + Advantech ADAM-6000 | ~$350-600 |
| Professional | Opto 22 groov RIO | ~$1,500-2,000 |

## Repository Structure

```
ai-homebrew/
├── blog/                    # Jekyll blog (GitHub Pages)
│   ├── _posts/              # Published blog posts
│   └── _drafts/             # Work in progress
├── docs/                    # Engineering documentation
│   ├── drawings/            # Draw.io panel layouts, system architecture
│   ├── ccw-analysis/        # Extracted PLC program analysis
│   └── photos/              # Build photos
├── baseline/                # Original system artifacts
│   ├── ccw-project/         # CCW project export (reference only)
│   └── ignition/            # Ignition project backup
├── new-platform/            # New controller code (when we get there)
└── assets/                  # Shared images and resources
```

## Tools Used

- **Allen-Bradley CCW** (Connected Components Workbench) — original PLC programming
- **Inductive Automation Ignition** — SCADA/HMI
- **Claude (Anthropic)** — AI-assisted code analysis, translation, and documentation
- **Draw.io** — Engineering drawings
- **Jekyll + GitHub Pages** — Blog hosting

## License

MIT — See [LICENSE](LICENSE) for details.

## About

Built by a controls engineer who wanted to see if AI could actually help with real automation work — not just toy demos, but production code with safety interlocks, PID loops, and proper mode control.
