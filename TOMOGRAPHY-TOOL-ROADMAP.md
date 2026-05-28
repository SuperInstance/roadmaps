# Conservation Tomography Tool — Roadmap

> **Point this at any API, database, or service. It sends carefully crafted probe requests, measures conservation of responses, and reconstructs the internal state graph. See how a black-box system is structured WITHOUT access to its source code.**

---

## The Pitch

You can't see inside the black box. But you can watch how information flows through it — what it preserves, what it destroys, what it mixes. Conservation Tomography exploits the mathematical relationship between a dynamical system's internal structure and the conservation ratios of its outputs. By probing with diverse inputs and measuring conservation, we reverse-engineer the state transition graph.

**One tool. Any black box. Internal structure revealed.**

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   ctom (CLI / SDK)                       │
├─────────────┬──────────────┬──────────────┬─────────────┤
│  Probes     │  Analysis    │  Recon.      │  Reporting  │
│             │              │              │             │
│ REST probe  │ Clustering   │ Spectral     │ HTML viz    │
│ DB probe    │ Conservation │ Laplacian    │ JSON report │
│ ML probe    │ measurement  │ Graph build  │ Diff view   │
│ Net probe   │ Eigenvalue   │ Validation   │ CI/CD       │
│ Compile pr. │ estimation   │ Confidence   │ Export      │
├─────────────┴──────────────┴──────────────┴─────────────┤
│                    Core Engine                           │
│  Spectral estimation │ Graph reconstruction │ Diff engine│
└─────────────────────────────────────────────────────────┘
```

---

## 1. Probing Protocol

The probe pipeline runs in four phases. Each phase feeds the next.

### Phase 1: Enumerate Observable States

**Goal:** Discover the output space of the system.

- Send a diverse corpus of inputs (randomized, fuzzed, hand-crafted, gradient-guided)
- Collect responses; extract features (JSON structure, semantic embeddings, timing, status codes, etc.)
- Cluster similar outputs using hierarchical density clustering (HDBSCAN)
- Each cluster centroid = a candidate observable state
- Estimate state space dimension `n` from the cluster count

**Parameters:**
- `--corpus` — input corpus path or built-in generator name
- `--corpus-size` — number of probe inputs (default: 500)
- `--cluster-method` — hdbscan | kmeans | spectral (default: hdbscan)
- `--min-cluster-size` — minimum cluster membership (default: 3)

### Phase 2: Build Transition Estimates

**Goal:** Discover how the system moves between states.

- Send *sequences* of inputs (ordered probe series) and observe state transitions
- Record `(input_sequence, observed_state_before, observed_state_after)` triples
- Estimate transition counts → empirical transition matrix `P_hat`
- Handle partial observability (hidden states) via HMM smoothing when needed
- Estimate stationary distribution from long probe chains

**Parameters:**
- `--sequence-length` — inputs per sequence (default: 20)
- `--num-sequences` — number of independent chains (default: 100)
- `--warmup` — inputs to discard before recording (default: 5)

### Phase 3: Conservation Measurement

**Goal:** Compute conservation ratios across eigenmodes.

- Define attributes on the output space (binary classifiers on clusters, continuous features, custom attribute functions)
- For each attribute `f`, measure conservation at multiple time scales `t`
- Compute empirical conservation ratio:
  ```
  CR(f, t) = Cov[f(X_0), f(X_t)] / Var[f(X_0)]
  ```
- Fit exponential decay model `CR(f, t) ≈ Σ_k a_k * exp(-λ_k * t)` via Prony/ESPRIT
- Extract eigenvalue spectrum `{λ_0, λ_1, ..., λ_{n-1}}`

**Parameters:**
- `--attributes` — built-in set name or Python module path for custom attributes
- `--time-scales` — list of delay values (default: [1,2,5,10,20,50,100])
- `--spectral-method` — prony | esprit | music | nn (default: esprit)
- `--num-attributes` — number of random attributes to generate (default: 2n)

### Phase 4: Graph Reconstruction

**Goal:** Solve the inverse problem — conservation → Laplacian → graph.

- Run **Iterative Spectral Matching Algorithm** (from §5 of the research proposal):
  1. Initialize random graph on `n` vertices with estimated edge density
  2. Compute candidate Laplacian spectrum
  3. Define spectral loss: `L = Σ(λ_k^candidate - λ_k^target)² + μ * Σ(CR_fit - CR_observed)²`
  4. Perturb edges via gradient `∂λ_k/∂w_uv = φ_k(u) * φ_k(v)` (spectral perturbation theory)
  5. Iterate until convergence (loss < ε) or budget exhausted
- Validate: re-simulate conservation on recovered graph, compare to observations
- Confidence scoring per edge via bootstrap resampling

**Parameters:**
- `--reconstruction-method` — spectral-matching | mcmc | gradient-descent (default: spectral-matching)
- `--max-iterations` — outer loop cap (default: 1000)
- `--tolerance` — convergence threshold ε (default: 1e-6)
- `--restarts` — number of random restarts (default: 10)
- `--validate` — run cross-validation on held-out conservation measurements

---

## 2. Probe Modules

Each target type gets a specialized probe that knows how to speak its language.

### 2.1 REST API Probe

```bash
ctom probe --type rest-api --target https://api.example.com \
  --openapi spec.yaml \           # optional: seed from OpenAPI spec
  --auth "Bearer $TOKEN" \
  --rate-limit 10/s \
  --depth 3
