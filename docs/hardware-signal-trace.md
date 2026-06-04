# Geist — Hardware Signal Trace

> Every signal that enters, exits, or moves inside Geist, explained at the wire level.

---

## The Complete Signal Chain

```
                    ┌─────────────────────────────────────────────────────┐
                    │                     GEIST                           │
                    │                                                     │
    USB-C ════╦═══► │  ┌─────────┐   ┌──────────┐   ┌──────────────────┐ │
    Cable     ║     │  │ AP2112K │──►│ ESP32-S3 │──►│ PCM5102A         │ │
  (from PC)   ║     │  │  (3.3V) │   │  (Brain) │   │ (Digital→Analog) │ │
              ║     │  └─────────┘   └──────────┘   └────────┬─────────┘ │
              ║     │  ┌─────────┐                           │           │
              ║     │  │ LM27762 │                     ┌─────┴─────┐     │
              ║     │  │ (±4.5V) │────────────────────►│ 2×OPA1656 │     │ ══► Headphones
              ║     │  └─────────┘                     │  (Amps)   │     │
              ║     │                                  └───────────┘     │
              ║     └─────────────────────────────────────────────────────┘
              ║
         5V Power
       + Data (D+/D-)
```

The signal passes through **five distinct stages**. Each stage transforms the signal in a fundamentally different way. Here's what happens at every single one.

---

## Stage 1 — The USB-C Connector: Power and Data Extraction

### The Physical Connection

```
USB-C Receptacle (J1) — 16-pin PCB mount
┌──────────────────────────────────────────────────┐
│  Pin    │ Name   │ Voltage        │ Role         │
├─────────┼────────┼────────────────┼──────────────┤
│  A4/B4  │ VBUS   │ +5.0V DC       │ Power rail   │
│  A1/B1  │ GND    │ 0V (reference) │ Return path  │
│  A7     │ D-     │ 0–3.3V diff.   │ USB data (−) │
│  A6     │ D+     │ 0–3.3V diff.   │ USB data (+) │
│  A5     │ CC1    │ via 5.1kΩ→GND  │ Orientation  │
│  B5     │ CC2    │ via 5.1kΩ→GND  │ detection    │
│  Shell  │ SHIELD │ → GND          │ EMI shield   │
└──────────────────────────────────────────────────┘
```

When you plug in the cable, two things happen simultaneously:

**Power**: The host PC sources +5.0V on the VBUS pins. This is the single power source for the entire device — every voltage rail on the board is derived from this. The 5.1kΩ pull-down resistors on CC1/CC2 (R1) tell the host "I am a device that wants power" — without these resistors, a USB-C port will not turn on VBUS at all. These resistors also let the host detect which orientation the cable is plugged in.

**Data**: The D+ and D- lines carry the USB Full-Speed differential signal. "Differential" means data is encoded as the *voltage difference* between D+ and D−, not the absolute voltage on either wire. When D+ is HIGH and D- is LOW (difference > +200mV), that's a "J" state. The reverse is a "K" state. By toggling between J and K, the host and device exchange data packets at 12 million bits per second.

> The USB data lines connect directly to the ESP32-S3's internal USB OTG transceiver on GPIO19 (D−) and GPIO20 (D+). No external PHY chip is needed — the transceiver, pull-up resistors, and signal conditioning are all inside the ESP32 silicon.

### The Decoupling Capacitor (C5: 4.7µF)

Sits right at the VBUS input. When the ESP32 suddenly draws a burst of current (e.g., a USB transaction), the voltage on VBUS would dip for a few nanoseconds because the USB cable has inductance (it resists sudden changes in current). C5 acts as a local energy reservoir — it discharges instantly to fill that dip, keeping the voltage stable. Think of it as a tiny battery that only activates during microsecond-scale demand spikes.

---

## Stage 2 — Power Delivery: One Source, Two Worlds

The +5V from USB splits into two completely independent supply rails — one for digital logic, one for analog audio. These two rails are designed to be electrically isolated from each other so that the digital noise from the ESP32 never contaminates the analog audio signal.

