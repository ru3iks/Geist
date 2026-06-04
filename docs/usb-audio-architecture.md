# USB Audio Deep Dive — Geist Edition

> [!NOTE]
> This document explains USB Audio Class 1.0 from the ground up, specifically in the context of Geist's hardware. Every concept is tied to your actual signal chain. Keep this as a reference when you start writing firmware.

---

## 1. The Big Picture: What Happens When You Plug Geist In

When you plug Geist's USB-C cable into your PC, the following happens in roughly 200ms:

1. The PC's USB host controller detects a device (pull-up resistor on D+)
2. The PC asks: **"What are you?"** — it reads your **Device Descriptor**
3. The PC asks: **"What can you do?"** — it reads your **Configuration Descriptor** (which contains all your interfaces)
4. The PC sees three things inside:
   - "I'm a sound card" → **Audio Class interface** → loads the built-in USB Audio driver
   - "I have media keys" → **HID interface** → loads the built-in HID driver  
   - "I have a serial port" → **CDC interface** → loads the built-in serial driver
5. The PC activates the configuration, and Geist is live — no custom drivers needed

This is what "class-compliant" means. The OS already has drivers for Audio, HID, and CDC. You just need to describe yourself correctly through **descriptors**.

---

## 2. The USB Descriptor Hierarchy

Think of descriptors as a nested data structure — a tree. The host reads the whole tree to understand your device. Here's Geist's complete tree:

```
Device Descriptor (1 device = Geist)
│
└── Configuration Descriptor (1 configuration)
    │
    ├── IAD (Interface Association Descriptor) ─── groups Audio interfaces
    │   │
    │   ├── Interface 0: Audio Control (AC)
    │   │   └── AC Header
    │   │   └── Input Terminal  (USB streaming in)
    │   │   └── Feature Unit    (volume control)
    │   │   └── Output Terminal (speaker/DAC out)
    │   │
    │   └── Interface 1: Audio Streaming (AS)
    │       ├── Alt Setting 0: Zero-bandwidth (idle — no endpoint)
    │       └── Alt Setting 1: Active streaming
    │           └── AS Header (links to Input Terminal)
    │           └── Format Descriptor (PCM, 24-bit, 48/96kHz, stereo)
    │           └── Isochronous Endpoint (the actual data pipe)
    │           └── Audio Endpoint descriptor (sample rate control)
    │
    ├── Interface 2: HID (media keys + volume knob)
    │   └── HID Descriptor
    │   └── Interrupt Endpoint IN (sends key events to PC)
    │
    └── Interface 3: CDC ACM (serial port for companion app)
        ├── CDC Control Interface
        │   └── Interrupt Endpoint IN (serial state notifications)
        └── CDC Data Interface
            ├── Bulk Endpoint IN  (data FROM Geist to PC)
            └── Bulk Endpoint OUT (data FROM PC to Geist)
```

Each of these is a packed binary struct that you define in your firmware. When the host reads the Configuration Descriptor, it gets **all** of this in one big blob. Let's break down each piece.

---

## 3. USB Audio Class 1.0 — The Sound Card

### 3.1 Why UAC1 and not UAC2?

| | UAC1 | UAC2 |
|---|---|---|
| Designed for | Full-Speed (12 Mbps) ✓ | High-Speed (480 Mbps) |
| Max practical quality | 24-bit / 96 kHz ✓ | 32-bit / 384 kHz |
| Windows driver | Built-in since XP | Built-in since Win10 1703 |
| macOS driver | Built-in since 10.2 | Built-in since 10.6 |
| Linux driver | Built-in since 2.4 | Built-in since 2.6 |
| Implementation complexity | Simpler | More complex (clock domains, feedback) |

Since the ESP32-S3 is Full-Speed only and you want plug-and-play on every machine, **UAC1 is the correct choice**. UAC2 would buy you nothing and risk Windows compatibility issues.

### 3.2 The Audio Control (AC) Interface — Your "Topology"

The AC interface doesn't carry audio data. It describes the **signal path** inside your device as a graph of **Terminals** and **Units**. The host reads this to understand what your device can do (adjust volume? select inputs? mute?).

Here's what Geist's topology looks like:

