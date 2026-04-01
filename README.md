# FPGA Logic Analyzer — Nexys A7-100T

A fully functional 8-channel logic analyzer implemented on the Digilent Nexys A7-100T FPGA board. Captures up to 1024 samples at programmable rates from 1 Hz to 100 MHz and streams them to a real-time Python waveform viewer over UART.

---

## Features

- **8 digital probe channels** (CH0–CH7) via on-board switches or external signals
- **18 programmable sample rates** from 1 Hz to 100 MHz
- **1024-sample capture depth** per frame stored in Block RAM
- **Configurable edge trigger** — rising or falling edge on any channel
- **Free-run mode** — continuous capture with no trigger required
- **Internal test signal generator** — 8 square waves at different frequencies for self-test
- **Real-time Python GUI** — dark-theme waveform viewer with dropdowns, pause, save, and reconnect
- **Robust UART framing** — 0xAA 0x55 sync header for automatic resync after any data loss
- **Single-byte command protocol** — sample rate and trigger config from PC to FPGA

---

## Repository Structure

```
fpga-logic-analyzer/
│
├── README.md
├── LICENSE
│
├── rtl/                        ← Verilog RTL source files
│   ├── system_top.v            ← Board top-level wrapper (Nexys A7-100T)
│   ├── logic_analyzer_top.v    ← Core top-level, submodule wiring
│   ├── capture_mem.v           ← 1024×8 Block RAM capture memory
│   ├── cmd_decoder.v           ← UART command → rate/trigger registers
│   ├── sample_divider.v        ← Programmable sample rate divider
│   ├── signal_generator.v      ← Internal test pattern generator
│   ├── sync_2stage.v           ← 2-stage metastability synchronizer
│   ├── trigger_unit.v          ← Configurable edge trigger engine
│   ├── tx_fsm.v                ← Capture memory → UART TX sender FSM
│   ├── uart_rx.v               ← UART receiver (8N1)
│   └── uart_tx.v               ← UART transmitter (8N1)
│
├── constraints/
│   └── Nexys_A7_100T.xdc       ← Pin assignments and clock constraints
│
├── software/
│   ├── logic_analyzer_viewer.py ← Real-time 8-channel waveform viewer
│   └── requirements.txt
│
├── sim/                        ← Testbenches (future)
└── docs/                       ← Architecture diagrams, notes (future)
```

---

## Hardware

| Item | Detail |
|---|---|
| Board | Digilent Nexys A7-100T |
| FPGA | Xilinx Artix-7 XC7A100T |
| Clock | 100 MHz on-board oscillator |
| UART | USB-UART via FT2232H bridge |
| Probe inputs | SW[7:0] (on-board switches) or external PMOD signals |
| Baud rate | 115200 8N1 |

### Switch Mapping

| Switch | Function |
|---|---|
| SW[7:0] | 8 external probe channels |
| SW[8] | Free-run mode (1 = continuous, 0 = wait for trigger) |
| SW[9] | Signal source (0 = SW[7:0] probes, 1 = internal test generator) |
| BTNC | Active-high reset |

---

## Architecture

```
system_top
└── logic_analyzer_top
    ├── sync_2stage       (×2)   signal + UART RX synchronizers
    ├── uart_rx                  UART receiver
    ├── uart_tx                  UART transmitter
    ├── cmd_decoder              command byte → rate_div / trig config
    ├── sample_divider           programmable rate divider → sample_en
    ├── trigger_unit             edge trigger FSM
    ├── capture_mem              1024×8 BRAM, write on sample_en+triggered
    └── tx_fsm                   reads BRAM, streams bytes over UART TX
```

Data flow:

```
Probe signals → sync_2stage → trigger_unit ──────────────────┐
                                                              ↓
                            sample_divider → sample_en → capture_mem → tx_fsm → UART TX → PC
                                                              ↑
                        UART RX → uart_rx → cmd_decoder ──────┘
                                            (rate_div, trig_ch, trig_edge)
```

---

## UART Command Protocol (PC → FPGA)

All commands are a **single byte**.

### Sample Rate Commands (0x01 – 0x12)