### The Digital Rail: AP2112K-3.3 (U2)

```
            AP2112K-3.3
            ┌──────────┐
  +5.0V ───►│ VIN  VOUT├───► +3.3V (Digital)
            │          │       │
  R7:10k ──►│ EN   GND ├───┐  ├── C6: 2.2µF (output stability)
            └──────────┘   │  │
                          GND GND
```

The AP2112K is a **Low-Dropout Regulator (LDO)**. It takes the 5V input and outputs a rock-stable 3.3V by continuously adjusting an internal pass transistor — essentially a variable resistor that burns the excess 1.7V as heat. The "low-dropout" part means it can regulate even when the input voltage sags close to the output (it only needs ~250mV of headroom).

**What does it power?**
- The ESP32-S3's core logic (CPU, memory, I2S peripheral)
- The ST7789 display controller
- The rotary encoder and button pull-ups
- The PCM5102A's *digital* supply pins (DVDD, CPVDD)

**C6 (2.2µF)**: The output capacitor. Every LDO requires one for stability — it's part of the feedback loop. Without it, the LDO's internal control loop oscillates and the output becomes a mess of high-frequency ringing instead of a clean DC rail. The 2.2µF value is specified in the AP2112K datasheet as the minimum for stable operation.

**R7 (10kΩ)**: The enable pull-up. Ties the EN pin high through VBUS so the LDO turns on as soon as USB power appears.

### The Analog Rail: LM27762 (U1)

This is the more interesting one. The OPA1656 op-amps need **both a positive and negative supply** to swing an audio waveform above and below 0V. USB only gives you +5V. The LM27762 solves this by generating both **+4.5V and −4.5V** from the single +5V input.

```
                        LM27762
                   ┌──────────────┐
  +5.0V ──────────►│ VIN    OUT+  ├──────────────────────────► +4.5V_Audio
                   │              │   │
  R3: 27.4kΩ ────►│ FB+     EN+  │   ├── C3: 2.2µF (positive output cap)
  R4: 26.7kΩ ────►│ FB-     EN-  │   ├── C12: 0.1µF (HF decoupling)
                   │              │
         C1 ──────│ C+      OUT- ├──────────────────────────► -4.5V_Audio
  (4.7µF)         │              │   │
         C2 ──────│ C-       CP  │   ├── C4: 2.2µF (negative output cap)
  (4.7µF)         │              │   ├── C14: 0.1µF (HF decoupling)
                   │         GND │
                   └──────┬──────┘
                          │
                         GND
```

**How the charge pump actually works:**

The LM27762 contains a set of internal switches and uses **C1 (the "flying capacitor")** as an energy shuttle. Here's the two-phase cycle, which repeats ~1 million times per second:

```
Phase 1 — CHARGE:                    Phase 2 — TRANSFER:

  +5V ──┐                             +5V     (disconnected)
        ├── Switch closes                    
        │                                    Switch closes
   ┌────┴────┐                         ┌────────┐
   │ C1      │  ← Charges to ~4.5V     │ C1     │  ← Dumps charge
   │ (flying)│                         │(flying)│     into C4
   └────┬────┘                         └────┬───┘
        │                                   │
        ├── Switch closes to GND            ├── Switch connects to OUT-
       GND                                  │
                                       ┌────┴────┐
                                       │ C4      │  ← Builds up
                                       │ (output)│     negative voltage
                                       └────┬────┘
                                            │
                                          -4.5V
```

In Phase 1, C1 charges to the positive output voltage (~4.5V) with its top plate connected to VIN and its bottom plate to GND. In Phase 2, the switches flip — the *top* plate of C1 is connected to GND and the *bottom* plate to the negative output. Since C1 is still holding its charge, the bottom plate is now forced to sit at approximately −4.5V relative to ground. This charge is dumped into C4, the negative output capacitor.

