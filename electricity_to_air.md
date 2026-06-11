# Electricity to Air: The Complete Signal Journey Inside Geist

> *From a USB bit to a pressure wave on your eardrum — every stage, every transistor-level mechanism, every physical transformation.*

This document goes **inside** the ICs. The [Hardware Signal Trace](./hardware_signal_trace.md) covered what happens *between* chips. This covers what happens *within* them.

---

## The Complete Chain at a Glance

```
 Spotify on laptop
       │
       │ USB cable (differential voltage)
       ▼
 ┌─────────────────┐
 │   ESP32-S3       │
 │  ┌─────────┐    │
 │  │ USB PHY │    │  Stage 1: Differential → Digital bits
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ USB SIE │    │  Stage 2: Protocol decode, extract PCM samples
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │   DMA   │    │  Stage 3: Move samples to memory without CPU
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ DSP/EQ  │    │  Stage 4: Biquad filters (your parametric EQ)
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ I2S TX  │    │  Stage 5: Serialize samples into I2S bitstream
 │  └─────────┘    │
 └────────┬────────┘
          │ I2S bus (3 wires: BCLK, LRCK, DATA)
          ▼
 ┌─────────────────┐
 │   PCM5102A       │
 │  ┌─────────┐    │
 │  │ I2S RX  │    │  Stage 6: Deserialize bitstream → parallel words
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ Interp. │    │  Stage 7: Upsample 48kHz → 6.144 MHz
 │  │ Filter  │    │
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │  Delta  │    │  Stage 8: Multi-bit → 1-bit PDM stream
 │  │  Sigma  │    │
 │  │  Mod    │    │
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ 1-bit   │    │  Stage 9: Current switches toggle at MHz rates
 │  │ DAC     │    │
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ Analog  │    │  Stage 10: Low-pass filter → smooth analog voltage
 │  │ Recon.  │    │
 │  └─────────┘    │
 └────────┬────────┘
          │ Analog voltage (~2.1 Vrms)
          ▼
 ┌─────────────────┐
 │   OPA1656 ×2     │
 │  ┌─────────┐    │
 │  │ Diff    │    │  Stage 11: Input differential transistor pair
 │  │ Pair    │    │
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ Gain    │    │  Stage 12: Voltage amplification stage
 │  │ Stage   │    │
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ Output  │    │  Stage 13: Class AB push-pull output
 │  │ Stage   │    │
 │  └─────────┘    │
 └────────┬────────┘
          │ Balanced analog (OUT+, OUT-)
          ▼
 ┌─────────────────┐
 │   Headphone      │
 │  ┌─────────┐    │
 │  │ Voice   │    │  Stage 14: Current → magnetic force
 │  │ Coil    │    │
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ Diaphr. │    │  Stage 15: Force → mechanical motion
 │  └────┬────┘    │
 │  ┌────┴────┐    │
 │  │ Air     │    │  Stage 16: Motion → pressure wave → your eardrum
 │  └─────────┘    │
 └─────────────────┘
```

---

## Chapter 1: The ESP32-S3 — Digital Signal Processing Engine

### 1.1 The USB PHY — Turning Voltage Into Bits

When you plug the USB cable in, the D+ and D- lines carry **differential signals**. This means the data isn't encoded as "high = 1, low = 0" on a single wire. Instead, two wires carry *opposite* voltages, and the receiver looks at the *difference* between them:

```
USB Full-Speed Signaling (12 Mbps):

D+ ───╲  ╱───╲  ╱───────╲  ╱───
       ╲╱    ╲╱         ╲╱
D- ───╱  ╲───╱  ╲───────╱  ╲───
       ╱╲    ╱╲         ╱╲

     J    K    J       J    K     ← line states

     D+ > D- = J state
     D- > D+ = K state
```

**Why differential?** Any electromagnetic noise picked up by the cable hits *both* wires equally. When the PHY subtracts D+ from D-, the noise cancels out:

```
D+ with noise:   signal + noise
D- with noise:  -signal + noise
─────────────────────────────────
D+ minus D-:    2 × signal         ← noise gone, signal doubled
```

Inside the ESP32-S3, the **USB PHY** (Physical Layer) is a small analog circuit at the very edge of the silicon die:

```
┌─── ESP32-S3 USB PHY (on-die analog) ───────────────────────┐
│                                                              │
│  D+ pin ──→ ┌──────────────┐                                │
│              │ Differential │──→ Raw bit    ┌───────────┐   │
│  D- pin ──→ │  Comparator  │    stream  ──→│ NRZI      │   │
│              └──────────────┘               │ Decoder   │──→│ byte stream
│                                             └───────────┘   │
│  Internal:                                                   │
│  • 1.5kΩ pull-up on D+ (identifies as Full-Speed device)    │
│  • ESD protection diodes                                     │
│  • 45Ω termination resistors (match cable impedance)        │
│  • Clock recovery PLL (extracts 12 MHz clock from data)     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

The **differential comparator** is literally two transistors arranged as a long-tailed pair:

```
                    Vdd
                   ╱   ╲
                  R     R      ← matched load resistors
                  │     │
          out- ───┤     ├─── out+
                  │     │
    D+ ──┤gate  Q1     Q2  gate├── D-
                  │     │
                  └──┬──┘
                     │
                     R         ← tail current source
                     │
                    GND