```

**What it does:**
- Enumerates endpoints from OpenAPI spec or via fuzzing
- Sends method+path+body combinations
- Measures response conservation (status codes, body structure, timing, headers)
- Detects hidden endpoints (states that only appear under certain input sequences)
- Tracks session/stateful behavior via cookies, auth tokens, headers

**Output states:** Each unique response signature = a state. Transitions = how responses change across sequential requests.

### 2.2 Database Probe

```bash
ctom probe --type database --target "postgres://host/db" \
  --schema-aware \                # use schema metadata to craft smarter queries
  --query-patterns select,join,aggregate \
  --depth 3
```

**What it does:**
- Sends systematic query patterns (point lookups, range scans, joins, aggregations)
- Measures result conservation (row counts, value distributions, query timing)
- Infers index structure, partition boundaries, join cardinalities
- Detects hidden tables/columns via side-channel (timing anomalies, error messages)

**Output states:** Query result profiles = states. Transitions = how query results change after write operations.

### 2.3 ML Model Probe

```bash
ctom probe --type ml-model --target "https://model.endpoint/predict" \
  --input-dim 768 \
  --perturbation gaussian \
  --depth 4
```

**What it does:**
- Sends input perturbations (Gaussian noise, adversarial deltas, interpolation)
- Measures output conservation (prediction stability, confidence scores, output class)
- Maps the model's decision boundary structure
- Reveals hidden clusters in the model's internal representation

**Output states:** Prediction profiles = states. Transitions = how outputs change across input neighborhood traversals.

### 2.4 Network Protocol Probe

```bash
ctom probe --type network --target 192.168.1.100:8080 \
  --protocol tcp \
  --packet-templates syn,ack,push,custom \
  --depth 3
```

**What it does:**
- Sends crafted packet sequences
- Measures behavioral conservation (response patterns, timing, state machine transitions)
- Reconstructs the protocol's state machine (e.g., TCP FSM, custom protocol FSM)
- Detects undocumented states or transitions

**Output states:** Observable response patterns = states. Transitions = protocol state changes.

### 2.5 Compiler Probe

```bash
ctom probe --type compiler --target "gcc -O2" \
  --source-templates loops,branches,math,memory \
  --depth 3