| Byte | Sample Rate | Divider |
|---|---|---|
| 0x01 | 100 MHz | 1 |
| 0x02 | 50 MHz | 2 |
| 0x03 | 25 MHz | 4 |
| 0x04 | 10 MHz | 10 |
| 0x05 | 5 MHz | 20 |
| 0x06 | 1 MHz | 100 |
| 0x07 | 500 KHz | 200 |
| 0x08 | 100 KHz | 1,000 |
| 0x09 | 50 KHz | 2,000 |
| 0x0A | 10 KHz | 10,000 |
| 0x0B | 5 KHz | 20,000 |
| 0x0C | 1 KHz | 100,000 |
| 0x0D | 500 Hz | 200,000 |
| 0x0E | 100 Hz | 1,000,000 |
| 0x0F | 50 Hz | 2,000,000 |
| 0x10 | 10 Hz | 10,000,000 |
| 0x11 | 5 Hz | 20,000,000 |
| 0x12 | 1 Hz | 100,000,000 |

### Trigger Commands (0x20 – 0x2F)

Upper nibble `0x2` marks a trigger config command.

```
Byte layout:  0 0 1 0 | E C2 C1 C0
                        │  └──────── channel select (0–7)
                        └─────────── edge: 0=rising, 1=falling
```

| Example | Meaning |
|---|---|
| 0x20 | Rising edge on CH0 |
| 0x21 | Rising edge on CH1 |
| 0x27 | Rising edge on CH7 |
| 0x28 | Falling edge on CH0 |
| 0x2F | Falling edge on CH7 |

---

## UART Stream Format (FPGA → PC)

Each capture frame is **1026 bytes**:

```
[0xAA] [0x55] [mem[0]] [mem[1]] ... [mem[1023]]
  ↑      ↑     └──────────────────────────────┘
  sync header        1024 data bytes
```

The Python viewer scans for the `0xAA 0x55` header to find frame boundaries. This makes the receiver immune to mid-stream connects and automatic resync after any data loss.

---

## Internal Test Signal Generator

When SW[9] = 1 the FPGA probes its own internal counter instead of the external switches. Useful for verifying the design without any external hardware.

| Channel | Approximate Frequency |
|---|---|
| CH0 | 48.8 kHz |
| CH1 | 12.2 kHz |
| CH2 | 3.05 kHz |
| CH3 | 763 Hz |
| CH4 | 191 Hz |
| CH5 | 47.7 Hz |
| CH6 | 5.96 Hz |
| CH7 | 0.75 Hz |

---

## Python Viewer

### Installation

```bash
pip install -r software/requirements.txt
```

### Run

```bash
# Auto-detect Nexys A7 USB port
python software/logic_analyzer_viewer.py

# Specify port manually
python software/logic_analyzer_viewer.py COM11        # Windows
python software/logic_analyzer_viewer.py /dev/ttyUSB0 # Linux
python software/logic_analyzer_viewer.py /dev/tty.usbserial-XXXX # macOS
```

### GUI Controls

| Control | Function |
|---|---|
| Sample Rate dropdown | Send rate command byte to FPGA (18 options) |
| Trig Channel dropdown | Select trigger channel CH0–CH7 |
| Trig Edge dropdown | Rising ↑ or Falling ↓ |
| \|\| Pause | Freeze display (FPGA keeps capturing) |
| X Clear | Clear waveform display |
| S Save | Save screenshot as PNG |
| R Reconnect | Re-establish USB connection without restarting |

---

## Vivado Project Setup

1. Create a new Vivado RTL project targeting `xc7a100tcsg324-1`
2. Add all `.v` files from `rtl/` as design sources
3. Add `constraints/Nexys_A7_100T.xdc` as a constraint source
4. Set `system_top` as the top module
5. Run Synthesis → Implementation → Generate Bitstream
6. Program the board via Hardware Manager

---

## Design Notes

**Block RAM inference** — `capture_mem` uses a registered (synchronous) read port with `(* ram_style = "block" *)` so Vivado infers true BRAM. An asynchronous read would fall back to distributed LUT RAM.

**Metastability protection** — all asynchronous inputs (probe signals, UART RX, control switches) pass through `sync_2stage` with `(* ASYNC_REG = "TRUE" *)` on both flip-flops. This tells Vivado to co-locate the FFs and disable optimisations that would break the synchronizer chain.

**Spurious-trigger protection** — `trigger_unit` seeds `prev_sample` from the actual signal level on the first `sample_en` after reset or after a trigger channel change, preventing false edge detection from stale history.

**UART guard period** — `uart_tx` holds `tx_busy` for one extra baud period after the stop bit. This prevents `tx_fsm` from driving the next start bit before the receiver has sampled the stop-bit centre.

---

## License

MIT License — see [LICENSE](LICENSE) for details.