If D+ > D-:  Q1 conducts more → out+ goes LOW, out- goes HIGH → logic "J"
If D- > D+:  Q2 conducts more → out+ goes HIGH, out- goes LOW → logic "K"
```

This tiny circuit — two transistors and two resistors — is the very first thing that touches your audio data.

The **NRZI decoder** then converts the J/K transitions into actual bits. In USB, a "0" bit is encoded as a *transition* (J→K or K→J), and a "1" bit is encoded as *no transition* (stay in current state). This is called NRZI (Non-Return-to-Zero Inverted):

```
Line state: J  K  K  J  K  J  J  J  K  K
Transition:    ↕  ·  ↕  ↕  ↕  ·  ·  ↕  ·
NRZI bit:      0  1  0  0  0  1  1  0  1
```

### 1.2 The USB SIE — Protocol Engine

After the PHY produces a raw byte stream, the **Serial Interface Engine** (SIE) handles the USB protocol:

```
┌─── USB SIE (hardware state machine) ───────────────────────┐
│                                                              │
│  Byte stream ──→ ┌──────────┐    ┌───────────┐             │
│                  │ Packet   │──→ │ CRC Check │             │
│                  │ Detector │    │ (CRC-16)  │             │
│                  └──────────┘    └─────┬─────┘             │
│                                        │                    │
│                                  ┌─────┴─────┐             │
│                                  │ Endpoint   │             │
│                                  │ Router     │──→ EP1 (Audio Isochronous)
│                                  │            │──→ EP2 (HID Interrupt)
│                                  │            │──→ EP3 (CDC Bulk)
│                                  └────────────┘             │
│                                                              │
│  The SIE automatically:                                      │
│  • Responds to SETUP packets (enumeration)                  │
│  • Routes data to correct endpoint buffers                  │
│  • Generates ACK/NAK handshakes                             │
│  • Handles SOF (Start of Frame) every 1ms                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**USB Audio Class** uses **isochronous transfers** — the host sends a packet of audio samples every 1ms (one USB frame). For 48kHz stereo 24-bit audio:

```
Every 1ms USB frame:
┌────────────────────────────────────────────────┐
│ 48 samples × 2 channels × 3 bytes = 288 bytes │
│                                                │
│ Sample 1L: [byte2][byte1][byte0]  ← 24 bits   │
│ Sample 1R: [byte2][byte1][byte0]               │
│ Sample 2L: [byte2][byte1][byte0]               │
│ Sample 2R: [byte2][byte1][byte0]               │
│ ...                                            │
│ Sample 48L: [byte2][byte1][byte0]              │
│ Sample 48R: [byte2][byte1][byte0]              │
└────────────────────────────────────────────────┘

48 samples/frame × 1000 frames/second = 48,000 samples/second = 48 kHz ✓
```

### 1.3 DMA — Data Movement Without the CPU

The audio data now sits in the USB endpoint buffer (a small hardware FIFO inside the SIE). Getting it to memory for processing requires the **DMA (Direct Memory Access) controller**:

```
┌─── DMA Controller ────────────────────────────────────┐
│                                                        │
│  USB EP1 Buffer ──→ ┌──────────┐ ──→ SRAM Buffer      │
│  (hardware FIFO)    │ DMA Ch.  │     (ring buffer)    │
│                     │ Engine   │                       │
│                     └──────────┘                       │
│                                                        │
│  The DMA engine is an independent hardware unit.       │
│  It has its own bus master interface to SRAM.          │
│  It copies data while the CPU does other things        │
│  (running your EQ, updating the display, etc.)         │
│                                                        │
│  When a transfer completes, it fires an interrupt      │
│  to notify the CPU: "48 new samples are ready"         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

Without DMA, the CPU would spend ~30% of its time just copying bytes from the USB buffer to memory. With DMA, the CPU spends 0% — it just gets an interrupt when new data arrives.

### 1.4 The Parametric EQ — Biquad Filters in Real-Time

Now the CPU gets involved. Your 10-band parametric EQ is a chain of **biquad IIR filters**. Each band is implemented as a second-order filter with 5 coefficients (b0, b1, b2, a1, a2):

```
One biquad filter (Direct Form I):

x[n] ──→(×b0)──→(+)──→ y[n]
              ↗   ↑
x[n-1]→(×b1)─┘   │
              ↗   │
x[n-2]→(×b2)─┘   │
                   │