```

**What it does:**
- Sends code variants (semantically equivalent programs with structural differences)
- Measures output conservation (binary size, execution time, assembly structure)
- Reveals optimization pass boundaries and internal pipeline stages
- Detects optimization bugs (same input semantics → different outputs = non-conservation)

**Output states:** Compilation result profiles = states. Transitions = how compilation behavior changes across code transformations.

---

## 3. Visualization

### 3.1 Reconstructed State Graph

Interactive force-directed or hierarchical layout showing:
- **Nodes** = inferred states (sized by stationary probability, colored by cluster)
- **Edges** = inferred transitions (width = probability, color = conservation ratio)
- **Edge confidence** = opacity proportional to bootstrap confidence
- **Hover** = full state detail (example inputs that reach this state, attribute profile)

```
ctom visualize report.json --format html --layout force
ctom visualize report.json --format svg --layout hierarchical
```

### 3.2 Conservation Heatmap

Matrix view: rows = attributes, columns = time scales, cells = conservation ratio.
- **Smooth regions** (high conservation) = well-behaved dynamics
- **Anomalous regions** (unexpectedly low or high conservation) = structural features of interest
- **Cluster boundaries** visible as sharp conservation gradients

```
ctom visualize report.json --type heatmap --attributes all --time-scales all
```

### 3.3 Comparison View (Diff Mode)

Side-by-side or overlay comparison of two systems:
- **Structural diff:** added/removed states and transitions
- **Spectral diff:** eigenvalue shifts, new/missing modes
- **Conservation diff:** attributes whose conservation changed
- **Spectral distance:** `d_spec(G1, G2) = sqrt(Σ(λ_k^(1) - λ_k^(2))²)`

```
ctom compare old-report.json new-report.json --diff --format html
ctom compare v1-report.json v2-report.json --spectral-distance
```

### 3.4 Confidence Overlay

Every element (state, transition, eigenvalue) annotated with confidence:
- **High confidence** (many observations, low variance): solid rendering
- **Medium confidence**: dashed/pulsing
- **Low confidence** (few observations, high variance): ghosted with warning icon
- Confidence derived from bootstrap resampling of probe data

```
ctom visualize report.json --confidence-overlay --min-confidence 0.7
```

---

## 4. Use Cases with Worked Examples

### 4.1 API Versioning: Detect Internal Changes

**Scenario:** Your dependency updated from API v2.3 to v2.4. Did the internal structure change?

```bash
# Probe both versions
ctom probe --target https://api.example.com/v2.3 --output v23.json
ctom probe --target https://api.example.com/v2.4 --output v24.json

# Compare
ctom compare v23.json v24.json --diff --format html
```

**What you learn:**
- New internal states (undocumented features?)
- Removed transitions (deprecated code paths?)
- Changed conservation ratios (modified caching, routing, or logic?)
- Spectral distance quantifies *how much* changed internally

### 4.2 Competitive Analysis: Reverse-Engineer Competitor Architecture

**Scenario:** You want to understand how a competitor's search ranking system works internally.

```bash
ctom probe --type rest-api --target https://competitor.example.com/search \
  --corpus search-queries.txt \
  --attributes semantic-category,sentiment,length-tier,freshness \
  --depth 4 \
  --output competitor-structure.json

ctom visualize competitor-structure.json --format html
```

**What you learn:**
- How many distinct ranking regimes exist (state clusters)
- How queries transition between ranking regimes (echo chambers? topic walls?)
- Which inputs cause unusual state transitions (gaming vectors?)
- Spectral fingerprint for future comparison

### 4.3 Security Auditing: Find Hidden Endpoints / Backdoors

**Scenario:** Audit a third-party service your company depends on.

```bash
ctom probe --type rest-api --target https://service.internal \
  --auth "$SERVICE_TOKEN" \
  --fuzz-endpoints \
  --detect-anomalies \
  --output audit.json

ctom visualize audit.json --confidence-overlay --highlight-anomalies
```

**What you learn:**
- States reachable only via specific input sequences (hidden functionality)
- Transitions with anomalously low conservation (timing side-channels)
- Unexpected state clusters (undocumented features, debug endpoints, backdoors)
- Comparison to vendor's documented state graph

### 4.4 Compliance: Verify Architecture Matches Documentation

**Scenario:** Regulator requires proof that the deployed system matches its certified architecture.

```bash
# Generate reference from design docs
ctom spec-from-docs architecture.md --output certified-graph.json

