<div align="center">

# GEIST

**A reference-grade balanced USB DAC, real-time DSP engine, and tactile media controller — designed and built from scratch.**

`ESP32-S3` · `PCM5102A` · `OPA1656` · `Rust` · `Tauri`

---

*24-bit · 96kHz · Balanced 4.4mm Pentaconn · 10-Band Parametric EQ · FFT Visualizer*

</div>

---

## What is Geist?

Geist is an open-source USB audio device that I designed from the ground up — the circuit, the PCB, the firmware, and the companion app. It plugs into a computer over USB-C and serves three functions simultaneously:

1. **Reference-Grade USB DAC** — Enumerates as a class-compliant USB Audio device (UAC1). Receives lossless 24-bit/96kHz audio, converts it through a hardware DAC, and drives headphones through a true balanced amplifier stage.

2. **Real-Time DSP Engine** — A 10-band parametric EQ processes audio in real-time on the ESP32-S3's vector DSP unit. An FFT spectrum visualizer renders live on the onboard display.

3. **Tactile Media Controller** — Physical rotary encoder, mechanical buttons, and a toggle switch send HID commands back to the PC for volume and playback control. No drivers needed.

The companion desktop app (built in Rust + Tauri) pushes track metadata and album art to the device, provides a graphical EQ editor, and displays real-time audio analysis.

> This project was built as a 2nd year B.Tech ECE student at Manipal Institute of Technology. Every part of it — analog circuit design, mixed-signal PCB layout, embedded Rust firmware, and desktop application — was learned and implemented from scratch.

---

## Hardware Architecture

### The Signal Chain

```
  USB-C ──► AP2112K (3.3V LDO) ──► ESP32-S3 ──► PCM5102A DAC ──► 2× OPA1656 ──► Headphones
       └──► LM27762 (±4.5V Charge Pump) ─────────────────────────────┘
```

Every component was selected to serve a specific role in the audio path:

| Block | Component | Role |
|---|---|---|
| **Power (Digital)** | AP2112K-3.3 | LDO regulator — steps 5V USB to clean 3.3V for all digital logic |
| **Power (Analog)** | LM27762 | Bipolar charge pump — generates ±4.5V dual rails for the op-amps |
| **Digital Brain** | ESP32-S3-WROOM-1 | Dual-core 240MHz processor — USB audio streaming, DSP, UI control |
| **DAC** | PCM5102A | 32-bit delta-sigma DAC — converts I2S digital audio to analog |
| **Amplifiers** | 2× OPA1656 | Ultra-low-noise dual op-amps — buffers and creates balanced output |
| **Display** | ST7789 (240×240) | SPI LCD — track info or FFT visualizer |
| **Input** | Alps EC11 + 3 push buttons + SPDT toggle | Volume/nav encoder, media keys, mode switch |
| **Output** | 4.4mm Pentaconn + 3.5mm TRS | True balanced and single-ended headphone outputs |

### The Life of an Audio Sample

This table traces a single sample from the moment it leaves the PC to the moment it moves air:

```
Step   Location                  Signal State                        Domain
─────────────────────────────────────────────────────────────────────────────
 1     PC audio engine           24-bit integer: 0x3FFFFF            Software
 2     USB cable (D+/D-)         NRZI-encoded differential packet    Digital
 3     ESP32 USB DMA buffer      Bytes in RAM                        Digital
 4     ESP32 I2S peripheral      Serial bitstream at 3.072 MHz       Digital (3.3V)
 5     PCM5102A delta-sigma      1-bit stream at 6.144 MHz           Digital
 6     PCM5102A OUTL pin         +2.05V (audio + 1.55V DC bias)      Analog
 7     OPA1656 buffer output     +2.05V → OUT_L+ (hot phase)         Analog
 8     OPA1656 inverter output   −0.50V → OUT_L− (cold phase)        Analog
 9     After 10Ω resistors       +2.04V / −0.49V                     Analog
 10    Headphone driver          ΔV = 2.53V differential → sound     Acoustic
```

### Why Balanced?

A balanced signal carries audio on two wires moving in opposite directions. Any noise that couples into the cable appears equally on both wires and is cancelled by subtraction at the receiver:

```
Single-Ended:                      Balanced:
Signal ── ╱╲╱~~╲╱╲ ── +V+noise     HOT  ── ╱╲╱~~╲╱╲ ── +V+noise
Ground ── ──────── ── 0V            COLD ── ╲╱╲~~╱╲╱ ── −V+noise

Amp sees: signal + noise            Amp subtracts: (+V+noise) − (−V+noise)
→ noise passes through ✗            = 2V signal, noise cancelled ✓
```

The OPA1656 op-amps create this balanced pair — one half buffers the DAC output (positive phase), the other half inverts it (negative phase, gain = −1).

---

## PCB Design Philosophy

Geist is laid out on a **2-layer PCB** with strict attention to analog/digital noise isolation:

