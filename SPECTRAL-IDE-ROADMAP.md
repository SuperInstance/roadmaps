# 🧬 Spectral IDE / Editor Plugin — Roadmap

**Killer App:** An editor extension (VS Code + Neovim) that gives developers **conservation awareness** while they code. Built on the moo-spectral analysis pipeline: tokenization → transition graph → Laplacian → eigenvectors → conservation → visualization.

---

## Table of Contents

1. [Vision](#vision)
2. [Architecture](#architecture)
3. [The Spectral Pipeline](#the-spectral-pipeline)
4. [TypeScript API Surface](#typescript-api-surface)
5. [UI Wireframe](#ui-wireframe)
6. [Configuration](#configuration)
7. [Roadmap](#roadmap)
8. [Performance Requirements](#performance-requirements)

---

## Vision

Every line of code has a spectral fingerprint. Well-structured code shows **high conservation** — token types transition smoothly along predictable paths. Buggy, obfuscated, or AI-injected code shows **anomalous conservation drops** — regions where the token grammar violates the Laplacian eigenstructure.

**Spectral IDE makes conservation visible in real time.**

- **Green gutter** = code that respects its own spectral grammar
- **Red gutter** = code that violates it (bugs, injection, mismatch)
- **Hover refactors** = spectral correction suggestions
- **Dependency graphs** = conservation across imports

This is moo-spectral turned from a lexer enhancement into a **universal code quality lens**.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     VS Code Extension                         │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────────┐ │
│  │ Spectral      │  │ Spectral      │  │ Dependency        │ │
│  │ Gutter        │  │ Refactor      │  │ Spectral View     │ │
│  │ Decorations   │  │ Code Actions  │  │ (Webview Panel)   │ │
│  └──────┬───────┘  └──────┬────────┘  └────────┬──────────┘ │
│         │                 │                     │            │
└─────────┼─────────────────┼─────────────────────┼────────────┘
          │                 │                     │
          ▼                 ▼                     ▼
┌──────────────────────────────────────────────────────────────┐
│              LSP Server (rust-spectral-ls)                    │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Token-  │→│ Transition│→│ Laplacian│→│ Conservation  │ │
│  │ izer    │  │ Graph     │  │ Eigensys │  │ Analyzer     │ │
│  └─────────┘  └──────────┘  └──────────┘  └──────┬───────┘ │
│                                                   │         │
│                    ┌──────────────────────────────┘         │
│                    ▼                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ WASM Core (Rust → wasm32-wasi)                      │   │
│  │ • SpectralAnalysisEngine                            │   │
│  │ • EigenvalueSolver (thin-sparse)                    │   │
│  │ • ConservationScore / AnomalyDetection              │   │
│  │ • LanguageFingerprint / LanguageRegistry            │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
          │
          ├─────────────────────────┐
          ▼                         ▼
┌──────────────────┐   ┌──────────────────────┐
│  Neovim Plugin   │   │ VSCode Extension     │
│  (lua/vimscript) │   │ (TypeScript)         │
│  • SpectralGutter │   │ • Same LSP client    │
│  • Diagnostics    │   │ • Plus Webview UIs   │
│  • VirtualText    │   │ • Rich decorations   │
└──────────────────┘   └──────────────────────┘
```

### Key Design Decisions

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Core** | Rust compiled to WASM | Shared across VSCode & Neovim; single audited implementation |
| **LSP** | Custom `rust-spectral-ls` | Extends LSP protocol with spectral-specific capabilities |
| **Gutter** | VSCode `TextEditorDecorationType` / Neovim `nvim_buf_set_extmark` | Native rendering, no canvas hacks |
| **Fingerprinting** | Bytes-based spectral signature | Works on any file regardless of extension |
| **Transport** | JSON-RPC over stdio (LSP) + custom notification | Standard LSP for diagnostics, custom for spectral data |

---

## The Spectral Pipeline

### Phase 1: Tokenization

For any file, the pipeline first needs a token grammar. The editor:

1. **Detects the language** via Language Fingerprinting (Phase 5) or file extension
2. **Loads or infers a token grammar** — for known languages, uses a pre-built moo-spectral grammar; for unknown languages, infers a grammar from repeated character-class patterns
3. **Tokenizes the full buffer** and emits a stream of `(type, range, text)` tokens

### Phase 2: Transition Graph

From the token stream, we build a **weighted directed graph**:

```
Vertices: token types (keyword, identifier, operator, literal, ...)
Edges:    P(type_j | type_i) — transition probability from type_i to type_j
Weights:  transition frequency × character-set overlap (tension)
```

This is the moo-spectral `buildTransitionMatrix()` applied to the running token stream rather than static rule definitions. The resulting matrix W captures the **grammatical flow** of the code.

### Phase 3: Laplacian → Eigensystem

```
L = D - W          (unnormalized Laplacian)
L_norm = D⁻¹² L D⁻¹²   (normalized — handles degree imbalance)

Eigensolve: L_norm × v = λ × v
  → λ₀ = 0 (trivial, constant eigenvector)
  → λ₁ = spectral gap (how connected the grammar is)
  → λ₂..λₖ = higher modes (grammar "harmonics")
  → v₁ = Fiedler vector (optimal rule ordering)
```

Implementation uses a **sparse Lanczos solver** for files >5K tokens, falling back to dense Jacobi (as in moo-spectral) for smaller files.

### Phase 4: Conservation Score

For each token at position `i` in the token stream:

```
projection[i][k] = eigenvector_i[k]   (type i's coordinate in eigenspace)
gradient[k][i]   = projection[i+1][k] - projection[i][k]
variance[k]      = Var(gradient[k]) over sliding window of size W
conservation[i]  = 1 / (mean variance across top-K eigenvectors + ε)
```

**High conservation** = token transitions follow the grammar's natural eigenstructure.
**Low conservation** = abrupt transitions, unusual token sequences, likely errors.

The conservation score maps to a **color gradient** for the gutter:
- Conservation > 0.8 → 🟢 green (coherent)
- Conservation 0.5–0.8 → 🟡 amber (acceptable)
- Conservation < 0.5 → 🔴 red (anomalous)

### Phase 5: Language Fingerprinting

A spectral signature for a language is the **eigenvalue profile** of its rule tension graph:

```typescript
interface LanguageFingerprint {
  name: string;
  eigenvalueProfile: number[];   // first 10 normalized eigenvalues
  spectralGap: number;
  cheegerConstant: number;
  tokenTypeDistribution: Record<string, number>;  // normalized histogram
}
```

To identify a file's language:
1. Tokenize with a generic character-class lexer
2. Compute spectral signature of the resulting token stream
3. Compare against the registry via cosine similarity in eigenvalue space
4. Return best match with confidence score

### Phase 6: Anomaly Detection

A token or region is anomalous when its conservation score deviates significantly from the file's own baseline:

```
delta = conservation_score / file_baseline
anomaly = delta > 2.25  (same threshold as moo-spectral ConservationTracker)
```

Anomaly types:
- **Grammatical anomaly** — token sequence violates the language's grammar
- **Fingerprint mismatch** — region has a different spectral signature than its surrounding file (injection)
- **Conservation drop** — sudden loss of coherence (typo, half-finished refactor)

---

## TypeScript API Surface

```typescript
// === Core Types ================================================

/** A single spectral analysis result for a file or region */
interface SpectralAnalysisResult {
  /** Unique analysis session ID */
  sessionId: string;
  /** URI of the analyzed document */
  documentUri: string;
  /** Language detected (file extension or fingerprint match) */
  detectedLanguage: string;
  /** Confidence in language detection (0–1) */
  languageConfidence: number;
  /** Per-token conservation data */
  tokenResults: TokenConservation[];
  /** Aggregate statistics */
  statistics: SpectralStatistics;
  /** Timestamp of analysis */
  analyzedAt: number;
}

interface TokenConservation {
  /** Token type name (keyword, identifier, operator, ...) */
  type: string;
  /** Conservation score (0 = chaotic, 1 = perfectly coherent) */
  conservation: number;
  /** Deviation from file baseline (1 = normal) */
  delta: number;
  /** Whether this token is flagged as anomalous */
  isAnomaly: boolean;
  /** Anomaly type, if any */
  anomalyType?: 'grammatical' | 'fingerprint-mismatch' | 'conservation-drop';
  /** Projection onto each of the top-K eigenvectors */
  eigenspaceProjection: number[];
  /** Source range in the document */
  range: DocumentRange;
}

interface DocumentRange {
  startLine: number;
  startColumn: number;
  endLine: number;
  endColumn: number;
}

interface SpectralStatistics {
  /** λ₁ = smallest positive eigenvalue */
  spectralGap: number;
  /** Cheeger constant (conductance of the grammar graph) */
  cheegerConstant: number;
  /** First 10 eigenvalues of the Laplacian */
  eigenvalues: number[];
  /** Conservation baseline (mean over whole file) */
  conservationBaseline: number;
  /** Variance of conservation scores across the file */
  conservationVariance: number;
  /** Fraction of tokens flagged as anomalous */
  anomalyRate: number;
  /** Language fingerprint of this file */
  fingerprint: LanguageFingerprint;
}

// === Configuration =============================================

interface SpectralConfig {
  /** Master enable/disable */
  enabled: boolean;

  /** Laplacian type */
  laplacianType: 'unnormalized' | 'normalized';

  /** Kernel width for tension similarity (σ in moo-spectral) */
  sigma: number;

  /** Sliding window size for conservation tracking */
  conservationWindow: number;

  /** Number of eigenvectors to use for conservation scoring */
  conservationComponents: number;

  /** Anomaly threshold (delta above baseline) */
  anomalyThreshold: number;

  /** Conservation score thresholds for gutter coloring */
  colorThresholds: {
    green: number;   // conservation >= this → green
    amber: number;   // amber zone
    red: number;     // conservation < this → red
  };

  /** Languages to analyze (empty = all) */
  includeLanguages: string[];

  /** Languages to skip */
  excludeLanguages: string[];

  /** Maximum file size (bytes) to analyze; skip files larger than this */
  maxFileSize: number;
}

const defaultSpectralConfig: SpectralConfig = {
  enabled: true,
  laplacianType: 'normalized',
  sigma: 1.0,
  conservationWindow: 20,
  conservationComponents: 5,
  anomalyThreshold: 2.25,
  colorThresholds: {
    green: 0.8,
    amber: 0.5,
    red: 0.5
  },
  includeLanguages: [],
  excludeLanguages: [],
  maxFileSize: 5_000_000  // 5MB
};

// === Refactor Suggestions ======================================

interface RefactorSuggestion {
  /** The suggested replacement text */
  replacement: string;
  /** Confidence score (0–1) from spectral projection */
  confidence: number;
  /** Suggested token type */
  suggestedType: string;
  /** Source range to apply the fix */
  range: DocumentRange;
  /** Description of the issue */
  description: string;
  /** The anomalous token type that triggered this suggestion */
  anomalousType: string;
}

// === Dependency Spectral Health ================================

interface DependencyGraphNode {
  /** Path or package name */
  name: string;
  /** Spectral fingerprint of this dependency (computed from its public API) */
  fingerprint: LanguageFingerprint;
  /** Conservation score of API usage in the importing file */
  apiUsageConservation: number;
  /** Children in the dependency tree */
  children: DependencyGraphNode[];
  /** Whether this dependency is well-behaved (high conservation) */
  isWellBehaved: boolean;
}

interface DependencySpectralGraph {
  /** Root node (the current file/package) */
  root: DependencyGraphNode;
  /** All edges as adjacency list */
  edges: Array<{ from: string; to: string; conservation: number }>;
  /** Global dependency health score (0–1) */
  globalConservation: number;
}

// === LSP Protocol Extensions ===================================

// Custom LSP methods (sent as window/logMessage + custom notification)

namespace SpectralLSP {
  /** 'textDocument/spectralAnalysis' — get full analysis for a URI */
  const METHOD_SPECTRAL_ANALYSIS = 'textDocument/spectralAnalysis';
  interface Params {
    textDocument: { uri: string };
  }
  type Result = SpectralAnalysisResult;

  /** 'textDocument/spectralRefactor' — get refactor suggestions at cursor */
  const METHOD_SPECTRAL_REFACTOR = 'textDocument/spectralRefactor';
  interface RefactorParams {
    textDocument: { uri: string };
    position: { line: number; character: number };
  }
  type RefactorResult = RefactorSuggestion[];

  /** 'textDocument/spectralDependencies' — get dependency spectral graph */
  const METHOD_SPECTRAL_DEPENDENCIES = 'textDocument/spectralDependencies';
  interface DepsParams {
    textDocument: { uri: string };
  }
  type DepsResult = DependencySpectralGraph;

  /** 'spectral/conservationUpdate' — server→client push notification */
  const NOTIFICATION_CONSERVATION_UPDATE = 'spectral/conservationUpdate';
  interface ConservationUpdate {
    uri: string;
    changes: Array<{
      range: DocumentRange;
      conservation: number;
    }>;
  }

  /** 'spectral/fingerprintMatch' — server→client language detection */
  const NOTIFICATION_FINGERPRINT_MATCH = 'spectral/fingerprintMatch';
  interface FingerprintMatchNotification {
    uri: string;
    detectedLanguage: string;
    confidence: number;
    candidates: Array<{ language: string; similarity: number }>;
  }
}

// === VS Code Extension API =====================================

// Contributes:
// - commands: spectral.enable, spectral.disable, spectral.showDependencies
// - views: spectralGutter (builtin decoration), spectralDependencies (webview)
// - configuration: spectral.* (maps to SpectralConfig above)

interface SpectralExtension {
  /** Toggle conservation analysis on/off */
  toggle(): void;
  /** Get full spectral analysis for active editor */
  getAnalysis(): SpectralAnalysisResult | null;
  /** Get refactor suggestions at cursor */
  getRefactorSuggestions(): RefactorSuggestion[];
  /** Show dependency spectral graph */
  showDependencyGraph(): void;
  /** Update config (hot-reload) */
  updateConfig(config: Partial<SpectralConfig>): void;
}

// === Neovim Plugin API =========================================

// Provides via Lua module:
// require('spectral').setup(opts)        — initialize
// require('spectral').toggle()          — toggle
// require('spectral').get_analysis()    — get current analysis
// require('spectral').show_deps()       — dependency view

// Autocommands:
// SpectralGutterUpdate — fired when gutter decorations change
// SpectralAnomaly      — fired on new anomaly detection
```

---

## UI Wireframe

```
╔══════════════════════════════════════════════════════════════════╗
║  ● SPECTRAL MODE  [Conservation: 0.82 ████████░░]  [σ: 1.0]  ║
║  ┌────────────────────────────────────────────────────────────┐ ║
║  │function fibonacci(n: number): number {                    │ ║
║  │🟢  if (n <= 1) return n                                   │ ║
║  │🟢  let a = 0, b = 1                                       │ ║
║  │🟡  for (let i = 2; i <= n; i++) {                         │ ║
║  │🟢    const c = a + b                                      │ ║
║  │🟢    a = b                                                 │ ║
║  │🟢    b = c                                                 │ ║
║  │🟢  }                                                       │ ║
║  │🟡  return b                                                │ ║
║  │🔴  } /* ← anomalous: mismatched closing brace? */          │ ║
║  │                                                            │ ║
║  │  /* Below: user types code with typo */                    │ ║
║  │  let x: strng = "hello"                                    │ ║
║  │🔴     ▔▔▔▔▔▔                                               │ ║
║  │  /* conservation drop — "strng" not in type ecosystem */   │ ║
║  │                                                            │ ║
║  │  /* AI-generated region with injection odor */             │ ║
║  │🔴🔴🔴  let password = decrypt(secret)                     │ ║
║  │  /* spectral fingerprint shift — code reads like Python */ │ ║
║  └────────────────────────────────────────────────────────────┘ ║
║                                                                ║
║  ┌─ DEPENDENCY SPECTRAL HEALTH ───────────────────────────────┐ ║
║  │  📦 my-app (conservation: 0.82)                           │ ║
║  │  ├── 📦 express@4.18  🟢 (API conservation: 0.91)        │ ║
║  │  ├── 📦 lodash@4.17  🟢 (API conservation: 0.87)         │ ║
║  │  ├── 📦 sequelize@6  🟡 (API conservation: 0.64)         │ ║
║  │  │    └── 📦 tedious@15 🔴 (API conservation: 0.31)      │ ║
║  │  │         ⚠ Unusual usage pattern detected               │ ║
║  │  └── 📦 left-pad@1.3  🟢 (API conservation: 0.99)        │ ║
║  │                                                            │ ║
║  │  Global dependency conservation: 0.74                      │ ║
║  │  Recommendations:   sequelize → drizzle-orm  (Δ+0.18)     │ ║
║  │                     tedious → mssql         (Δ+0.45)      │ ║
║  └────────────────────────────────────────────────────────────┘ ║
╚══════════════════════════════════════════════════════════════════╝
```

### Gutter Color Semantics

| Color | Conservation | Meaning |
|-------|-------------|---------|
| 🟢 Green | ≥ 0.80 | Normal, well-structured code |
| 🟡 Amber | 0.50–0.80 | Acceptable, minor structural tension |
| 🔴 Red | < 0.50 | Anomalous — likely bug, typo, or injection |
| 🔴🔴🔴 Flashing | Sustained low | Critical — obfuscation or code injection |

### Interaction

- **Hover on gutter marker** → tooltip shows: conservation score, delta from baseline, anomaly type, suggested fix
- **Click on gutter marker** → applies refactor suggestion (if `confidence > 0.7`)
- **Right-click gutter** → "Show spectral detail" opens inline diff of token projection
- **Spectral dependency panel** → auto-opens when file's dependency conservation drops below threshold

---

## Configuration

```typescript
// settings.json (VS Code) / setup({...}) (Neovim)
{
  "spectral.enabled": true,
  "spectral.laplacianType": "normalized",  // "unnormalized" | "normalized"
  "spectral.sigma": 1.0,                   // tension kernel width
  "spectral.conservationWindow": 20,       // sliding window (tokens)
  "spectral.conservationComponents": 5,    // top-K eigenvectors
  "spectral.anomalyThreshold": 2.25,       // delta threshold from baseline
  "spectral.colorThresholds": {
    "green": 0.8,
    "amber": 0.5,
    "red": 0.5
  },
  "spectral.includeLanguages": [
    "typescript", "javascript", "python", "rust", "go", "c", "cpp"
  ],
  "spectral.excludeLanguages": [
    "json", "yaml", "markdown", "css"
  ],
  "spectral.maxFileSize": 5000000,          // 5 MB
  "spectral.showGutter": true,
  "spectral.showDependencyPanel": true,
  "spectral.fingerprintUnknownFiles": true  // detect language even if .txt
}
```

### Tuning Guide

| Parameter | Effect | Recommended |
|-----------|--------|-------------|
| `sigma` | Higher = smoother tension, less sensitive | 0.5–2.0 (default 1.0) |
| `conservationWindow` | Larger = more stable but slower to react | 10–50 (default 20) |
| `conservationComponents` | More components = finer-grained analysis | 3–10 (default 5) |
| `anomalyThreshold` | Higher = fewer false positives | 2.0–3.0 (default 2.25) |

---

## Roadmap

### Phase 0: Foundation (Weeks 1–4)

**Goal:** Core engine, zero plugin UI, prove the performance budget.

- [ ] **Rust core crate `spectral-core`**
  - [ ] Sparse transition-matrix builder from token streams
  - [ ] Lanczos eigensolver for sparse symmetric matrices
  - [ ] Conservation scorer (sliding-window variant)
  - [ ] Language fingerprint computation + registry (initially: 10 languages)
  - [ ] `suggestCorrection()` — spectral-projection nearest-neighbor
  - [ ] WASM build target (`wasm32-wasi`)
- [ ] **Benchmark suite**
  - [ ] Synthetic files from 100 to 100K tokens
  - [ ] Verify <100ms for 100K tokens
  - [ ] Regression test against moo-spectral.js results
- [ ] **LSP server skeleton `spectral-ls`**
  - [ ] stdio JSON-RPC listener
  - [ ] File-change watcher (debounced, 200ms)
  - [ ] Text sync (full buffer on save, incremental on edit)
  - [ ] Basic `textDocument/spectralAnalysis` handler

**Deliverable:** `spectral-ls --stdio` accepts an LSP client, returns spectral analysis on request. No visualization yet.

---

### Phase 1: Spectral Gutter (MVP) (Weeks 5–8)

**Goal:** Working gutter decorations in both editors.

- [ ] **VS Code extension**
  - [ ] `SpectralGutterProvider` — subscribes to LSP conservation updates
  - [ ] `TextEditorDecorationType` per token — green/amber/red
  - [ ] Hover tooltip shows conservation score + delta
  - [ ] Status bar item: "Spectral: 🟢 0.82"
  - [ ] Command `spectral.toggle` — enable/disable
  - [ ] Debounced reanalysis on file edit (300ms idle)
- [ ] **Neovim plugin**
  - [ ] Lua client module `spectral.nvim`
  - [ ] `nvim_buf_set_extmark` for gutter signs (`SpectralGreen`, etc.)
  - [ ] Virtual text summary on current line
  - [ ] `:SpectralToggle` command
- [ ] **Configuration UI**
  - [ ] VS Code settings panel
  - [ ] Neovim `setup()` with defaults

**Deliverable:** Open any supported language file, see gutter colors. Green where code is clean, red where it's suspicious. Already useful for code review.

---

### Phase 2: Refactor Suggestions (v1.0) (Weeks 9–12)

**Goal:** Ctrl+. shows spectral refactor suggestions.

- [ ] **LSP `textDocument/spectralRefactor`**
  - [ ] Returns `RefactorSuggestion[]` at cursor position
  - [ ] Uses ConservationTracker.suggestCorrection() logic
  - [ ] Multi-token suggestions (replace region, not just token)
- [ ] **VS Code Code Actions**
  - [ ] Register `Spectral Fix` code action provider
  - [ ] Lightbulb on anomalous tokens
  - [ ] Preview inline diff
- [ ] **Neovim**
  - [ ] `vim.lsp.buf.code_action()` integration
  - [ ] Virtual text showing suggestion on hover
- [ ] **Special case: AI-generated code smell**
  - [ ] Flag regions whose spectral fingerprint doesn't match the file
  - [ ] Suggested fix: "Refactor region to match file's spectral style"
  - [ ] Threshold: conservation delta > 3.0 over any 10-line window

**Deliverable:** Anomaly detection + one-click fixes. The editor tells you "this line doesn't fit" and offers the most likely correction.

---

### Phase 3: Language Fingerprinting (v1.5) (Weeks 13–14)

**Goal:** Detect language even when the file extension is wrong or missing.

- [ ] **Language registry** — pre-computed fingerprints for ~30 languages
  - [ ] TypeScript, JavaScript, Python, Rust, Go, C, C++, Java, Kotlin
  - [ ] Ruby, PHP, Swift, Scala, Haskell, Elixir, Clojure
  - [ ] Lua, Zig, Nim, OCaml, F#, R, Julia, MATLAB
  - [ ] SQL, GraphQL, Protocol Buffers, TOML, YAML (yes, config langs have fingerprints too)
- [ ] **Fingerprint matching** — cosine similarity in eigenvalue space
  - [ ] Fallback: token-type histogram Jaccard similarity
  - [ ] Confidence threshold: must exceed 0.6 to auto-detect
- [ ] **Only in `SpectralLexer` mode** — requires spectral analysis
  - [ ] If `fingerprintUnknownFiles: true`, run detection on non-standard extensions
  - [ ] If confidence > 0.8, offer "Set language to [detected]?" prompt
- [ ] **UI**
  - [ ] Status bar shows detected language + confidence
  - [ ] Hover tooltip: "Detected: Python (confidence: 0.93)"

**Deliverable:** Open a file with `.tmp` or no extension → Spectral IDE tells you what language it probably is.

---

### Phase 4: Anomaly Detection Hardening (v2.0) (Weeks 15–18)

**Goal:** Detect injection, obfuscation, and code quality erosion.

- [ ] **Injection detection**
  - [ ] Compare spectral fingerprint of each function/block against the file baseline
  - [ ] Flag regions where fingerprint shifts significantly (different eigenvalue profile)
  - [ ] Common patterns: SQL-injected strings in Go, eval()-encased obfuscation
- [ ] **Obfuscation detection**
  - [ ] Minified code: very high conservation (all same token types) → too uniform
  - [ 】Encoded strings: sudden spike in string-type tokens → red flag
  - [ ] Dead code: region with near-zero conservation (random token types)
- [ ] **AI-generated code detection**
  - [ ] Train a binary classifier on: eigenvalue profiles × conservation variance
  - [ ] AI-generated code often has slightly-too-uniform conservation
  - [ ] Heuristic: conservation variance < 0.1 over a function → suspicious
  - [ ] Threshold: user-configurable "AI sensitivity" slider
- [ ] **Gutter upgrade**
  - [ ] Double-wide gutter markers for critical anomalies
  - [ ] Animated (pulsing) on sustained critical
  - [ ] Click to highlight all related anomalous regions

**Deliverable:** Security tooling built into the editor. Conservation becomes a code quality gate.

---

### Phase 5: Dependency Spectral Health (v3.0) (Weeks 19–24)

**Goal:** Multi-file analysis — spectral health of your entire dependency tree.

- [ ] **Dependency graph builder**
  - [ ] Parse import/require statements for the language
  - [ ] Walk node_modules, vendor, or crates
  - [ ] Compute spectral fingerprint for each dependency's public API token stream
  - [ ] Edge weight = API usage conservation (how well the import uses the API)
- [ ] **LSP `textDocument/spectralDependencies`**
  - [ ] Returns `DependencySpectralGraph`
  - [ ] Push updates on dependency changes
- [ ] **VS Code Webview Panel**
  - [ ] Interactive tree view (as shown in UI wireframe)
  - [ ] Color-coded: green = good API conservation, red = bad
  - [ ] Click to open dependency file with its spectral analysis
  - [ ] "Recommendations" section: suggest alternative packages with better conservation
- [ ] **Neovim**
  - [ ] `:SpectralDeps` — opens quickfix list with dependency health
  - [ ] `:SpectralDepsTree` — ASCII tree in a split buffer
- [ ] **Monorepo support**
  - [ ] Workspace-level spectral analysis
  - [ ] Cross-package conservation patterns (common APIs that degrade together)

**Deliverable:** See the conservation health of your entire codebase. Spot the package with bad API patterns pulling down the whole graph.

---

### Phase 6: Ecosystem (Future) (Post v3.0)

- **CI integration** — `spectral-ci` as GitHub Action / GitLab CI job
  - Fail build if conservation drops below threshold in diff
- **Git blame integration** — show conservation trend per author
- **Learning mode** — spectrally analyze commit history to find when conservation began degrading
- **Team dashboards** — aggregate spectral health across repos
- **Language learning plugin** — upload your own codebase, Spectral IDE tells you "these 3 developers write code with the same spectral fingerprint, these 2 are outliers"

---

## Performance Requirements

| Metric | Target | Notes |
|--------|--------|-------|
| **Tokenization** | <10ms for 100K lines | Rust + `memchr`-based scanning |
| **Transition graph** | <20ms for 10K unique token types | Sparse CSR format |
| **Eigensolve (Lanczos)** | <40ms for 1000×1000 sparse | Krylov subspace, 50 iterations max |
| **Conservation scoring** | <10ms for 100K tokens | O(nk) with k=numComponents |
| **Total latency** | <100ms for 100K lines | Sync on open, debounced async while typing |
| **Memory (single file)** | <50MB for 100K lines | Sparse storage, no full matrix copies |
| **Memory (LSP server)** | <200MB baseline | Pooled analysis objects |
| **Startup** | <500ms to first gutter draw | Warm WASM cache, lazy language registry |

### Degradation Strategy

| File Size | Analysis Mode | Gutter Behavior |
|-----------|---------------|-----------------|
| < 1000 lines | Full spectral (all tokens) | Per-token colors |
| 1000 – 10K | Windowed spectral (sliding window, step=1) | Per-line average color |
| 10K – 100K | Windowed spectral (window=20, step=5) | Per-paragraph blocks |
| > 100K | Sampling (every 10th line) | Block-level overview |
| > 1M | Disabled (show info banner) | Message: "File too large" |

When the user is typing, reanalysis is debounced at **300ms**. During idle, full reanalysis runs in the background with priority below keystroke handling.

---

## Implementation Notes

### Shared WASM Core

The Rust core compiles to `wasm32-wasi` and is loaded by both:
- **`spectral-ls`** (Rust host) — via `wasmtime` runtime, ~1μs call overhead
- **Web version** (future) — via browser WASM, same core binary

The WASM boundary passes only flat arrays (no serde overhead):
```rust
// Core interface (wasm exports)
#[no_mangle]
pub extern "C" fn spectral_analyze(
  tokens_ptr: *const Token,
  tokens_len: usize,
  config_ptr: *const AnalysisConfig,
) -> AnalysisResult;

#[repr(C)]
pub struct Token {
  pub type_id: u16,      // token type index
  pub line: u32,         // for range mapping
  pub col: u32,
}

#[repr(C)]
pub struct AnalysisConfig {
  pub sigma: f32,
  pub window_size: u32,
  pub num_components: u32,
  pub laplacian_type: u8,  // 0 = unnormalized, 1 = normalized
  pub anomaly_threshold: f32,
}
```

### LSP Transport

The LSP server sends spectral data as **custom notifications** (`spectral/conservationUpdate`) so the gutter can update without explicit polling. The full analysis is available via request (`textDocument/spectralAnalysis`).

### Comparing to moo-spectral.js

| Feature | moo-spectral.js | Spectral IDE |
|---------|----------------|--------------|
| Domain | Lexer rule ordering | Token stream conservation |
| Source | Static rule definitions | Live file contents |
| Eigensolver | Dense Jacobi (N² operations) | Sparse Lanczos (N^1.5 typical) |
| Language coverage | Single lexer grammar at a time | 30+ languages dynamically |
| Performance target | N/A (setup-time only) | <100ms for 100K lines |
| UI | None | Gutter colors, code actions, panels |
| Dependency analysis | None | Whole dependency graph |

---

## Appendix: Language Fingerprint Registry (Abbreviated)

```
Language      | SpecGap | Cheeger | Eigenvalue Profile (first 5)
──────────────┼─────────┼────────┼────────────────────────────────
TypeScript    |  0.42   |  0.31  | [0, 0.42, 0.67, 0.89, 1.12]
Python        |  0.38   |  0.29  | [0, 0.38, 0.61, 0.83, 1.05]
Rust          |  0.51   |  0.37  | [0, 0.51, 0.78, 0.95, 1.21]
Go            |  0.34   |  0.25  | [0, 0.34, 0.55, 0.72, 0.94]
C             |  0.55   |  0.41  | [0, 0.55,