# Probe live system
ctom probe --target https://production.internal --output live-report.json

# Verify
ctom compare certified-graph.json live-report.json --compliance-mode --tolerance 0.05
```

**What you learn:**
- Spectral distance between certified and observed structure
- Specific divergences (extra states, missing transitions, different conservation)
- Confidence-weighted compliance score
- Audit trail for regulators

### 4.5 Migration: Verify New System Matches Old

**Scenario:** You're migrating from legacy System A to new System B. Verify behavioral equivalence.

```bash
ctom probe --target https://system-a.internal --output system-a.json
ctom probe --target https://system-b.internal --output system-b.json
ctom compare system-a.json system-b.json --diff --format html \
  --spectral-distance --min-confidence 0.8
```

**What you learn:**
- Whether the new system has the same number of internal states
- Whether transitions are preserved
- Where behavioral differences exist (and whether they matter)
- Quantified similarity score for sign-off

---

## 5. CLI Design

### Installation

```bash
# Rust binary (primary)
cargo install ctom

# Python package (SDK + CLI wrapper)
pip install conservation-tomography

# Docker
docker run -v $(pwd)/reports:/reports ctom ctom probe --target https://api.example.com
```

### Core Commands

```bash
# Probe a target and generate a report
ctom probe --target <url|connstring|command> \
  --type <rest-api|database|ml-model|network|compiler> \
  [--corpus <path|generator>] \
  [--depth <1-5>] \
  [--output <path>] \
  [--rate-limit <rate>] \
  [--auth <credentials>] \
  [--config <path>]

# Visualize a report
ctom visualize <report.json> \
  --format <html|svg|png|json> \
  [--type <graph|heatmap|spectrum|confidence>] \
  [--layout <force|hierarchical|circular>] \
  [--output <path>]

# Compare two reports
ctom compare <report-a.json> <report-b.json> \
  --diff \
  [--format <html|json|markdown>] \
  [--spectral-distance] \
  [--compliance-mode] \
  [--tolerance <float>] \
  [--output <path>]

# Extract just the eigenvalue spectrum
ctom spectrum <report.json> \
  [--format <json|csv|plot>]

# Generate a reference spec from documentation
ctom spec-from-docs <architecture.md> \
  --output <spec.json>

# Validate a report (re-simulate conservation)
ctom validate <report.json> \
  [--target <url>] \
  [--samples <n>]

# Configuration management
ctom config set rate-limit 10/s
ctom config set default-time-scales 1,2,5,10,20,50
ctom config set probes.rest-api.user-agent "ctom/1.0"
```

### Config File (~/.ctom/config.toml)

```toml
[defaults]
depth = 3
time_scales = [1, 2, 5, 10, 20, 50, 100]
spectral_method = "esprit"
num_attributes = "2n"
rate_limit = "10/s"

[probes.rest-api]
timeout = 30
follow_redirects = true
user_agent = "ctom/1.0"

[probes.database]
query_timeout = 10
schema_aware = true

[reconstruction]
method = "spectral-matching"
max_iterations = 1000
tolerance = 1e-6
restarts = 10

[visualization]
default_format = "html"
default_layout = "force"

[ethics]
respect_robots_txt = true
respect_ctom_opt_out = true  # X-Ctom-Opt-Out: true header
max_request_ratio = 0.01     # never exceed 1% of target's estimated capacity
require_consent_flag = false  # set true for production deployments
```

---

## 6. SDK — Programmatic API

### Python SDK

```python
from ctom import Probe, Report, Compare

# Probe a REST API
report = Probe.rest_api(
    target="https://api.example.com",
    depth=3,
    corpus="search-queries.txt",
    attributes=["semantic-category", "sentiment", "length-tier"],
    rate_limit="10/s",
    auth={"Authorization": f"Bearer {token}"},
)

