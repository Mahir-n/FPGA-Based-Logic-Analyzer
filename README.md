# FPGA Logic Analyzer — Nexys A7-100T

A fully functional **8-channel real-time logic analyzer** implemented in Verilog HDL on the Digilent Nexys A7-100T FPGA development board, with a companion Python waveform viewer and a second Nexys A7-100T acting as a configurable signal generator.

> **EEE 304 Lab Project — Bangladesh University of Engineering and Technology (BUET)**

---

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Repository Structure](#repository-structure)
- [Hardware Requirements](#hardware-requirements)
- [Board 1 — Logic Analyzer](#board-1--logic-analyzer)
- [Board 2 — Signal Generator](#board-2--signal-generator)
- [PMOD Interconnect](#pmod-interconnect)
- [UART Command Protocol](#uart-command-protocol)
- [Python Waveform Viewer](#python-waveform-viewer)
- [Getting Started](#getting-started)

---

## Overview

This project transforms two commodity FPGA development boards into a
self-contained digital instrumentation system:

- **Board 1 (Logic Analyzer)** captures 8 parallel digital channels simultaneously at programmable sample rates from **1 Hz to 100 MHz**, stores 1024 samples per frame in on-chip Block RAM, and streams the data to a host PC over UART at 115,200 baud.
- **Board 2 (Signal Generator)** produces 8 independent square wave outputs whose frequencies are individually adjustable from **1 Hz to 10 MHz** in real time using on-board push-buttons, with the current frequency shown on the 7-segment display.
- A **Python GUI** on the host PC renders all 8 channels in real time, and lets the user change the sample rate and trigger settings interactively without reprogramming the FPGA.

---

## System Architecture

```
┌─────────────────────────────────┐          ┌──────────────────────────────────────────┐
│   Board 2 — Signal Generator    │          │         Board 1 — Logic Analyzer         │
│         (Nexys A7-100T)         │          │            (Nexys A7-100T)               │
│                                 │  PMOD JA │                                          │
│  8 × Square Wave Outputs ───────┼──────────┼──▶ 2-Stage Sync ──▶ Sample Register     │
│  (1 Hz – 10 MHz per channel)    │  8 wires │         │                  │             │
│                                 │          │   Sample Divider      Trigger Unit       │
│  SW[2:0]  → Channel Select      │          │         │                  │             │
│  BTNU     → Frequency Up        │          │         └──────────────────▼             │
│  BTND     → Frequency Down      │          │                    Capture Memory        │
│  7-Seg    → Channel & Freq      │          │                    (1024×8 BRAM)         │
│                                 │          │                         │                │
└─────────────────────────────────┘          │                      TX FSM             │
                                             │                         │                │
                                             │                    UART TX               │
                                             │                         │                │
                                             └─────────────────────────┼────────────────┘
                                                                       │ USB-UART (115200 baud)
                                                         ┌─────────────▼──────────────┐
                                                         │   Host PC — Python Viewer  │
                                                         │   Real-time 8-ch display   │
                                                         │   Sample rate control      │
                                                         │   Trigger configuration    │
                                                         └────────────────────────────┘
```

---

## Repository Structure

```
fpga-logic-analyzer/
│
├── fpga/
│   ├── board1_analyzer/          # Logic Analyzer firmware (Board 1)
│   │   ├── system_top.v          # Board-level top module
│   │   ├── logic_analyzer_top.v  # Core analyzer top
│   │   ├── uart_rx.v             # UART receiver (8N1)
│   │   ├── uart_tx.v             # UART transmitter (8N1)
│   │   ├── cmd_decoder.v         # UART command interpreter
│   │   ├── sample_divider.v      # Programmable rate divider
│   │   ├── trigger_unit.v        # Edge trigger engine
│   │   ├── capture_mem.v         # 1024×8 BRAM capture buffer
│   │   ├── tx_fsm.v              # BRAM-to-UART sender FSM
│   │   └── sync_2stage.v         # 2-stage metastability synchronizer
│   │
│   └── board2_siggen/            # Signal Generator firmware (Board 2)
│       ├── siggen_top.v          # Signal generator top module
│       ├── freq_divider.v        # Per-channel square wave generator
│       ├── btn_debounce.v        # Push-button debouncer
│       └── seg7_driver.v         # 7-segment display driver
│
├── constraints/
│   ├── Nexys_A7_100T.xdc         # XDC constraints for Board 1
│   └── siggen_top.xdc            # XDC constraints for Board 2
│
├── python/
│   └── logic_analyzer_viewer.py  # Host PC waveform viewer
│
├── report/                       # LaTeX report source
│
└── README.md
```

---

## Hardware Requirements

| Item | Quantity | Notes |
|------|----------|-------|
| Digilent Nexys A7-100T | 2 | One per board |
| USB Type-A to Micro-B cable | 2 | Power + JTAG/UART |
| PMOD jumper wires (8-pin) | 1 set | PMOD JA interconnect |
| Host PC | 1 | Windows / Linux / macOS |

**Software:**
- Xilinx Vivado Design Suite (WebPACK edition — free)
- Python 3.x with `pyserial`, `matplotlib`, `numpy`

```bash
pip install pyserial matplotlib numpy
```

---

## Board 1 — Logic Analyzer

### Features
- **8 channels**, 1-bit per channel, packed into 1 byte per sample
- **18 discrete sample rates**: 1 Hz → 100 MHz (6 decades)
- **1024-sample capture depth** stored in a single BRAM tile (RAMB18)
- **Configurable edge trigger**: rising or falling, on any of the 8 channels
- **Free-run mode**: continuous capture without waiting for a trigger
- **UART streaming**: 115,200 baud, 8N1, with `0xAA 0x55` sync header
- **Two-stage synchronizers** on all asynchronous inputs (probes + UART RX)

### Module Hierarchy

```
system_top
├── sync_2stage        [u_sync_free]   SW_FREE synchronizer (1-bit)
├── logic_analyzer_top [u_analyzer]
│   ├── sync_2stage    [u_sig_sync]    Probe bus synchronizer (8-bit)
│   ├── sync_2stage    [u_rx_sync]     UART RX synchronizer (1-bit)
│   ├── uart_rx        [u_uart_rx]     UART receiver
│   ├── uart_tx        [u_uart_tx]     UART transmitter
│   ├── cmd_decoder    [u_cmd_dec]     Command byte interpreter
│   ├── sample_divider [u_samp_div]    Rate divider
│   ├── trigger_unit   [u_trigger]     Edge trigger engine
│   ├── capture_mem    [u_cap_mem]     1024×8 BRAM buffer
│   └── tx_fsm         [u_tx_fsm]      Memory-to-UART FSM
```

### Switch / Button Mapping (Board 1)

| Control | Function |
|---------|----------|
| `SW[8]` (SW_FREE) | Free-run mode — ON = continuous capture |
| `BTNC` | Active-high system reset |

> Probe inputs (CH0–CH7) come from Board 2 via PMOD JA in the two-board setup.

---

## Board 2 — Signal Generator

### Features
- **8 independent square wave outputs** on PMOD JA[7:0]
- **Per-channel frequency control**: 8 discrete steps per channel
- **Frequency range**: 1 Hz, 10 Hz, 100 Hz, 1 KHz, 10 KHz, 100 KHz, 1 MHz, 10 MHz
- **7-segment display**: shows selected channel and its current frequency
- **LED indicators**: Hz / KHz / MHz unit display

### Default Frequencies (after reset)

| Channel | PMOD Pin | Default Frequency |
|---------|----------|-------------------|
| CH0 | JA[0] | 10 MHz |
| CH1 | JA[1] | 1 MHz |
| CH2 | JA[2] | 100 KHz |
| CH3 | JA[3] | 10 KHz |
| CH4 | JA[4] | 1 KHz |
| CH5 | JA[5] | 100 Hz |
| CH6 | JA[6] | 10 Hz |
| CH7 | JA[7] | 1 Hz |

### Switch / Button Mapping (Board 2)

| Control | Function |
|---------|----------|
| `SW[2:0]` | Channel select (CH0–CH7) |
| `BTNU` | Increase selected channel frequency one step |
| `BTND` | Decrease selected channel frequency one step |
| `BTNC` | Reset all channels to default frequencies |

---

## PMOD Interconnect

Connect the **PMOD JA** header of Board 2 (signal generator) directly to the **PMOD JA** header of Board 1 (logic analyzer) using 8 jumper wires:

```
Board 2  PMOD JA        Board 1  PMOD JA
─────────────────        ─────────────────
JA[0]  (CH0) ──────────▶  JA[0]  (CH0 probe)
JA[1]  (CH1) ──────────▶  JA[1]  (CH1 probe)
JA[2]  (CH2) ──────────▶  JA[2]  (CH2 probe)
JA[3]  (CH3) ──────────▶  JA[3]  (CH3 probe)
JA[4]  (CH4) ──────────▶  JA[4]  (CH4 probe)
JA[5]  (CH5) ──────────▶  JA[5]  (CH5 probe)
JA[6]  (CH6) ──────────▶  JA[6]  (CH6 probe)
JA[7]  (CH7) ──────────▶  JA[7]  (CH7 probe)
GND          ──────────▶  GND
```

> Both boards must share a common ground. All signals are 3.3 V LVCMOS, compatible with both boards.

---

## UART Command Protocol

All configuration is sent from the Python GUI to Board 1 as **single-byte commands** over UART (115,200 baud, 8N1).

### Sample Rate Commands (0x01 – 0x12)

| Byte | Sample Rate | Byte | Sample Rate |
|------|-------------|------|-------------|
| `0x01` | 100 MHz | `0x0A` | 10 KHz |
| `0x02` | 50 MHz | `0x0B` | 5 KHz |
| `0x03` | 25 MHz | `0x0C` | 1 KHz |
| `0x04` | 10 MHz | `0x0D` | 500 Hz |
| `0x05` | 5 MHz | `0x0E` | 100 Hz |
| `0x06` | 1 MHz | `0x0F` | 50 Hz |
| `0x07` | 500 KHz | `0x10` | 10 Hz |
| `0x08` | 100 KHz | `0x11` | 5 Hz |
| `0x09` | 50 KHz | `0x12` | 1 Hz |

### Trigger Commands (0x20 – 0x2F)

```
Byte layout:  0 0 1 0 | E | C2 C1 C0
                        │    └──────── Channel (0–7)
                        └──────────── Edge: 0=Rising, 1=Falling
```

| Example | Meaning |
|---------|---------|
| `0x20` | Rising edge, CH0 |
| `0x28` | Falling edge, CH0 |
| `0x25` | Rising edge, CH5 |
| `0x2F` | Falling edge, CH7 |

### Data Stream Format (Board 1 → PC)

Each capture frame transmitted by Board 1:

```
[ 0xAA ] [ 0x55 ] [ mem[0] ] [ mem[1] ] ... [ mem[1023] ]
  ──────────────    ──────────────────────────────────────
   2-byte header         1024 data bytes
        └─────────────────────────────── 1026 bytes total
```

Each data byte contains all 8 channel states at one sample point (bit 0 = CH0, bit 7 = CH7).

---

## Python Waveform Viewer

```bash
python logic_analyzer_viewer.py [COM_PORT]
```

If no port is specified, the script auto-detects a connected Nexys A7 board. On Linux the port is typically `/dev/ttyUSB1`; on Windows it is `COMn`.

### GUI Controls

| Control | Function |
|---------|----------|
| Sample Rate dropdown | Select from 18 rates (1 Hz – 100 MHz) |
| Trig Channel dropdown | Select trigger channel (CH0–CH7) |
| Trig Edge dropdown | Rising ↑ or Falling ↓ |
| `\|\| Pause` | Freeze display (acquisition continues on FPGA) |
| `X Clear` | Clear all waveforms |
| `S Save` | Save current view as PNG |
| `R Reconnect` | Reconnect after USB disconnect without restarting |

### Dependencies

```bash
pip install pyserial matplotlib numpy
```

---

## Getting Started

### 1. Program Board 1 (Logic Analyzer)

1. Open Vivado and create a new project targeting `XC7A100T-1CSG324C`.
2. Add all source files from `fpga/board1_analyzer/` and the constraint file `constraints/Nexys_A7_100T.xdc`.
3. Set `system_top` as the top module.
4. Run Synthesis → Implementation → Generate Bitstream.
5. Open Hardware Manager, connect Board 1, and program the device.

### 2. Program Board 2 (Signal Generator)

1. Create a new Vivado project targeting the same device.
2. Add all source files from `fpga/board2_siggen/` and `constraints/siggen_top.xdc`.
3. Set `siggen_top` as the top module.
4. Run Synthesis → Implementation → Generate Bitstream.
5. Program Board 2.

### 3. Connect the Boards

Wire PMOD JA of Board 2 to PMOD JA of Board 1 as described in the [PMOD Interconnect](#pmod-interconnect) section. Ensure GND is shared.

### 4. Run the Python Viewer

```bash
cd python
pip install pyserial matplotlib numpy
python logic_analyzer_viewer.py COM3   # replace COM3 with your port
```

### 5. Operate

- Use the **Sample Rate** dropdown to choose a rate where channels show visible waveforms (start with **10 KHz** or **100 KHz**).
- Use **SW[8]** on Board 1 for free-run mode (no trigger wait).
- Adjust individual channel frequencies on Board 2 with **SW[2:0]** + **BTNU/BTND** and watch the waveforms update live.

---