y[n-1]→(×a1)──────┘
              ↗
y[n-2]→(×a2)─┘

y[n] = b0·x[n] + b1·x[n-1] + b2·x[n-2] - a1·y[n-1] - a2·y[n-2]

Operations per sample per band:
  5 multiplications + 4 additions = 9 operations

For 10 bands, stereo, 48kHz:
  9 × 10 × 2 × 48,000 = 8,640,000 operations/second

ESP32-S3 at 240 MHz with single-cycle multiply:
  240,000,000 / 8,640,000 = 27.8× headroom  ← plenty
```

The companion app computes the filter coefficients on the PC and sends them over CDC serial. The ESP32 just applies them — multiply and add, multiply and add, 10 times per sample.

```
10-band EQ chain:

Sample in → [Band 1] → [Band 2] → [Band 3] → ... → [Band 10] → Sample out
             60 Hz     150 Hz     400 Hz              16 kHz

Each band:
  • Center frequency (set by user): 60 Hz to 16 kHz
  • Gain (set by user): -12 dB to +12 dB
  • Q factor (set by user): 0.5 to 10.0
  
  These three parameters → companion app computes b0,b1,b2,a1,a2
                         → sends 5 floats over USB serial
                         → ESP32 stores them and applies in real-time
```

### 1.5 The I2S Transmitter — Serializing to the DAC

After EQ processing, the samples need to get to the PCM5102A. The ESP32's **I2S peripheral** serializes 32-bit words into a precisely-timed bitstream:

```
┌─── I2S TX Peripheral ─────────────────────────────────────┐
│                                                            │
│  SRAM ──→ ┌──────┐ ──→ ┌──────────────┐ ──→ DATA pin     │
│   (DMA)   │ FIFO │     │ Shift        │                   │
│           │ 32×  │     │ Register     │                   │
│           └──────┘     │ (parallel →  │                   │
│                        │  serial)     │                   │
│                        └──────────────┘                   │
│                                                            │
│  Clock generator:                                          │
│  MCLK (master) ──÷──→ BCLK (bit clock) ──→ BCLK pin      │
│                  ──÷──→ LRCK (L/R select) ──→ LRCK pin   │
│                                                            │
│  For 48kHz, 32-bit, stereo:                               │
│    LRCK = 48,000 Hz (one toggle per sample)               │
│    BCLK = 48,000 × 32 × 2 = 3,072,000 Hz (3.072 MHz)    │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

What the three wires actually look like on an oscilloscope:

```
LRCK:  ┌──────────────────┐                    ┌──────────────────┐
       │   LEFT channel   │   RIGHT channel    │   LEFT channel
  ─────┘                  └────────────────────┘

BCLK:  ┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐
  ─────┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└

DATA:  ─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─╳─
        MSB...............LSB  padding   MSB...............LSB  padding
        ◄── 24 data bits ──►  ◄─ 8 ──►  ◄── 24 data bits ──►  ◄─ 8 ──►
        ◄──── LEFT sample ─────────────► ◄──── RIGHT sample ──────────►

Each data bit is clocked on the FALLING edge of BCLK.
The DAC reads each bit on the RISING edge of BCLK.
LRCK LOW = left channel. LRCK HIGH = right channel.
```

---

## Chapter 2: The PCM5102A — Turning Numbers Into Voltage

This is where the magic happens. A 24-bit integer becomes a precise analog voltage. The PCM5102A does this through a multi-stage pipeline that is genuinely elegant.

### 2.1 The I2S Receiver — Reassembling the Bitstream

The first stage reverses what the ESP32's I2S TX did. A shift register clocked by BCLK captures each incoming bit and assembles them back into a parallel 24-bit word:

```
┌─── I2S Receiver ──────────────────────────────┐
│                                                │
│  DATA pin ──→ ┌────────────────────────────┐  │
│               │ 24-bit Shift Register      │  │
│  BCLK pin ──→ │ clocked on rising edge     │──→ 24-bit parallel word
│               └────────────────────────────┘  │
│                                                │
│  LRCK pin ──→ ┌────────────────┐              │
│               │ L/R Demux      │──→ Left sample register
│               │                │──→ Right sample register
│               └────────────────┘              │
│                                                │
│  When LRCK transitions:                        │
│    1. Current shift register → output latch    │
│    2. Shift register resets for next sample    │
│                                                │
└────────────────────────────────────────────────┘
```

### 2.2 The Interpolation Filter — Upsampling

Here's the first non-obvious thing the DAC does. Your audio is sampled at 48 kHz, meaning the highest frequency it can represent is 24 kHz (Nyquist theorem). But if you try to convert 48 kHz samples directly to analog, you'd need a very steep analog low-pass filter to remove the "images" above 24 kHz. Steep analog filters are expensive and introduce phase distortion.