The positive output (+4.5V) is generated by a separate internal LDO from VIN, with the feedback resistor R3 (27.4kΩ) setting the exact voltage.

**The feedback resistors R3/R4**: These set the output voltages via a resistor divider. The LM27762's internal reference compares the divided output to a fixed 1.2V bandgap reference and adjusts the regulation to maintain the target voltage. With the values you've chosen (27.4kΩ and 26.7kΩ), the positive rail lands at approximately +4.5V.

**What does ±4.5V power?**
- Only the OPA1656 op-amps. Nothing else. This rail exists solely to give the op-amps enough voltage swing to drive headphones with a clean, undistorted waveform.

**The 0.1µF capacitors (C12, C13, C14, C15)**: High-frequency decoupling. The 2.2µF electrolytic/ceramic output caps handle bulk energy storage, but they're too physically large (internally) to respond to very fast transients. The 0.1µF ceramics have extremely low ESR (effective series resistance) and can absorb high-frequency noise spikes in the 1–100 MHz range — exactly where the charge pump's switching harmonics live.

> **Why ±4.5V and not ±5V?** Headroom. The LM27762 has internal dropout, so it can't output the full input voltage. Additionally, keeping the rails slightly below 5V gives the op-amps thermal margin and keeps them well within their rated supply range (max ±9V for the OPA1656).

---

## Stage 3 — The Digital Brain: ESP32-S3

The ESP32-S3 is where data becomes audio-shaped. But it does no analog processing — everything here is pure digital math on binary numbers.

### What Happens Inside (Electrically)

```
USB D+/D- ──► USB OTG          I2S Peripheral ──► I2S_BCLK  (GPIO21)
              Peripheral              │        ──► I2S_LRCK  (GPIO35)
                  │                   │        ──► I2S_DATA  (GPIO36)
                  │            ┌──────┘
                  ▼            │
           ┌─────────────┐    │
           │  DMA Engine  │───┘   (Direct Memory Access — moves data
           │              │        without CPU intervention)
           └──────┬───────┘
                  │
           ┌──────▼──────┐
           │ Dual Xtensa │    240 MHz, vector instructions
           │   LX7 CPU   │    for FFT/DSP processing
           └─────────────┘
```

1. **USB packets arrive** as isochronous transfers — 288 bytes every 1ms (at 48kHz/24-bit stereo). The USB peripheral's DMA engine writes these directly into a RAM buffer without the CPU lifting a finger.

2. **The CPU reads these samples** and places them into the I2S peripheral's transmit DMA buffer. Again, DMA handles the actual byte-shuffling — the CPU just manages the pointers.

3. **The I2S peripheral outputs three signals**, continuously, at a precise clock rate:

### The I2S Bus — Three Wires, One Audio Stream

```
I2S_BCLK (Bit Clock)     ──── 48kHz × 2ch × 32bits = 3.072 MHz square wave
                                ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
                               ─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘

I2S_LRCK (L/R Clock)     ──── 48 kHz, tells the DAC which channel is active
                               ┌───────────────────┐
                          LEFT ─┘                   └───────────────────
                                                     RIGHT

I2S_DATA                  ──── Serial bitstream, MSB first
                                Each sample: 24 data bits + 8 padding bits
                               ┌─Bit23─┬─Bit22─┬─Bit21─┬─ ... ─┬─Bit0─┬─pad─┐
                               │  MSB  │       │       │       │ LSB  │ 0s  │
```

**BCLK** (Bit Clock): A continuous 3.072 MHz square wave. Each rising edge means "read the next bit from the DATA line." This clock is generated by the ESP32's I2S peripheral, derived from its internal PLL. Every edge of this clock matters — if an edge arrives even a few nanoseconds early or late (jitter), the DAC will reconstruct a slightly wrong voltage, manifesting as a faint hiss or distortion. This is why I2S clock quality is critical in audio design.

**LRCK** (Left/Right Clock): A 48 kHz square wave (= sample rate). When HIGH, the data on I2S_DATA belongs to the Left channel. When LOW, it's the Right channel. This clock defines the sample rate. The DAC uses this edge as its trigger to latch a complete sample and begin conversion.

