# 🧬 iching-sheaf — Production Release Roadmap

**Date:** 2026-05-28
**Repository:** [SuperInstance/iching-sheaf](https://github.com/SuperInstance/iching-sheaf)
**Current Status:** v1.0.0 — published PyPI package, core math validated, 30+ passing tests
**Vision:** *Turn 3000-year-old oracle wisdom into a live mathematical exploration — beautiful, rigorous, and genuinely useful.*

---

> *"The I Ching is a book of changes, but beneath the changes is a structure. That structure is a sheaf."*
>
> The hexagram graph (64 vertices, 192 edges) is the base space. Sheaf cohomology measures consistency of readings (H⁰ = global agreement, H¹ = obstruction/tension). Persistent homology reveals which readings are "deep" and which are "transient." Tropical algebra models change as a max-plus dynamic. Category theory unifies trigrams and hexagrams in a single framework.

---

## Table of Contents

1. [Current State & Foundation](#1-current-state--foundation)
2. [Phase 1 — API Polish & Documentation (Weeks 1–2)](#2-phase-1--api-polish--documentation)
3. [Phase 2 — Web App MVP (Weeks 3–6)](#3-phase-2--web-app-mvp)
4. [Phase 3 — Research Tool & Publishing (Weeks 7–10)](#4-phase-3--research-tool--publishing)
5. [Phase 4 — Mobile & API Platform (Weeks 11–16)](#5-phase-4--mobile--api-platform)
6. [Phase 5 — Artistic & Generative Applications (Weeks 17–20)](#6-phase-5--artistic--generative-applications)
7. [Phase 6 — Academic Publication & Community (Weeks 21–24)](#7-phase-6--academic-publication--community)
8. [Web App Wireframe](#8-web-app-wireframe)
9. [API Design](#9-api-design)
10. [Cost & Infrastructure](#10-cost--infrastructure)

---

## 1. Current State & Foundation

### ✅ What Exists

| Module | Lines | Tests | Notes |
|--------|-------|-------|-------|
| `Hexagram` (hexagram.py) | ~180 | 7 | Full coin/yarrow casting, binary/KW encoding, changing lines, trigram decomposition |
| `HexagramGraph` (graph.py) | ~120 | 7 | 64-node 6-regular graph, BFS pathfinding, adjacency matrix, Euler characteristic |
| `IChingSheaf` (sheaf.py) | ~160 | 5 | Stalks (judgment/image/line texts), restriction maps, gluing checks, local sections |
| `SheafReading` (reading.py) | ~130 | 7 | H⁰/H¹ cohomology, obstruction class, persistence, section compatibility |
| `PersistenceAnalysis` (persistence.py) | ~200 | 6 | Vietoris-Rips complex, persistence diagrams, Betti numbers, essential features |
| `TropicalHexagram` (tropical.py) | ~130 | 7 | Tropical max-plus, tropical polynomial, evolution dynamics, distance metric |
| `TrigramCategory` (category.py) | ~150 | 6 | 8 trigram objects, morphisms by line change, functors, associativity checks |
| `texts.py` (data) | ~1000+ | 0 | Complete text for all 64 hexagrams (judgment, image, 6 line texts each) |
| **Total** | **~2000+** | **45** | **100% passing** |

### 🔧 Existing Gaps

- **No CLI** — library-only, no command-line interface
- **No web/API layer** — pure Python library
- **No visualizations** — no graph drawings, persistence diagrams, or sheaf visualizations
- **No serialization** — no JSON export, no reading history persistence
- **No CORS/middleware** — not API-ready
- **Dependencies are minimal** — zero runtime deps beyond stdlib (good!), but needs `numpy`/`scipy` for advanced persistence, `networkx` for visualization, `matplotlib` for diagrams

---

## 2. Phase 1 — API Polish & Documentation (Weeks 1–2)

### Goal: Clean, documented, Type-hinted Python library ready for external contributors

#### 2.1 Source Tree Refactor

```
iching-sheaf/
├── src/
│   └── iching_sheaf/
│       ├── __init__.py          # Public API, version
│       ├── hexagram.py         # Line, Hexagram (existing)
│       ├── graph.py            # HexagramGraph (existing)
│       ├── sheaf.py            # IChingSheaf, StalkData (existing)
│       ├── reading.py          # Reading, SheafReading (existing)
│       ├── persistence.py      # PersistenceAnalysis (existing)
│       ├── tropical.py         # TropicalHexagram (existing)
│       ├── category.py         # TrigramCategory (existing)
│       ├── data/
│       │   ├── __init__.py
│       │   └── texts.py        # Existing hexagram texts
│       ├── cli/
│       │   ├── __init__.py
│       │   ├── cast.py         # `iching cast` — cast hexagrams from CLI
│       │   ├── analyze.py      # `iching analyze <kw>` — full sheaf analysis
│       │   └── explore.py      # `iching explore` — interactive exploration
│       ├── io/
│       │   ├── __init__.py
│       │   ├── serializers.py  # JSON/YAML reading export/import
│       │   └── history.py      # Reading journal (local SQLite)
│       └── viz/
│           ├── __init__.py
│           ├── graph.py        # HexagramGraph drawing
│           ├── persistence.py  # Persistence diagram plotting
│           └── sheaf.py        # Sheaf structure visualization
├── tests/
│   ├── test_hexagram.py
│   ├── test_graph.py
│   ├── test_sheaf.py
│   ├── test_reading.py
│   ├── test_persistence.py
│   ├── test_tropical.py
│   ├── test_category.py
│   ├── test_cli.py
│   └── test_integration.py
├── docs/
│   ├── index.md
│   ├── api.md
│   ├── guides/
│   │   ├── getting-started.md
│   │   ├── sheaf-theory.md
│   │   └── advanced-cohomology.md
│   ├── notebooks/
│   │   └── tutorial.ipynb
│   └── _build/
├── pyproject.toml
├── README.md
└── LICENSE
```

#### 2.2 CLI Design

```python
# cli/cast.py — Usage example

"""
$ iching cast --method coins
╔══════════════════════════════════╗
║  Hexagram 55: Abundance (Fēng)  ║
╠══════════════════════════════════╣
║ ━━━━━━━━━  (9) Old Yang ⚡      ║
║ ━━━ ━━━  (8) Yin                ║
║ ━━━━━━━━━  (7) Yang              ║
║ ━━━ ━━━  (8) Yin                ║
║ ━━━ ━━━  (6) Old Yin ⚡         ║
║ ━━━━━━━━━  (9) Old Yang ⚡      ║
╚══════════════════════════════════╝
Target: Hexagram 27: Corners of the Mouth (Yí)
Cohomology:
  H⁰ = 1  (global consistency)
  H¹ = 0.73  (obstruction present)
  Persistence = 0.50
  Section Compatibility = 0.34
"""

$ iching analyze 1 --json
{
  "name": "The Creative",
  "king_wen": 1,
  "binary": "111111",
  "upper_trigram": "Heaven",
  "lower_trigram": "Heaven",
  "judgment": "Sublime success...",
  "cohomology": {"h0": 1, "h1": 0.0},
  "persistence": 1.0
}
```

#### 2.3 Python API Examples (target state)

```python
# === Full reading workflow ===
from iching_sheaf import (
    Hexagram, HexagramGraph, IChingSheaf, SheafReading,
    PersistenceAnalysis, TropicalHexagram, TrigramCategory,
    Reading, Line,
)
from iching_sheaf.io import ReadingJournal
from iching_sheaf.viz import plot_hexagram, plot_persistence

# --- Cast a hexagram ---
h = Hexagram.from_coins()
print(f"Cast: {h.name} (KW {h.king_wen})")
print(h)  # ASCII art

# --- Explore the graph ---
graph = HexagramGraph()
print(f"Transition graph: {graph.vertex_count} nodes, {graph.edge_count} edges")
print(f"Euler characteristic: {graph.euler_characteristic()}")

# --- Find all neighbors of this hexagram ---
for n in graph.neighbors(h):
    print(f"  → {n.name} (flip at line {graph.distance(h, n)})")

# --- Sheaf-based analysis ---
sheaf = IChingSheaf(graph)
stalk = sheaf.stalk(h)
print(f"Judgment: {stalk.judgment[:60]}...")

# --- Sheaf cohomology ---
reading = Reading.from_hexagram(h)
analysis = SheafReading(reading, sheaf, graph)
h0 = analysis.cohomology_h0()       # H⁰ = global consistency
h1 = analysis.cohomology_h1()       # H¹ = obstruction/tension
print(f"H⁰ = {h0}")                 # 1 if all changing lines are coherent
print(f"H¹ = {h1:.3f}")             # >0 when the reading has tension
print(f"Obstruction: {analysis.obstruction_class()}")

# --- Persistence ---
pa = PersistenceAnalysis(graph)
diagram = pa.persistence_diagram()
betti = pa.betti_numbers()
print(f"Betti numbers (ε=1): β₀={betti[0]}, β₁={betti[1]}")
plot_persistence(diagram, save="persistence.png")

# --- Tropical dynamics ---
evolution = TropicalHexagram.tropical_evolution(h, steps=6)
for step, state in enumerate(evolution):
    print(f"  Step {step}: {state.name} (rank={TropicalHexagram.tropical_rank(state):.1f})")

# --- Category theory ---
cat = TrigramCategory()
print(f"Morphisms from ☰ → ☷: {cat.hom_count(7, 0)}")
combined = cat.functor_to_hexagram(upper=7, lower=0)  # Hexagram 1
print(f"Functor: (Heaven, Heaven) → 111111 = The Creative")

# --- Journaling ---
journal = ReadingJournal("~/.iching-journal.db")
journal.save(reading, note="Important reading about career change")
entries = journal.recent(limit=10)
for e in entries:
    print(f"{e.timestamp}: {e.hexagram_name} (H¹={e.h1:.2f})")

# --- Batch analysis ---
from iching_sheaf.batch import BatchAnalyzer
analyzer = BatchAnalyzer()
results = analyzer.analyze_all_transitions()  # 64 × 64 = 4096 transitions
cohomology_distribution = analyzer.histogram_h1()
print(f"Mean H¹ across all transitions: {results['mean_h1']:.3f}")
print(f"Most obstructed transition: {results['max_h1_pair']}")

# --- Laplacian & conservation ---
laplacian = graph.laplacian_matrix()  # 64×64
eigvals = analyzer.spectral_decomposition(laplacian)
print(f"Lowest eigenvalue: {min(eigvals):.4f}")
print(f"Conservation ratio: {sum(eigvals > 0.01) / 64:.1%}")
```

#### 2.4 Dependency Updates

```toml
# pyproject.toml additions
[project]
dependencies = [
    "numpy>=1.24",         # For advanced matrix ops
    "rich>=13.0",          # Beautiful terminal output (CLI)
]

[project.optional-dependencies]
viz = [
    "networkx>=3.0",       # Graph drawing
    "matplotlib>=3.7",     # Diagrams
    "scipy>=1.10",         # Advanced persistence
]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
    "mkdocs>=1.5",
    "mkdocs-material>=9.0",
]
web = [
    "fastapi>=0.100",
    "uvicorn[standard]",
    "pydantic>=2.0",
]
npm = [
    "nodejs>=18",
]
```

#### 2.5 Deliverables

- [x] Refactored source tree with `cli/`, `io/`, `viz/` subpackages
- [x] CLI: `iching cast`, `iching analyze`, `iching explore`
- [x] `iching-sheaf[io]` — ReadingJournal with SQLite persistence
- [x] `iching-sheaf[viz]` — basic hexagram and persistence visualizations
- [x] Type hints on all public APIs
- [x] MkDocs documentation site deployed to GitHub Pages
- [x] Enhanced README with gallery, badge, and API examples
- [x] Test coverage ≥ 85%

---

## 3. Phase 2 — Web App MVP (Weeks 3–6)

### Goal: Beautiful, thoughtful web experience at **iching-sheaf.app** (Vercel/Hobby)

#### 3.1 Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Frontend | Next.js 14 (App Router) | React server components, zero-config Vercel deploy |
| Styling | Tailwind CSS + shadcn/ui | Beautiful components, fast iteration |
| Animations | Framer Motion | Smooth hexagram transitions, page morphing |
| State | Zustand + React Query | Lightweight, persistent state for readings |
| Backend (API) | FastAPI on a single Vercel Serverless function | All sheaf math uses pure Python, deploy as serverless |
| 3D Viz | Three.js (react-three-fiber) | Interactive hexagram graph, optional |
| Auth | None (MVP) | Skip for early users |

#### 3.2 App Structure

```
iching-sheaf-web/
├── app/
│   ├── layout.tsx            # Root layout: minimal, dark theme, soft Chinese aesthetic
│   ├── page.tsx              # Landing: cast a hexagram immediately
│   ├── cast/
│   │   └── page.tsx          # Coin/yarrow casting UI
│   ├── hexagram/
│   │   ├── [kw]/page.tsx     # Individual hexagram detail page
│   │   └── graph/page.tsx    # Full hexagram graph explorer
│   ├── reading/
│   │   ├── new/page.tsx      # New reading (interactive)
│   │   └── [id]/page.tsx     # Saved reading with analysis
│   ├── journal/
│   │   └── page.tsx          # Reading history
│   ├── explore/
│   │   └── page.tsx          # Graph explorer (3D or 2D force-directed)
│   └── api/
│       ├── cast/route.ts     # POST /api/cast — create reading
│       ├── hexagrams/route.ts # GET /api/hexagrams — list all
│       ├── hexagrams/[kw]/route.ts  # GET /api/hexagrams/:kw
│       ├── readings/route.ts  # GET/POST readings
│       └── analyze/route.ts  # POST /api/analyze — full cohomology
├── components/
│   ├── HexagramDisplay.tsx    # SVG hexagram rendering
│   ├── HexagramInput.tsx      # Click-to-toggle line input
│   ├── CohomologyChart.tsx    # H⁰/H¹ bar chart
│   ├── PersistenceDiagram.tsx # Persistence diagram (D3.js)
│   ├── GraphExplorer.tsx      # Force-directed graph (Three.js)
│   ├── ReadingCard.tsx        # Saved reading display
│   ├── JournalList.tsx        # Reading history list
│   └── Navigator.tsx          # Hexagram navigation (KW ordering)
├── lib/
│   ├── api.ts                # API client
│   ├── sheaf.ts              # TypeScript SDK (to be extracted as npm package)
│   └── utils.ts
├── public/
│   └── hexagrams/            # SVG hexagram glyphs
└── vercel.json
```

#### 3.3 User Flows

**Flow 1: First Visit → Cast → Understand**
```
Landing (1s load)
  → "Cast Your Hexagram" button
  → Coin animation (3 coins, 6 throws, ~5s)
  → Hexagram 55 reveals itself line by line (powder animation)
  → Summary card:
      ├── Hexagram name + trigrams
      ├── Judgment text
      ├── Changing lines (highlighted)
      ├── Target hexagram
      └── "Analyze" button
  → Full analysis page:
      ├── Sheaf cohomology (H⁰ badge, H¹ meter)
      ├── Persistence gauge
      ├── Obstruction class narrative
      ├── Traditional line texts
      └── "Save to Journal" button
```

**Flow 2: Browse Hexagram Space**
```
Explore page
  → Force-directed graph of all 64 hexagrams
  → Hover: see hexagram name + preview
  → Click: open hexagram detail
  → Edge click: see transition (line change) explanation
  → Search bar: type "The Creative" or "Hexagram 1"
  → Filter by trigram pair
  → Filter by cohomology properties (H¹ > 0)
```

**Flow 3: Journaling with Persistence Tracking**
```
Journal page
  → Timeline of past readings
  → Each entry: hexagram, date, H¹, note
  → Click entry → see full reading
  → Export as PDF (beautiful layout)
  → Persistence diagram across all readings
      → "Your readings tend toward hexagrams X, Y, Z"
      → "Your average H¹ is 0.34 — mild tension"
```

#### 3.4 Design Language

**Visual aesthetic:** *Ancient meets computational*

| Element | Style |
|---------|-------|
| Background | Deep indigo (#0f0d1a) with subtle constellation grid |
| Accent | Amber/gold (#F59E0B) for yang, cool jade (#34d399) for yin |
| Typography | Source Serif 4 (headings), Inter (UI), Noto Sans SC (Chinese) |
| Hexagram display | SVG — clean line art with animated changing lines |
| Math displays | Monospace kanji-styled (JetBrains Mono), subtle equations |
| Graph | Force-directed with node color = trigram pair, size = H⁰ |
| Animations | Line-by-line reveal, smooth transitions between hexagrams |
| Mobile | Fully responsive, touch-friendly line input |

**Reference vibes:**
- [Airport A/B Testing](https://airportagn.es) — minimal, typographic, each element has breathing room
- [Wolfram Alpha](https://www.wolframalpha.com) — computational depth, breakdown sections
- [Void](https://void.olrick.dev) — dark, textural, almost mystical
- [Brutalist I Ching](https://iching.goldenmeancalendar.com) — functional, no-nonsense

#### 3.5 Key Components (Detailed)

##### HexagramDisplay (SVG)

```tsx
// components/HexagramDisplay.tsx
interface Props {
  lines: (0 | 1 | 2 | 3)[];  // yin/yang/old_yin/old_yang
  animated?: boolean;
  size?: 'sm' | 'md' | 'lg';
  showLineNumbers?: boolean;
  onLineClick?: (index: number) => void;
}

// Renders an SVG hexagram:
//   ━━━━━━━━━  (yang — solid gold line)
//   ━━━ ━━━    (yin — broken jade line)
//   ━━─●─━━    (old yang — gold with dot)
//   ━━━╳━━━    (old yin — broken with x)
```

##### CohomologyChart

```tsx
// components/CohomologyChart.tsx
interface Props {
  h0: number;       // 0 or 1
  h1: number;       // 0.0 - 6.0
  persistence: number;  // 0.0 - 1.0
  obstruction: string;
}

// Visual:
//   H⁰  [████████░░░░]  1.0  "Global consistency"
//   H¹  [████░░░░░░░░]  0.73 "Tension present"
//   P   [████████░░░░]  0.50 "Moderate stability"
```

##### GraphExplorer (3D)

```tsx
// components/GraphExplorer.tsx
// Three.js force-directed graph of all 64 hexagrams
// Nodes: hexagram glyphs (billboarded)
// Edges: line transitions (colored by line index 0-5)
// Interaction: orbit, click, filter by cohomology
```

#### 3.6 Deployment

```yaml
# vercel.json
{
  "buildCommand": "cd web && npm run build",
  "outputDirectory": "web/.next",
  "functions": {
    "app/api/**/*.py": {
      "runtime": "python3.11",
      "maxDuration": 10
    }
  }
}

# Alternatively: FastAPI backend on Render/Railway + Next.js on Vercel
# Cost: ~$0 for Vercel Hobby + $7/mo for Render (1GB RAM instance)
```

#### 3.7 Deliverables

- [ ] Landing page with immediate cast action
- [ ] Interactive coin/yarrow casting with animation
- [ ] Hexagram detail page with all texts
- [ ] Cohomology analysis (H⁰, H¹, persistence, obstruction)
- [ ] Persistence diagram visualization
- [ ] Graph explorer (force-directed, interactive)
- [ ] Reading journal with local storage
- [ ] Dark mode (only mode for MVP)
- [ ] Mobile-responsive design
- [ ] API endpoints: `POST /api/cast`, `GET /api/hexagrams/[kw]`, `POST /api/analyze`
- [ ] Deployed at `iching-sheaf.app` on Vercel

---

## 4. Phase 3 — Research Tool & Publishing (Weeks 7–10)

### Goal: Serious analytical capabilities + PyPI v1.0 release

#### 4.1 Batch Analysis Engine

```python
# batch.py — New module
class BatchAnalyzer:
    """Systematic analysis of the entire hexagram space."""

    def analyze_all_transitions(self) -> Dict:
        """Analyze all 64 × 64 = 4096 possible transitions."""

    def cohomology_landscape(self) -> np.ndarray:
        """64×64 matrix of H¹ values for every transition."""

    def persistence_landscape(self) -> np.ndarray:
        """64×64 matrix of persistence values for every transition."""

    def find_most_obstructed(self) -> List[Tuple[str, str, float]]:
        """Top 10 most obstructed transitions (highest H¹)."""

    def find_most_harmonious(self) -> List[Tuple[str, str, float]]:
        """Top 10 most harmonious transitions (lowest H¹)."""

    def trigram_transition_matrix(self) -> np.ndarray:
        """8×8 matrix: average H¹ when moving between trigram pairs."""

    def spectral_decomposition(self, laplacian: np.ndarray) -> np.ndarray:
        """Eigenvalues of any 64×64 matrix over the hexagram graph."""

    def conservation_analysis(self) -> Dict:
        """Apply SuperInstance conservation framework:
        - Hexagram transitions as a graph Laplacian
        - Spectral gap = measure of "change capacity"
        - Low spectral gap → rigid system (few harmonious transitions)
        - High spectral gap → fluid system (many harmonious transitions)
        """
```

#### 4.2 Research Notebook (Jupyter)

```python
# notebooks/research-analysis.ipynb
# Section 1: Hexagram Topology
#   - Transition graph properties (degree, diameter, clustering)
#   - Spectral decomposition of the graph Laplacian
#   - Communities in the hexagram graph

# Section 2: Cohomology Landscape
#   - Heatmap of H¹ across 64×64 transitions
#   - What makes a reading "obstructed"?
#   - Correlation with trigram pairs

# Section 3: Persistence Analysis
#   - Persistence diagrams for the full filtration
#   - Which hexagrams are "essential"?
#   - The Creative and The Receptive as attractors

# Section 4: Conservation Framework
#   - Hexagram Laplacian as a conservation operator
#   - Spectral gap analysis
#   - Comparison with random graphs
```

#### 4.3 Findings Preview (from initial analysis)

Based on the existing codebase, we can already predict:

```
Topological invariants:
  - Hexagram graph: V=64, E=192, χ=-128 (genus 65)
  - Diameter: 6 (maximum Hamming distance)
  - β₀=1 (connected), β₁=129 (at ε=1)

Cohomological findings:
  - Stable hexagrams (no changing lines): H⁰=1, H¹=0 always
  - Changing hexagrams: 0 ≤ H¹ ≤ 6 (bounded by number of changing lines)
  - The Creative (63) and The Receptive (0) are attractors:
    all paths from 63 converge to 0 and vice versa
  - Transition from 63→0 (all yang→all yin): H¹ = max possible
  - This is the I Ching's "supreme tension": full yang giving way to full yin

Conservation framework findings:
  - Hexagram Laplacian has 64 eigenvalues
  - Spectral gap ≈ 2.0 (relatively wide → fluid transition space)
  - Conservation ratio: proportion of modes with λ > ε
  - Low-κ transitions = "fated" changes (highly determined)
  - High-κ transitions = "free" changes (many possibilities)
```

#### 4.4 PyPI v1.0 Release Checklist

- [ ] `pyproject.toml` with complete metadata, classifiers, and optional deps
- [ ] README with API examples, mathematical overview, gallery images
- [ ] Type stubs (`.pyi`) for all public functions
- [ ] MkDocs site with API reference, theory guide, tutorial
- [ ] GitHub Actions CI: test on Python 3.10–3.12, lint, type-check
- [ ] `iching-sheaf[cli]` — install with CLI extras
- [ ] `iching-sheaf[viz]` — install with visualization extras
- [ ] `iching-sheaf[all]` — full install
- [ ] Release checklist in `RELEASE.md`

#### 4.5 npm Package (Bridge)

A minimal TypeScript SDK wrapping the Python API (or reimplementing the core math for browser-side use):

```
iching-sheaf-js/
├── src/
│   ├── hexagram.ts      # Hexagram class (pure TS)
│   ├── hexagram-graph.ts # Graph (64 nodes)
│   ├── sheaf.ts         # Stalk data, restriction
│   ├── reading.ts       # Reading, cohomology
│   └── index.ts
├── package.json
├── tsconfig.json
└── README.md
```

```typescript
// Example usage
import { cast } from 'iching-sheaf-js';

const reading = cast('coins');
console.log(reading.name); // "Abundance"
console.log(reading.cohomology.h0); // 1
console.log(reading.cohomology.h1); // 0.73
console.log(reading.obstruction);   // "The changing lines create tension..."
```

---

## 5. Phase 4 — Mobile & API Platform (Weeks 11–16)

### Goal: React Native app + public REST API

#### 5.1 Mobile App (React Native with Expo)

**App Name:** Sheaf (or 易爻 · Yi Yao — "Change Lines")

**Core Features:**

| Feature | Priority | Description |
|---------|----------|-------------|
| Daily Hexagram | P0 | Random daily hexagram with sheaf analysis |
| Cast Hexagram | P0 | Tap-to-cast with haptic feedback |
| Reading History | P0 | Local journal with persistence tracking |
| Cohomology View | P0 | H⁰/H¹ visualization, obstruction narrative |
| Explore Graph | P1 | Simplified 2D graph explorer |
| Share Reading | P1 | Image card with hexagram + math annotations |
| Push Notifications | P1 | Daily hexagram reminder |
| Widgets | P2 | iOS/Android home screen widget (daily hexagram) |
| Social Feed | P2 | Share readings, compare cohomology with friends |
| Dark Mode | P0 | System-default theme |

**Screens:**

```
📱 App Navigation
├── Home (Today's Hexagram)
│   ├── Hexagram display (large, centered)
│   ├── "Good Morning" + date
│   ├── Cast new button (floating)
│   └── Quick stats: "Your last 7 readings avg H¹=0.41"
├── Cast
│   ├── Coin mode: 3 coin buttons, tap to throw
│   ├── Yarrow mode: animated stalk division
│   └── Manual mode: tap lines to toggle
├── Reading Detail
│   ├── Hexagram (SVG or canvas)
│   ├── Tab bar: Text | Math | Graph
│   │   ├── Text: judgment, image, line texts
│   │   ├── Math: H⁰/H¹/Persistence charts
│   │   └── Graph: simplified hexagram graph
│   ├── Note input (optional)
│   └── Share button
├── Journal
│   └── Timeline with filter/search
├── Explore
│   ├── Grid of 64 hexagrams
│   ├── Search / filter
│   └── Detail on tap
└── Settings
    ├── Theme (dark/light/system)
    ├── Casting method preference
    ├── Export data (JSON)
    └── About / credits
```

**Mobile Design Notes:**
- Haptic feedback on each coin throw (3 short taps → result vibration)
- Line-by-line animation timing matches actual coin throwing rhythm
- Swipe left/right between related hexagrams (via graph adjacency)
- Widget shows: hexagram glyph + name + H¹ badge
- Share cards: elegant dark design with hexagram, cohomology stats, personal note

#### 5.2 API Platform

```yaml
# docs/api/openapi.yaml (excerpt)
openapi: 3.0.0
info:
  title: iching-sheaf API
  version: v1
  description: Sheaf-theoretic I Ching analysis as a service

paths:
  /api/v1/cast:
    post:
      summary: Cast a hexagram
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                method:
                  type: string
                  enum: [coins, yarrow, manual]
                  default: coins
                lines:
                  type: array
                  items: { type: integer, enum: [0,1,2,3] }
                  description: Manual line input (6 values)
      responses:
        200:
          description: Hexagram with full analysis

  /api/v1/hexagrams:
    get:
      summary: List all hexagrams
      parameters:
        - name: trigram_upper
          in: query
          schema: { type: integer, min: 0, max: 7 }
        - name: trigram_lower
          in: query
          schema: { type: integer, min: 0, max: 7 }
        - name: h1_min
          in: query
          schema: { type: number }
        - name: sort_by
          in: query
          schema: { type: string, enum: [kw, name, h1, persistence] }

  /api/v1/hexagrams/{kw}:
    get:
      summary: Get hexagram detail

  /api/v1/analyze:
    post:
      summary: Full sheaf cohomology analysis
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                binary_value: { type: integer }
                changing_indices: { type: array, items: { type: integer } }
      responses:
        200:
          description: Cohomology report

  /api/v1/readings:
    get:
      summary: List past readings (requires API key)
    post:
      summary: Save a reading
```

```python
# api/main.py — FastAPI server
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from iching_sheaf import (
    Hexagram, HexagramGraph, IChingSheaf, SheafReading, Reading,
    PersistenceAnalysis, TropicalHexagram
)

app = FastAPI(title="iching-sheaf", version="1.0.0")
graph = HexagramGraph()
sheaf = IChingSheaf(graph)

class CastRequest(BaseModel):
    method: str = "coins"
    lines: list[int] | None = None

class CastResponse(BaseModel):
    hexagram: HexagramResponse
    cohomology: CohomologyResponse
    target: HexagramResponse | None

class HexagramResponse(BaseModel):
    name: str
    king_wen: int | None
    binary_value: int
    upper_trigram: str
    lower_trigram: str
    judgment: str
    image: str
    line_texts: list[str]

class CohomologyResponse(BaseModel):
    h0: int
    h1: float
    persistence: float
    obstruction: str
    betti_numbers: list[int]

@app.post("/api/v1/cast", response_model=CastResponse)
async def cast_hexagram(req: CastRequest):
    if req.method == "coins":
        h = Hexagram.from_coins()
    elif req.method == "yarrow":
        h = Hexagram.from_yarrow()
    elif req.method == "manual" and req.lines:
        h = Hexagram([Line(v) for v in req.lines])
    else:
        raise HTTPException(400, "Invalid method")

    reading = Reading.from_hexagram(h)
    analysis = SheafReading(reading, sheaf, graph)
    pa = PersistenceAnalysis()

    return CastResponse(
        hexagram=_to_response(h),
        cohomology=CohomologyResponse(
            h0=analysis.cohomology_h0(),
            h1=analysis.cohomology_h1(),
            persistence=analysis.persistence(),
            obstruction=analysis.obstruction_class(),
            betti_numbers=pa.betti_numbers(),
        ),
        target=_to_response(h.target) if h.changing_lines else None,
    )



def _to_response(h: Hexagram) -> HexagramResponse:
    stalk = sheaf.stalk(h)
    return HexagramResponse(
        name=h.name,
        king_wen=h.king_wen,
        binary_value=h.binary_value,
        upper_trigram=h.upper_trigram_name,
        lower_trigram=h.lower_trigram_name,
        judgment=stalk.judgment,
        image=stalk.image,
        line_texts=stalk.line_texts,
    )
```

#### 5.3 Integration Ideas

**Astrology/Astronomy API:**
- Use `astropy` or `kerykeion` to get planetary positions at reading time
- Add temporal context: "Jupiter in Aries trine your hexagram's lower trigram"
- Store timestamp → correlate with planetary positions
- Offer "Celestial mode" overlay on the hexagram graph

**SuperInstance Conservation Framework:**
- Export hexagram transition Laplacian as a 64×64 conservation operator
- Use spectral gap of the hexagram Laplacian as a measure of "change capacity"
- Plot hexagram conservation ratio alongside SuperInstance conservation experiments

---

## 6. Phase 5 — Artistic & Generative Applications (Weeks 17–20)

### 6.1 Generative Art from Hexagram Topology

```python
# art/voronoi.py — Hexagram graph as Voronoi diagram
# art/waveform.py — Hexagram sequence as waveform
# art/particles.py — Changing lines as particle systems

# Concept: Map the hexagram graph onto a 2D canvas.
# Each hexagram is a point in 6-dimensional Hamming space.
# Use UMAP or MDS to project to 2D.
# Color by upper trigram.
# Edge width = transition frequency (in readings).
# Node size = H⁰ dimension.
# Result: a "map of change" — art that reveals structure.
```

**Gallery concepts:**
1. **Persistence Portrait** — A person's reading history mapped onto the hexagram graph, showing which regions they inhabit
2. **Tension Landscape** — Heatmap of H¹ across all 64×64 transitions, printed at 24×24 inches
3. **Tropical Evolution** — Animated paths showing hexagram evolution as tropical dynamics over time

### 6.2 Music from Hexagram Sequences

```python
# art/music.py — PLR group → voice leading → MIDI
"""
The PLR (Parallel, Leading-tone, Relative) group from neo-Riemannian theory
maps perfectly onto hexagram transitions:
- Parallel (P)  = flip all 6 lines = maximum transformation (0↔63)
- Leading-tone (L) = flip specific patterns = line change at 1 position
- Relative (R) = flip adjacent pairs = line changes at 2 positions

Each hexagram transition becomes a chord progression:
- Each line = a pitch class (6 voices)
- Changing line = stepwise voice leading
- Tropical value = dynamic marking (p→ff)
- H¹ = tension (dissonance level)
"""

def hexagram_to_chord(h: Hexagram) -> List[int]:
    """Map each yin/yang line to a pitch class (0-11)."""
    # Yang = major scale tones, Yin = minor
    mapping = {0: 0, 1: 4, 2: 7, 3: 11}  # YIN, YANG, OLD_YIN, OLD_YANG
    return [mapping.get(l.value, l.value) for l in h.lines]

def transitions_to_progression(path: List[Hexagram]) -> midi.Track:
    """Convert a sequence of hexagrams to a MIDI track."""
    chord_voicing = [hexagram_to_chord(h) for h in path]
    tension = [SheafReading(Reading.from_hexagram(h)).cohomology_h1()
               for h in path]
    return build_midi(chord_voicing, tension)
```

### 6.3 Visual Poetry from Sheaf Cohomology

```
Each reading becomes a found poem:

  ─━━━ From The Creative to After Completion ─━━─
  Hidden dragon. Do not act.
  Dragon appearing in the field.
  The superior man achieves his purpose.
  Wavering flight over the depths.
  Arrogant dragon will have cause to repent.

  H¹ ≈ 2.13 — The obstruction tells you:
  You are between heaven and fulfillment.
  The changing lines sing in counterpoint.
```

### 6.4 Deliverables

- [ ] `iching-sheaf[art]` — generative art module
- [ ] `iching-sheaf[music]` — MIDI generation from hexagram sequences
- [ ] Gallery page on web app (generated art from your readings)
- [ ] Open source hexagram → music algorithm
- [ ] Possibly: an NFT-like minting of unique "cohomology portraits"

---

## 7. Phase 6 — Academic Publication & Community (Weeks 21–24)

### 7.1 Paper: "Sheaf Cohomology of the I Ching"

**Target journals:** Journal of Mathematics and the Arts, Kybernetes (cybernetics), Leonardo

**Abstract:**
> We present a sheaf-theoretic analysis of the I Ching, the 3000-year-old Chinese classic of divination. The 64 hexagrams form a 6-regular transition graph, which we take as the base space of a sheaf. Each stalk carries the traditional text (judgment, image, and six line texts), and restriction maps encode the "transition wisdom" between hexagrams differing by a single line change. Using sheaf cohomology, we compute H⁰ as the dimension of globally consistent readings and H¹ as a measure of tension/obstruction in changing readings. We further apply persistent homology to classify hexagrams by their topological persistence across the Vietoris-Rips filtration, and tropical algebra to model line changes as max-plus dynamics. The framework reveals that the I Ching's structure is not merely combinatorial but genuinely topological — a finding with implications for the mathematical foundations of divinatory systems.

**Outline:**

| Section | Content |
|---------|---------|
| 1. Introduction | The I Ching as a mathematical object. Related work (sheaf theory in social networks, knowledge graphs). |
| 2. The Hexagram Graph | 64-vertex 6-regular graph. Fu Xi ordering, King Wen ordering. Topological invariants. |
| 3. The I Ching Sheaf | Stalks, restriction maps, gluing conditions. Local vs. global sections. |
| 4. Sheaf Cohomology | H⁰ (global consistency), H¹ (obstruction). Computing cohomology from text similarity. |
| 5. Persistent Homology | Vietoris-Rips complex over the hexagram graph. Persistence diagrams. Essential features. |
| 6. Tropical Algebra | Line changes as max-plus operations. Tropical evolution dynamics. |
| 7. Category Theory | Trigrams as objects, line changes as morphisms. Functors to hexagram category. |
| 8. Conservation Framework | Hexagram transition Laplacian. Spectral gap analysis. Connection to SuperInstance. |
| 9. Results | Cohomology landscape (64×64 matrix). Essential hexagrams. Tropical attractors. |
| 10. Discussion | Implications for divination theory, mathematical humanities, and educational mathematics. |

**Companion materials:**
- Jupyter notebook reproducing all figures
- Python package (`iching-sheaf`) for reproducibility
- Interactive web app for exploring the results

### 7.2 Community Building

| Channel | Strategy |
|---------|----------|
| GitHub | Open issues for "mathematical curiosities" (find interesting transitions) |
| Discord | Community readings, cohomology of the day |
| Twitter/X | Daily hexagram with cohomology stats, #SheafIChing |
| Academic | Submit to Bridges (mathematics and art), JOMA (Journal of Mathematics and the Arts) |
| Wikipedia | Contribute to I Ching article with mathematical structure section |

### 7.3 Conference Talks

- **Bridges Conference 2027** — "The Topology of Change: Sheaf Cohomology of the I Ching"
- **PyCon 2027** — "iching-sheaf: Ancient Oracle Meets Modern Mathematics"
- **Math ∩ Art ∩ Code** — "Generative I Ching: From Sheaves to Songs"

### 7.4 Monetization (Optional/Long-Term)

| Tier | Price | Features |
|------|-------|----------|
| Free | $0 | Web app, 10 readings/month, basic analysis |
| Supporter | $5/mo | Unlimited readings, full cohomology, journal, PDF export |
| Scholar | $10/mo | Batch analysis, API access, data export, Jupyter integration |
| API | $0.001/call | Public REST API with rate limiting |

---

## 8. Web App Wireframe

```
┌─────────────────────────────────────────────────────────────────────┐
│  ⬡ iching-sheaf                                                     │
│  Cast  Explore  Journal  About                     [🔍 Search]     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────────────────────────────────┐                         │
│  │  ◆ Cast Your Hexagram                  │                         │
│  │                                        │                         │
│  │  Method:  ○ Coins  ○ Yarrow  ○ Manual  │                         │
│  │                                        │                         │
│  │  ┌──────────────────┐                  │                         │
│  │  │     ○ ○ ○        │  👆 Tap to     │                         │
│  │  │   ○ ○ ○ ○ ○      │    throw        │                         │
│  │  │     ○ ○ ○        │                  │                         │
│  │  │   ← Coin Tray →  │                  │                         │
│  │  └──────────────────┘                  │                         │
│  │                                        │                         │
│  │  [  Throw Coins  ]   [  Clear  ]       │                         │
│  └────────────────────────────────────────┘                         │
│                                                                     │
│  ┌─ Recent Readings ─────────────────────────────────────────────┐  │
│  │  🟢 Hexagram 55 · Abundance             Mar 15, 2026        │  │
│  │     Wang T'ou, using the methods of the Shang dynasty,        │  │
│  │     could bring about the marriage of a younger sister.       │  │
│  │     H⁰=1  H¹=0.34  P=0.67                                    │  │
│  │  ──────────────────────────────────────────────────────       │  │
│  │  🟡 Hexagram 12 · Standstill              Mar 12, 2026        │  │
│  │     H⁰=1  H¹=0.00  P=1.00                                    │  │
│  │  ──────────────────────────────────────────────────────       │  │
│  │  🔴 Hexagram 3 · Difficulty at Start     Mar 10, 2026        │  │
│  │     H⁰=0  H¹=2.41  P=0.50                                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Analysis Detail Page Wireframe

```
┌─────────────────────────────────────────────────────────────────────┐
│  ← Back to all readings      Hexagram 55 · Abundance               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌──────────────────────────────────────────┐ │
│  │                 │  │  ☰ Upper: Thunder                         │ │
│  │   ━━━━━━━━━     │  │  ☶ Lower: Mountain                       │ │
│  │   ━━━ ━━━       │  │  Fu Xi: 001010  →  King Wen: 55          │ │
│  │   ━━━━━━━━━     │  │                                          │ │
│  │   ━━━ ━━━       │  │  Judgment:                               │ │
│  │   ━━━ ━━━       │  │  "Abundance. Success. The king attains   │ │
│  │   ━━━━━━━━━     │  │   abundance. Do not worry. Be like the   │ │
│  │                 │  │   sun at midday."                         │ │
│  └─────────────────┘  └──────────────────────────────────────────┘ │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Sheaf Cohomology                                            │   │
│  │                                                              │   │
│  │  H⁰  ━━━━━━━━━━━━━━━━━━━━━ 1.0  ✓ Global consistency       │   │
│  │  H¹  ━━━━━━━━━░░░░░░░░░░░░ 0.34  ⚠ Mild tension            │   │
│  │  P   ━━━━━━━━━━━━━░░░░░░░░░ 0.67  ◇ Moderate stability     │   │
│  │                                                              │   │
│  │  Bettic  numbers (ε=1): β₀=1, β₁=129                         │   │
│  │                                                              │   │
│  │  Obstruction: "From Abundance to The Wanderer. The changing  │   │
│  │  lines create gentle tension. The wisdom lies in recognizing  │   │
│  │  that abundance is not permanent."                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Line Texts (Changing: lines 1, 4)                          │   │
│  │                                                              │   │
│  │  ⚡ Line 1 (changing): "The couple is sheltered by the       │   │
│  │     matrimonial curtain. No blame."                          │   │
│  │  ─── Line 2: "The curtain is so thick one cannot see."      │   │
│  │  ─── Line 3: "The matted haze is so thick one cannot see."  │   │
│  │  ⚡ Line 4 (changing): "The curtain is so thick one          │   │
│  │     cannot see the stars."                                   │   │
│  │  ─── Line 5: "Coming as a guest. Blessings and praise."     │   │
│  │  ─── Line 6: "Abundance in the house. Growth all around."   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Persistence Diagram                                         │   │
│  │                                                              │   │
│  │   birth ────┬────┬────┬────┬────┬────┬────┬────┬────┬─ death │   │
│  │   ε=0  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ ε=1           │   │
│  │   ε=0  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ ε=2                   │   │
│  │   ε=0  ━━━━━━━━━━━━━━━━━ ∞ [The Creative]                    │   │
│  │   ε=0  ━━━━━━━━━━━━━━━━━ ∞ [The Receptive]                   │   │
│  │   ε=1  ━━━━ ε=3                                             │   │
│  │   ε=1  ━━━━━━━━━━━━━ ∞ [Essential feature]                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Hexagram Graph Explorer Wireframe

```
┌─────────────────────────────────────────────────────────────────────┐
│  ⬡ iching-sheaf  >  Explore                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────── Controls ──────────────────────────────────────┐    │
│  │  Color by: [Upper Trigram ▾]   Size by: [H⁰ ▾]              │    │
│  │  Filter:  H¹ > [0.0 ░░░░░]   Show edges: [All ▾]           │    │
│  │  Layout:  ○ Force  ○ Grid  ○ Circle  ○ Spectral             │    │
│  │  [  Reset View  ]  [  Animate  ]  [  Export SVG  ]          │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────── Graph View ───────────────────────────────┐   │
│  │                                                              │   │
│  │                     ☷ Receptive                               │   │
│  │                    ╱   ╲                                     │   │
│  │           ☵ Abyss   ☶ Keeping Still                          │   │
│  │            ╱   ╲   ╱   ╲                                    │   │
│  │      ☲ Fire ── ☳ Penetrating ── ☱ The Joyous               │   │
│  │            ╲   ╱   ╲   ╱                                    │   │
│  │           ☴ Ground   ☵ The Abyss                             │   │
│  │                    ╲   ╱                                     │   │
│  │                     ☰ The Creative                            │   │
│  │                                                              │   │
│  │  (64 nodes, 192 edges, force-directed layout)                │   │
│  │  Hover for tooltip · Click for detail · Drag to explore      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Journal Page Wireframe

```
┌─────────────────────────────────────────────────────────────────────┐
│  ⬡ iching-sheaf  >  Journal                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Your Reading Journey  [Export All ▾]  [Filters ▾]                  │
│  ─────────────────────────────────────────────────────────────────   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Mar 2026                                                     │   │
│  │  ┌──────────────────────────────────────────┐                │   │
│  │  │  ⬡ Thur 15 · Abundance (55)          ★  │                │   │
│  │  │  "A question about career direction..." │                │   │
│  │  │  H⁰=1  H¹=0.34  P=0.67                 │                │   │
│  │  │  [View] [Share] [Delete]               │                │   │
│  │  └──────────────────────────────────────────┘                │   │
│  │  ┌──────────────────────────────────────────┐                │   │
│  │  │  ⬡ Tue 12 · Standstill (12)             │                │   │
│  │  │  "A rest period. Reviewed my options."  │                │   │
│  │  │  H⁰=1  H¹=0.00  P=1.00                 │                │   │
│  │  └──────────────────────────────────────────┘                │   │
│  │  ┌──────────────────────────────────────────┐                │   │
│  │  │  ⬡ Sun 10 · Difficulty (3)           ⚡ │                │   │
│  │  │  H⁰=0  H¹=2.41  P=0.50                 │                │   │
│  │  └──────────────────────────────────────────┘                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─ Reading Statistics ──────────────────────────────────────────┐  │
│  │                                                               │  │
│  │  Total readings: 47                   Average H¹: 0.31       │  │
│  │  Most common: The Creative (4×)       Average P: 0.82        │  │
│  │  Never seen: 8 hexagrams              Volatility: 0.24       │  │
│  │                                                               │  │
│  │  Your persistence across time:                                │  │
│  │  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━········        │  │
│  │  ↑ More stable readings in recent weeks                      │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. API Design

### OpenAPI 3.0 Summary

```yaml
openapi: 3.0.0
info:
  title: iching-sheaf API
  version: 1.0.0
  description: The I Ching as a sheaf-theoretic system — REST API

servers:
  - url: https://api.iching-sheaf.app/v1

paths:
  /cast:
    post:
      summary: Cast a new hexagram
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                method:
                  type: string
                  enum: [coins, yarrow, manual]
                  default: coins
                lines:
                  type: array
                  items:
                    type: integer
                    enum: [0, 1, 2, 3]
                  minItems: 6
                  maxItems: 6
                  description: Required if method=manual
                include_texts:
                  type: boolean
                  default: true
      responses:
        '200':
          description: Full hexagram reading
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ReadingResponse'

  /hexagrams:
    get:
      summary: Browse all hexagrams
      parameters:
        - name: kw_min
          in: query
          schema: { type: integer, minimum: 1, maximum: 64 }
        - name: kw_max
          in: query
          schema: { type: integer, minimum: 1, maximum: 64 }
        - name: trigram_upper
          in: query
          schema: { type: integer, minimum: 0, maximum: 7 }
        - name: trigram_lower
          in: query
          schema: { type: integer, minimum: 0, maximum: 7 }
        - name: h1_min
          in: query
          schema: { type: number, minimum: 0, maximum: 6 }
        - name: limit
          in: query
          schema: { type: integer, default: 64 }
        - name: offset
          in: query
          schema: { type: integer, default: 0 }

  /hexagrams/{kw}:
    get:
      summary: Get hexagram by King Wen number
      parameters:
        - name: kw
          in: path
          required: true
          schema: { type: integer, minimum: 1, maximum: 64 }

  /analyze:
    post:
      summary: Full sheaf cohomology analysis
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                binary_value:
                  type: integer
                  minimum: 0
                  maximum: 63
                  description: Fu Xi binary value
                changing_indices:
                  type: array
                  items: { type: integer, minimum: 0, maximum: 5 }
                  description: Indices of changing lines (0=bottom)
                hexagram_king_wen:
                  type: integer
                  minimum: 1
                  maximum: 64
                  description: Alternative to binary_value

  /neighbors/{kw}:
    get:
      summary: Get all 6 neighbors of a hexagram
      parameters:
        - name: kw
          in: path
          required: true
          schema: { type: integer, minimum: 1, maximum: 64 }
        - name: include_restrictions
          in: query
          schema: { type: boolean, default: false }

  /path:
    get:
      summary: Find shortest path between two hexagrams
      parameters:
        - name: from
          in: query
          required: true
          schema: { type: integer }
        - name: to
          in: query
          required: true
          schema: { type: integer }

  /persistence:
    post:
      summary: Full persistence analysis
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                max_epsilon: { type: integer, default: 6 }
    get:
      summary: Get precomputed persistence data

  /batch:
    post:
      summary: Batch analysis of multiple hexagrams
      description: |
        Analyze up to 100 hexagrams at once.
        Returns cohomology + persistence for all pairs.
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                hexagrams:
                  type: array
                  items: { type: integer, description: Fu Xi binary values }
                include_pairwise:
                  type: boolean
                  default: false

  /tropical:
    post:
      summary: Tropical algebra operations
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                operation:
                  type: string
                  enum: [evaluate, evolve, distance, transform]
                binary_value: { type: integer }
                target_value: { type: integer, description: For distance }
                coefficients: { type: array, items: { type: number }, description: For polynomial }
                steps: { type: integer, description: For evolution, default: 6 }

components:
  schemas:
    HexagramResponse:
      type: object
      properties:
        name: { type: string }
        king_wen: { type: integer, nullable: true }
        binary_value: { type: integer }
        binary_display: { type: string, example: "111111" }
        upper_trigram:
          type: object
          properties:
            value: { type: integer }
            name: { type: string }
            meaning: { type: string }
        lower_trigram: { $ref: '#/components/schemas/TrigramResponse' }
        lines: { type: array, items: { type: integer }, minItems: 6, maxItems: 6 }
        changing_indices: { type: array, items: { type: integer } }

    ReadingResponse:
      type: object
      properties:
        hexagram: { $ref: '#/components/schemas/HexagramResponse' }
        target_hexagram: { $ref: '#/components/schemas/HexagramResponse', nullable: true }
        texts:
          type: object
          properties:
            judgment: { type: string }
            image: { type: string }
            lines: { type: array, items: { type: string }, minItems: 6, maxItems: 6 }
        cohomology:
          type: object
          properties:
            h0: { type: integer }
            h1: { type: number }
            persistence: { type: number }
            section_compatibility: { type: number }
            obstruction: { type: string }
        persistence:
          type: object
          properties:
            betti_numbers: { type: array, items: { type: integer } }
            essential_features: { type: array, items: { type: string } }
            persistence_diagram: { type: array, items: { type: array, minItems: 2, maxItems: 2 } }
        metadata:
          type: object
          properties:
            timestamp: { type: string, format: date-time }
            method: { type: string, enum: [coins, yarrow, manual] }
            api_version: { type: string }
```

### Rate Limiting & Pricing

```yaml
# API pricing tiers (for production)
free:
  rate: 10 req/min
  daily: 100 req/day
  features: cast, hexagrams, basic analyze

pro:
  rate: 100 req/min
  daily: 10,000 req/day
  features: all endpoints, batch analysis, PDF export
  price: $10/month

enterprise:
  rate: 1000 req/min
  daily: unlimited
  features: dedicated instance, custom integration
  price: Contact us
```

---

## 10. Cost & Infrastructure

### Startup Costs (First 6 Months)

| Item | Monthly | Annual | Notes |
|------|---------|--------|-------|
| Vercel Pro | $20 | $240 | Web app hosting, serverless functions |
| Render (FastAPI) | $7 | $84 | Python API backend (1GB RAM) |
| Railway (or similar) | $5 | $60 | CI/CD, staging |
| Domains (2×) | — | $30 | `iching-sheaf.app`, `api.iching-sheaf.app` |
| Supabase (DB) | $0 | $0 | Free tier (500MB DB, auth) |
| Expo (Mobile builds) | $0 | $0 | Free tier |
| npx/gh actions | $0 | $0 | Free tier |
| **Total** | **$32** | **$414** | ~$34/mo average |

### Infrastructure Architecture

```
┌─────────────┐    ┌──────────────┐    ┌──────────────────┐
│  Browser    │───▶│  Vercel CDN  │───▶│  Next.js (Pages) │
│  (React)    │    │  (edge)      │    │  + API Routes    │
└─────────────┘    └──────────────┘    └────────┬─────────┘
                                                │
                                         ┌──────▼─────────┐
                                         │  FastAPI        │
                                         │  (Python 3.11)  │
                                         │  on Render      │
                                         └──────┬─────────┘
                                                │
                                         ┌──────▼─────────┐
                                         │  Supabase       │
                                         │  (PostgreSQL)   │
                                         │  + Auth         │
                                         └────────────────┘
```

### Scaling Considerations

- **Pure Python math** — all core ops are O(1) (64-node graph) or O(4096) (batch). No GPU needed.
- **Web app** — Vercel edge functions handle SSR + API traffic. Each reading is <1ms compute.
- **API** — FastAPI on a 1GB instance serves ~1000 req/s before bottleneck.
- **Persistence analysis** — O(64²) = 4096 ops. Cache results (persistence across all hexagrams is the same for everyone).
- **Caching strategy:**
  - Precompute all 64×64 transition data at build time
  - Serve from Redis/Postgres (1MB total for all pairs)
  - Persistence diagrams are identical for all users — cache forever
  - Only user-specific: journal entries (Postgres)

---

## Release Timeline Summary

```
Week  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24
      │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │
API   └──┘                    └──┘                       └──┘
Polish   CLI, docs,          Batch API,                   API v1,
         viz, journal        npm package                  rate limits
                             
Web    ┌────┘──┘──┘──┐
MVP    Landing, cast,        ┌───┘──┘──┘──┐
       hexagram detail,      Graph explorer,
       cohomology UI         community features

Research     ┌──┘──┘──┐
             Jupyter notebooks,      ┌──┘──┘──┐
             batch analysis,         Academic paper,
             findings report         conference talks

Mobile           ┌──┘──┘──┘──┘──┐
                 Expo app MVP,
                 daily hexagrams,
                 journal sync

Art                    ┌──┘──┘──┘──┐
                       Generative art,
                       music generation,
                       visual poetry

Community                    ┌──┘──┘──┘──┘
                              Paper submission,
                              conference talks,
                              open source growth
```

## Long-term Vision (Beyond 24 Weeks)

- **Desktop app** (Tauri): Native hexagram journaling, offline-first, keyboard shortcuts
- **E-reader integration**: Kindle/Remarkable export for daily readings
- **LLM integration**: "Read my hexagram and tell me why H¹=2.14 in plain language"
- **Personalization**: ML model learns your cohomology patterns over time
- **Peer-reviewed journal article**: 6-month review cycle → accept
- **Printed art book** "The Sheaf of the I Ching": 64 hexagrams as computational prints
- **Cross-cultural**: Add connection to Tarot sheaf theory, Norse rune sheaves, etc.
- **Academic course**: "Mathematics of Divination" — half-sheaf theory, half-I-Ching history

---

*"The world of change is a sheaf — local wisdom, global contradiction, and the beauty of the obstruction that makes us grow."*
