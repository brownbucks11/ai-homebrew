---
layout: post
title: "Part 1: The Baseline — What We're Starting With"
date: 2026-02-28
categories: [baseline, hardware]
tags: [micro820, ccw, plc, brewing, automation]
excerpt: "Before we let AI anywhere near the code, let's document what we're working with — a fully operational 3-zone electric brewery controller running on an Allen-Bradley Micro820."
---

# Part 1: The Baseline

Before we let AI anywhere near the control logic, we need to thoroughly document what we're starting with. This isn't a greenfield project — it's a migration of a **fully operational** brewing automation system that's been running real brew days since 2019.

## The System

The controller runs a 3-vessel electric brewery: Hot Liquor Tank (HLT), Mash Tun, and Boil Kettle. Each vessel can be assigned any of four temperature sensors and routed to either of two heating elements via PID control with PWM output.

### Hardware

The brain is an **Allen-Bradley Micro820** (2080-LC20-20QBB) — a compact PLC with:
- 12 discrete inputs (panel buttons, selectors, E-STOP)
- 7 discrete outputs (2× SSR PWM for elements, 2× pump contactors)
- 4 analog inputs (NTC thermistors through voltage divider networks)
- 1 analog output
- Ethernet port running Modbus TCP

The power section handles 240V heating elements through **Magnecraft W6240ASX-1** solid-state relays (40A) mounted on heatsinks, protected by C32 breakers and HC1-63 contactors. An ASI safety relay handles the hardwired E-STOP circuit.

The operator interface is a custom face panel with color-coded pushbuttons, amber selector switches for mode control, a key switch for local/remote, and a digital temperature display. An HP Slate 21 Pro touchscreen mounted on top runs the SCADA/HMI.

### Software

The PLC is programmed in **Function Block Diagram (FBD)** using Connected Components Workbench (CCW). The application contains:

- **16 programs** handling everything from raw temperature scaling to PID control to mash step sequencing
- **39 custom function blocks** — a reusable library including PID auto-tune, pauseable timers, multi-step mash controllers, and ISA-style mode control
- **~100 Modbus TCP tags** mapped to Ignition Community Edition for the HMI

The control architecture follows proper ISA patterns: Remote/Local and Auto/Manual mode selection, HMI command/status separation, setpoint range limiting, and over-temperature protection with debounce timers.

## Why Migrate?

The Micro820 works. It brews beer reliably. So why change?

1. **CCW is a dead end for AI assistance.** The FBD programs are stored in proprietary binary formats. There's no text-based representation that AI can read, modify, or generate.

2. **The platform is aging.** Rockwell has moved on from the Micro800 line. Getting replacement hardware or software updates is increasingly painful.

3. **I want to test a hypothesis.** As a controls engineer, I want to know: can AI actually help with real automation work? Not just writing a Python script, but understanding process control architecture and translating it faithfully?

## What's Next

In Part 2, we'll feed the CCW project to Claude and see what it can understand. Can it parse the program structure? Identify the PID loops? Understand the mode control architecture? That analysis will determine whether AI-assisted migration is viable or just hype.

---

*All engineering drawings, CCW analysis, and source code are available in the [project repository](https://github.com/brownbucks11/ai-homebrew).*