Instead, the PCM5102A **upsamples digitally** — it increases the sample rate by 128×, inserting interpolated samples between each original sample:

```
Original signal at 48 kHz (only 48,000 points per second):

  ●         ●                   ●         ●
      ●         ●           ●       ●
          ●         ●   ●               ●
              ●       ●                     ●
  └──────────────────────────────────────────┘
  One period of a 1 kHz sine wave = 48 samples

After 128× interpolation (6,144,000 points per second):

  ●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●
  ●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●
  ●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●●
  └──────────────────────────────────────────┘
  Same period, but now 6,144 samples — much smoother

This is done with a multi-stage FIR (Finite Impulse Response) filter.
Each stage doubles the sample rate while filtering out spectral images.
```

The PCM5102A uses a **3-stage cascaded half-band FIR filter**:

```
48 kHz ──→ [Stage 1: ×2] ──→ 96 kHz ──→ [Stage 2: ×2] ──→ 192 kHz
  ──→ [Stage 3: ×2] ──→ 384 kHz ──→ ... ──→ [Final: ×32] ──→ 6.144 MHz

Each stage:
  1. Insert a zero between every sample (zero-stuffing)
  2. Apply a low-pass FIR filter to remove the spectral image
  3. Result: double the sample rate with no aliasing artifacts

Why multi-stage? A single 128× interpolation would need a 
~10,000-tap FIR filter. Cascading 7 stages of ×2 needs only
~100 taps total. Same result, 100× less computation.
```

### 2.3 The Delta-Sigma Modulator — The Heart of the DAC

This is the most conceptually difficult stage and the most brilliant. The interpolated 24-bit samples at 6.144 MHz need to become an analog voltage. Building a 24-bit accurate voltage reference with 16 million distinct levels is essentially impossible with practical electronics.

Instead, the PCM5102A uses a **delta-sigma modulator** to convert the multi-bit signal into a **1-bit stream** at very high speed. The key insight: you can trade **amplitude resolution** for **time resolution**.

```
The core idea:

  24-bit sample = 0.75 (¾ of full scale)

  Instead of outputting exactly 0.75V once:
  
  Output a stream of 1s and 0s where 75% are 1s:
  
  1 1 1 0 1 1 1 0 1 1 1 0 1 1 1 0 ...
  
  Average value = 12/16 = 0.75 ✓
  
  But we need the PATTERN of 1s and 0s to be shaped
  so that the quantization error (noise) is pushed
  to ultrasonic frequencies where it's inaudible.
```

The delta-sigma modulator is a **feedback loop** that continuously compares its 1-bit output to the desired multi-bit input, and shapes the error:

```
┌─── 4th-Order Delta-Sigma Modulator ──────────────────────────────┐
│                                                                    │
│  24-bit      ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌─────────┐       │
│  input ──(+)─┤ ∫ ├─(+)─┤ ∫ ├─(+)─┤ ∫ ├─(+)─┤ ∫ ├──→│ 1-bit   │──→ 1-bit
│           ↑  └───┘  ↑  └───┘  ↑  └───┘  ↑  └───┘  │ Quantizer│   output
│           │         │         │         │          └─────────┘       │
│           │         │         │         │               │           │
│           └─────────┴─────────┴─────────┴───── DAC ─────┘           │
│                                                 (1-bit feedback)    │
│                                                                      │
│  The ∫ blocks are integrators (accumulators).                       │
│  4th-order means 4 integrators in cascade.                          │
│  Each integrator adds another 20 dB/decade of noise shaping.       │
│                                                                      │
│  The quantizer is literally a comparator:                           │
│    input > 0 → output = 1                                           │
│    input ≤ 0 → output = 0                                           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**What "noise shaping" means visually:**

```
Frequency spectrum of quantization noise:

Without noise shaping (traditional DAC):

  Noise ▲
  level │████████████████████████████████████████
        │████████████████████████████████████████
        │████████████████████████████████████████  ← flat noise everywhere
        └────────────────────────────────────────→ Frequency
        20Hz                                  3MHz

With 4th-order delta-sigma noise shaping:

  Noise ▲
  level │                                ████████
        │                           █████████████
        │                      ██████████████████
        │                 ███████████████████████  ← noise pushed HIGH
        │░░░░░░░░░░░█████████████████████████████
        └────────────────────────────────────────→ Frequency
        20Hz    24kHz                         3MHz
         ↑                                    ↑
    Audio band: nearly                   Ultrasonic:
    zero noise (112 dB SNR)              lots of noise (filtered out later)