**DATA**: A serial bitstream. Each audio sample is 24 bits sent MSB-first (most significant bit first), packed into a 32-bit I2S word slot. The extra 8 bits are zero-padded. At 48kHz stereo, the DATA line carries 48,000 × 2 × 32 = 3,072,000 bits per second — exactly synchronized to BCLK.

> These three wires are the boundary between the digital and analog worlds. Everything before this point is square-wave logic. Everything after this point is continuous voltage.

---

## Stage 4 — The DAC: PCM5102A (U4)

The PCM5102A is where numbers become sound. It takes the 24-bit integers from the I2S bus and outputs a proportional analog voltage.

### Pin-Level Connections

```
                         PCM5102A (U4)
                    ┌────────────────────┐
  I2S_LRCK  ──────►│ LRCK    OUTL (6) ──├────────────► DAC_LEFT
  I2S_DATA  ──────►│ DIN     OUTR (7) ──├────────────► DAC_RIGHT
  I2S_BCLK  ──────►│ BCK               │
       GND  ──────►│ SCK     CPVDD (20)├──── 3.3V + C8: 0.1µF
                   │                    │
      3.3V  ──────►│ FLT     DVDD      ├──── 3.3V
      3.3V  ──────►│ DEMP    AVDD (8)  ├──── internally regulated
      3.3V  ──────►│ XSMT              │
       GND  ──────►│ FMT     CAPP (2)  ├──── C10: 2.2µF
                   │         CAPM (4)  ├──── C11: 2.2µF
                   │         VNEG (9)  ├──── (internal negative rail)
                   │         LDOO (18) ├──── (internal regulator out)
                   └────────────────────┘

  Config pins (active during power-up, latched permanently):
    FLT  = HIGH → Normal latency filter (linear phase)
    DEMP = HIGH → De-emphasis OFF
    XSMT = HIGH → Soft mute OFF (output active)
    FMT  = LOW  → Standard I2S format
    SCK  = GND  → Use internal PLL (derive system clock from BCK)
```

### What Happens Inside the DAC

The PCM5102A is a **delta-sigma DAC**. This is not a simple R-2R ladder — it's a fundamentally different architecture that trades speed for precision:

```
24-bit      ┌──────────────┐    ┌──────────┐    ┌──────────────┐
samples ───►│ Interpolation│───►│ Delta-   │───►│ Analog       │───► OUTL/OUTR
(48kHz)     │ Filter       │    │ Sigma    │    │ Reconstruction│    (analog)
            │              │    │ Modulator│    │ Filter        │
            │ Upsamples to │    │          │    │               │
            │ ~6.144 MHz   │    │ 1-bit    │    │ Smooths the   │
            │ (128× over-  │    │ stream   │    │ 1-bit stream  │
            │  sampling)   │    │ at very  │    │ into a clean  │
            │              │    │ high rate│    │ continuous     │
            └──────────────┘    └──────────┘    │ waveform      │
                                                └──────────────┘
```

**Step 1 — Interpolation (Digital Oversampling)**: The incoming 48,000 samples/second stream is digitally upsampled to ~6.144 million samples/second (128× oversampling). This is done by inserting zeros between original samples and then applying a digital lowpass filter. The purpose is to push the quantization noise far above the audible frequency range (20 kHz).

**Step 2 — Delta-Sigma Modulation**: The upsampled stream is converted to a 1-bit bitstream running at 6.144 MHz. Instead of each sample being a 24-bit number, each sample is just 0 or 1, but switching incredibly fast. Through noise shaping, the modulator pushes quantization errors to ultrasonic frequencies where they're inaudible. This is the core trick of delta-sigma — you trade bit depth for speed.

**Step 3 — Analog Reconstruction Filter**: A simple analog lowpass filter smooths the rapid 1-bit switching into a continuous voltage waveform. Because the quantization noise was pushed far above 20 kHz by the modulator, a gentle filter slope is enough to remove it cleanly without affecting the audible signal.