```
◄──── DIGITAL ZONE ──────────►│◄──────── ANALOG ZONE ──────────►

┌──────────┐  ┌──────────────┐ │  ┌───────────┐  ┌───────────┐
│ ESP32-S3 │  │ Display,     │ │  │ OPA1656×2 │  │ Audio     │
│          │  │ Encoder,     │ │  │           │  │ Jacks     │
│          │  │ Buttons      │ │  │           │  │           │
└──────────┘  └──────────────┘ │  └───────────┘  └───────────┘
                               │
                         ┌─────┴─────┐
                         │ PCM5102A  │ ← sits on the border
                         └───────────┘

Bottom layer: UNBROKEN ground copper pour — no splits.
```

### Key Layout Rules

- **Zonal Segregation** — All digital components (ESP32, display, buttons) on one side. All analog components (op-amps, output jacks) on the other. The PCM5102A sits on the boundary with its digital input pins facing left and analog outputs facing right.

- **Solid Ground Plane** — No split grounds. A single unbroken copper pour on the bottom layer allows return currents to flow directly beneath their source traces (path of least inductance), minimizing loop area and radiated noise. This is the modern best practice endorsed by TI, Analog Devices, and EMC engineers — not the outdated "split and bridge" approach.

- **Tight Decoupling** — Every bypass capacitor is placed physically touching the power pins of its parent IC. A decoupling cap 15mm away is electrically useless — the trace inductance defeats its purpose.

- **Antenna Isolation** — The ESP32's Wi-Fi/Bluetooth antenna overhangs the board edge with a keepout zone beneath it. No copper, no traces, no components under the antenna.

---

## Firmware Architecture

The firmware is written in **Rust** using the Embassy async runtime — bare-metal, no RTOS, no garbage collector.

```
geist-firmware/
├── src/
│   ├── main.rs                ← Embassy entry, spawns all tasks
│   ├── usb/
│   │   ├── descriptors.rs     ← USB composite device (UAC1 + HID + CDC)
│   │   ├── audio.rs           ← Isochronous audio reception, buffer management
│   │   ├── hid.rs             ← Media key HID reports
│   │   └── cdc.rs             ← Serial link to companion app
│   ├── audio/
│   │   ├── i2s.rs             ← I2S DMA driver for PCM5102A
│   │   ├── buffer.rs          ← Lock-free ring buffer (USB → I2S)
│   │   └── dsp.rs             ← Biquad EQ engine + FFT analysis
│   ├── ui/
│   │   ├── display.rs         ← ST7789 SPI driver
│   │   ├── encoder.rs         ← EC11 rotary encoder with debounce
│   │   ├── buttons.rs         ← Debounced media buttons
│   │   └── renderer.rs        ← Track info view + spectrum visualizer
│   └── app/
│       └── state.rs           ← Global state machine
```

### USB Composite Device

Geist enumerates as three USB interfaces simultaneously — no custom drivers required:

| Interface | USB Class | Function |
|---|---|---|
| Audio Control + Audio Streaming | UAC1 | Appears as a sound card — receives PCM audio from the OS |
| HID (Consumer Control) | HID | Sends media keys (play/pause/next/prev) and volume from the encoder |
| CDC ACM | CDC | Virtual serial port for companion app communication |

### Real-Time DSP

The 10-band parametric EQ runs as a chain of biquad filters — each band is 5 multiplies and 4 adds per sample:

```rust
// One EQ band — runs for every sample, every channel
fn process(&mut self, x: f32) -> f32 {
    let y = self.b0 * x + self.b1 * self.x1 + self.b2 * self.x2
          - self.a1 * self.y1 - self.a2 * self.y2;
    self.x2 = self.x1; self.x1 = x;
    self.y2 = self.y1; self.y1 = y;
    y
}
```

At 48kHz stereo with 10 bands, this amounts to ~5 million operations per second — less than 2% of the ESP32-S3's capacity. The biquad coefficients are computed by the companion app from human-readable parameters (frequency, gain, Q) and sent over the CDC serial link.

### Task Model

```
Core 0 (Real-time audio — never blocked):
├── usb_audio_task()       ← Isochronous USB reception
├── i2s_feed_task()        ← Drains ring buffer → I2S DMA → PCM5102A
└── usb_hid_cdc_task()     ← HID reports + serial I/O

Core 1 (UI + DSP — lower priority):
├── dsp_task()             ← Biquad EQ processing + FFT analysis
├── display_task()         ← ST7789 rendering @ 30fps
└── input_task()           ← Encoder + button polling + debounce
```

---

## Companion App

Built with **Tauri** (Rust backend + web frontend) — a ~5MB native app, not a 150MB Electron wrapper.

### Features

- **Now Playing** — Reads track info from the OS media session (Windows SMTC / macOS MediaRemote / Linux MPRIS2) and pushes title, artist, and album art to Geist's display over CDC serial
- **10-Band Parametric EQ** — Interactive frequency response curve editor. Drag bands to set frequency, gain, and Q. Coefficients are computed locally and sent to the ESP32 in real-time
- **Real-Time Visualizer** — Captures system audio via loopback and renders spectrum analyzer, waveform, and spectrogram views in the browser frontend
- **Device Dashboard** — Connection status, sample rate, firmware version, output mode
- **System Tray** — Runs silently in the background, auto-connects when Geist is plugged in
- **Firmware Update** — Flash new firmware to the ESP32 over USB without opening the case
- **Listening Stats** — Tracks hours, volume levels, and session history in a local SQLite database