```

The noise hasn't been eliminated — it's been **moved** to frequencies you can't hear. This is the fundamental trick that makes affordable hi-res DACs possible.

### 2.4 The 1-Bit DAC — Current Switches

The 1-bit output stream from the delta-sigma modulator drives a deceptively simple circuit — a pair of matched current sources that toggle at ~6 MHz:

```
                    +3.3V (AVDD)
                      │
                ┌─────┴─────┐
                │   PMOS    │
                │  current  │
                │  source   │
                │  (Iref)   │
                └─────┬─────┘
                      │
            ┌─────────┼─────────┐
            │         │         │
     ┌──────┴──┐            ┌──┴──────┐
     │  SW_P   │            │  SW_N   │
     │ (PMOS)  │            │ (NMOS)  │
     └────┬────┘            └────┬────┘
          │                      │
          ├──────── VOUT ────────┤
          │                      │
     ┌────┴────┐            ┌────┴────┐
     │  SW_N   │            │  SW_P   │
     │ (NMOS)  │            │ (PMOS)  │
     └────┬────┘            └────┬────┘
          │         │         │
          └─────────┼─────────┘
                    │
                ┌───┴─────┐
                │  NMOS   │
                │ current │
                │  sink   │
                │ (-Iref) │
                └────┬────┘
                     │
                    GND

When 1-bit output = 1:  SW_P closes top-left, SW_N closes bottom-right
                        → Current flows into VOUT → positive voltage

When 1-bit output = 0:  SW_N closes top-right, SW_P closes bottom-left
                        → Current flows out of VOUT → negative voltage

The switches toggle at ~6 MHz. The current sources are matched 
to <0.01% accuracy using laser-trimmed thin-film resistors on-die.
```

The key to DAC linearity: with only 1 bit, there are only 2 levels (Iref and -Iref). You don't need 16 million matched levels — you just need 2 perfectly matched current sources. A 1-bit DAC is inherently perfectly linear. All the complexity is in the delta-sigma modulator that decides *when* to toggle.

### 2.5 The Analog Reconstruction Filter — Smoothing

The 1-bit stream at 6 MHz is now an analog signal, but it's a square wave switching between +Iref and -Iref at megahertz rates. It needs to be low-pass filtered to extract the audio band:

```
┌─── Internal Reconstruction Filter ──────────────────┐
│                                                       │
│  1-bit switching ──→ ┌───────────────┐ ──→ OUTL pin  │
│  current            │ 3rd-order     │     (smooth    │
│                     │ Butterworth   │      analog)   │
│                     │ LPF          │                 │
│                     │ fc ≈ 100 kHz │                 │
│                     └───────────────┘                 │
│                                                       │
│  The filter removes everything above ~100 kHz.        │
│  Your audio (20 Hz - 24 kHz) passes through.         │
│  The shaped quantization noise (above 24 kHz) is      │
│  heavily attenuated.                                  │
│                                                       │
│  What remains is a smooth analog voltage that is      │
│  faithful to the original digital samples within      │
│  112 dB of the full-scale output.                     │
│                                                       │
│  External caps CAPP and CAPM (C10, C11 on your PCB)  │
│  provide additional filtering of the charge pump's    │
│  internal voltage doubler.                            │
│                                                       │
└───────────────────────────────────────────────────────┘
```

After this filter, the signal at OUTL and OUTR is a clean analog voltage:
- **Full scale:** 2.1 Vrms (297 mVp-p with a 1 kHz sine at -6 dBFS)
- **SNR:** 112 dB (A-weighted) — meaning the noise floor is 5.3 µV
- **THD:** -93 dB — harmonic distortion is 0.002% of the signal
- **Output impedance:** ~200Ω (needs buffering for headphones — that's what the OPA1656 is for)

---

## Chapter 3: The OPA1656 — Precision Analog Amplification

The DAC output is too weak to drive headphones directly (200Ω output impedance into 32Ω headphones would lose most of the signal as voltage division). The OPA1656 op-amp buffers and converts to balanced output.

### 3.1 Inside an Op-Amp — Three Stages

The OPA1656 contains roughly ~80 transistors organized into three stages:

```
┌─── OPA1656 Internal Architecture ───────────────────────────────────┐
│                                                                       │
│  IN+ ──┐    ┌─────────────────┐   ┌───────────────┐   ┌──────────┐ │
│         ├──→│  Input Stage    │──→│  Gain Stage   │──→│  Output  │──→ OUT
│  IN- ──┘    │  (Differential  │   │  (Voltage     │   │  Stage   │ │
│             │   Pair)         │   │   Amplifier)  │   │  (Class  │ │
│             │                 │   │               │   │   AB)    │ │
│             │  Gain: ~60 dB   │   │  Gain: ~40 dB │   │  Gain:  │ │
│             │  Input bias:    │   │               │   │  ~1 (0  │ │
│             │  0.2 pA (JFET)  │   │               │   │   dB)   │ │
│             └─────────────────┘   └───────────────┘   └──────────┘ │
│                                                                       │
│  Total open-loop gain: ~100 dB (100,000×)                            │
│  With negative feedback, closed-loop gain = set by external R ratio  │
│                                                                       │
│  Miller compensation capacitor between gain stage and output:        │
│  This small (~15 pF) internal cap prevents the op-amp from           │
│  oscillating by rolling off the gain smoothly at high frequencies.   │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### 3.2 The Input Stage — JFET Differential Pair

