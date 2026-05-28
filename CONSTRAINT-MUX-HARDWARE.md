# Constraint-Mux Hardware Ecosystem

> Turning serial-port consonance analysis into a plug-and-play musical instrument platform.

**Status:** Design Phase  
**Repository:** constraint-mux (Rust serial port multiplexer + real-time consonance analysis)  
**Last Updated:** 2026-05-28

---

## Table of Contents

1. [Overview](#overview)
2. [Hardware Interfaces](#hardware-interfaces)
3. [Consonance Firmware (Embedded Rust)](#consonance-firmware-embedded-rust)
4. [Binary Protocol](#binary-protocol)
5. [Web Dashboard](#web-dashboard)
6. [Collaborative Modes](#collaborative-modes)
7. [Killer Demo](#killer-demo)
8. [Open Hardware Specification](#open-hardware-specification)
9. [Go-to-Market](#go-to-market)
10. [Architecture Diagram](#architecture-diagram)
11. [Roadmap & Milestones](#roadmap--milestones)

---

## Overview

Constraint-mux is a Rust serial port multiplexer that performs real-time consonance analysis. Multiple musicians on separate devices connect to the same serial port (via the multiplexer) and see live consonance heatmaps — a shared musical consciousness.

The **Hardware Ecosystem** extends this from software into a tangible instrument platform: custom controllers, embedded firmware, wireless instruments, and a web dashboard that makes music theory visible and tactile.

**Core promise:** *It's impossible to play "wrong" notes — the lattice catches you.*

---

## Hardware Interfaces

### 1. USB-to-Serial Adapters (Analog Sensors)

The simplest entry point. Any analog sensor + USB-to-serial adapter = instrument.

| Sensor Type | Signal | Use Case | Adapter |
|---|---|---|---|
| Photoresistor (light) | Resistance → voltage | Theremin-like, hand-waving | FTDI FT232RL |
| Force-sensitive resistor (pressure) | Resistance → voltage | Pressure pads, drum surfaces | CH340G |
| Ultrasonic distance (HC-SR04) | PWM pulse width | Spatial control, proximity | CP2102 |
| Potentiometer (rotation) | Voltage divider | Knobs, sliders, faders | Any USB-serial |

**Signal chain:**
```
Sensor → ADC (10-bit, built into adapter's MCU or external MCP3008) → 
  UART framing (9600-115200 baud) → constraint-mux → consonance analysis
```

**Firmware on adapter MCU (ATmega328P or similar):**
- Sample ADC at 1kHz
- Apply exponential moving average (EMA, α=0.1) for smoothing
- Map to frequency range (configurable, default 65-2093 Hz, C2-C7)
- Send frequency as text line: `FREQ:<hz>\n` or binary packet
- Deadband: only send if Δfreq > 2Hz or Δt > 200ms

### 2. MIDI-to-Serial Bridge

Connects existing MIDI gear (keyboards, drum machines, sequencers) to constraint-mux.

**Hardware options:**
- **DIY:** Arduino Nano + MIDI IN circuit (optoisolator 6N138, 220Ω resistor, DIN-5 jack) → serial out
- **Off-the-shelf:** Roland UM-ONE mk2 or similar USB-MIDI adapter, piped through a MIDI→serial converter daemon

**Bridge firmware/software:**
```
MIDI IN → Note On/Off events → Extract note number → 
  Convert to frequency (A4=440Hz, equal temperament) → 
  Send to constraint-mux via serial
```

**Enhanced mode:**
- Velocity → amplitude mapping
- Pitch bend → frequency glide (smooth, not stepped)
- CC messages → lattice parameter control (shift the consonance window)
- Program change → preset lattice configurations

### 3. Raspberry Pi Pico (RP2040) — Custom Controllers

The sweet spot: powerful enough for onboard analysis, cheap enough to embed anywhere.

**Specs:**
- Dual-core ARM Cortex-M0+ @ 133MHz
- 264KB SRAM, 2MB Flash
- 26 GPIO pins (4 ADC channels, many PWM)
- Hardware UART × 2, SPI, I2C
- ~$4 USD

**Firmware architecture (Rust, `rp2040-hal`):**
```
┌─────────────────────────────────────┐
│  Core 0: Sensor I/O + Frequency     │
│    ADC sampling (4 channels)         │
│    Capacitive touch (GPIO)           │
│    Frequency detection               │
│    Deadband filtering                │
├─────────────────────────────────────┤
│  Core 1: Consonance + Communication  │
│    Lattice snap (lookup table)       │
│    Consonance scoring                │
│    UART TX to constraint-mux         │
│    USB CDC (debug/alt output)        │
└─────────────────────────────────────┘
```

**Peripheral support:**
- Capacitive touch pads (copper tape on cardboard works)
- Rotary encoders (infinitely turning knobs)
- OLED display (SSD1306, I2C) — shows current note, consonance score
- NeoPixel LEDs (WS2812) — color-code consonance (red=dissonant → green=consonant)
- Speaker output (PWM → RC filter → 3.5mm jack) for standalone audio

### 4. ESP32 — Wireless Constraint-Aware Instruments

Cut the cable. ESP32 adds WiFi + Bluetooth to the constraint-mux ecosystem.

**Specs:**
- Xtensa LX6 dual-core @ 240MHz (or ESP32-S3, ESP32-C3 RISC-V)
- 520KB SRAM, 4-16MB Flash
- WiFi 802.11 b/g/n, Bluetooth 4.2/BLE
- 18 ADC channels, capacitive touch × 10
- ~$5-8 USD

**Wireless architecture:**
```
ESP32 Device                        constraint-mux Server
┌──────────────┐                   ┌──────────────────┐
│ Sensors →    │  ── WebSocket ──→ │  Consonance       │
│ Freq detect  │     (port 8765)   │  analysis engine  │
│ Lattice snap │  ←─────────────── │  Broadcast to all │
│ NeoPixel FB  │  consonance data  │  connected clients│
└──────────────┘                   └──────────────────┘
```

**Power options:**
- USB-C (modern, 5V)
- 18650 LiPo battery + TP4056 charge module (4+ hours)
- 3×AA battery pack (simple, replaceable)

**Wireless protocol:**
- Primary: WebSocket (low overhead, firewall-friendly, works with `tokio-tungstenite`)
- Fallback: UDP broadcast on LAN (sub-ms latency, no TCP handshake)
- Discovery: mDNS (`constraint-mux.local`) or UDP broadcast beacon

---

## Consonance Firmware (Embedded Rust)

### Real-Time Frequency Detection

Two-stage approach optimized for microcontrollers:

**Stage 1: Zero-Crossing Count (Fast, Coarse)**
```rust
/// Count zero crossings over a window. O(n) time, O(1) memory.
fn zero_crossing_freq(samples: &[i16], sample_rate: u32) -> Option<f32> {
    let mut crossings = 0u32;
    for i in 1..samples.len() {
        if (samples[i - 1] < 0) != (samples[i] < 0) {
            crossings += 1;
        }
    }
    if crossings < 2 { return None; }
    Some(sample_rate as f32 * crossings as f32 / (2.0 * samples.len() as f32))
}
```
- Window: 256-1024 samples (at 8kHz sample rate → 32-128ms latency)
- Good for: single sustained tones, sensor-derived frequencies
- Accuracy: ±5Hz for frequencies above 100Hz

**Stage 2: Autocorrelation Refinement (Accurate)**
```rust
/// Normalized autocorrelation. Finds the fundamental period.
fn autocorrelation_freq(samples: &[i16], sample_rate: u32) -> Option<f32> {
    let n = samples.len();
    let min_period = (sample_rate as f32 / 2000.0) as usize; // max 2kHz
    let max_period = (sample_rate as f32 / 50.0) as usize;   // min 50Hz
    
    let mut best_period = 0usize;
    let mut best_corr = i64::MIN;
    
    for period in min_period..=max_period.min(n / 2) {
        let mut corr = 0i64;
        for i in 0..(n - period) {
            corr += samples[i] as i64 * samples[i + period] as i64;
        }
        if corr > best_corr {
            best_corr = corr;
            best_period = period;
        }
    }
    
    if best_period == 0 { return None; }
    Some(sample_rate as f32 / best_period as f32)
}
```
- Runs only when zero-crossing detects a signal (saves CPU)
- Accuracy: ±1Hz for clean signals

**For sensor-derived frequencies (not audio):** Skip detection entirely — the ADC value directly maps to a frequency via configurable curve (linear, exponential, logarithmic).

### Lattice Snap (Pythagorean Lookup Table)

The magic. Precomputed lookup table fits in <1KB of flash.

**Pythagorean comma = 531441/524288 ≈ 23.46 cents**

The lattice is 3D: coordinates (a, b, c) represent:
- `a` = perfect fifths (3/2) — the "x" axis
- `b` = major thirds (5/4) — the "y" axis  
- `c` = octave shifts (2/1) — the "z" axis

**Precomputed table (12×12×3 = 432 entries):**
```rust
/// Precomputed Pythagorean lattice points.
/// Each entry: (fifth_index, third_index, octave_offset) → frequency ratio
const LATTICE: &[f32] = &include!(concat!(env!("OUT_DIR"), "/lattice_table.rs"));

/// Snap a raw frequency to the nearest lattice point.
/// Returns (snapped_freq, lattice_coords, distance_in_cents).
fn lattice_snap(raw_freq: f32, base_freq: f32) -> (f32, (i8, i8, i8), f32) {
    let ratio = raw_freq / base_freq;
    let semitones = 12.0 * ratio.log2();
    
    // Search nearby lattice points (±2 fifths, ±2 thirds)
    let mut best = (raw_freq, (0i8, 0i8, 0i8), 100.0f32);
    
    for a in -2i8..=2 {
        for b in -2i8..=2 {
            for c in -1i8..=1 {
                let idx = lattice_index(a, b, c);
                let lattice_freq = base_freq * LATTICE[idx];
                let cents_diff = 1200.0 * (raw_freq / lattice_freq).log2().abs();
                
                if cents_diff < best.2 {
                    best = (lattice_freq, (a, b, c), cents_diff);
                }
            }
        }
    }
    
    best
}

/// Map lattice coordinates to flat array index.
/// Range: a ∈ [-5,5], b ∈ [-3,3], c ∈ [-2,2]
fn lattice_index(a: i8, b: i8, c: i8) -> usize {
    let a = (a + 5) as usize;
    let b = (b + 3) as usize;
    let c = (c + 2) as usize;
    c * 7 * 6 + b * 7 + a // 7*6*5 = 210 entries, ~840 bytes for f32s
}
```

**Memory budget:**
- 210 × 4 bytes (f32 ratios) = 840 bytes
- 210 × 3 bytes (i8 coordinates) = 630 bytes
- **Total: ~1.5KB** — fits easily in RP2040 or ESP32

### Consonance Scoring

Simple harmonic concordance between two frequencies:

```rust
/// Score consonance between two frequencies (0.0 = noise, 1.0 = perfect unison).
/// Based on Euler's Gradus Suavitatis (degree of sweetness).
fn consonance_score(freq_a: f32, freq_b: f32) -> f32 {
    if freq_a <= 0.0 || freq_b <= 0.0 { return 0.0; }
    
    let ratio = if freq_a > freq_b { freq_a / freq_b } else { freq_b / freq_a };
    
    // Reduce to simplest fraction (limit denominator to avoid infinite search)
    let (num, den) = approximate_ratio(ratio, 16); // denominator ≤ 16
    
    // Euler's formula: G = 1 + Σ(p-1) for prime factors of num*den
    let product = (num * den) as u32;
    let gradus = euler_gradus(product);
    
    // Normalize: unison (1/1) → 1.0, tritone (45/32, gradus=14) → ~0.3
    1.0 / (1.0 + (gradus - 1) as f32 * 0.15)
}

/// Euler's Gradus Suavitatis
fn euler_gradus(n: u32) -> u32 {
    let mut result = 1u32;
    let mut remaining = n;
    let mut factor = 2u32;
    
    while factor * factor <= remaining {
        while remaining % factor == 0 {
            result += factor - 1;
            remaining /= factor;
        }
        factor += 1;
    }
    if remaining > 1 { result += remaining - 1; }
    result
}
```

**Consonance benchmarks:**

| Interval | Ratio | Gradus | Score |
|---|---|---|---|
| Unison | 1/1 | 1 | 1.00 |
| Octave | 2/1 | 2 | 0.87 |
| Perfect fifth | 3/2 | 4 | 0.69 |
| Perfect fourth | 4/3 | 5 | 0.63 |
| Major third | 5/4 | 7 | 0.52 |
| Minor third | 6/5 | 8 | 0.48 |
| Major sixth | 5/3 | 8 | 0.48 |
| Tritone | 45/32 | 14 | 0.33 |

### Deadband Compression

Only transmit when something meaningful changes. Critical for serial bandwidth.

```rust
struct DeadbandFilter {
    last_freq: f32,
    last_consonance: f32,
    last_send_ms: u32,
    freq_threshold: f32,    // Hz — default 2.0
    consonance_threshold: f32, // default 0.05
    min_interval_ms: u32,    // default 50ms (20 msg/sec max)
}

impl DeadbandFilter {
    fn should_send(&mut self, freq: f32, consonance: f32, now_ms: u32) -> bool {
        let freq_delta = (freq - self.last_freq).abs();
        let cons_delta = (consonance - self.last_consonance).abs();
        let time_delta = now_ms - self.last_send_ms;
        
        // Force send if it's been too long (keepalive)
        if time_delta > 1000 { return true; }
        
        // Respect minimum interval
        if time_delta < self.min_interval_ms { return false; }
        
        // Send on meaningful change
        if freq_delta > self.freq_threshold || cons_delta > self.consonance_threshold {
            self.last_freq = freq;
            self.last_consonance = consonance;
            self.last_send_ms = now_ms;
            return true;
        }
        
        false
    }
}
```

---

## Binary Protocol

Low-latency binary framing for real-time consonance data over serial and WebSocket.

### Packet Format

```
┌──────┬──────┬───────┬───────┬──────┬───────────┬───────┐
│ 0xAA │ LEN  │ TYPE  │ TS    │ DEV  │ PAYLOAD   │ CRC8  │
│ Sync │ u8   │ u8    │ u32   │ u8   │ variable  │ u8    │
│ 1B   │ 1B   │ 1B    │ 4B LE │ 1B   │ N bytes   │ 1B    │
└──────┴──────┴───────┴───────┴──────┴───────────┴───────┘
```

**Header (8 bytes fixed):**
- `0xAA` — Sync byte (always, for frame detection)
- `LEN` — Total packet length (header + payload + CRC)
- `TYPE` — Message type:
  - `0x01` FREQUENCY_UPDATE
  - `0x02` CONSONANCE_UPDATE
  - `0x03` LATTICE_SNAP
  - `0x04` FULL_STATE (freq + consonance + lattice)
  - `0x10` SESSION_JOIN
  - `0x11` SESSION_LEAVE
  - `0x20` HEARTBEAT
  - `0xFF` ERROR
- `TS` — Timestamp (milliseconds since session start, u32 LE)
- `DEV` — Device ID (assigned by server on join, 0-255)

### Payload by Type

**FREQUENCY_UPDATE (0x01): 4 bytes**
```
┌──────────┐
│ freq     │
│ f32 LE   │
│ 4B       │
└──────────┘
```

**CONSONANCE_UPDATE (0x02): 5 bytes**
```
┌──────┬──────────┐
│ peer │ score    │
│ u8   │ f32 LE   │
│ 1B   │ 4B       │
└──────┴──────────┘
```

**FULL_STATE (0x04): 12 bytes**
```
┌──────────┬──────────┬──────┬──────┬──────┐
│ freq     │ cons     │ a    │ b    │ c    │
│ f32 LE   │ f32 LE   │ i8   │ i8   │ i8   │
│ 4B       │ 4B       │ 1B   │ 1B   │ 1B   │
└──────────┴──────────┴──────┴──────┴──────┘
```
Total packet: 8 (header) + 12 (payload) + 1 (CRC) = **21 bytes**

**At 20 msg/sec per device: 420 bytes/sec** — easily fits in 9600 baud serial.

### CRC8 Calculation
```rust
const CRC8_TABLE: [u8; 256] = /* standard CRC-8/CCITT */;

fn crc8(data: &[u8]) -> u8 {
    data.iter().fold(0xFF, |crc, &byte| CRC8_TABLE[(crc ^ byte) as usize])
}
```

### WebSocket Transport

Over WebSocket, the same binary packets are base64-encoded in JSON envelopes for browser compatibility:

```json
{
  "type": "consonance",
  "data": "base64-encoded-packet",
  "ts": 12345
}
```

Or, for bandwidth-sensitive scenarios, raw binary WebSocket frames (`opcode=0x02`).

---

## Web Dashboard

Real-time visualization of the shared consonance space. Built with React + WebSocket + Canvas/WebGL.

### 1. Consonance Heatmap

```
Freq (Hz)
2093 ┤▓▓░░▓▓▓░░▓░▓▓░░▓▓▓░░▓░▓▓░░▓▓▓░
1760 ┤░░▓▓░░░▓▓▓░░░░▓▓░░░▓▓▓░░░░▓▓░
1479 ┤▓░░░▓▓▓░░░▓▓░░▓░▓▓▓░░░▓▓░░▓░▓
1318 ┤░▓▓▓░░░▓▓░░▓▓▓░░░░▓▓░░▓▓▓░░░
1108 ┤▓▓░░▓▓▓░▓░▓▓░░▓▓▓░░▓░▓▓░░▓▓▓
 880 ┤░░▓▓░░░▓▓▓░░░▓▓░░░▓▓▓░░░▓▓░░
 659 ┤▓▓░▓░▓▓░░▓▓▓░▓░▓▓░░▓▓▓░▓░▓▓
 440 ┤░▓▓▓▓░▓▓▓░░░▓▓▓░░░░▓▓░░░▓▓▓
 220 ┤▓░░░▓▓░░▓▓▓░▓░░▓▓▓░▓░▓▓▓░▓░
 131 ┤░▓▓▓░░▓▓░░░▓▓▓░░▓▓▓░░░▓▓░░░
  65 ┤▓▓░▓▓▓░▓▓▓░▓░▓▓▓░▓░▓▓▓░▓░▓▓
     └──────────────────────────────────→ Time
      Device A (red) vs Device B (blue)
      Color intensity = consonance score (brighter = more consonant)
```

**Implementation:**
- Canvas 2D or WebGL for rendering (60fps target)
- Rolling window: last 30 seconds visible, configurable
- X-axis: time (auto-scrolling)
- Y-axis: frequency (logarithmic scale)
- Color: consonance score mapped to a perceptual colormap (viridis or custom warm/cool)
- Overlay: current lattice snap points as diamond markers

### 2. Lattice Visualization (3D)

Interactive 3D Pythagorean lattice using Three.js or similar.

```
         Major thirds (5/4)
          ↑
          │   ● ─── ● ─── ●
          │  /│     /│     /│
          │ ● ─── ● ─── ● │    ← Active notes glow
          │  │    /  │    / │       + pulse animation
          │  ● ─── ● ─── ● │       + connecting lines
          │  │    /  │    / │       show intervals
          │ ● ─── ● ─── ● │
          │  │    /  │    / │
          │  ● ─── ● ─── ● │
          │  │    /  │    / │
          └──●─────●─────●─┼──→ Perfect fifths (3/2)
             │    /  │    / │
             ●──────●─────●  │
              /     /     / │
             ●──────●─────●──┘
                                   Octaves (2/1) extends into/out of screen
```

**Features:**
- Active points: glowing spheres with color = consonance score
- Lines between active points: thickness ∝ consonance
- Rotation: orbit controls (drag to rotate, scroll to zoom)
- Click a point to "pin" it and see its ratio/name
- Animation: new notes appear with a growing/pulsing animation

### 3. Conservation Tracker

Rolling window showing how consonance evolves over time.

```
Consonance
1.0 ┤ ╭─╮     ╭────╮        ╭──╮
0.8 ┤╯  ╰──╮ ╭╯    ╰─╮  ╭──╯  ╰─╮
0.6 ┤      ╰─╯       ╰──╯       ╰──
0.4 ┤                                 ← Dissonant moments visible
0.2 ┤
0.0 ┤────────────────────────────────→ Time
    │←── 60s window ──→│
```

**Stats overlay:**
- Current consonance: 0.73
- Rolling average (30s): 0.68
- Peak: 0.95 (at T+12.3s)
- Lattice utilization: 8/210 points active
- "Consonance budget" remaining: 67%

---

## Collaborative Modes

### Session Mode (Default)

Multiple musicians share a consonance space. Everyone contributes to the lattice.

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Musician │  │ Musician │  │ Musician │
│   A      │  │   B      │  │   C      │
│ (guitar) │  │ (synth)  │  │ (drums)  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │serial        │wifi          │midi
     ▼              ▼              ▼
┌──────────────────────────────────────────┐
│          constraint-mux server            │
│  ┌──────────────────────────────────┐    │
│  │   Shared Lattice State           │    │
│  │   • A: (2,1,0) → 660Hz          │    │
│  │   • B: (0,0,1) → 440Hz          │    │
│  │   • C: (0,0,0) → 220Hz          │    │
│  │   Consonance: A-B=0.69 A-C=0.52 │    │
│  └──────────────────────────────────┘    │
│         │ broadcast (WebSocket)           │
└─────────┼────────────────────────────────┘
          ▼
   ┌─────────────┐
   │  Dashboard   │  ← All musicians + audience see this
   │  (browser)   │
   └─────────────┘
```

**Protocol:**
- Each musician gets a device_id on connect
- Their frequency/consonance updates are broadcast to all
- The lattice is the shared truth — all snapping happens server-side
- Musicians can "hear" each other's influence (lattice shifts)

### Conductor Mode

One person controls the lattice constraints. The musicians respond.

**Conductor controls:**
- **Lattice window:** Shift which region of the lattice is "active" (e.g., only allow notes within ±2 fifths of center)
- **Snap strength:** 0% (free pitch) to 100% (hard snap to lattice)
- **Consonance target:** Set a minimum consonance threshold — dissonant notes are silently pushed toward consonance
- **Key/scale overlay:** Map the lattice to traditional keys (C major, D dorian, etc.)
- **Time constraints:** Force changes at bar boundaries, beat boundaries, or free

**Use case:** Music education. Teacher sets constraints, students explore within them. The system makes it impossible to play "wrong."

### Audience Mode

Read-only view of the consonance landscape.

- No device_id, no input capability
- Full dashboard access (heatmap, lattice, tracker)
- Optional: "vote" system where audience can influence the conductor's lattice (democratic constraints)
- Stream-friendly: OBS-compatible overlay mode (transparent background, just the heatmap + lattice)

---

## Killer Demo

**The $40 Impossible-To-Play-Wrong Duet**

### What You Need

| Item | Qty | Cost Each | Total | Notes |
|---|---|---|---|---|
| Raspberry Pi Pico (RP2040) | 2 | $4 | $8 | Or clone boards at $2 each |
| Force-sensitive resistor (FSR 402) | 2 | $3 | $6 | Or 4 for $10 on AliExpress |
| 10kΩ resistor (pull-down) | 2 | $0.10 | $0.20 | |
| USB cable (micro or USB-C) | 2 | $1 | $2 | |
| Cardboard + copper tape | - | $1 | $1 | Enclosure + touch pads |
| NeoPixel strip (8 LEDs) | 2 | $2 | $4 | Visual feedback |
| Jumper wires | - | $1 | $1 | |
| **Total** | | | **~$22** | |

### Build Instructions (30 minutes)

**Step 1: Wire the FSR**
```
Pico GPIO26 (ADC0) ──┬── FSR ── 3.3V
                     │
                    10kΩ
                     │
                    GND
```

**Step 2: Wire the NeoPixels**
```
Pico GPIO15 ── 330Ω resistor ── NeoPixel DIN
Pico 3.3V(VBUS) ── NeoPixel VCC
Pico GND ── NeoPixel GND
```

**Step 3: Flash firmware** (`constraint-pad` binary, ~50KB)

**Step 4: Connect both Picos to host computer via USB**

**Step 5: Start constraint-mux**
```bash
$ constraint-mux --ports /dev/ttyACM0,/dev/ttyACM1 --web :8765
```

**Step 6: Open dashboard** → `http://localhost:8765`

### What Happens

1. **Player A** presses their pad → FSR resistance changes → Pico reads ADC → maps to frequency
2. **Player B** does the same on their pad
3. Both frequencies flow to constraint-mux → server snaps to nearest lattice points → computes consonance
4. **Dashboard** shows:
   - Both frequencies on the heatmap
   - Active lattice points glowing
   - Consonance line connecting them (thick/bright = consonant, thin/dim = dissonant)
5. **NeoPixels** on each pad show:
   - Color: green (consonant) → yellow (mild) → red (dissonant)
   - Brightness: pressure level
6. **The magic:** If Player A is on C and Player B presses toward F♯, the lattice gently nudges the frequency to G instead (perfect fifth). It *feels* like the instrument wants to be in tune.

### Demo Script (2 minutes)

```
0:00 - "These are two pressure pads. $10 each."
0:10 - "Each one reads how hard you press and turns it into a note."
0:20 - "Player A presses... C. Simple."
0:30 - "Player B presses randomly... and the system snaps to a perfect fifth."
0:40 - "Watch the dashboard — the lattice caught it."
0:50 - "Now they both play. Every combination is consonant."
1:10 - "It's impossible to play wrong notes."
1:20 - "The lattice is a safety net for harmony."
1:30 - "Open source. Hardware, firmware, software. Build yours today."
```

---

## Open Hardware Specification

### PCB Design: "Constraint Pad" v1.0

**Design tool:** KiCad 8.0+

**Board specs:**
- 2-layer PCB, 50mm × 70mm
- Mounting holes: 4× M3 (corners)
- USB-C connector (through-hole, robust)
- Designed for JLCPCB minimum order (5 boards for $2)

**Schematic blocks:**

```
┌──────────────────────────────────────────┐
│              Constraint Pad v1.0          │
│                                          │
│  ┌─────────┐   ┌──────────┐  ┌────────┐ │
│  │ USB-C   │──→│ RP2040   │──│ NeoPixel│ │
│  │ Conn    │   │          │  │ 8-LED   │ │
│  └─────────┘   │          │  └────────┘ │
│                │  ADC0 ───┤             │
│  ┌─────────┐   │  ADC1 ───┤  ┌────────┐ │
│  │ FSR 1   │──→│  ADC2    │  │ SSD1306│ │
│  ├─────────┤   │          │──│ 128×64 │ │
│  │ FSR 2   │──→│  GPIO ───┤  │ OLED   │ │
│  ├─────────┤   │          │  └────────┘ │
│  │ FSR 3   │──→│  I2C ────┤             │
│  ├─────────┤   │          │  ┌────────┐ │
│  │ FSR 4   │──→│          │  │ MIDI   │ │
│  └─────────┘   │  UART ───┤──│ OUT    │ │
│                │          │  │ (opt.) │ │
│  ┌─────────┐   │          │  └────────┘ │
│  │ Encoder │──→│  GPIO    │             │
│  └─────────┘   └──────────┘             │
│                                          │
│  BOM total: ~$12 (qty 1)                │
│          ~$8  (qty 100)                 │
└──────────────────────────────────────────┘
```

### Bill of Materials

| Ref | Part | Qty | Unit Price (1pc) | Unit Price (100pc) | Source |
|---|---|---|---|---|---|
| U1 | RP2040 | 1 | $1.20 | $0.80 | LCSC |
| U2 | W25Q16JVSSIQ (2MB Flash) | 1 | $0.35 | $0.20 | LCSC |
| J1 | USB-C 16-pin, THT | 1 | $0.40 | $0.25 | LCSC |
| D1 | WS2812B-2020 (8-LED strip) | 1 | $1.50 | $0.80 | AliExpress |
| OLED1 | SSD1306 128×64 I2C | 1 | $2.50 | $1.50 | AliExpress |
| R1-4 | 10kΩ 0805 | 4 | $0.01 | $0.005 | Any |
| R5 | 330Ω 0805 | 1 | $0.01 | $0.005 | Any |
| C1 | 1µF 0805 (decouple) | 1 | $0.02 | $0.01 | Any |
| C2 | 100nF 0805 (decouple) | 1 | $0.02 | $0.01 | Any |
| SW1 | Tactile switch (boot) | 1 | $0.05 | $0.03 | Any |
| J2 | 4-pin JST-XH (FSR input) | 1 | $0.10 | $0.06 | Any |
| PCB | 50×70mm 2-layer | 1 | $0.40 | $0.20 | JLCPCB |
| | **Total** | | **~$6.57** | **~$3.63** | |

*Add FSRs ($3 each, ×4 = $12) and enclosure → under $20 complete.*

### 3D-Printable Enclosure

**Design:** OpenSCAD parametric model

```
┌─────────────────────────┐ ← Top plate (1.5mm)
│  ┌───┐  ┌───┐  ┌───┐   │    - Cutouts for FSR pads
│  │Pad│  │Pad│  │Pad│   │    - OLED window (acrylic or PETG)
│  └───┘  └───┘  └───┘   │    - NeoPixel diffuser strip
│         ┌───┐           │
│         │Pad│           │
│         └───┘           │
│  ┌────────────────────┐ │
│  │    OLED Display     │ │
│  └────────────────────┘ │
└─────────────────────────┘
│  USB-C   │              │ ← Side wall (3mm)
└──────────┘              │
     Bottom plate (2mm)
```

- Material: PLA or PETG
- Print time: ~2 hours
- No supports needed
- Parametric: adjust pad count, spacing, orientation in OpenSCAD

### License

**Triple license for maximum openness:**
- **Hardware (schematics, PCB, enclosure):** CERN-OHL-S v2 (Strongly Reciprocal)
- **Firmware:** MIT OR Apache-2.0 (Rust standard dual-license)
- **Software (constraint-mux + dashboard):** MIT OR Apache-2.0

---

## Go-to-Market

### Phase 1: Maker Community (Month 1-2)

**Channels:**
- **Hackaday:** Project log series (one post per week during build)
  - Week 1: "Musical Lattice — A Serial Port Consonance Engine"
  - Week 2: "Building the Constraint Pad — $12 Open Hardware Synth Controller"
  - Week 3: "Firmware Deep Dive — Lattice Snap on a $4 Microcontroller"
  - Week 4: "The Killer Demo — Playing a Duet Where Wrong Notes Don't Exist"
- **Adafruit forums / Discord:** Cross-post builds, emphasize CircuitPython compatibility layer
- **r/rust, r/synthesizers, r/Arduino, r/raspberry_pi:** Reddit posts with video demos
- **YouTube:** 5-minute build video + 2-minute demo video

**Assets needed:**
- Build video (timelapse + narration)
- Demo video (two people playing, dashboard visible)
- PCB gerber files (ready-to-order)
- Pre-built firmware binaries (UF2 for Pico — drag-and-drop flash)

### Phase 2: Music Tech (Month 3-4)

**Channels:**
- **Modular synth community:** 
  - ModWiggler forums — emphasize the "quantizer that understands harmony" angle
  - Eurorack module concept: 4HP constraint quantizer (lattice snap in hardware)
  - Compatibility with existing modules (MIDI/CV)
- **NAMM (if feasible):** Maker area booth with demo station
- **Sonic Pi / TidalCycles community:** Integration examples (live coding + lattice constraints)
- **Music tech YouTubers:** Send demo units to Look Mum No Computer, Hainbach, Andrew Huang types

**Positioning:** "The quantizer that understands music theory — not just scales, but the actual mathematical structure of consonance."

### Phase 3: Education (Month 5-6)

**Channels:**
- **Music educators:** Lesson plan template (grades 8-12)
  - Lesson 1: "What makes notes sound good together?" (intervals, ratios)
  - Lesson 2: "Building a Pythagorean lattice" (math → music)
  - Lesson 3: "Programming the constraint pad" (Rust / embedded programming)
  - Lesson 4: "Playing together — why the lattice works" (ensemble playing)
- **STEM programs:** 
  - Integrates math (ratios, primes), physics (acoustics), CS (embedded programming), music (theory)
  - Complete project-based learning unit (20+ hours of material)
- **University music tech programs:** Syllabus integration, lab modules
- **Code.org / CS education:** Cross-list as a creative coding project

**Education pricing:**
- Classroom kit (10 constraint pads + teacher controller): $150
- Bulk PCB + component kits (solder-it-yourself): $8/student

### Ongoing Community

- **Discord server:** #build-help, #firmware, #music-sharing, #education
- **Monthly challenges:** "Build an instrument that only plays prime-numbered harmonics"
- **Patch library:** Community-submitted lattice configurations, sensor mappings, visualizations
- **Concerts / livestreams:** Monthly "constraint sessions" where musicians play together remotely

---

## Architecture Diagram

```
                    ┌─────────────────────────────────────────┐
                    │           constraint-mux server          │
                    │              (Rust/Tokio)                │
                    │                                         │
                    │  ┌───────────┐  ┌───────────────────┐  │
┌─────────┐ serial │  │ Serial    │  │ Consonance Engine  │  │
│ FSR Pad │────────│→│ Multiplex │→│ • Lattice snap     │  │
│ (Pico)  │        │  │ (tokio-   │  │ • Euler scoring    │  │
└─────────┘        │  │  serial)  │  │ • Deadband filter  │  │
                    │  └───────────┘  └────────┬──────────┘  │
┌─────────┐ WiFi   │  ┌───────────┐           │              │
│ Wireless │───────│→│ WebSocket │           │              │
│ (ESP32)  │        │  │ Server    │←──────────┘              │
└─────────┘        │  │ (port 8765)│                          │
                    │  └─────┬─────┘                          │
┌─────────┐ MIDI   │        │                                │
│ Keyboard │───────│→│ MIDI→Serial Bridge                    │
└─────────┘        │                                         │
                    └────────────┬────────────────────────────┘
                                 │ WebSocket broadcast
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │ Musician  │ │ Musician  │ │ Audience  │
              │ Dashboard │ │ Dashboard │ │ View      │
              │ (session) │ │ (session) │ │ (read-only│
              └──────────┘ └──────────┘ └──────────┘)
```

---

## Roadmap & Milestones

### Milestone 1: Firmware MVP (Week 1-2)
- [ ] RP2040 firmware: ADC read → frequency map → UART output
- [ ] Binary protocol encoder/decoder (Rust)
- [ ] Deadband filter
- [ ] Basic frequency detection (zero-crossing)
- **Deliverable:** Two Picos sending frequency data to constraint-mux over serial

### Milestone 2: Lattice Engine (Week 3-4)
- [ ] Pythagorean lookup table generator (build.rs)
- [ ] Lattice snap algorithm
- [ ] Consonance scoring (Euler's Gradus)
- [ ] Full binary protocol (all message types)
- **Deliverable:** Server snaps frequencies to lattice, computes pairwise consonance

### Milestone 3: Dashboard (Week 5-6)
- [ ] WebSocket server in constraint-mux
- [ ] Consonance heatmap (Canvas 2D)
- [ ] 3D lattice visualization (Three.js)
- [ ] Conservation tracker
- **Deliverable:** Full web dashboard showing live data from two connected devices

### Milestone 4: Killer Demo (Week 7-8)
- [ ] Build two pressure pad controllers
- [ ] Record demo video
- [ ] Write Hackaday project log
- [ ] PCB design in KiCad
- **Deliverable:** Complete demo kit + video + documentation

### Milestone 5: Open Release (Week 9-10)
- [ ] Publish GitHub repo (firmware, software, hardware)
- [ ] PCB gerber files
- [ ] 3D-printable enclosure files (OpenSCAD + STL)
- [ ] Build guide (with photos)
- [ ] Pre-built firmware binaries (UF2)
- **Deliverable:** Someone can build a constraint pad from scratch in one evening

### Milestone 6: Community (Week 11-16)
- [ ] ESP32 firmware (WiFi support)
- [ ] MIDI bridge
- [ ] Conductor mode
- [ ] Audience mode
- [ ] Education materials (lesson plans)
- **Deliverable:** Full ecosystem, ready for maker community adoption

---

*This document is a living roadmap. Update as the ecosystem evolves.*