### Tech Stack

| Layer | Technology |
|---|---|
| Backend | Rust (serialport, windows-rs, cpal, rustfft, rusqlite) |
| Frontend | Svelte + HTML5 Canvas for visualizations |
| Framework | Tauri v2 |
| Protocol | JSON messages + binary RGB565 art over CDC serial |

---

## Bill of Materials

| Component | Part Number | Quantity | Description |
|---|---|---|---|
| MCU | ESP32-S3-WROOM-1 | 1 | Dual-core 240MHz, USB OTG, vector DSP |
| DAC | PCM5102A (PWR) | 1 | 32-bit delta-sigma, I2S input, 112dB SNR |
| Op-Amp | OPA1656IDR | 2 | Ultra-low-noise dual op-amp, 2.8 nV/√Hz |
| LDO | AP2112K-3.3TRG1 | 1 | 3.3V, 600mA, low-dropout regulator |
| Charge Pump | LM27762DSSR | 1 | Bipolar ±output from single supply |
| USB Connector | USB-C 16-pin | 1 | PCB-mount receptacle |
| Display | ST7789 240×240 | 1 | SPI TFT LCD module |
| Encoder | Alps EC11 | 1 | Rotary encoder with push switch |
| Toggle | SPDT | 1 | Mode select switch |
| Buttons | Tactile push | 3 | Prev / Play-Pause / Next |
| Balanced Jack | 4.4mm Pentaconn | 1 | Panel-mount, 5-pole |
| SE Jack | 3.5mm TRS | 1 | Panel-mount, 3-pole |
| Resistors | 0805 | 15 | Various values (see schematic) |
| Capacitors | 0805 | 15 | 0.1µF, 2.2µF, 4.7µF, 10µF |

---

## Project Structure

```
Geist/
├── hardware/                    ← KiCad project
│   ├── Geist.kicad_pro
│   ├── Geist.kicad_sch          ← Schematic
│   ├── Geist.kicad_pcb          ← PCB layout
│   └── fabrication/             ← Gerber files for manufacturing
│
├── firmware/                    ← ESP32-S3 Rust firmware
│   ├── Cargo.toml
│   └── src/
│
├── companion/                   ← Tauri desktop app
│   ├── src-tauri/               ← Rust backend
│   ├── src/                     ← Svelte frontend
│   └── package.json
│
├── enclosure/                   ← 3D-printable case (STEP/STL)
│
├── docs/
│   ├── hardware-signal-trace.md ← Complete electrical signal walkthrough
│   └── usb-audio-architecture.md← USB Audio Class deep dive
│
└── README.md
```

---

## Design Specifications

| Parameter | Value |
|---|---|
| Audio Format | PCM, 24-bit, 48kHz / 96kHz |
| USB Class | UAC1 (class-compliant, driverless) |
| DAC SNR | 112 dB (A-weighted) |
| DAC THD+N | −93 dB |
| Amp Noise | 2.8 nV/√Hz |
| Amp THD+N | −132 dB (0.000025%) |
| Output (Balanced) | 4.4mm Pentaconn, ~4 Vrms differential |
| Output (SE) | 3.5mm TRS |
| DSP | 10-band parametric EQ (biquad IIR) |
| Visualization | 1024-point FFT, real-time spectrum |
| Power | USB bus-powered (5V, <500mA) |
| PCB | 2-layer, solid ground plane |
| Firmware | Rust + Embassy (bare-metal async) |
| Companion App | Rust + Tauri + Svelte |

---

## Building

> 🚧 Build instructions will be added as each subsystem is completed.

### Hardware
1. Open `hardware/Geist.kicad_pro` in KiCad 9.0+
2. Generate Gerbers and order from JLCPCB or PCBWay (2-layer, 1.6mm, 1oz copper)
3. Source components from LCSC/Mouser/Digikey (see BOM above)
4. Hand-solder with hot air station (TSSOP/QFN packages) and soldering iron (0805 passives, connectors)

### Firmware
```bash
# Prerequisites: Rust toolchain + ESP32 target
rustup target add xtensa-esp32s3-none-elf
cargo install espflash

# Build and flash
cd firmware
cargo build --release
espflash flash target/xtensa-esp32s3-none-elf/release/geist-firmware
```

### Companion App
```bash
# Prerequisites: Rust, Node.js 18+, pnpm
cd companion
pnpm install
pnpm tauri dev     # development
pnpm tauri build   # production binary
```

---

## License

This project is open-source under the [MIT License](LICENSE).

---

<div align="center">

Designed and built by **Arihant** · MIT Manipal · 2026

*A project born from curiosity about what sits between the music and the ear.*

</div>