### The Output

The OUTL and OUTR pins output a **single-ended analog voltage** centered at roughly half the internal analog supply (~1.55V DC bias), with the audio signal swinging above and below that bias point:

```
Voltage at OUTL pin (playing a sine wave):

  ~2.65V  ─ ─ ─╲─ ─ ─ ─ ─ ─ ─ ─╲─ ─ ─ ─ ─ ─     peak
               ╱ ╲             ╱ ╲
              ╱   ╲           ╱   ╲
  ~1.55V ───╱─────╲─────────╱─────╲──────────     DC bias (quiescent)
           ╱       ╲       ╱       ╲
          ╱         ╲     ╱         ╲
  ~0.45V ╱───────────╲───╱───────────╲────────     trough
                      ╲╱

  Full-scale output: ~2.1 Vrms (specified in the datasheet)
```

This signal has three properties:
1. It rides on a **DC offset** (~1.55V) — this bias must be removed before it reaches headphones, or it would push the driver cone to one side permanently and waste power as heat.
2. It's **single-ended** — referenced to ground. The signal voltage moves, ground stays at 0V.
3. It's relatively **high-impedance** — the PCM5102A's output stage can't drive a headphone directly with any authority. It needs a buffer.

This is where the op-amps come in.

### The Internal Charge Pump (CAPP/CAPM/VNEG)

The PCM5102A has its own tiny charge pump inside, separate from the LM27762. The external capacitors on CAPP (C10: 2.2µF) and CAPM (C11: 2.2µF) are the flying and output capacitors for this internal pump. It generates a negative voltage (VNEG) that powers the DAC's internal analog output stage, giving it enough voltage swing to reach the full ±2.1 Vrms output.

> This is why the PCM5102A datasheet says you don't need a negative supply externally — it makes its own. But its output stage is weak (high impedance), which is why you still need the external OPA1656 op-amps to drive headphones.

---

## Stage 5 — The Amplifiers: OPA1656 (U5, U6)

The OPA1656 is a dual op-amp — two independent amplifiers in one 8-pin package. Geist uses **two packages** (four total op-amp halves), configured to produce a **true balanced output** for each stereo channel.

### What "Balanced" Means Electrically

A single-ended signal has one wire that moves (signal) and one wire that stays still (ground). A balanced signal has **two wires that both move, in opposite directions**:

```
Single-Ended (3.5mm):                Balanced (4.4mm Pentaconn):

  Signal ──── ╱╲╱╲╱╲ ──── +V          HOT  (L+) ──── ╱╲╱╲╱╲ ──── +V
  Ground ──── ──────── ──── 0V         COLD (L-) ──── ╲╱╲╱╲╱ ──── -V  (inverted!)
                                       Ground    ──── ──────── ──── 0V

  If noise couples into the cable:     If the SAME noise couples in:

  Signal ──── ╱╲╱~~╲╱╲ ── +V+noise    HOT  ──── ╱╲╱~~╲╱╲ ──── +V+noise
  Ground ──── ──────── ──── 0V         COLD ──── ╲╱╲~~╱╲╱ ──── -V+noise
                                       
  Amplifier sees: signal + noise       Amp subtracts: (HOT) - (COLD)
  → noise passes through ✗             = (+V+noise) - (-V+noise)
                                       = 2V  ← signal DOUBLED
                                       noise CANCELLED ✓
```

The receiver (your headphone amp, or the amplifier in balanced IEMs) subtracts COLD from HOT. Any noise that coupled equally into both wires (called "common-mode" noise) gets subtracted to zero. The signal, being opposite in polarity on each wire, gets *doubled*. This is why balanced connections have higher output voltage and better noise rejection.

### The Op-Amp Topology (Per Channel)

Each channel uses one OPA1656 package. One half buffers the DAC output (positive/hot phase). The other half inverts it (negative/cold phase):