```
                        ┌──────────────────────────────────┐
                        │     Audio Control Interface      │
                        │         (Interface 0)            │
                        │                                  │
  USB audio data ──►    │  ┌──────────────┐                │
  from the PC           │  │ Input        │  ID = 1        │
                        │  │ Terminal     │  Type: USB      │
                        │  │ (USB Stream) │  Streaming      │
                        │  └──────┬───────┘                │
                        │         │                        │
                        │         ▼                        │
                        │  ┌──────────────┐                │
                        │  │ Feature      │  ID = 2        │
                        │  │ Unit         │  Controls:      │
                        │  │ (Volume/Mute)│  - Master Vol   │
                        │  │              │  - Mute         │
                        │  └──────┬───────┘                │
                        │         │                        │
                        │         ▼                        │
                        │  ┌──────────────┐                │
                        │  │ Output       │  ID = 3        │     ──► to I2S
                        │  │ Terminal     │  Type:          │     ──► to PCM5102A
                        │  │ (Speaker)    │  Speaker        │     ──► to OPA1656
                        │  └──────────────┘                │     ──► to your ears
                        │                                  │
                        └──────────────────────────────────┘
```

Each block is a **descriptor struct** you define in firmware:

- **Input Terminal (ID=1)**: "I receive audio from the USB host." Terminal type = `USB Streaming (0x0101)`. 2 channels (L+R).
- **Feature Unit (ID=2)**: "I can adjust volume and mute." Source = Terminal 1. This is what makes the OS volume slider work with your device. When you drag the slider, the OS sends a `SET_CUR` control request to this unit.
- **Output Terminal (ID=3)**: "Audio comes out of a speaker." Terminal type = `Speaker (0x0301)`. Source = Unit 2. In reality, "speaker" here means your DAC+amp chain — the OS doesn't care about the actual hardware.

> [!IMPORTANT]
> The Feature Unit is what makes Windows/macOS show a volume slider for Geist in Sound Settings. Without it, the OS can only do volume in software (lower bit depth). With it, your firmware receives the volume level and you can control the PCM5102A's soft mute (via XSMT GPIO) or apply digital gain before sending to I2S.

### 3.3 The Audio Streaming (AS) Interface — The Data Pipe

The AS interface is where actual PCM audio bytes flow. It has **two alternate settings**:

**Alt Setting 0 (Zero-Bandwidth):**
- No endpoint defined
- No USB bandwidth reserved
- This is the "idle" state — Geist is plugged in but no app is playing audio
- The host selects this when audio stops

**Alt Setting 1 (Active Streaming):**
- Contains an **isochronous OUT endpoint** (data flows FROM host TO device)
- USB bandwidth is reserved
- The host selects this when audio playback begins

The Format Descriptor inside Alt Setting 1 tells the host exactly what you accept:

```
Format: PCM (raw samples, no compression)
Channels: 2 (stereo)
Bit Depth: 24
Sample Rates: 48000, 96000 (you list which ones you support)
```

> [!TIP]
> You can list multiple sample rates (48kHz AND 96kHz). The OS will negotiate the best one. Most music is 44.1/48kHz; hi-res files are 96kHz. Supporting both covers everything you'd realistically play.

### 3.4 Isochronous Transfers — How Audio Actually Moves

Regular USB transfers (bulk, control) are **reliable but not real-time** — they retry on errors, which means unpredictable latency. Audio can't tolerate that. Instead, USB Audio uses **isochronous transfers**:

- The host sends one packet **every 1ms** (the USB frame period)
- Each packet contains exactly the right number of samples for that 1ms window
- **No retries, no handshakes, no ACKs** — if a packet is corrupted, it's gone
- This guarantees consistent timing at the cost of potential (rare) data loss

For 24-bit stereo at 48 kHz:
```
Samples per 1ms frame = 48,000 / 1,000 = 48 samples
Bytes per sample = 3 bytes (24-bit) × 2 channels = 6 bytes
Bytes per frame = 48 × 6 = 288 bytes   ← well within the 1,023 byte limit
```

For 24-bit stereo at 96 kHz:
```
Samples per 1ms frame = 96 samples
Bytes per frame = 96 × 6 = 576 bytes   ← still fine
```

Your firmware's job: every 1ms, the ESP32-S3's USB peripheral receives a 288-byte (or 576-byte) isochronous packet into a DMA buffer. You feed those samples into the I2S peripheral's DMA buffer for output to the PCM5102A. The critical firmware challenge is managing the **ring buffer** between these two asynchronous clocks (USB frame clock vs. I2S bit clock).

### 3.5 The Clock Synchronization Problem

This is subtle but important. Two clocks are at play:

1. **The USB host's 1ms frame clock** — driven by the PC's USB controller crystal
2. **The ESP32's I2S bit clock** — derived from the ESP32's 40MHz crystal via PLL

These two crystals are **not synchronized**. They'll run at very slightly different rates (±50 ppm typical). Over minutes, one clock will drift ahead of the other. If the host sends samples slightly faster than you consume them, your buffer will overflow. Slightly slower, and it will underflow (causing audio glitches/clicks).