# Access results
print(f"Discovered {report.num_states} states")
print(f"Eigenvalue spectrum: {report.spectrum}")
print(f"Spectral gap: {report.spectral_gap}")
print(f"Algebraic connectivity: {report.algebraic_connectivity}")
print(f"Mixed time estimate: {report.mixing_time}")

# Access the reconstructed graph
graph = report.graph  # NetworkX DiGraph with weights and confidence
for u, v, data in graph.edges(data=True):
    print(f"  {u} → {v}: weight={data['weight']:.3f}, conf={data['confidence']:.3f}")

# Compare two reports
diff = Compare(report_v1, report_v2)
print(f"Spectral distance: {diff.spectral_distance:.4f}")
print(f"Added states: {diff.added_states}")
print(f"Removed states: {diff.removed_states}")
print(f"Changed transitions: {len(diff.changed_transitions)}")

# Export
report.to_html("report.html")
report.to_json("report.json")
graph.to_graphml("graph.graphml")

# Custom attribute function
def my_attribute(response):
    """Return a scalar attribute value for a response."""
    return 1.0 if "premium" in response.body else 0.0

report = Probe.rest_api(
    target="https://api.example.com",
    custom_attributes=[my_attribute],
)

# Low-level: build your own probe pipeline
from ctom.pipeline import (
    StateEnumerator,
    TransitionEstimator,
    ConservationMeasurer,
    GraphReconstructor,
)

states = StateEnumerator(cluster_method="hdbscan").fit(responses)
transitions = TransitionEstimator().fit(sequences, states)
conservation = ConservationMeasurer(
    time_scales=[1, 2, 5, 10, 20, 50, 100],
    spectral_method="esprit",
).fit(transitions)
graph = GraphReconstructor(
    method="spectral-matching",
    restarts=10,
    tolerance=1e-6,
).fit(conservation)
```

### Rust SDK

```rust
use ctom::{Probe, ProbeConfig, Report, Compare};

// Probe a REST API
let config = ProbeConfig::builder()
    .target("https://api.example.com")
    .probe_type(ProbeType::RestApi)
    .depth(3)
    .rate_limit("10/s")
    .auth("Bearer", &token)
    .build()?;

let report = Probe::run(config).await?;

// Access results
println!("States: {}", report.num_states());
println!("Spectrum: {:?}", report.spectrum());
println!("Spectral gap: {:.4}", report.spectral_gap());

// Reconstructed graph (petgraph)
let graph = report.graph();
for edge in graph.edge_references() {
    println!(
        "  {} → {}: weight={:.3}, conf={:.3}",
        graph.node_weight(edge.source()).unwrap(),
        graph.node_weight(edge.target()).unwrap(),
        edge.weight().transition_prob,
        edge.weight().confidence,
    );
}

// Compare
let diff = Compare::new(&report_v1, &report_v2)?;
println!("Spectral distance: {:.4}", diff.spectral_distance());
println!("Added states: {:?}", diff.added_states());

// Export
report.to_html("report.html")?;
report.to_json("report.json")?;
```

---

## 7. Ethical Safeguards

Conservation tomography is powerful. With power comes responsibility.

### 7.1 Built-in Rate Limiting

- Default: 10 requests/second, never more than 1% of estimated target capacity
- Adaptive backoff: if latency increases or errors appear, automatically slow down
- User-configurable with hard floors (cannot set below 1 req/s against non-localhost targets)
- Random jitter to avoid synchronized probe storms

### 7.2 Opt-Out Headers

Systems can opt out of tomography via standard headers:

```
# Server response headers:
X-Ctom-Opt-Out: true
X-Robots-Tag: noai, noctom

# ctom respects these by default
# Override requires --skip-opt-out-check flag (logs a warning)
```

### 7.3 Responsible Disclosure Mode

When probing discovers unexpected structure (hidden endpoints, undocumented states):

```bash
ctom probe --disclosure-mode responsible \
  --disclosure-contact security@example.com \
  --disclosure-template templates/disclosure.md