```
                        ┌──── +4.5V_Audio (V+)
                        │
   DAC_LEFT ────────────┤
   (from PCM5102A       │       U5A: Unity-Gain Buffer
    OUTL pin)           │       (Non-Inverting, Gain = +1)
                        │
                        │         ┌─────────┐
                        ├────────►│ +       │
                        │         │  U5A    ├─────┬──────► OUT_L+  (Hot)
                        │    ┌───►│ -       │     │
                        │    │    └─────────┘     │
                        │    │                    │
                        │    └────────────────────┘  ← 100% negative feedback
                        │                               (output wired to inverting input)
                        │
                        │
                        │       U5B: Inverting Amplifier
                        │       (Gain = -1)
                        │
                        │    R8: 10kΩ (input)
                        ├────┤►──────────┐
                        │                │
                        │         ┌──────▼──┐
                        │    ┌───►│ -       │
                        │    │    │  U5B    ├─────┬──────► OUT_L-  (Cold)
                        │    │    │ +       │     │
                        │    │    └──┬──────┘     │
                        │    │       │            │
                        │    │      GND           │
                        │    │                    │
                        │    └────────┤►──────────┘
                        │         R8: 10kΩ (feedback)
                        │
                        └──── -4.5V_Audio (V-)
```

**U5A — The Buffer (Non-Inverting, Gain = +1)**:
The DAC output connects to the non-inverting (+) input. The output is wired directly back to the inverting (−) input — this is 100% negative feedback, which forces the output to exactly follow the input. Gain = 1. The signal is not amplified; it's **buffered**.

*Why buffer if the gain is 1?* Because the OPA1656's output can deliver far more current than the PCM5102A's output pin. The DAC's output is high-impedance (~1kΩ source impedance) — it can barely push a few milliamps. The OPA1656's output impedance is a fraction of an ohm and can drive ~40mA. Headphones need current, especially low-impedance IEMs. The buffer provides that current without loading the DAC.

**U5B — The Inverter (Gain = −1)**:
This takes the same DAC signal and creates a phase-inverted copy. Two equal resistors (both R8: 10kΩ) form a classic inverting amplifier. The gain is −R_feedback / R_input = −10k/10k = **−1**. The output is the exact mirror image of the input — when U5A outputs +0.5V, U5B outputs −0.5V.

The non-inverting (+) input of U5B is tied to GND. This sets the DC operating point at 0V — the op-amp inverts around ground. This also means **the DC bias from the DAC output (~1.55V) is removed** by this stage. The inverting amplifier's virtual ground at the (−) input naturally rejects DC offset. For U5A (the buffer), you'd typically use a DC-blocking capacitor on the output to remove the bias before reaching the headphone. 

> **The ±4.5V supply rails** are why the LM27762 charge pump exists. If the op-amps only had a positive supply (say, +5V and GND), the output could only swing from ~0V to ~3.5V — it would clip the negative half of the audio waveform. With ±4.5V rails, the output can swing from roughly −3.8V to +3.8V (accounting for ~0.7V headroom to each rail), giving a clean, symmetrical waveform with no clipping.

### Why the OPA1656 Specifically?

The OPA1656 is characterised by:

| Parameter | Value | Why It Matters |
|---|---|---|
| Input noise voltage | 2.8 nV/√Hz | Exceptionally quiet — below the noise floor of the PCM5102A itself. The amp won't add audible hiss. |
| THD+N | 0.000025% (−132 dB) | Distortion is ~50 dB below what human ears can detect. The amp is transparent. |
| GBW (Gain-Bandwidth) | 28 MHz | Can accurately track signals up to hundreds of kHz — massively overqualified for 20kHz audio, which ensures zero phase distortion in the audible band. |
| Output current | ±40 mA | Enough to drive low-impedance IEMs (like your Titan S2 at 12Ω) to deafening levels. |
| Supply range | ±2.25V to ±18V | Comfortable at ±4.5V with plenty of margin. |

---