**UAC1's solution: Adaptive Sync.** You declare your isochronous endpoint as "adaptive" — meaning your device adjusts its playback clock to match the host's rate. The ESP32 can do this by monitoring the buffer fill level and nudging its I2S clock divider. This is the simplest approach and is what most USB DACs use.

```
Endpoint synchronization type: Adaptive (0b01 in bmAttributes)
```

> [!NOTE]
> The alternative is "asynchronous" mode where YOUR device is the clock master and you send feedback packets telling the host to speed up or slow down. This gives better jitter performance (your MCLK drives everything) but is significantly harder to implement and is more of a UAC2 pattern. Stick with adaptive for V1.

---

## 4. HID — Media Keys and Volume Knob

### 4.1 How HID Works

HID (Human Interface Device) is the simplest USB class. Your device sends small **reports** (just a few bytes) to the PC whenever something happens — a button is pressed, the encoder is turned. The OS interprets these as keyboard/media key events.

For Geist, you need a **Consumer Control** HID report. This is the USB standard for media keys (as opposed to regular keyboard keys). It lets you send:

- Play/Pause
- Next Track
- Previous Track  
- Volume Up / Volume Down

### 4.2 The HID Report Descriptor

The HID Report Descriptor is a weird mini-program written in a stack-based bytecode language. It tells the host the format of the reports you'll send. For Geist's media controls:

```
// Pseudocode — the actual descriptor is packed binary
Usage Page: Consumer (0x0C)         // "I'm a media device"
Usage: Consumer Control (0x01)
Collection: Application
    // --- Buttons (bitmap) ---
    Usage: Play/Pause (0xCD)
    Usage: Scan Next Track (0xB5)
    Usage: Scan Previous Track (0xB6)
    Logical Min: 0
    Logical Max: 1
    Report Size: 1 bit
    Report Count: 3
    Input: Variable                 // 3 bits, one per button
    
    // --- Padding ---
    Report Size: 5 bits
    Report Count: 1
    Input: Constant                 // pad to byte boundary
    
    // --- Volume (relative) ---
    Usage: Volume (0xE0)
    Logical Min: -1
    Logical Max: 1
    Report Size: 8 bits (2 used)
    Report Count: 1
    Input: Variable, Relative      // rotary encoder delta
End Collection
```

The actual HID report your firmware sends is just **2 bytes**:
```
Byte 0: [0 0 0 0 0 | Prev | Next | Play]   ← button bitmap
Byte 1: [-1 / 0 / +1]                        ← volume delta from encoder
```

When you press the Play button → send `[0x01, 0x00]`, then immediately send `[0x00, 0x00]` (release).  
When you rotate the encoder one click CW → send `[0x00, 0x01]` (volume +1).

The PC's OS receives this and adjusts master volume / controls media playback. No drivers needed.

### 4.3 Mapping to Geist Hardware

| Physical Input | GPIO | HID Action |
|---|---|---|
| Alps EC11 rotation (KNOB_A, KNOB_B) | IO4, IO5 | Volume Up/Down (relative) |
| Alps EC11 button (KNOB_SW) | IO6 | Mute toggle (or Play/Pause) |
| Push button (BUTTON_PLAY) | IO16 | Play/Pause |
| Push button (BUTTON_NEXT) | IO10 | Next Track |
| Push button (BUTTON_PREV) | IO8 | Previous Track |
| SPDT Toggle (TOGGLE_MODE) | IO7 | Mode switch (internal only, not HID) |

> [!TIP]
> The rotary encoder needs **debouncing** in firmware. The EC11 is mechanical and will produce bounce noise on each detent. Either debounce in software (ignore transitions faster than ~2ms) or add RC filters (10kΩ + 100nF) on KNOB_A and KNOB_B. Hardware debouncing is cleaner for real-time audio firmware where you don't want CPU time spent on GPIO polling.

---

## 5. CDC ACM — The Companion App Link

### 5.1 What CDC ACM Is

CDC ACM (Communications Device Class — Abstract Control Model) is just a USB serial port. When the host sees this interface, it creates a virtual COM port (Windows: `COM3`, Linux: `/dev/ttyACM0`, macOS: `/dev/cu.usbmodem*`). Your companion app opens this port and sends/receives data — just like UART, but over USB.

### 5.2 The Protocol You Design

This is entirely up to you. You own both ends (firmware + companion app). A simple approach:

```
// Companion app → Geist (PC sends track info)
{"cmd": "track", "title": "Resonance", "artist": "HOME", "album": "Odyssey"}

// Companion app → Geist (PC sends album art)
{"cmd": "art", "width": 240, "height": 240, "format": "rgb565"}
<raw binary pixel data follows>

// Geist → Companion app (device sends status)
{"status": "playing", "volume": 72, "sample_rate": 96000, "mode": "visualizer"}
```

JSON over serial is easy to debug. For album art, you'd send a header then raw RGB565 pixel data (the ST7789 speaks RGB565 natively). A 240×240 image at 2 bytes/pixel = 115,200 bytes — takes about 1 second at 1 Mbps baud.

### 5.3 How the Companion App Gets Track Info

On the PC side, your companion app needs to read the current media session:

- **Windows**: `Windows.Media.Control.GlobalSystemMediaTransportControlsSessionManager` (SMTC API) — C#/.NET, Python via `winsdk`, or Rust via `windows-rs`
- **macOS**: `MRMediaRemoteGetNowPlayingInfo` (private framework, or use `osascript`)
- **Linux**: D-Bus `org.mpris.MediaPlayer2` interface (MPRIS2)

The companion app polls/subscribes to media changes, then pushes updates to Geist over the CDC serial port.

---

## 6. The Complete Data Flow — Putting It All Together

Here is the full picture of everything happening simultaneously inside Geist:

```
┌─────────────────────────────────────────────────────────────────────┐
│                              HOST PC                                │
│                                                                     │
│  ┌─────────────┐   ┌────────────────┐   ┌──────────────────────┐   │
│  │ Music Player │   │ OS Media Keys  │   │ Companion App        │   │
│  │ (Tidal/foobar│   │ Handler        │   │ (reads SMTC/MPRIS)   │   │
│  │  /Spotify)   │   │                │   │                      │   │
│  └──────┬───────┘   └───────▲────────┘   └──────────┬───────────┘   │
│         │ PCM audio         │ key events            │ serial data   │
│         ▼                   │                       ▼               │
│  ┌──────────────────────────┴───────────────────────────────────┐   │
│  │                    USB Host Controller                        │   │
│  │    Isochronous OUT    Interrupt IN       Bulk IN/OUT          │   │
│  └──────────┬────────────────┬─────────────────┬────────────────┘   │
└─────────────┼────────────────┼─────────────────┼────────────────────┘
              │ USB Cable      │                 │
              ▼                ▼                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         ESP32-S3 (Geist)                            │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    USB Device Peripheral                     │   │
│  │   EP1 OUT (Isoch)     EP2 IN (Interrupt)   EP3 IN/OUT (Bulk)│   │
│  └────┬──────────────────────▲─────────────────────▲──┬────────┘   │
│       │                      │                     │  │            │
│       ▼                      │                     │  ▼            │
│  ┌──────────┐          ┌─────┴──────┐        ┌────┴──────────┐    │
│  │ Ring     │          │ HID Report │        │ CDC Serial    │    │
│  │ Buffer   │          │ Builder    │        │ Parser        │    │
│  │ (DMA)    │          │            │        │               │    │
│  └──┬───┬───┘          └─────▲──────┘        └───────┬───────┘    │
│     │   │                    │                       │            │
│     │   │  ┌─────────┐      │                       ▼            │
│     │   └─►│ FFT     │   ┌──┴──────────┐    ┌──────────────┐    │
│     │      │ Engine  │   │ GPIO        │    │ ST7789 SPI   │    │
│     │      │ (DSP)   │   │ Debounce    │    │ Display      │    │
│     │      └────┬────┘   │             │    │              │    │
│     │           │        │ Encoder     │    │ Mode 1: Track│    │
│     │           └───────►│ Buttons     │    │ Mode 2: FFT  │    │
│     │            spectrum│ Toggle      │    └──────────────┘    │
│     │                    └─────────────┘                        │
│     ▼                                                           │
│  ┌──────────┐                                                   │
│  │ I2S      │                                                   │
│  │ Peripheral│──► BCK, LRCK, DATA ──► PCM5102A ──► OPA1656     │
│  │ (DMA)    │                          (DAC)        (Amp)       │
│  └──────────┘                                        │          │
│                                                      ▼          │
│                                              4.4mm / 3.5mm      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Timing Contract (per millisecond):

```
Every 1ms USB frame:
├── Receive ~288 bytes (48 stereo samples @ 24-bit) from isochronous EP
├── Write samples into the I2S DMA ring buffer
├── (if Mode 2) Copy samples to FFT input buffer  
├── (if FFT buffer full) Run 1024-point FFT (~0.1ms on ESP32-S3 @ 240MHz)
├── (if FFT done) Render spectrum bars to ST7789 framebuffer
├── Poll GPIO state, debounce, queue HID report if changed
└── Check CDC RX buffer, parse any incoming track info
```

This all fits comfortably. The ESP32-S3 has 240MHz dual-core — Core 0 can handle USB+I2S+HID, Core 1 can handle FFT+Display. Embassy's async executor makes this natural.

---

## 7. Sample Rate & Bit Depth — What to Actually Advertise

In your AS Format Descriptor, list these supported formats:

| Sample Rate | Bit Depth | Bytes/Frame | Use Case |
|---|---|---|---|
| 48,000 Hz | 24-bit | 288 | Default — all music, games, voice |
| 96,000 Hz | 24-bit | 576 | Hi-res FLAC files |

**Skip 44,100 Hz.** The ESP32-S3's PLL cannot generate a clean 44.1kHz-family I2S clock (it derives from 40MHz, which doesn't divide cleanly into 44.1k). You'd get fractional dividers and increased jitter. 48kHz is the USB/multimedia standard anyway. If the OS has a 44.1k source, it will resample to 48k — this is normal and every USB DAC does this.

> [!CAUTION]
> If you DO list 44,100 Hz as a supported rate, the OS will send 44.1k samples and expect your I2S clock to be exact. The ESP32's PLL jitter at 44.1k will manifest as audible distortion on sustained tones. Better to not advertise it at all.

---

## 8. Firmware Architecture Overview (Rust + Embassy)

Here is how all of this maps to your Rust firmware project structure:

```
geist-firmware/
├── Cargo.toml
├── src/
│   ├── main.rs              ← Embassy entry point, spawns all tasks
│   │
│   ├── usb/
│   │   ├── mod.rs
│   │   ├── descriptors.rs   ← All USB descriptors (Device, Config, AC, AS, HID, CDC)
│   │   ├── audio.rs         ← UAC1 handler: isochronous RX, buffer management
│   │   ├── hid.rs           ← HID report builder, sends media key events
│   │   └── cdc.rs           ← CDC serial handler, parses companion app messages
│   │
│   ├── audio/
│   │   ├── mod.rs
│   │   ├── i2s.rs           ← I2S DMA driver, feeds PCM5102A
│   │   ├── buffer.rs        ← Lock-free ring buffer between USB and I2S
│   │   └── dsp.rs           ← FFT engine, spectrum analysis
│   │
│   ├── ui/
│   │   ├── mod.rs
│   │   ├── display.rs       ← ST7789 SPI driver, framebuffer management
│   │   ├── encoder.rs       ← EC11 rotary encoder with debounce state machine
│   │   ├── buttons.rs       ← Debounced button handler (prev/play/next)
│   │   └── renderer.rs      ← Screen layouts: track info view, spectrum view
│   │
│   └── app/
│       ├── mod.rs
│       └── state.rs         ← Global state machine (mode, volume, track info)
```

### Task Model (Embassy async):

```
Core 0 (Real-time audio):
  ├── usb_audio_task()      ← Highest priority. Handles isochronous USB RX.
  ├── i2s_feed_task()       ← Drains ring buffer into I2S DMA.
  └── usb_hid_cdc_task()    ← Handles HID reports + CDC serial.