The OPA1656 uses **JFET (Junction Field-Effect Transistor)** inputs instead of BJT or CMOS. This is why it's special for audio:

```
                        +4.5V
                       ╱   ╲
                     Rd     Rd     ← drain resistors (matched)
                      │     │
            Vout- ────┤     ├──── Vout+
                      │     │
    IN+ ──┤gate  J1     J2  gate├── IN-        ← JFET differential pair
                      │     │
                      └──┬──┘
                         │
                        Itail       ← constant current source (~1 mA)
                         │
                       -4.5V

Why JFETs for audio:
  • Input bias current: 0.2 pA (picoamps!)
    BJT input: ~100 nA = 500,000× more current drawn from source
    JFET: essentially zero loading on the DAC output
    
  • Input voltage noise: 4.5 nV/√Hz
    This means at audio bandwidth (20 kHz):
    Total input noise = 4.5 × √20000 = 636 nV = 0.636 µV
    That's 70 dB below a 1 mV signal — inaudible
    
  • No 1/f noise corner issues in audio band
    BJTs have popcorn noise. JFETs don't.
```

The differential pair works by **steering current**. When IN+ is slightly more positive than IN-, transistor J1 conducts slightly more current than J2. The difference in drain currents creates a voltage difference at the outputs. This voltage difference is the amplified version of the input difference.

The gain of this stage alone is about 60 dB (1000×). But the op-amp needs to be stable with feedback, so the overall gain is rolled off by the Miller compensation.

### 3.3 The Output Stage — Driving Headphones

The output stage is a **Class AB push-pull** arrangement that can both source and sink current:

```
                    +4.5V
                      │
                ┌─────┴─────┐
                │   Q_NPN   │
                │  (source  │  ← conducts during positive half-cycles
                │  current) │
                └─────┬─────┘
                      │
        Gain stage ───┤──── OUT pin ──→ to headphones
                      │
                ┌─────┴─────┐
                │   Q_PNP   │
                │  (sink    │  ← conducts during negative half-cycles
                │  current) │
                └─────┬─────┘
                      │
                    -4.5V

Class AB biasing:
  Both transistors are biased to conduct a small "idle" current
  (~1 mA) even with no signal. This eliminates "crossover 
  distortion" — the glitch that would occur when one transistor
  turns off and the other turns on at the zero-crossing point.

  Without Class AB bias (Class B):
  
  Output: ╱╲  ╱╲  ╱╲     ← clean sine
          ╱ ╲╱ ╲╱ ╲
          
  Becomes: ╱╲  ╱╲  ╱╲   ← distorted at zero crossings  
           ╱·╲╱·╲╱·╲    
           ↑ glitch (crossover distortion)

  With Class AB: smooth handoff, no glitch.
```

### 3.4 How Feedback Sets the Gain — Your Circuit

In Geist, each OPA1656 package contains two op-amps (A and B). They're configured as:

```
Channel A — Unity-Gain Buffer (non-inverting, gain = +1):

                    ┌─────────┐
  DAC_OUT ──────────┤+   OPA  ├──────────── OUT+ (hot)
                    │   1656A │
              ┌─────┤-        │
              │     └─────────┘
              │          │
              └──────────┘     ← 100% feedback = gain of exactly 1
              
  Vout = Vin × (1 + Rf/Rg)
       = Vin × (1 + 0/∞)
       = Vin × 1
       = Vin                   ← buffer, no voltage gain

  Purpose: transform the DAC's 200Ω output to ~0.001Ω output
           so headphones see a stiff voltage source


Channel B — Inverting Unity-Gain (gain = -1):

                         Rf (10kΩ, R8/R10)
                    ┌───/\/\/\/───┐
                    │             │
                    │ ┌─────────┐ │
  DAC_OUT ──/\/\/──┤─┤-   OPA  ├─┴──────── OUT- (cold)
           Rin      │ │   1656B │
          (10kΩ,    │ ┤+        │
          R9/R11)   │ └────┬────┘
                    │      │
                    │     GND
                    │
                    
  Vout = -Vin × (Rf / Rin)
       = -Vin × (10k / 10k)
       = -Vin                  ← inverted copy of the signal

  Purpose: create the "cold" leg of the balanced signal
```

The balanced output (OUT+ and OUT-) carries the same signal, but one is inverted. The headphone driver in your IEMs subtracts them:

```
Signal received by headphone driver:

  OUT+ = +V (hot)
  OUT- = -V (cold)
  
  Driver sees: OUT+ minus OUT- = V - (-V) = 2V
  
  Any noise picked up equally on both wires:
  OUT+ = +V + noise
  OUT- = -V + noise
  
  Driver sees: (+V + noise) - (-V + noise) = 2V + 0
  
  Noise cancelled. Signal doubled.
```