```

- Findsings are encrypted and held for 90 days before the user can export them
- Automatic disclosure draft generated
- CVSS-style severity rating for discovered anomalies

### 7.4 Consent and Scope

```bash
# Explicit consent file
ctom probe --consent-file consent.json --target ...

# consent.json
{
  "target": "https://api.example.com",
  "authorized_by": "Jane Smith, CTO",
  "date": "2026-05-28",
  "scope": "v2 endpoints only",
  "expires": "2026-08-28"
}
```

### 7.5 Audit Trail

Every probe run generates an immutable audit log:

```json
{
  "timestamp": "2026-05-28T11:08:00Z",
  "target": "https://api.example.com",
  "total_requests": 3421,
  "duration_seconds": 342,
  "peak_rps": 8.7,
  "opt_out_respected": true,
  "consent_file": "consent.json",
  "operator": "user@company.com",
  "ctom_version": "1.0.0"
}
```

### 7.6 Safety Guardrails

- **No destructive probes**: ctom never sends PUT/POST/DELETE unless explicitly enabled via `--allow-writes`
- **Scope limiting**: `--url-regex`, `--endpoint-allowlist`, `--endpoint-denylist`
- **Payload limits**: max request body size, max response storage
- **PII detection**: automatic redaction of potential PII in stored responses
- **Network scope**: refuses to probe localhost/private IPs by default (requires `--allow-private`)

---

## 8. Technical Architecture

### Crates / Modules (Rust)

```
ctom/
├── crates/
│   ├── ctom-core/          # Core algorithms (spectral estimation, graph reconstruction)
│   ├── ctom-probe/         # Probing infrastructure (HTTP client, rate limiting, sequencing)
│   ├── ctom-probes/        # Probe implementations
│   │   ├── rest-api/       # REST API probe
│   │   ├── database/       # Database probe
│   │   ├── ml-model/       # ML model probe
│   │   ├── network/        # Network protocol probe
│   │   └── compiler/       # Compiler probe
│   ├── ctom-analysis/      # Conservation measurement, spectral analysis
│   ├── ctom-reconstruct/   # Graph reconstruction algorithms
│   ├── ctom-visualize/     # Visualization engine (HTML/SVG output)
│   ├── ctom-compare/       # Diff engine for report comparison
│   ├── ctom-ethics/        # Rate limiting, opt-out, consent, audit
│   └── ctom-cli/           # CLI binary
├── ctom-python/            # Python bindings (PyO3)
└── docs/
```

### Key Dependencies

- **nalgebra** / **faer** — Linear algebra (eigendecomposition, SVD)
- **petgraph** — Graph data structures
- **reqwest** — HTTP client (REST probes)
- **tokio** — Async runtime
- **serde** — Serialization (JSON, TOML config)
- **clap** — CLI argument parsing
- **handlebars** / **askama** — HTML report templating
- **PyO3** — Python bindings

### Algorithm Details

**Spectral Estimation** (Phase 3):
- ESPRIT (Estimation of Signal Parameters via Rotational Invariance Techniques) for exponential fitting
- MUSIC (Multiple Signal Classification) as alternative for noisy measurements
- Compressed sensing for undersampled time-scale measurements
- Automatic model order selection via MDL (Minimum Description Length)

**Graph Reconstruction** (Phase 4):
- Iterative Spectral Matching with spectral perturbation gradients
- Multiple random restarts with best-of-k selection
- Simulated annealing for escaping local minima
- Constraint: row-stochastic transition matrix (sum-to-one)
- Sparsity prior: L1 regularization on edge weights (real graphs are sparse)
- Bootstrap confidence: resample probe data → re-reconstruct → measure edge stability

**Scalability Targets:**
| System Size | States (n) | Probe Time | Reconstruction Time | Total |
|---|---|---|---|---|
| Small | ≤50 | ~1 min | ~1 sec | ~1 min |
| Medium | ≤500 | ~10 min | ~30 sec | ~11 min |
| Large | ≤5000 | ~1 hr | ~10 min | ~1.1 hr |
| XL | ≤50K | ~6 hr | ~2 hr | ~8 hr |

---

## 9. Development Phases

### Phase 1: Core Engine (Weeks 1-6)
- [ ] Spectral estimation module (ESPRIT, Prony)
- [ ] Graph reconstruction (iterative spectral matching)
- [ ] Transition estimator from sequential probe data
- [ ] Bootstrap confidence engine
- [ ] Unit tests with synthetic graphs (known ground truth)

### Phase 2: REST API Probe (Weeks 4-8)
- [ ] HTTP probing infrastructure with rate limiting
- [ ] Response clustering (feature extraction → HDBSCAN)
- [ ] Attribute library for common API response types
- [ ] OpenAPI spec ingestion for guided probing
- [ ] Integration tests against test APIs

### Phase 3: CLI + Visualization (Weeks 7-10)
- [ ] CLI binary with all core commands
- [ ] HTML visualization (graph, heatmap, spectrum)
- [ ] Diff/comparison view
- [ ] Confidence overlay
- [ ] Config file support

### Phase 4: Additional Probes (Weeks 10-16)
- [ ] Database probe (PostgreSQL, MySQL, SQLite)
- [ ] ML model probe (HTTP endpoint + local ONNX models)
- [ ] Network protocol probe (TCP/UDP state machine)
- [ ] Compiler probe (GCC, Clang, rustc)
- [ ] Pluggable probe API for custom targets

### Phase 5: Python SDK + Polish (Weeks 14-20)
- [ ] PyO3 bindings for all core functionality
- [ ] Pythonic API design
- [ ] pip-installable package
- [ ] Documentation + examples
- [ ] Performance benchmarking + optimization

### Phase 6: Advanced Features (Weeks 18-24)
- [ ] Compliance mode (compare to documented architecture)
- [ ] Responsible disclosure mode
- [ ] CI/CD integration (ctom as regression test)
- [ ] Distributed probing (multi-machine for large targets)
- [ ] Spectral obfuscation detection (is the system hiding its structure?)

---

## 10. Comparison: ctom vs. Alternatives

| Feature | ctom | Fuzzing (AFL, etc.) | API Testing (Postman) | Reverse Engineering (Ghidra) |
|---|---|---|---|---|
| Reveals internal structure | ✅ Full state graph | ❌ Crash paths only | ❌ Functional only | ✅ Full (needs binary) |
| No source/binary needed | ✅ Black-box only | ✅ | ✅ | ❌ Requires binary |
| Quantified structural diff | ✅ Spectral distance | ❌ | ❌ Ad-hoc | ❌ |
| Conservation-based | ✅ Novel signal | ❌ | ❌ | ❌ |
| Multi-target (API, DB, ML) | ✅ | ❌ Binary-focused | ❌ API-only | ❌ Binary-only |
| Confidence scoring | ✅ Bootstrap CI | ❌ | ❌ | ❌ |
| Ethical safeguards | ✅ Built-in | ❌ | ❌ | ❌ |
| CI/CD integration | ✅ Regression mode | ✅ | ✅ Newman | ❌ |

---

## 11. Research Connections

This tool is the practical realization of the theoretical framework in [`INVERSE-CONSERVATION-PROBLEM.md`](../docs/INVERSE-CONSERVATION-PROBLEM.md). Key theoretical underpinnings:

- **Forward pipeline:** Dynamics → Graph → Laplacian → Eigenvalues → Conservation
- **Inverse pipeline:** Conservation → Eigenvalues → Laplacian → Graph → Dynamics
- **Spectral uniqueness:** Most graphs are uniquely determined by their spectrum (cospectral graphs are exponentially rare for large n)
- **Gradient:** `∂λ_k/∂w_uv = φ_k(u) · φ_k(v)` enables efficient graph reconstruction
- **Multiple regimes:** Probing under different dynamical parameters breaks cospectral degeneracies

The tool makes the inverse problem operational: probe → measure → reconstruct → visualize → compare.

---

*Conservation Tomography: Seeing in the dark.*