Core 1 (UI + DSP):
  ├── fft_task()            ← Runs FFT when buffer is ready (~every 21ms for 1024 samples at 48k).
  ├── display_task()        ← Renders to ST7789 via SPI DMA (~30fps target).
  └── input_task()          ← Polls encoder + buttons, debounces, updates state.
```

---

## 9. Key Firmware Crates You'll Need

| Crate | Purpose |
|---|---|
| `esp-hal` | Low-level ESP32-S3 peripheral access (I2S, SPI, GPIO, USB) |
| `embassy-executor` | Async task runtime with multi-core support |
| `embassy-usb` | USB device stack — composite device, control endpoints |
| `embassy-time` | Timers, delays, debounce timing |
| `heapless` | Stack-allocated ring buffers, strings (no `alloc` needed) |
| `microfft` | `no_std` FFT implementation, fixed-size transforms |
| `embedded-graphics` | 2D rendering primitives for the ST7789 |
| `mipidsi` | ST7789 display driver (SPI) |
| `serde-json-core` | `no_std` JSON parser for CDC companion protocol |

> [!NOTE]
> You will likely need to implement the UAC1 descriptors manually using `embassy-usb`'s raw descriptor builder. There's no ready-made UAC1 crate. The TinyUSB C library has a working audio device example — use it as your descriptor reference, then translate the structs to Rust.