---

## Chapter 4: Voltage to Sound — The Transduction

### 4.1 The Voice Coil — Electricity Becomes Force

Inside every headphone driver (dynamic, BA, or planar) is a **conductor in a magnetic field**. When current flows through it, the Lorentz force pushes it:

```
┌─── Dynamic Driver Cross-Section ───────────────────┐
│                                                      │
│            Diaphragm (thin mylar/bio-cellulose)     │
│         ╱─────────────────────────────────╲         │
│        ╱                                   ╲        │
│       ╱                                     ╲       │
│      │          ┌───────────────┐            │      │
│      │          │  Voice Coil   │ ← current  │      │
│      │          │  (copper wire │   flows     │      │
│      │          │   wound in    │   here      │      │
│      │          │   a cylinder) │            │      │
│      │          └───────┬───────┘            │      │
│      │                  │                    │      │
│      │    N ════════════╪═══════════ S       │      │
│      │    ↑   permanent │ magnetic   ↑       │      │
│      │    magnet        │ field      magnet  │      │
│      │                  │                    │      │
│      └──────────────────┴────────────────────┘      │
│                         │                            │
│                      Surround                        │
│                   (flexible edge)                    │
│                                                      │
│  F = B × I × L                                      │
│                                                      │
│  F = force on the coil (Newtons)                    │
│  B = magnetic field strength (~1 Tesla)             │
│  I = current through coil (milliamps)               │
│  L = length of wire in the field (~1 meter wound)   │
│                                                      │
│  For a 32Ω headphone at 1 Vrms:                    │
│  I = V/R = 1/32 = 31.25 mA                         │
│  F = 1 × 0.03125 × 1 = 31.25 milliNewtons          │
│                                                      │
│  That tiny force, applied to a <1 gram diaphragm,   │
│  produces accelerations of ~30 m/s² — enough to     │
│  vibrate the diaphragm hundreds to thousands of      │
│  times per second.                                   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 4.2 The Diaphragm — Force Becomes Air Pressure

The voice coil is glued to a thin diaphragm (typically 6-50mm diameter, 10-50 µm thick). When the coil moves, the diaphragm moves with it, pushing and pulling air molecules:

```
Audio signal voltage over time (1 kHz sine):

  +V ─── ╱╲      ╱╲      ╱╲
          ╱  ╲    ╱  ╲    ╱  ╲
  0  ────╱────╲──╱────╲──╱────╲──
         ╱    ╲╱╱    ╲╱╱    ╲╱
  -V ───╱      ╱      ╱

Maps to diaphragm position:

  Forward ─── ╱╲      ╱╲      ╱╲
              ╱  ╲    ╱  ╲    ╱  ╲
  Rest   ────╱────╲──╱────╲──╱────╲──    ← ~10 µm displacement
              ╱    ╲╱╱    ╲╱╱    ╲╱
  Backward ──╱      ╱      ╱

Maps to air pressure at your eardrum:

  High   ─── ╱╲      ╱╲      ╱╲
              ╱  ╲    ╱  ╲    ╱  ╲
  Ambient ───╱────╲──╱────╲──╱────╲──    ← ~0.1 Pa pressure variation
              ╱    ╲╱╱    ╲╱╱    ╲╱        (vs 101,325 Pa atmospheric)
  Low    ────╱      ╱      ╱

The pressure variation that constitutes "loud music" is about
0.00001% of atmospheric pressure. Your ear is absurdly sensitive.
```

### 4.3 The Numbers — Full Chain

Let's trace an actual sample value through the entire chain:

```
Step 1: Spotify sends sample value 0x5A8000 (24-bit, left channel)
        Decimal: 5,931,008 out of 8,388,608 maximum
        Fraction: 0.707 of full scale = -3 dBFS

Step 2: ESP32 receives via USB, applies EQ (no change at flat EQ)
        Sends 0x5A8000 via I2S to PCM5102A

Step 3: PCM5102A interpolates: 48k → 6.144M samples/sec
        Delta-sigma modulates to 1-bit stream:
        Average density of 1s = 70.7%

Step 4: PCM5102A analog output:
        VOUT = 0.707 × 2.1 Vrms = 1.485 Vrms
        Peak voltage = 1.485 × √2 = 2.1 Vpeak

Step 5: OPA1656A buffer: OUT+ = +1.485 Vrms
        OPA1656B inverter: OUT- = -1.485 Vrms
        
        Balanced differential: 2 × 1.485 = 2.97 Vrms

Step 6: Through 2.2Ω output resistors (R12-R15):
        Negligible voltage drop at headphone currents

