# Dial-Space Explorer — Killer App Design

> *"The conservation law was the hypothesis. The parameter space is the map. The Innovation Cycle is the story the map tells over time."*

**Version:** v0.9  
**Status:** Design specification  
**Dependencies:** [DIALS-NOT-LAWS.md](../docs/DIALS-NOT-LAWS.md), [INNOVATION-CYCLE.md](../docs/INNOVATION-CYCLE.md)

---

## Table of Contents

1. [Product Vision](#1-product-vision)
2. [The Parameter Space](#2-the-parameter-space)
3. [3D Visualization Engine](#3-3d-visualization-engine)
4. [Audio Synthesis Engine](#4-audio-synthesis-engine)
5. [AI Composition System](#5-ai-composition-system)
6. [Research Tools](#6-research-tools)
7. [Social & Community Features](#7-social--community-features)
8. [Architecture](#8-architecture)
9. [Foundation: Data Model & API Schema](#9-foundation-data-model--api-schema)
10. [Wireframes](#10-wireframes)
11. [MVP → Full Product Roadmap](#11-mvp--full-product-roadmap)
12. [Edge Cases & Innovation](#12-edge-cases--innovation)

---

## 1. Product Vision

### 1.1 The One-Sentence Pitch

**Dial-Space Explorer is a 3D interactive map of every musical tradition in the known universe — and the places no tradition has ever gone — where you can hear, generate, and share music at any point in the parameter space of possible music.**

### 1.2 Who It's For

| Persona | What They Get | Why They Care |
|---------|---------------|---------------|
| **Ethnomusicologist** | Empirical map of 10+ traditions with spectral fingerprints, Laplacian eigenvectors, conservation ratios | Test hypotheses about cultural evolution across parameter space |
| **AI Music Researcher** | Real-time synthesis at ANY dial position, including unexplored regions | Generate genuinely new music; test artificial creativity against the structure surplus filter |
| **Producer/Composer** | Style interpolation between any two traditions | Fusion genres with mathematical grounding; find "the sound between Carnatic and Gagaku" |
| **Educator** | Guided expeditions through dial space with narration | Teach music theory, cultural evolution, the Innovation Cycle with visceral 3D exploration |
| **Curious Listener** | Intuitive 3D interface to explore music from every culture | Discover traditions they've never heard; understand why music sounds the way it does |
| **AI Agent (this system)** | The dial framework is the native ontology | Systematic browsing, generation, and analysis of musical possibility space |

### 1.3 Core Interaction Model

```
User sees:  3D point cloud of traditions     (the geography)
User does:  Drag a cursor through the space  (navigation)
User hears: Morphing audio at cursor position (the experience)
User gets:  Spectral fingerprint of position  (the analysis)
User can:   Generate new music at any point   (the creation)
User can:   Share, compare, and discover      (the community)
```

---

## 2. The Parameter Space

### 2.1 The Three Axes

| Axis | Label | Domain | What It Measures | Source Traditions |
|------|-------|--------|------------------|-------------------|
| X | **I_vert** | 1.5 – 4.0 | Vertical/pitch information: microtonal density, scale complexity, key gradients, interval system richness | All 10 traditions (Carnatic: 2.77, Gagaku: 2.38, etc.) |
| Y | **I_horiz** | 1.0 – 4.5 | Horizontal/rhythmic information: meter complexity, polyrhythm density, syncopation, cycle framework richness | All 10 traditions (West African: 3.63, Western CP: 2.05) |
| Z | **I_spectral** | 0.0 – 4.0 | Spectral/timbral information: inharmonicity, partial evolution, microtonal inflection, beating patterns | Extrapolated from Gamelan, Gagaku, and spectral analysis |

### 2.2 The 10 Known Traditions (Seed Data)

These are the measured coordinate points from DIALS-NOT-LAWS (exp4_conservation_law.json). Each becomes a labeled cluster in the 3D space.

```
┌──────────────┬────────┬─────────┬──────────┬──────────────┐
│ Tradition    │ I_vert │ I_horiz │ I_spect  │ Cluster      │
├──────────────┼────────┼─────────┼──────────┼──────────────┤
│ Carnatic     │  2.77  │  3.63   │  1.8     │ Maximal      │
│ Hindustani   │  2.77  │  3.45   │  1.8     │ Maximal      │
│ Turkish Makam│  2.83  │  3.28   │  2.0     │ Maximal      │
│ Arabic Maqam │  2.94  │  3.10   │  2.0     │ Maximal      │
│ W. African   │  2.41  │  3.63   │  0.5     │ Rhythmic     │
│ Balinese Gam │  2.31  │  3.10   │  3.5     │ Balanced     │
│ Javanese Gam │  2.31  │  2.75   │  3.5     │ Balanced     │
│ Western CP   │  2.72  │  2.05   │  1.2     │ Harmonic     │
│ Chinese Trad │  2.32  │  2.05   │  2.5     │ Presence     │
│ Japanese Gag │  2.38  │  1.70   │  3.8     │ Presence     │
└──────────────┴────────┴─────────┴──────────┴──────────────┘
```

*Note: I_spectral values are initial estimates. The real measurements come from live FFT analysis of source recordings.*

### 2.3 The 5 Clusters (Color-Coded)

| Cluster | Color | Hex | Members |
|---------|-------|-----|---------|
| **Maximal** (High I_vert, High I_horiz) | 🔴 Red | `#FF4136` | Carnatic, Hindustani, Turkish, Arabic |
| **Rhythmic** (Low I_vert, High I_horiz) | 🟡 Yellow | `#FFDC00` | West African |
| **Balanced** (Mid I_vert, Mid I_horiz, High I_spectral) | 🟢 Green | `#2ECC40` | Balinese, Javanese |
| **Harmonic** (High I_vert, Low I_horiz) | 🔵 Blue | `#0074D9` | Western CP |
| **Presence** (Low I_vert, Low I_horiz, High I_spectral) | 🟣 Violet | `#B10DC9` | Chinese, Gagaku |

### 2.4 Occupied vs. Unexplored Space

From DIALS-NOT-LAWS §1.3 and §4.4:

| Region | I_vert | I_horiz | I_spectral | Status |
|--------|--------|---------|------------|--------|
| **Maximal cluster** | 2.5 – 3.0 | 3.0 – 3.8 | 1.5 – 2.5 | Occupied (4 traditions) |
| **Rhythmic outlier** | 2.3 – 2.5 | 3.5 – 3.7 | 0.3 – 0.7 | Occupied (1 tradition) |
| **Gamelan region** | 2.2 – 2.4 | 2.5 – 3.2 | 3.0 – 4.0 | Occupied (2 traditions) |
| **Western/Harmonic** | 2.5 – 2.9 | 1.8 – 2.3 | 1.0 – 1.5 | Occupied (1 tradition) |
| **Presence cluster** | 2.2 – 2.5 | 1.5 – 2.2 | 2.0 – 4.0 | Occupied (2 traditions) |
| **Position A** | 3.0 – 4.0 | 1.0 – 1.8 | 3.0 – 4.0 | 🚀 EMPTY — Extreme pitch + spectral, minimal rhythm |
| **Position B** | 1.5 – 2.2 | 3.8 – 4.5 | 0.0 – 1.5 | 🚀 EMPTY — Extreme rhythm, minimal pitch |
| **Position C** | 2.8 – 3.2 | 2.8 – 3.2 | 2.8 – 3.2 | 🚀 EMPTY — Maximum everything (cognitive limit region) |
| **The void** | Everything else | | | Dark — predicted S ≈ 0, no stable style possible |

**Total filled space:** ~18% (estimated)  
**Total empty space:** ~82% (the frontier)

### 2.5 Coordinate System Details

- Origin: (1.5, 1.0, 0.0) — absolute minimum information (pure silence / monotone)
- Bounding box: (4.0, 4.5, 4.0) — absolute theoretical maximum
- Traditions occupy the range: I_vert [2.3 – 2.94], I_horiz [1.7 – 3.63] — a sub-volume of ~18% of the full space
- Empty space (82%) is rendered as a dark void with sparse star-like anchor points
- "Frontier regions" (positions near occupied clusters with S > 0 but no tradition) are highlighted in aurora glow

---

## 3. 3D Visualization Engine

### 3.1 Technology Stack: Three.js via React Three Fiber

```
┌──────────────────────────────────────────────────────┐
│                    DIAL-SPACE EXPLORER               │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │            3D VIEWPORT (Canvas)              │   │
│  │                                              │   │
│  │    ● Carnatic (red)       ● West African     │   │
│  │    ● Hindustani (red)        (yellow)        │   │
│  │    ● Turkish (red)        ● Gamelan (green)  │   │
│  │    ● Arabic (red)         ● Western (blue)   │   │
│  │                            ● Gagaku (violet) │   │
│  │                                              │   │
│  │    ═══ Cursor ═══►  (user navigates here)   │   │
│  │                                              │   │
│  │    [dark void in unexplored regions 82%]     │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌─────── SIDEBAR ──────────────────────────┐        │
│  │ Current Position: (2.65, 3.12, 2.45)     │        │
│  │ Nearest: Carnatic (0.61 away)            │        │
│  │ Style: "Near Maximal, leaning Balanced"  │        │
│  │ [Generate] [Save] [Share]                │        │
│  └──────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────┘
```

### 3.2 Key Visual Features

#### Tradition Clusters
- **Spheres:** Each tradition rendered as a glowing sphere, radius proportional to tradition's total information (I_total)
- **Density clouds:** Gaussian kernel density estimates around each cluster, opacity based on local tradition density
- **Labels:** Tradition name in 3D text (Billboard-mode) with hover tooltip showing coordinates
- **Trajectory trails:** For traditions with known evolutionary paths (e.g., Western: meantone → ET → modern), animate an arc through parameter space

#### The Cursor
- **Primary interaction:** Drag a glowing cursor through 3D space (OrbitControls + DragControls hybrid)
- **Visualization:** Crosshair sphere with polarizing ring, adapts color based on nearest tradition
- **Snap-to-tradition:** Click a tradition sphere to snap cursor to its exact coordinates and hear its sound
- **Free exploration:** Drag cursor freely through the space — audio morphs in real-time
- **Geodesic lines:** When cursor is between traditions, show shortest-path curves through the space

#### The Void / Frontier Regions
- **Dark regions:** 82% of the space rendered as a near-black void with subtle pointillist stars
- **Frontier glow:** Regions with predicted S > 0 but no tradition shown as aurora-like shimmering patches
- **Innovation hotspots:** Predictive model overlays heatmap where the NEXT innovation is likely (empty regions adjacent to active clusters)
- **Phase overlays:** Toggle Innovation Cycle phases (which tradition is in discovery, codification, boredom, rebellion?)

#### Controls

```
┌─────────────────────────────────────┐
│   NAVIGATION                       │
│                                     │
│  🖱️ Drag = Move cursor in 3D       │
│  🔄 Scroll = Zoom in/out           │
│  🖐️ Right-drag = Rotate view       │
│  🎯 Click tradition = Snap cursor   │
│                                     │
│   OVERLAYS                          │
│  [✓] Tradition labels               │
│  [✓] Density clouds                 │
│  [ ] Innovation Cycle phases        │
│  [ ] Spectral heatmap               │
│  [ ] Frontier predictions           │
│  [ ] Geodesic paths                 │
└─────────────────────────────────────┘
```

### 3.3 WebGPU Considerations

For the MVP, Three.js with WebGL renderer is sufficient (works on all modern browsers). WebGPU paths:
- **Phase 2:** WebGPU renderer for 10x particle count (density clouds, thermal heatmaps)
- **Phase 3:** Real-time ray-marched volume rendering for the frontier aurora effects
- **Phase 4:** WebGPU compute shaders for on-device spectral analysis (WASM fallback)

### 3.4 Performance Targets

| Metric | MVP (WebGL) | Phase 2 (WebGPU) | Phase 3 (Compute) |
|--------|-------------|-------------------|-------------------|
| Traditions rendered | 10 | 50+ | 500+ |
| Particle count | 10K | 100K | 1M+ |
| Audio latency | < 50ms | < 20ms | < 10ms |
| FPS (mid-range GPU) | 60 | 60 | 30 |
| FPS (integrated GPU) | 30 | 60 | 30 |

---

## 4. Audio Synthesis Engine

### 4.1 Architecture

```
┌──────────────────────────────────────────────────────────┐
│              AUDIO PIPELINE                              │
│                                                          │
│  Dial Position ──→ Parameter Interpolation ──→ Synthesis │
│  (2.65,3.12,2.45)       │                                │
│                          ▼                                │
│                   Nearest Traditions                     │
│                   (Carnatic, Turkish, Balinese)          │
│                          │                                │
│                          ▼                                │
│              Weighted Interpolation (geodesic)           │
│                          │                                │
│                  ┌───────┴───────┐                       │
│                  ▼               ▼                       │
│           Pitch Engine    Rhythm Engine                  │
│           (Tuning, scale,  (Meter, tempo,               │
│            harmony)         polyrhythm)                  │
│                  │               │                       │
│                  └───────┬───────┘                       │
│                          ▼                               │
│                     Spectral Engine                      │
│                    (Timbre, inharmonicity,               │
│                     partial distribution)                │
│                          │                               │
│                          ▼                               │
│                     Master Mix                          │
│                          │                               │
│                          ▼                               │
│                    Web Audio Out                        │
└──────────────────────────────────────────────────────────┘
```

### 4.2 Parameter Interpolation (Geodesic)

When the cursor is at position **P** = (I_v, I_h, I_s), the system:

1. **Finds the k-nearest traditions** (k=3 by default, configurable)
2. **Computes geodesic weights** using inverse distance weighting in the 3D space:

   ```
   w_i = 1 / |P - tradition_i|²
   ```

3. **Interpolates ALL synthesis parameters** along geodesic curves between nearest tradition centroids
4. **Applies conservation-aware normalization**: ensures the synthesized parameters respect the Laplacian of nearby traditions (prevents unnatural combinations)

### 4.3 Synthesis Sub-engines

#### Pitch Engine

Generates the tonal/harmonic layer at any dial position:

| Dial Range | How I_vert Maps | Synthesis Method |
|------------|-----------------|------------------|
| 1.5 – 2.0 | Minimal pitch: pentatonic, few pitches | Simple oscillator, pentatonic scale, open harmonies |
| 2.0 – 2.5 | Moderate: diatonic, modal | Diatonic scale generator, functional harmony engine |
| 2.5 – 3.0 | Complex: microtonal, 22-śruti | Microtonal generator, just-intonation ratios, comma bends |
| 3.0 – 4.0 | Extreme: ultra-dense pitch space | Continuous pitch field, granular synthesis across frequency space |

**Key parameters:** Scale degrees (base set + microtonal inflection), tonic frequency, harmony density (chord thickness), voice count, pitch entropy per event

#### Rhythm Engine

Generates the temporal layer:

| Dial Range | How I_horiz Maps | Synthesis Method |
|-------------|-------------------|-------------------|
| 1.0 – 1.8 | Minimal rhythm: free, pulse only | Drone pulse, minimal percussion, no meter |
| 1.8 – 2.5 | Regular meter: 4/4, 3/4, 6/8 | Meter engine with accent patterns, simple subdivisions |
| 2.5 – 3.5 | Complex meter: additive, irregular | Additive meter engine (3+2+3, 7/8, 9/8), polyrhythm generator (3:2, 4:3, 5:4) |
| 3.5 – 4.5 | Extreme rhythm: cross-rhythm, timeline | Polyrhythm superposition, bell pattern generators, phase-shifted cycles |

**Key parameters:** Meter signature, subdivision ratio (binary:ternary), cross-rhythm density, syncopation index, tempo, cycle length

#### Spectral Engine

Generates the timbral layer:

| Dial Range | How I_spectral Maps | Synthesis Method |
|-------------|----------------------|-------------------|
| 0.0 – 1.0 | Pure harmonic timbre | Sine waves, simple additive, clean wave shapes |
| 1.0 – 2.0 | Warm harmonic with formants | Subtractive synthesis, resonant filters, vocal formants |
| 2.0 – 3.0 | Inharmonic partials, beating | FM synthesis, ring modulation, harmonic detuning |
| 3.0 – 4.0 | Extreme spectral: complex partial clouds | Granular synthesis, spectral warping, FM with high modulation index |

**Key parameters:** Harmonicity ratio, spectral centroid, partial count, beating rate, microtonal jitter, inharmonicity coefficient

### 4.4 Conservation-Aware Synthesis

From DIALS-NOT-LAWS §2.2: The conservation pattern holds *locally* for certain transitions (meantone→ET) but fails globally. The synthesis engine should:

1. **Honor local conservation** when interpolating between nearby traditions (I_v + I_h ≈ local constant)
2. **Allow global non-conservation** when moving to distant regions of parameter space
3. **Flag conservation violations** to the user: "This position has +15% more total information than either nearby tradition — you are genuinely expanding the space"

The conservation constraint is a *rubber band*, not a *steel beam*. The further the cursor moves from any tradition, the looser the constraint.

### 4.5 Live Spectral Visualization

```
┌─────────────────────────────────────────┐
│  LIVE SPECTRAL DISPLAY                 │
│                                         │
│  Amplitude                              │
│    │ ╱╲                                 │
│    │╱╱╲╲     ╱╲     ╱╲                  │
│    │╱╱╲╲╱╲╱╲╱╲╱╲╱╲╱╲                  │
│    └───────────────────────────────►    │
│    0Hz                    Nyquist       │
│                                         │
│  Conservation Overlay:                  │
│  ┌───┐ Tradition A (Carnatic)          │
│  ┌──────┐ Tradition B (Turkish)        │
│  ┌───┐ Current (interpolated)          │
│                                         │
│  Spectral centroid: 1.2 kHz             │
│  Inharmonicity: 0.23                    │
│  Beating rate: 3.4 Hz                   │
└─────────────────────────────────────────┘
```

The FFT display updates at ~30 fps, showing:
- **Current spectrum** (white line, filled)
- **Upper/lower envelope** from nearest traditions (ghosted lines)
- **Conservation overlay**: the Laplacian of nearby traditions — does the current position deviate smoothly?
- **Real-time metrics**: centroid, inharmonicity, roughness, beating rate

### 4.6 Audio Smoothing

To prevent clicks/pops during cursor movement:
- **Cross-fade window:** 100ms Hann window when cursor moves > threshold
- **Parameter smoothing:** First-order IIR on all synthesis parameters (τ = 50ms)
- **Phase continuity:** Voice count changes use voice-stealing with retained phases

---

## 5. AI Composition System

### 5.1 Generation Modes

Dial-Space Explorer offers three AI generation modes, each corresponding to a different relationship with the parameter space:

#### Mode 1: Conservation Mode

**Purpose:** Generate music that maximizes continuity with existing traditions. The AI acts as a conservative composer — it stays within known regions of parameter space and produces music that could plausibly belong to an existing tradition.

**How it works:**
1. User selects a dial position **within** or **very near** (< 0.3 distance) an existing cluster
2. System retrieves the k-nearest tradition's training data (melodic patterns, rhythm templates, spectral profiles)
3. Model generates music using a **constrained** diffusion process — the constraint is the Laplacian of nearby traditions
4. Output preserves: scale structure, typical cadences, rhythmic grammar, timbral character

**Use case:** "I want a Carnatic piece but with my melody." Or "Show me what Arabic maqam would sound like if it had slightly more rhythmic complexity."

#### Mode 2: Innovation Mode

**Purpose:** Generate music in **empty regions** of parameter space — genuinely new styles that no tradition has developed. The AI acts as an explorer-composer finding new dial positions that produce coherent structure.

**How it works:**
1. User selects a dial position in a **frontier region** (predicted S > 0, no tradition within 1.0 distance)
2. System generates with **no style constraint** — only the physics of sound and the structure surplus target
3. Synthesis parameters are sampled from a learned distribution over the parameter space, with noise injected proportional to distance from nearest tradition
4. Output is evaluated by the structure surplus filter: generated music must have S > 0 (i.e., more structure than random)

**Use case:** "I want to hear what music at Position C (3.0, 3.0, 3.0) sounds like — the 'maximum everything' region that no human tradition occupies."

#### Mode 3: Rebellion Mode

**Purpose:** Deliberately break conservation and produce music that defies local patterns. The AI acts as a Rule-Breaker — it targets positions where the Laplacian of nearby traditions is maximally violated.

**How it works:**
1. User selects ANY dial position
2. System identifies what the **expected** sound is (based on nearest traditions)
3. Then generates music that **violates** the expectation — inverts the Laplacian gradient
4. If nearby traditions suggest I_vert + I_horiz should be constant, Rebellion Mode makes one spike while the other drops
5. If nearby traditions use harmonic timbre, Rebellion Mode uses inharmonic
6. Structure surplus S can be negative here — the music is deliberately anti-structured

**Use case:** "I want to hear what happens when you take Carnatic's pitch complexity and pair it with Gagaku's rhythmic sparsity at Gagaku's I_spectral." Or "Make something that sounds wrong."

### 5.2 Model Architecture

```
┌──────────────────────────────────────────────────────┐
│           AI SYNTHESIS PIPELINE                      │
│                                                      │
│  Dial Position ──→ ONNX Inference ──→ Audio Output    │
│                                                      │
│  Models (all ONNX):                                  │
│                                                      │
│  1. Position → Parameter Model (MLP)                 │
│     Input: (I_v, I_h, I_s)                           │
│     Output: 128 synthesis parameters                 │
│     Training: Denoising autoencoder on tradition data │
│                                                      │
│  2. Melody Generator (Transformer + Diffusion)        │
│     Input: (parameters, mode, seed)                  │
│     Output: MIDI-like note sequence (pitch, time,     │
│              velocity, microtonal bend)              │
│     Constraint: Structure surplus target             │
│                                                      │
│  3. Timbre Synthesizer (Neural Vocoder)              │
│     Input: (parameters, note sequence)               │
│     Output: Raw audio (44.1kHz)                       │
│     Architecture: HiFi-GAN or EnCodec decoder        │
│                                                      │
│  Fallback (no model):                                │
│  → Rule-based synthesis engine                       │
│    (see §4.3 for detail)                             │
│  → Continuous interpolation between nearest          │
│    tradition's sample sounds                         │
└──────────────────────────────────────────────────────┘
```

### 5.3 The Structure Surplus Filter

From DIALS-NOT-LAWS §4.1:

$$S(I_v, I_h, I_s) = I_{\text{structure}}_{\text{generated}} - I_{\text{structure}}_{\text{random}}$$

The filter:
1. Computes $S$ for generated output in real-time
2. Displays $S$ to user with interpretation:

| S value | Label | Meaning |
|---------|-------|---------|
| > 0.5 | 🟢 **Coherent** | Genuine musical structure detected — this dial position supports a stable style |
| 0.0 – 0.5 | 🟡 **Mildly structured** | Some coherence but weak — might be a transitional style |
| -0.5 – 0.0 | 🟠 **Borderline random** | Barely distinguishable from noise |
| < -0.5 | 🔴 **Random/Noise** | This dial position is musically barren — no stable style exists here |

### 5.4 Generation Controls

```
┌──────────────────────────────────────────────┐
│  AI GENERATION CONTROLS                      │
│                                              │
│  Mode: ● Conservation  ○ Innovation  ○ Rebellion│
│                                              │
│  Parameters:                                 │
│  Duration:  [===|=====] 30s                  │
│  Complexity: [  =|=====] Moderate-High       │
│  Temperament:[===|=====] Stable              │
│                                              │
│  Structure Surplus Target: [===|=====] 0.4   │
│                                              │
│  [🔊 Generate]  [♻️ Re-roll]  [💾 Save]      │
│                                              │
│  Generation ID: dse-20260528-47a3            │
│  Status: Generating... 67%                    │
└──────────────────────────────────────────────┘
```

---

## 6. Research Tools

### 6.1 Tradition Inspector

Click any tradition sphere to open the Inspector panel:

```
┌──────────────────────────────────────────────────┐
│  TRADITION INSPECTOR: CARNATIC                   │
│                                                   │
│  ┌─── Coordinates ───────────────────────┐       │
│  │ I_vert:    2.767  (high — śruti system)│       │
│  │ I_horiz:   3.626  (high — tala system) │       │
│  │ I_spectral: 1.8   (moderate/high est.) │       │
│  │ I_total:   6.393  (highest in dataset) │       │
│  └──────────────────────────────────────────┘       │
│                                                   │
│  ┌─── Spectral Fingerprint ──────────────┐       │
│  │ ╱╲╱╲    ╱╲                             │       │
│  │╱╱╲╲╱╲╱╲╱╲╱╲╱╲╱╲                        │       │
│  │      ╱╱╲╲╱╲                             │       │
│  │    ╱╱╲╲╱╲╱╲                            │       │
│  └──────────────────────────────────────────┘       │
│                                                   │
│  ┌─── Laplacian Eigenvectors ───────────┐         │
│  │ λ₁: 0.87 (92% variance — pitch/rhythm│         │
│  │      coupling)                       │          │
│  │ λ₂: 0.06 (6% — spectral drift)       │          │
│  │ λ₃: 0.03 (2% — noise)                │          │
│  │ Key: Carnatic is essentially 2D in   │          │
│  │ (I_vert, I_horiz) plane              │          │
│  └──────────────────────────────────────────┘       │
│                                                   │
│  ┌─── Conservation Ratios ─────────────┐          │
│  │ I_vert / I_horiz = 0.763             │         │
│  │ I_total / mean = 1.173 (17% above avg)│         │
│  │ Structure surplus S = 0.82            │         │
│  │ Cluster: Maximal (nearest: Hindustani │         │
│  │          0.175 away)                  │         │
│  └──────────────────────────────────────────┘       │
│                                                   │
│  ┌─── Innovation Cycle ─────────────────┐         │
│  │ Current phase: Discovery (Phase 1)   │         │
│  │ Last major innovation: ~1850         │         │
│  │ (Tyagaraja, Muthuswami Dikshitar)    │         │
│  │ Status: Active, still generating new │         │
│  │         compositions within system   │         │
│  └──────────────────────────────────────────┘       │
│                                                   │
│  [▶ Play reference recording]  [↗ Open in new tab] │
└──────────────────────────────────────────────────────┘
```

### 6.2 Innovation Cycle Overlay

Toggle the Innovation Cycle overlay to see WHERE traditions are in their lifecycle:

```
┌─────────────────────────────────────────────────────┐
│  INNOVATION CYCLE OVERLAY (toggle: ON)               │
│                                                      │
│  ● Carnatic    → Discovery (active innovation)      │
│  ● Arabic      → Codification (system being written) │
│  ● Western CP  → Boredom (school adoption: 1850)    │
│  ● Hip-hop     → Boredom (school adoption: 2015)    │
│  ● AI Music    → Rebellion (2023-present)           │
│                                                      │
│  Phase colors in 3D view:                            │
│  🟢 Discovery  🟡 Codification  🔵 Ubiquity         │
│  🟣 Boredom    🔴 Rebellion     ⚪ Restart           │
│                                                      │
│  ┌── Next Innovation Prediction ─────────┐          │
│  │ Likely position: (3.2, 1.5, 3.5)     │           │
│  │ Region: Position A (extreme pitch +   │           │
│  │         spectral, minimal rhythm)     │           │
│  │ Reason: AI can easily occupy this     │           │
│  │         space; humans need sustained  │           │
│  │         practice to get here          │           │
│  └──────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────┘
```

### 6.3 Comparative Analysis

Select two or more traditions to compare:

```
┌──────────────────────────────────────────────────────┐
│  COMPARISON: CARNATIC vs. GAGAKU                     │
│                                                      │
│  Metric            Carnatic  Gagaku   Diff           │
│  ──────────────────────────────────────────          │
│  I_vert             2.77     2.38    +0.39 (16%)     │
│  I_horiz            3.63     1.70    +1.93 (113%)    │
│  I_spectral         1.8      3.8     -2.0  (53%)    │
│  I_total            6.39     4.08    +2.31 (57%)    │
│  Structure S        0.82     0.91    -0.09 (11%)    │
│                                                      │
│  Distance in dial space: 2.79 (vastly different)    │
│                                                      │
│  The difference: Carnatic is "music as system"       │
│  (high everywhere). Gagaku is "music as presence"    │
│  (low pitch+rhythm, high spectral). Their total      │
│  distance is larger than any other pair in our       │
│  dataset.                                            │
│                                                      │
│  [Generate interpolation]  [Save comparison]         │
└──────────────────────────────────────────────────────┘
```

### 6.4 Predictive Mode

From INNOVATION-CYCLE §5.5: Rebellion moves to unexplored dial positions. The predictive mode:

1. Renders a heatmap over the 3D space showing **probability of next innovation**
2. Factors: distance from bored traditions (high), proximity to frontier with S > 0 (high), distance from stable traditions (low)
3. Each occupied tradition emits "push" vectors toward the nearest empty region
4. The intersection of these vectors is the most likely next innovation position
5. Shows the top 3 predicted innovation positions with confidence intervals

---

## 7. Social & Community Features

### 7.1 Save & Share Dial Positions

Any point in dial space can be saved as a **Dial Position**:

```json
{
  "id": "dse-pos-a3f7",
  "name": "Carnatic Fusion Core",
  "creator": "user_42",
  "coordinates": { "I_vert": 2.85, "I_horiz": 3.40, "I_spectral": 2.2 },
  "tags": ["carnatic", "fusion", "maximal"],
  "audio_url": "https://dse.ai/audio/a3f7.mp3",
  "description": "The sweet spot between Carnatic and Hindustani, slightly more spectral.",
  "generation_params": { "mode": "conservation", "duration": 30, "seed": 4242 },
  "structure_surplus": 0.82,
  "saved_at": "2026-05-28T11:00:00Z",
  "plays": 142
}

```

#### Sharing
- **Direct link:** `https://dse.ai/pos/a3f7` — opens the explorer at this exact coordinate
- **Embed:** `<iframe>` widget that plays the dial position's audio
- **Social card:** Auto-generated OG image showing the 3D view with the position highlighted
- **Export:** MIDI, WAV, spectral JSON, dial position JSON

### 7.2 Community Constellations

Users can create **Constellations** — named collections of dial positions that form meaningful patterns in the space:

```json
{
  "id": "constellation-golden-age",
  "name": "Golden Age of Polyphony",
  "creator": "ethnomusicologist_kim",
  "description": "The evolution of Western polyphony from organum (c. 900) through the ars nova (c. 1320) to the high Renaissance (c. 1500). Each point is a composer, each trail is a century.",
  "positions": ["pos-organum", "pos-notre-dame", "pos-ars-antiqua", "pos-ars-nova", "pos-flemish", "pos-renaissance"],
  "color": "#FFD700",
  "is_featured": true
}
```

**Constellation types:**
- **Chronological:** Evolution of a tradition through time (points with timestamps)
- **Geographic:** Map of related traditions from a cultural region
- **Genre trees:** How one genre branched into many (e.g., jazz → bebop, cool, free, fusion)
- **Experimental:** User-created paths blending distant traditions

### 7.3 Expeditions (Guided Tours)

Curated, narrated journeys through dial space:

```
┌──────────────────────────────────────────────┐
│  EXPEDITION: "From Gagaku to West African"   │
│  Curator: @ethnomusicologist_kim             │
│  Duration: 15 min  │ 7 stops                │
│                                              │
│  Stop 1/7: Gagaku (I_v:2.38, I_h:1.70)      │
│  "Listen to the shō — the mouth organ        │
│   clusters. Every partial is intentional.    │
│   Note the silence — ma — between phrases."  │
│                                              │
│  ▶ Playing: Gagaku sample (15s remaining)    │
│  [▶ Next]  [⏸ Pause]  [Exit expedition]      │
└──────────────────────────────────────────────┘
```

**Expedition features:**
- Curator's narration (text or recorded audio)
- Auto-advance through dial positions with smooth audio morphing
- Pauseable — explore the current position before moving on
- Q&A at each stop ("Why does Gagaku have low I_horiz?" → answer + link to research)
- Community rating and comments
- Users can create their own expeditions

### 7.4 Social Feed

```
┌──────────────────────────────────────────────┐
│  DIAL-SPACE FEED                             │
│                                              │
│  🎵 @alice_synth just discovered Position A  │
│     (3.2, 1.5, 3.5) — "Extreme microtonal   │
│      drone with inharmonic shimmer"         │
│     ♫ Listen · ♥ 42 · 💬 7                   │
│                                              │
│  🌌 @bob_theorist created constellation:     │
│     "Tala Systems of the World" (8 positions) │
│     ♫ Explore · ♥ 28 · 💬 3                  │
│                                              │
│  🧭 @carol_guide published expedition:       │
│     "From Blues to Hip-Hop: The Rhythm       │
│      Revolution" (12 stops, 22 min)          │
│     ♫ Start · ♥ 91 · 💬 15                   │
└──────────────────────────────────────────────┘
```

### 7.5 Leaderboards & Challenges

- **Most surprising Position:** Position with the highest structure surplus S at the greatest distance from any tradition (voted weekly)
- **Constellation curator:** Most starred constellation of the month
- **Expedition guide:** Highest-rated expedition
- **Innovation bounty:** "Generate music at Position C that passes the S > 0.5 filter" — community challenges

---

## 8. Architecture

### 8.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      FRONTEND (React)                          │
│                                                                │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ 3D Viewport  │  │ Audio Engine  │  │ Research/Social Panels │ │
│  │ (R3F/Three)  │  │ (Web Audio)   │  │ (React components)    │ │
│  └──────┬───────┘  └───────┬──────┘  └────────────┬───────────┘ │
│         │                  │                       │              │
│         └──────────────────┼───────────────────────┘              │
│                            │                                     │
│                    ┌───────▼──────┐                              │
│                    │ API Client   │                              │
│                    │ (fetch/WS)   │                              │
│                    └───────┬──────┘                              │
└────────────────────────────┼──────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   API Gateway   │
                    │  (nginx/caddy)  │
                    └────────┬────────┘
                             │
┌────────────────────────────┼──────────────────────────────────────┐
│                    ┌───────▼──────┐                              │
│                    │   Axum       │        BACKEND (Rust)        │
│                    │   (REST)     │                              │
│                    └───────┬──────┘                              │
│                            │                                      │
│              ┌─────────────┼─────────────┐                       │
│              ▼             ▼             ▼                       │
│       ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│       │PostgreSQL│  │  Redis   │  │ ONNX     │                   │
│       │(persist) │  │(cache)   │  │Runtime   │                   │
│       └──────────┘  └──────────┘  └──────────┘                   │
│                                            │                     │
│                                     ┌──────▼──────┐              │
│                                     │ WASM Module │              │
│                                     │ (Rust→WASM │              │
│                                     │  spectral   │              │
│                                     │  analysis)  │              │
│                                     └─────────────┘              │
└──────────────────────────────────────────────────────────────────┘
```

### 8.2 Frontend Stack

| Component | Technology | Rationale |
|-----------|-----------|----------|
| Framework | React 19 + Vite | Universal, fast HMR, rich ecosystem |
| 3D Engine | React Three Fiber (Three.js) | Declarative 3D, composable scenes |
| Audio | Web Audio API + WASM | Low-latency, real-time synthesis |
| State | Zustand + Immer | Minimal boilerplate, performant |
| Routing | TanStack Router | Type-safe, parallel loads |
| Styling | Tailwind CSS 4 + Radix UI | Utility-first + accessible components |
| Real-time | WebSocket (via Phoenix Channels) | Live collaboration, sync |

### 8.3 Backend Stack

| Component | Technology | Rationale |
|-----------|-----------|----------|
| API Server | Rust + Axum | Performance, type safety, WASM integration |
| Database | PostgreSQL 17 + pgvector | Relational + vector search for nearest-tradition queries |
| Cache | Redis 7 | Session state, generation queue, position cache |
| ML Runtime | ONNX Runtime (Rust bindings) | Cross-platform, optimized inference |
| File Store | S3-compatible (MinIO) | Audio samples, generated tracks |
| Queue | RabbitMQ | Async generation jobs (long-running AI synthesis) |

### 8.4 WASM Core (Rust → WASM)

The spectral analysis engine runs in-browser as a WASM module:

```rust
// lib.rs — Spectral analysis module compiled to WASM

#[wasm_bindgen]
pub struct SpectralAnalyzer {
    sample_rate: f32,
    fft_size: usize,
}

#[wasm_bindgen]
pub struct SpectralSignature {
    pub centroid: f32,
    pub inharmonicity: f32,
    pub partial_count: u32,
    pub beating_rate: f32,
    pub spectral_flatness: f32,
}

#[wasm_bindgen]
impl SpectralAnalyzer {
    pub fn new(sample_rate: f32, fft_size: usize) -> Self { /* ... */ }
    
    pub fn analyze(&self, buffer: &[f32]) -> SpectralSignature { /* ... */ }
    
    /// Compute structure surplus S: compare mutual information
    /// of generated audio to a random baseline
    pub fn structure_surplus(&self, buffer: &[f32]) -> f32 { /* ... */ }
    
    /// Compute the Laplacian of nearby traditions for
    /// conservation-aware synthesis
    pub fn conservation_laplacian(
        &self,
        position: &[f32; 3],
        traditions: &[[f32; 4]]  // [I_v, I_h, I_s, weight]
    ) -> f32 { /* ... */ }
}
```

### 8.5 API Design

Full API schemas in [§9: Foundation](#9-foundation-data-model--api-schema).

---

## 9. Foundation: Data Model & API Schema

### 9.1 PostgreSQL Schema

```sql
-- Traditions (the 10 seed traditions + user-added)
CREATE TABLE traditions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    slug        TEXT UNIQUE NOT NULL,
    cluster     TEXT NOT NULL CHECK (cluster IN (
                    'maximal', 'rhythmic', 'balanced',
                    'harmonic', 'presence', 'unclassified'
                )),
    i_vert      NUMERIC(5,3) NOT NULL,
    i_horiz     NUMERIC(5,3) NOT NULL,
    i_spectral  NUMERIC(5,3) NOT NULL,
    i_total     NUMERIC(5,3) GENERATED ALWAYS AS (i_vert + i_horiz + i_spectral) STORED,
    rhythm_complexity NUMERIC(3,2),
    description TEXT,
    origin_region TEXT,
    century_emerged INTEGER,
    audio_sample_url TEXT,
    metadata    JSONB DEFAULT '{}',
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- traditions ↔ pgvector embedding for nearest-neighbor search
CREATE TABLE tradition_embeddings (
    tradition_id UUID PRIMARY KEY REFERENCES traditions(id),
    embedding    vector(16) NOT NULL  -- learned embedding from (I_v, I_h, I_s, rhythm_complexity, cluster_one_hot...)
);

CREATE INDEX idx_tradition_embeddings ON tradition_embeddings USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);


-- Dial positions (saved user positions)
CREATE TABLE dial_positions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    creator_id  UUID REFERENCES users(id),
    i_vert      NUMERIC(5,3) NOT NULL,
    i_horiz     NUMERIC(5,3) NOT NULL,
    i_spectral  NUMERIC(5,3) NOT NULL,
    tags        TEXT[] DEFAULT '{}',
    description TEXT,
    structure_surplus NUMERIC(5,3),
    nearest_tradition_id UUID REFERENCES traditions(id),
    nearest_distance NUMERIC(6,3),
    generation_params JSONB DEFAULT '{}',
    audio_url    TEXT,
    play_count   INTEGER DEFAULT 0,
    is_public    BOOLEAN DEFAULT false,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_dial_positions_coords ON dial_positions (i_vert, i_horiz, i_spectral);
CREATE INDEX idx_dial_positions_creator ON dial_positions (creator_id);


-- Constellations
CREATE TABLE constellations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    slug        TEXT UNIQUE NOT NULL,
    creator_id  UUID REFERENCES users(id),
    description TEXT,
    color       TEXT DEFAULT '#FFFFFF',
    is_featured BOOLEAN DEFAULT false,
    star_count  INTEGER DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE constellation_positions (
    constellation_id UUID REFERENCES constellations(id),
    position_id      UUID REFERENCES dial_positions(id),
    sort_order       INTEGER NOT NULL,
    PRIMARY KEY (constellation_id, position_id)
);


-- Expeditions (guided tours)
CREATE TABLE expeditions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title       TEXT NOT NULL,
    creator_id  UUID REFERENCES users(id),
    description TEXT,
    duration_min INTEGER,
    stop_count  INTEGER DEFAULT 0,
    rating_avg  NUMERIC(2,1) DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE expedition_stops (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    expedition_id   UUID REFERENCES expeditions(id),
    position_id     UUID REFERENCES dial_positions(id),
    sort_order      INTEGER NOT NULL,
    narration_text  TEXT,
    narration_audio_url TEXT,
    auto_advance_ms INTEGER DEFAULT 30000
);


-- Generation jobs (for async AI synthesis)
CREATE TABLE generation_jobs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id  UUID REFERENCES users(id),
    status      TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                    'pending', 'processing', 'completed', 'failed'
                )),
    mode        TEXT NOT NULL CHECK (mode IN (
                    'conservation', 'innovation', 'rebellion'
                )),
    i_vert      NUMERIC(5,3) NOT NULL,
    i_horiz     NUMERIC(5,3) NOT NULL,
    i_spectral  NUMERIC(5,3) NOT NULL,
    params      JSONB DEFAULT '{}',
    result_url  TEXT,
    structure_surplus NUMERIC(5,3),
    error       TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

CREATE INDEX idx_generation_jobs_status ON generation_jobs (status, created_at);


-- Social
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        TEXT UNIQUE NOT NULL,
    display_name    TEXT,
    email           TEXT UNIQUE NOT NULL,
    avatar_url      TEXT,
    bio             TEXT,
    role            TEXT DEFAULT 'user' CHECK (role IN ('user', 'curator', 'admin')),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE likes (
    user_id     UUID REFERENCES users(id),
    target_type TEXT NOT NULL CHECK (target_type IN (
                    'position', 'constellation', 'expedition', 'comment'
                )),
    target_id   UUID NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, target_type, target_id)
);

CREATE TABLE comments (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id),
    target_type TEXT NOT NULL,
    target_id   UUID NOT NULL,
    body        TEXT NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### 9.2 REST API Endpoints

```
GET    /api/v1/traditions              # List all traditions (with optional filters)
GET    /api/v1/traditions/:slug        # Get tradition detail (fingerprint, eigenvectors, etc.)
GET    /api/v1/traditions/nearest      # Nearest traditions to (I_v, I_h, I_s)
       ?i_vert=2.77&i_horiz=3.63&i_spectral=1.8&k=3

GET    /api/v1/positions               # List dial positions (public, paginated)
POST   /api/v1/positions               # Save a dial position
GET    /api/v1/positions/:id           # Get dial position detail
DELETE /api/v1/positions/:id           # Delete a dial position

POST   /api/v1/positions/:id/like     # Like a position
POST   /api/v1/positions/:id/comment  # Comment on a position

GET    /api/v1/constellations          # List constellations
POST   /api/v1/constellations          # Create constellation
GET    /api/v1/constellations/:slug    # Get constellation detail

POST   /api/v1/constellations/:slug/like  # Star a constellation

GET    /api/v1/expeditions             # List expeditions
POST   /api/v1/expeditions             # Create expedition
GET    /api/v1/expeditions/:id         # Get expedition with stops

POST   /api/v1/generate                # Submit generation job
       Body: { mode, i_vert, i_horiz, i_spectral, params }
       Response: { job_id, status: "pending" }
GET    /api/v1/generate/:job_id        # Poll generation status
       Response: { job_id, status, result_url, structure_surplus }

GET    /api/v1/explore/frontier        # Get frontier regions (empty areas with predicted S > 0)
GET    /api/v1/explore/predict         # Get next innovation predictions
       Response: [{ position: [f32;3], confidence: f32, rationale: str }]
GET    /api/v1/explore/geodesic        # Get geodesic path between two positions
       ?from=2.77,3.63,1.8&to=2.38,1.70,3.8&steps=10

GET    /api/v1/compare                 # Compare two or more traditions
       ?ids=id1,id2

GET    /api/v1/users/:username         # Get user profile
GET    /api/v1/users/:username/positions  # User's saved positions
GET    /api/v1/users/:username/constellations  # User's constellations
```

### 9.3 WebSocket Events (Real-Time Collaboration)

```
# User joins a collaborative exploration session
{
  "event": "session:join",
  "payload": { "session_id": "sess_abc123", "user_id": "user_42" }
}

# Cursor position broadcast (throttled to ~10 Hz)
{
  "event": "cursor:move",
  "payload": {
    "user_id": "user_42",
    "position": [2.77, 3.63, 1.8],
    "timestamp": 1716890000
  }
}

# Generation started/completed notification
{
  "event": "generation:completed",
  "payload": {
    "job_id": "job_def456",
    "result_url": "https://dse.ai/audio/job_def456.mp3"
  }
}

# Feed event (new position, constellation, expedition)
{
  "event": "feed:new",
  "payload": {
    "type": "position",
    "id": "pos_a3f7",
    "creator": "user_42",
    "name": "Carnatic Fusion Core"
  }
}
```

---

## 10. Wireframes

### 10.1 Main View (Desktop)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  ◀ 🎵 DIAL-SPACE EXPLORER                              [Sign In] [⚙️]  │
├─────────────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────┐ ┌──────────────────────────────┐ │
│ │                                    │ │  📍 POSITION                 │ │
│ │         3D VIEWPORT                │ │                              │ │
│ │                                    │ │  I_vert:    2.65 ◀═══▶     │ │
│ │    ● Carnatic          ● Gagaku    │ │  I_horiz:   3.12 ◀═══▶     │ │
│ │                                    │ │  I_spectral:2.45 ◀═══▶     │ │
│ │    ● West African                  │ │                              │ │
│ │                     ● Chinese      │ │  Nearest: Carnatic (0.61)   │ │
│ │    ● Gamelan                       │ │  S:        0.72 🟢          │ │
│ │                  ● Western         │ │                              │ │
│ │                                    │ │  [🔊 Preview] [💾 Save]     │ │
│ │       ═══  Cursor  ═══            │ │                              │ │
│ │                                    │ │  ────────────────            │ │
│ │      [dark void - unexplored]      │ │  🎛️ GENERATION              │ │
│ │      82% of space empty            │ │  Mode: ○ Conserv ● Innov ○ │ │
│ │                                    │ │         Rebel                │ │
│ │    [Frontier glow: Position A]     │ │  Duration: [==|======] 30s  │ │
│ │                                    │ │  Seed:     [ 42        ]    │ │
│ └────────────────────────────────────┘ │                              │ │
│ ┌────────────────────────────────────┐ │  [▶ Generate]                │ │
│ │  ▶ PREVIEW: Carnatic Fusion Core  │ └──────────────────────────────┘ │
│ │  ════════════════════════════ 0:23 │                                   │
│ │  ◀⏸▶  □                            │                                   │
│ └────────────────────────────────────┘                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  🌌 Discover  🧭 Expeditions  🔬 Research  👥 Community  🚀 Frontier   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Tradition Inspector (Modal/Drawer)

```
┌──────────────────────────────────────────────────────────┐
│  🔍 CARNATIC            [✕ Close]  [↗ Open in new tab]  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌───────────────┐   I_vert:   2.77  ────◉───────────  │
│  │  SPECTRAL     │   I_horiz:  3.63  ──────────◉─────  │
│  │  FINGERPRINT  │   I_spectral:1.80  ────◉─────────── │
│  │               │   I_total:  6.39  (highest of all) │
│  │ ╱╲╱╲    ╱╲   │                                     │
│  │╱╱╲╲╱╲╱╲╱╲╱╲╱╲│  Cluster: Maximal                    │
│  │      ╱╱╲╲╱╲  │  Origin: South India (c. 500 CE)     │
│  └───────────────┘  S: 0.82 🟢                          │
│                                                          │
│  ┌─── Conservation Analysis ───────────────────┐        │
│  │ Carnatic does NOT conserve I_vert + I_horiz  │        │
│  │ (both are high — it compounds complexity).  │        │
│  │ The high-high dial position is possible      │        │
│  │ because śruti + tala systems developed       │        │
│  │ TOGETHER over 2000+ years.                  │        │
│  └──────────────────────────────────────────────┘       │
│                                                          │
│  ┌─── Innovation Cycle ────────────────────────┐        │
│  │ Phase: Discovery (Phase 1)                  │        │
│  │ Last major innovation: 1850 (Tyagaraja)     │        │
│  │ Status: Active innovation continues within  │        │
│  │         the raga system framework           │        │
│  └──────────────────────────────────────────────┘       │
│                                                          │
│  ■ Carnatic                                      [▶▶]  │
│  ■ Hindustani                                    [▶▶]  │
│  ■ Compare both                                  [📊]  │
└──────────────────────────────────────────────────────────┘
```

### 10.3 Mobile View (Simplified)

```
┌─────────────────┐
│  📍 POSITION    │
│  ─────────────  │
│  I_v:  2.7 ◀══▶ │
│  I_h:  3.6 ◀══▶ │
│  I_s:  1.8 ◀══▶ │
│  ─────────────  │
│  Carnatic 0.61  │
│  S: 0.72 🟢     │
│  ─────────────  │
│  [🔊 Preview]   │
│  [💾 Save] [↗]  │
│  ─────────────  │
│  [▶ Generate]   │
├─────────────────┤
│  ════════════   │
│  3D Viewport    │
│  (viewport fills │
│  background     │
│  behind panel)  │
│                 │
│  ● ●● ● ● ● ●  │
│  ═══ Cursor ═══ │
│                 │
├─────────────────┤
│ 🌌 🧭 🔬 👥 🚀│
└─────────────────┘
```

---

## 11. MVP → Full Product Roadmap

### 11.1 Phase 0: Foundation (Weeks 1-2) 🎯 **MVP**

**Goal:** A working 3D space with 10 traditions, cursor navigation, and real-time audio morphing.

| Task | Tech | Effort |
|------|------|--------|
| Set up React + R3F + Vite project | Frontend | 1 day |
| Render 3D space with 10 tradition spheres, labels, axes | R3F | 2 days |
| Implement OrbitControls + DragControls for cursor | Three.js | 2 days |
| Build rule-based synthesis engine (pitch + rhythm + spectral) | Web Audio | 3 days |
| Implement basic geodesic interpolation between traditions | WASM/Rust | 2 days |
| Compute nearest-tradition and display coordinates | Zustand | 1 day |
| Console-like log showing cursor position, nearest tradition, distance | React | 0.5 day |
| Seed database with 10 traditions (JSON → PostgreSQL) | Rust/Axum | 1 day |
| Basic API: tradition list, nearest, single tradition | Axum | 2 days |

**Deliverable:** `dse.phoenix.local:5173` — navigate the 3D space, hear morphing audio, saved positions work.

**Exit criteria:** Can drag cursor from Carnatic to Gagaku and hear smooth audio transition.

### 11.2 Phase 1: Core App (Weeks 3-4)

**Goal:** Full research tools, AI generation, social features.

| Task | Tech | Effort |
|------|------|--------|
| Tradition Inspector panel (fingerprint, eigenvectors, ratios) | React + WASM | 3 days |
| AI generation pipeline (ONNX model serving) | Rust + ONNX | 5 days |
| Conservation/Innovation/Rebellion modes | Backend | 2 days |
| Structure surplus filter | WASM | 2 days |
| Save/load dial positions + sharing links | API + DB | 1 day |
| User authentication (email or OAuth) | Axum + JWT | 2 days |
| Basic social feed | API | 1 day |
| Spectral heatmap overlay | R3F | 1 day |
| Innovation Cycle overlay | R3F | 1 day |
| Geodesic path visualization | R3F | 1 day |
| Audio transport controls (play, pause, scrub, export) | Web Audio | 1 day |

**Deliverable:** Full research app with social features.

**Exit criteria:** A user can discover an empty region, generate AI music there, save it, and share it with a friend.

### 11.3 Phase 2: Community (Months 2-3)

| Task | Effort |
|------|--------|
| Constellations CRUD + visualization in 3D space | 1 week |
| Expeditions (create, narrate, share) | 1 week |
| Collaborative exploration sessions (WebSocket) | 1 week |
| User profiles, follows, notifications | 1 week |
| Leaderboards + weekly challenges | 3 days |
| WebGPU renderer (improved particles, density clouds) | 2 weeks |
| Embeddable widgets (iframe for external sites) | 3 days |
| Mobile-responsive UI | 1 week |
| Internationalization (i18n) | 1 week |

### 11.4 Phase 3: Scale (Months 3-6)

| Task | Effort |
|------|--------|
| Add 50+ traditions from academic research | 2 weeks (research) + 2 weeks (integration) |
| User-contributed traditions (submit new coordinate data) | 1 week |
| Real-time spectral analysis of uploaded audio | 2 weeks |
| Advanced ML: fine-tune ONNX models on new tradition data | 2 weeks |
| Predictive mode v2: ML-based innovation prediction | 2 weeks |
| Community API (read-only for external apps) | 1 week |
| Export: MIDI, WAV, Ableton Live project, Max/MSP patch | 2 weeks |
| WebGPU compute: on-device training of structure surplus models | 3 weeks |

### 11.5 Phase 4: Platform (Months 6-12)

| Task | Effort |
|------|--------|
| **DialSpace SDK** — third-party developers build on the platform | 1 month |
| **DialSpace Studio** — DAW plugin (VST3/AU) with dial controls | 1 month |
| **VR/AR mode** — explore dial space in immersive VR | 1 month |
| **DialSpace Live** — real-time performance mode (MIDI input + dial synthesis) | 1 month |
| **DialSpace API Pro** — commercial API for AI music generation at any dial position | 1 month |
| **Academic partnerships** — ethnomusicology departments contribute tradition data | Ongoing |
| **DialSpace Exchange** — marketplace for AI-generated dial positions | 1 month |
| Hardware: **DialSpace Controller** — physical MIDI controller with 3-axis joystick mapped to (I_v, I_h, I_s) | 2 months |

### 11.6 Risk Register

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Users find the 3D interface confusing | Medium | High | Onboarding tour; tooltips every step; expeditions teach the interface |
| AI generation quality is poor at empty positions | High | Medium | Fallback to rule-based synthesis; show S score so users know what to expect |
| Spectral analysis values for seed traditions are inaccurate | Medium | Medium | Publish as "beta" values; provide tools for researchers to refine; version the data |
| Users share "bad" positions (S ≈ 0) that sound like noise | Low | Low | S score shown prominently; "This region may not support stable music" warning |
| WebGPU not available on all devices | Medium | Low | Graceful fallback to WebGL; core features work on all browsers |
| ONNX model inference is too slow for real-time | Medium | Medium | Offload to server; provide preview with rule-based while server generates |
| Server costs for AI generation are high | High | Medium | Queue system; GPU autoscaling; rate limits for free tier |
| Copyright/training data questions for ML models | Medium | High | Train only on public-domain/CC-licensed data; clear provenance display |

---

## 12. Edge Cases & Innovation

### 12.1 Edge Cases

#### Edge Case 1: Cursor at Origin (1.5, 1.0, 0.0)

**What happens:** Pure silence / monotonic drone. Minimum information in all dimensions.
**System behavior:** Renders a single point at the bottom of the space. Audio output is a sine wave at 220 Hz. Structure surplus S approaches 0. The system explicitly flags: "This is the origin — musical silence. No tradition exists here."
**Why it matters:** Defines the mathematical boundary of the space. Useful for calibration.

#### Edge Case 2: Cursor at Maximum (4.0, 4.5, 4.0)

**What happens:** Maximum everything — theoretical limit.
**System behavior:** Renders at the farthest corner. No tradition is within measurable distance. Audio generation attempts Position C synthesis but the structure surplus filter flags S < 0 for most seeds — this region may be **cognitively barren** as predicted in DIALS-NOT-LAWS §4.4.
**Why it matters:** Tests the upper bound of human musical cognition. If Position C produces S > 0, it means music can exist beyond anything any tradition has developed.

#### Edge Case 3: Cursor Between Traditions (Equal Distance)

Two traditions equally close (e.g., exactly midway between Carnatic and Gagaku). The geodesic interpolation should produce a 50/50 blend.
**System behavior:** Both traditions weighted equally. Interpolation explores smooth style blend. The audio should sound like "Carnatic-infused Gagaku" or "Gagaku-infused Carnatic" — genuinely alien to both traditions.
**Why it matters:** These midpoints are the most interesting unexplored positions — genuinely novel fusions no human composer would naturally produce.

#### Edge Case 4: New Tradition Added Mid-Session

An ethnomusicologist uploads spectral data for a tradition not in our dataset (e.g., Tuvan throat singing, Andean panpipes, Inuit throat singing).
**System behavior:** Real-time re-indexing of nearest-neighbor vectors. The new tradition appears as a glowing point that "lands" in the space. All nearest-tradition computations update. If it falls in a previously empty region, the Innovation Cycle overlay updates: "New tradition discovered in previously empty region — this may be Position A."
**Why it matters:** The space is alive. New data shouldn't require a rebuild.

#### Edge Case 5: Tradition Cluster Collapse

If enough traditions are added and they DON'T follow the cluster structure (e.g., a newly measured tradition falls far from all existing clusters), the system detects a model violation:
**System behavior:** Flags the anomaly: "⚠️ This tradition is 2.4 standard deviations from the nearest cluster centroid. The cluster model may need revision for [culture/region]."
**Why it matters:** The cluster model is a hypothesis, not a law. The system should be self-correcting.

#### Edge Case 6: Generation Job Timeout

AI model inference takes longer than expected (e.g., 30+ seconds for a complex position).
**System behavior:** After 5 seconds, serve a **rule-based preview** (lower quality but instant). When the full generation completes, swap audio seamlessly. Notify user: "Full generation ready — preview was used while waiting."
**Why it matters:** Users should never wait. Preview-first is better than spinner.

### 12.2 Innovations Beyond the Spec

These are features that emerged during design that weren't in the original scope:

#### Innovation 1: The "What If" Engine

A dedicated mode where the system asks "what if?" questions:
- "What if Western music had developed Gamelan's inharmonic spectra?"
- "What if Carnatic had West African rhythmic complexity?"
- "What if Gagaku's ma (silence) was applied to bebop?"
The system then generates music at the hypothetical dial position and displays the "divergence vector" — what changed and why.

#### Innovation 2: Dial-Space Chess

A multiplayer game where two players take turns moving the cursor. Player A sets I_vert, Player B sets I_horiz. The AI generates audio at the resulting position. Players vote on which tradition is closest. Scoring based on accuracy. Educational and competitive.

#### Innovation 3: The Structure Surplus as Musical "Taste"

The S metric can be gamified: "Your generated piece achieved S = 0.82 — 15% above the average for your selected region. You found structure where others found noise." Creates a virtuous cycle of exploration → generation → evaluation.

#### Innovation 4: Dial-Space Sonification

The space itself can be sonified. As the cursor moves, the parameter space produces a continuous drone/ambient soundscape that reflects the local density:
- Near occupied clusters: warm, harmonically rich drone
- In empty regions: sparse, cold, inharmonic
- In frontier regions: shimmering, evolving, hints of structure

This turns the *space itself* into a musical instrument.

#### Innovation 5: Temporal Dial (I_time)

A potential fourth dimension: how information content changes *over time* within a single piece. Some traditions have high internal variation (Carnatic: I_time = 0.7 — lots of dynamic range within a performance). Others are more static (Gagaku: I_time = 0.3 — meditative stasis). Adding I_time creates a 4D hypercube with even more frontier to explore.

#### Innovation 6: Auto-Expeditions from ML

Given a user's saved positions and browsing history, the system generates a personalized expedition: "Based on your interest in the Carnatic-Gamelan boundary, here's a 5-stop tour through the polyrhythmic traditions of the world." The AI becomes a tour guide.

#### Innovation 7: Dial-Space as a Musician's Instrument (Live Mode)

A real-time performance mode where the (I_v, I_h, I_s) joystick is MIDI-mappable. A musician performs by moving through the space while playing their instrument. The AI harmonizes/accompanies based on the current dial position. The performer is the cursor — every move changes the musical landscape.

---

## A. Appendix: The Secret Sauce

### A.1 Why This App Wins

| Factor | What It Does | Why It's Killer |
|--------|-------------|-----------------|
| **The intellectual foundation** | Grounded in real research (10 traditions, 3-axis parameter space, Shannon entropy measurements) | Not a toy — a research tool AND a creative platform |
| **The unexplored 82%** | Empty space is the feature, not a bug | Endless discovery; the app has more content the further you go |
| **The Innovation Cycle** | Not just where traditions are, but WHY they move | Predictive power: "the next innovation will be HERE" |
| **Three AI modes** | Conservation, Innovation, Rebellion | Every user finds their mode; no single right way to use it |
| **It sounds good** | Real-time synthesis at any dial position | Can demo in 5 seconds: drag cursor, hear the change |
| **Social/community** | Constellations, expeditions, sharing | Network effects: more users → more content → more users |
| **It will never be complete** | New traditions added continuously | "The map is never finished" is a feature, not a bug |

### A.2 The One-Sentence Pitch (Short Version)

> *"Google Maps for music — but 82% of the map is undiscovered territory, and you can hear what's there before anyone else."*

### A.3 Why This Can't Be Built Without the Dial Framework

Before DIALS-NOT-LAWS, the parameter space didn't exist. There was no way to say "Carnatic is at (2.77, 3.63, 1.8)" or "Gagaku is at (2.38, 1.70, 3.8)." The conservation law was a dead end — trying to fit everything into a single constraint.

The dial framework unlocks:
1. **Measurable coordinates** for any musical tradition
2. **Geodesic interpolation** between traditions (smooth morphing)
3. **Empty region identification** (82% of the space)
4. **Structure surplus** as a quality metric (is this position musically viable?)
5. **Innovation Cycle phase detection** (is this tradition mature or still evolving?)

Without these, Dial-Space Explorer would be a generic "explore music" app. With them, it's a **scientific instrument** that happens to make beautiful sounds.

---

*The conservation law was the hypothesis. The parameter space is the map. The Innovation Cycle is the story. Dial-Space Explorer is the vehicle.*

**Let's build it.**