## Stage 6 — The Output Jacks

### The 10Ω Series Resistors (R12, R13, R14, R15)

Before the signals reach the jacks, each line passes through a 10Ω resistor:

```
  OUT_L+  ───┤ R12: 10Ω ├──── pin on J2 (4.4mm Pentaconn)
  OUT_L-  ───┤ R13: 10Ω ├──── pin on J2
  OUT_R+  ───┤ R14: 10Ω ├──── pin on J2
  OUT_R-  ───┤ R15: 10Ω ├──── pin on J2
```

These resistors serve two purposes:

1. **Short-circuit protection**: If the headphone plug is inserted or removed while the amp is live, the tip momentarily shorts to the sleeve (ground). Without series resistance, the op-amp's output would be forced directly to ground — essentially driving into 0Ω. The OPA1656 would current-limit internally but could still experience thermal stress. The 10Ω resistor limits the short-circuit current to 4.5V / 10Ω = 450mA, well within safe bounds.

2. **Cable capacitance isolation**: Headphone cables are coaxial wires — they have capacitance (~100–500pF for a 1.2m cable). A capacitive load on an op-amp output can cause the feedback loop to become unstable and oscillate at ultrasonic frequencies. The 10Ω resistor decouples the cable capacitance from the op-amp's feedback loop, preventing oscillation.

> At 10Ω, the power loss is negligible. Driving 12Ω IEMs, the 10Ω resistor adds ~45% to the source impedance, which could slightly affect the IEM's frequency response (damping factor). If you find the bass getting loose, you could lower these to 4.7Ω. For 32Ω+ headphones, 10Ω is perfect.

### The 4.4mm Pentaconn Balanced Jack (J2)

```
  J2 — 4.4mm Pentaconn (5-pole)
  ┌───────────────────────────┐
  │  Tip     → L+  (Left hot) │
  │  Ring 1  → L-  (Left cold)│
  │  Ring 2  → R+  (Right hot) │
  │  Ring 3  → R-  (Right cold)│
  │  Sleeve  → GND            │
  └───────────────────────────┘
```

The headphone's balanced amplifier (or balanced IEM cable) receives L+ and L−, subtracts them, and drives the transducer with the resulting doubled, noise-free signal.

### The 3.5mm Single-Ended Jack (J3)

```
  J3 — 3.5mm TRS (3-pole)
  ┌──────────────────────────────┐
  │  Tip    → Left signal        │ ← from OUT_L+ (or DAC_LEFT direct)
  │  Ring   → Right signal       │ ← from OUT_R+ (or DAC_RIGHT direct)
  │  Sleeve → GND                │
  └──────────────────────────────┘
```

For single-ended output, only the positive phase (the buffered, non-inverted signal) is used. The cold/inverted phase is left unconnected for this jack.

> **Design consideration**: If you're tapping the buffered output (OUT_L+, OUT_R+) for the 3.5mm jack, be aware it still carries the DAC's DC bias (~1.55V). You'll need **DC-blocking capacitors** in the signal path to the 3.5mm jack — otherwise you'd push DC current through the headphone driver. A 100µF or 220µF electrolytic in series with each signal line would work. The capacitor forms a high-pass filter with the headphone impedance, with a cutoff frequency well below 20Hz (220µF into 12Ω = fc ≈ 60Hz — you may want to go larger, or use the approach below).

> Alternatively, a more elegant solution: **AC-couple at the DAC output** before the op-amp stage, so the op-amps operate around a 0V DC bias from the start. This means both the balanced and single-ended outputs would be DC-free without extra components at the jacks.

---

## The Complete Voltage Journey — One Audio Sample

Here is the life of a single audio sample — say, a 24-bit value representing a +0.5V peak in a sine wave on the left channel:

```
Step  │ Location             │ Voltage / Data                │ Domain
──────┼──────────────────────┼───────────────────────────────┼────────
  1   │ PC audio engine      │ 0x3FFFFF (integer, ≈ +0.5V   │ Software
      │                      │  at full-scale = 2.1Vrms)     │
  2   │ USB cable (D+/D-)    │ NRZI-encoded differential     │ Digital
      │                      │ bit pattern in isoch. packet   │
  3   │ ESP32 USB DMA buffer │ 0x3FFFFF in RAM at address    │ Digital
      │                      │ 0x3FC8_XXXX                    │
  4   │ ESP32 I2S DMA output │ Serial bits on I2S_DATA wire  │ Digital
      │                      │ 0011 1111 1111 1111 1111 1111 │ (3.3V logic)
  5   │ PCM5102A internal    │ Delta-sigma bitstream: a      │ Digital
      │                      │ rapid 1/0 pattern at 6.144MHz │
  6   │ PCM5102A OUTL pin    │ +2.05V (on 1.55V bias)        │ Analog
      │                      │ = +0.5V audio + 1.55V DC      │
  7   │ OPA1656 U5A output   │ +2.05V → OUT_L+ (hot)         │ Analog
      │ (buffer)             │                               │
  8   │ OPA1656 U5B output   │ -0.50V → OUT_L- (cold)        │ Analog
      │ (inverter)           │ (DC bias rejected by          │
      │                      │  virtual ground topology)      │
  9   │ After R12/R13 (10Ω)  │ +2.04V / -0.49V              │ Analog
      │                      │ (tiny drop across resistor)    │
  10  │ Headphone driver     │ ΔV = (+2.04) − (−0.49)        │ Acoustic
      │ (balanced IEM)       │    = 2.53V differential       │  → sound
      │                      │ → driver cone moves,          │
      │                      │   compresses air, you hear it │
```

---

## Noise Isolation Strategy — Why the Layout Matters

Everything described above only works cleanly if the PCB layout respects the electrical boundaries between digital and analog:

```
  ◄──────── DIGITAL ZONE ────────►│◄──────── ANALOG ZONE ────────►
                                  │
  ┌──────────┐  ┌──────────────┐  │  ┌───────────┐  ┌───────────┐
  │ ESP32-S3 │  │ Buttons,     │  │  │ OPA1656×2 │  │ Audio     │
  │ (U3)     │  │ Encoder,     │  │  │ (U5, U6)  │  │ Jacks     │
  │          │  │ Display      │  │  │           │  │ (J2, J3)  │
  └──────────┘  └──────────────┘  │  └───────────┘  └───────────┘
         │                        │         ▲
         │        I2S bus         │         │
         └────────────────────────┼─────────┘
                                  │
                            ┌─────┴─────┐
                            │ PCM5102A  │  ← sits on the border
                            │ (U4)      │     digital input side faces left,
                            └───────────┘     analog output side faces right

  Bottom layer: UNBROKEN ground copper pour across the entire board.
  No splits, no cuts, no traces on the ground plane.
```

**Why no split ground?** A common myth says you should split the ground into "digital ground" and "analog ground." In reality, a split forces return currents to take long, indirect paths around the gap — creating exactly the ground loop antenna you were trying to avoid. A **solid, unbroken ground plane** allows every return current to flow directly underneath its source trace (the path of least inductance), minimising loop area and therefore minimising radiated and received noise.

**Why zonal segregation on the top layer?** The noise coupling you're fighting isn't through the ground — it's through **electromagnetic radiation** from digital traces and **capacitive coupling** between adjacent traces. By keeping all high-speed switching (ESP32's 240MHz clock, SPI bus, USB) physically far from the sensitive analog traces (DAC output, op-amp inputs), you reduce the field strength at the analog traces to negligible levels. Distance is your most effective shield.

**Why tight decoupling?** Every IC has parasitic inductance in its power pins (the bond wires and package leads). When the IC switches states, it demands a spike of current through these inductors, causing a brief voltage drop (Ldi/dt). The 0.1µF ceramic capacitors placed *touching* the IC's power pins provide that current spike locally, so the disturbance doesn't propagate down the power trace to contaminate other ICs.