Step 7: At 32Ω headphone:
        Current = 2.97V / 32Ω = 92.8 mA peak (!)
        Power = 2.97² / 32 = 275 mW

        Actually, the OPA1656 current limits at ~40 mA,
        so max power into 32Ω ≈ (40mA × 32Ω)² / 32 = 51 mW
        That's still VERY loud — about 120 dB SPL with typical IEMs

Step 8: Voice coil force:
        F = B × I × L = 1T × 0.04A × 1m = 40 mN

Step 9: Diaphragm acceleration:
        a = F/m = 0.04N / 0.001kg = 40 m/s²
        
Step 10: Air pressure variation at eardrum:
         ~2 Pa peak (about 100 dB SPL)
         Your brain interprets this as music.
```

---

## Chapter 5: The Invisible Clocks — Why Timing Matters

Every stage in this chain is governed by clocks, and clock quality directly affects audio quality.

### 5.1 Jitter — The Enemy of Digital Audio

**Jitter** is the random variation in clock timing. If the I2S BCLK edge arrives 100 picoseconds early on one cycle and 100 picoseconds late on the next, the DAC converts the sample at slightly the wrong instant:

```
Ideal BCLK (perfect timing):
  ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐
  │  │  │  │  │  │  │  │  │  │
──┘  └──┘  └──┘  └──┘  └──┘  └──

Real BCLK (with jitter):
  ┌──┐  ┌──┐ ┌──┐   ┌──┐ ┌──┐
  │  │  │  │ │  │   │  │ │  │
──┘  └──┘  └─┘  └───┘  └─┘  └──
              ↑        ↑
              early    late   ← timing error = jitter

Effect on a 20 kHz sine wave with 1 ns RMS jitter:
  SNR degradation = 20 × log₁₀(1 / (2π × 20000 × 1e-9))
                  = 20 × log₁₀(7958)
                  = 78 dB

  So 1 ns of jitter limits your SNR to 78 dB at 20 kHz.
  For 112 dB SNR, you need jitter below ~15 picoseconds RMS.
```

The PCM5102A solves this with its **internal PLL (Phase-Locked Loop)**. It regenerates a clean clock from the incoming BCLK, filtering out most of the jitter:

```
┌─── PCM5102A Internal PLL ────────────────────────┐
│                                                    │
│  BCLK ──→ ┌──────────┐   ┌──────┐   ┌────────┐  │
│  (jittery) │ Phase    │──→│ Loop │──→│  VCO   │──→ Internal master clock
│            │ Detector │   │Filter│   │(voltage│   (clean, low jitter)
│  ┌────────→│          │   │(LPF) │   │control │  │
│  │         └──────────┘   └──────┘   │osc.)   │  │
│  │                                    └───┬────┘  │
│  │                                        │       │
│  │         ┌──────────┐                   │       │
│  └─────────┤ Divider  │←──────────────────┘       │
│            │ (÷N)     │                           │
│            └──────────┘                           │
│                                                    │
│  The PLL locks to the average frequency of BCLK   │
│  but its VCO output has much lower jitter because  │
│  the loop filter averages out the random timing    │
│  variations.                                       │
│                                                    │
└────────────────────────────────────────────────────┘
```

This is why the PCM5102A's SCK pin can be left unconnected (tied to GND via R7 on your board) — the internal PLL generates its own master clock from BCLK alone. Convenient and effective.

---

## Summary: The Complete Transform

```
USB cable:     Differential voltage swings (±400 mV) at 12 MHz
                            │
                            ▼
ESP32 USB PHY: Comparator → bits → NRZI decode → bytes
                            │
                            ▼
ESP32 SIE:     Protocol decode → extract 24-bit PCM samples
                            │
                            ▼
ESP32 DMA:     Hardware copy to SRAM ring buffer
                            │
                            ▼
ESP32 CPU:     Biquad IIR filter chain (10-band parametric EQ)
                            │
                            ▼
ESP32 I2S TX:  Parallel → serial shift register → 3-wire bus
                            │
                            ▼
PCM5102A RX:   Serial → parallel → 24-bit sample words
                            │
                            ▼
PCM5102A:      128× interpolation → delta-sigma → 1-bit @ 6.144 MHz
                            │
                            ▼
PCM5102A:      Matched current switches → analog reconstruction filter
                            │
                            ▼
OPA1656:       JFET differential pair → gain stage → Class AB output
                            │
                            ▼
Headphone:     Current → magnetic force → diaphragm motion → air pressure
                            │
                            ▼
Your eardrum:  Pressure variation → mechanical motion → nerve impulses
                            │
                            ▼
Your brain:    "Oh, I like this song"
```

> From a ±400 millivolt differential wiggle on a USB cable to a ±2 Pascal pressure variation on a membrane in your inner ear — 16 transformations across 5 different physical domains (electrical, digital, analog, mechanical, acoustic) — all happening 48,000 times per second, within 2 milliseconds end-to-end.
>
> That's what Geist does.
