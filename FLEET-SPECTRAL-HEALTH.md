# Fleet Spectral Health — Monitoring & Optimization Platform

> **Status:** Design Document  
> **Date:** 2026-05-28  
> **Scope:** Unified monitoring for the Cocapn SuperInstance fleet (40+ Rust crates) using spectral graph analysis (Tension-Graph Laplacian) to detect, diagnose, and remediate fleet health issues.

---

## 1. Fleet Tension Graph

The foundation of spectral health monitoring is a continuously-updated weighted graph encoding fleet behavior.

### 1.1 Nodes

Each node represents a deployable agent or service instance.

| Node Attribute | Type | Source | Description |
|---|---|---|---|
| `agent_id` | UUID | Registry | Unique instance identifier |
| `service_type` | enum | Registry | e.g., `deliberation`, `equipment`, `emergence`, `necropolis` |
| `task_success_rate` | f32 ∈ [0,1] | Metrics bus | Rolling 1h success ratio |
| `mean_latency_ms` | f32 | Metrics bus | P50 response latency |
| `confidence_score` | f32 ∈ [0,1] | cuda-equipment | Model confidence on recent outputs |
| `load_factor` | f32 ∈ [0,1] | Metrics bus | CPU/memory pressure |
| `last_heartbeat` | timestamp | Health pings | Time since last alive signal |

### 1.2 Edges

Edges encode inter-agent interaction strength. Edge weight between nodes *i* and *j*:

```
w_ij = α · comm_frequency(i,j) + β · response_similarity(i,j)
```

- **`comm_frequency(i,j)`** — normalized message count between agents *i* and *j* over the trailing window (e.g., 5 min), scaled to [0,1].
- **`response_similarity(i,j)`** — cosine similarity of recent output embedding vectors from agents *i* and *j*. Agents producing similar outputs (shared context/expertise) get stronger edges.
- **`α`, `β`** — tunable weights (default α=0.6, β=0.4). Emphasize communication topology but incorporate semantic coherence.

Edges are **sparse**: if `w_ij < ε` (default ε=0.01), the edge is dropped. This keeps the Laplacian tractable for 40+ nodes.

### 1.3 Laplacian Computation

Given the weighted adjacency matrix **W** and degree matrix **D** = diag(d_i) where d_i = Σ_j w_ij:

```
L = D - W                          # Combinatorial Laplacian
L_norm = D^{-1/2} · L · D^{-1/2}  # Normalized Laplacian (recommended)
```

The **normalized Laplacian** is preferred because it's scale-invariant — agents with high connectivity don't dominate the spectrum.

**Update cadence:** Every 10 seconds (configurable). The graph is small enough (≤64 nodes typically) that eigen-decomposition is sub-millisecond.

### 1.4 Tension Extension

The **Tension-Graph Laplacian** enriches L with a tension field **T** encoding constraint violations:

```
T_ij = |attr_i - attr_j| · w_ij    # Per-edge tension
L_tension = L + λ · T              # Tension-augmented Laplacian
```

Where `attr_i` is a scalar attribute (e.g., confidence score). High tension between strongly-connected agents signals disagreement — a key health indicator.

`λ` controls tension sensitivity (default 0.3).

---

## 2. Health Indicators

Spectral analysis of L_tension yields several fleet-level health signals.

### 2.1 Conservation Ratio

The **conservation ratio** measures how well the fleet's behavior aligns with a conserved quantity (stable energy distribution):

```
conservation = λ_2 / λ_max
```

- `λ_2` = second-smallest eigenvalue (algebraic connectivity / Fiedler value)
- `λ_max` = largest eigenvalue

| Range | Meaning |
|---|---|
| **> 0.4** | Fleet is coherent. Agents agree, communicate well, tasks succeed. |
| **0.2 – 0.4** | Mild degradation. Some agents drifting. |
| **< 0.2** | Fragmentation. Subgroups operating independently, consensus failing. |

### 2.2 Spectral Gap

```
spectral_gap = λ_3 - λ_2
```

A **large spectral gap** means the Fiedler partition (2 clusters) is the dominant structure — clear healthy/unhealthy split. A **small gap** means the fleet has many overlapping subgroups — everything is entangled and hard to diagnose.

### 2.3 Fiedler Vector

The eigenvector **v₂** corresponding to λ₂ partitions the fleet:

```
Healthy cluster:  { i | v₂[i] > 0 }
Unhealthy cluster: { i | v₂[i] ≤ 0 }
```

The **magnitude** |v₂[i]| indicates how strongly agent *i* belongs to its partition. Agents near zero are "borderline" — they could tip either way and are early-warning targets.

### 2.4 Anomaly Detection

An agent is **anomalous** if it violates fleet conservation:

```
anomaly_score(i) = |v₂[i]| · (1 - conservation) · tension_local(i)
```

Where `tension_local(i)` is the sum of tension on agent *i*'s edges. High anomaly score = the agent is both structurally distant and actively disagreeing with neighbors.

Additionally, monitor the **Rayleigh quotient** for each agent's local subgraph:

```
rayleigh(i) = (x^T L x) / (x^T x)    where x = indicator vector for node i's neighborhood
```

Spikes in Rayleigh quotient indicate local structural instability.

---

## 3. Alert System

### 3.1 Alert Levels

```
┌─────────┬───────────────────────────────────┬───────────────────────────┐
│ Level   │ Trigger                            │ Response                  │
├─────────┼───────────────────────────────────┼───────────────────────────┤
│ GREEN   │ conservation > 0.4                │ Monitor only              │
│         │ spectral gap healthy              │ Log metrics               │
│         │ no anomaly scores > threshold     │                           │
├─────────┼───────────────────────────────────┼───────────────────────────┤
│ YELLOW  │ conservation 0.2–0.4              │ Increase monitoring       │
│         │ OR anomaly_score > 0.6 for any    │ Flag anomalous agents     │
│         │   agent                           │ Prepare remediation plans │
│         │ OR spectral gap shrinking > 30%   │                           │
├─────────┼───────────────────────────────────┼───────────────────────────┤
│ RED     │ conservation < 0.2                │ Auto-remediation          │
│         │ OR ≥3 agents anomaly > 0.8        │ Alert operators           │
│         │ OR Fiedler partition shows         │ Trigger deliberation      │
│         │   >60% of fleet unhealthy          │                           │
└─────────┴───────────────────────────────────┴───────────────────────────┘
```

### 3.2 Automated Actions

| Action | Trigger | Crate |
|---|---|---|
| **Restart anomalous agent** | Single agent anomaly > 0.8 for 5 min | cuda-necropolis |
| **Rebalance load** | Fiedler partition skewed > 70/30 | cuda-equipment |
| **Trigger fleet deliberation** | RED alert sustained > 2 min | cuda-deliberation |
| **Quarantine agent** | Agent causing tension spike > 3σ | cuda-necropolis |
| **Expand MoE routing** | Expert utilization uneven (coefficient of variation > 0.5) | cuda-emergence |

### 3.3 Hysteresis

Alerts use hysteresis to avoid flapping:

- YELLOW→GREEN: conservation must stay > 0.4 for 5 consecutive cycles (50s)
- RED→YELLOW: conservation must stay > 0.2 for 3 consecutive cycles (30s)
- Any level jumps to RED immediately if conservation drops below 0.15

---

## 4. Dashboard Design

### 4.1 ASCII Wireframe

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║  FLEET SPECTRAL HEALTH — Cocapn SuperInstance           [●GREEN] 2026-05-28    ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                ║
║  ┌─ Conservation Ratio (1h) ──────────────────────────────────────────────┐    ║
║  │ 0.5│ ╭─╮                   ╭──────────────╮                             │    ║
║  │ 0.4│╯  ╰──╮         ╭─────╯              ╰───╮                         │    ║
║  │ 0.3│      ╰──╮   ╭──╯                        ╰──╮                      │    ║
║  │ 0.2│         ╰───╯ ▲                             ╰──╮                 │    ║
║  │ 0.1│                                 ● anomaly     ╰──                │    ║
║  │ 0.0└─────────────────────────────────────────────────────────────────  │    ║
║  │     10:00  10:15  10:30  10:45  11:00  11:15  11:30  11:45  12:00     │    ║
║  └───────────────────────────────────────────────────────────────────────┘    ║
║                                                                                ║
║  ┌─ Fleet Topology (Fiedler Partition) ────────┐  ┌─ Spectral Gap ────────┐  ║
║  │                                              │  │ 0.15│ ╭──╮           │  ║
║  │     [A]──────[B]──────[C]     ← healthy     │  │ 0.12│╯  ╰──╮        │  ║
║  │      │ ╲      │       │                     │  │ 0.09│      ╰──╮     │  ║
║  │      │   ╲    │      │                     │  │ 0.06│         ╰──    │  ║
║  │     [D]──────[E]──●──[F]     ← borderline  │  │ 0.03│               │  ║
║  │                ╲                             │  │ 0.00└──────────────  │  ║
║  │                 [G]────[H]   ← unhealthy    │  └─────────────────────┘  ║
║  │                                              │                           ║
║  │  ● healthy  ● borderline  ● unhealthy       │                           ║
║  └──────────────────────────────────────────────┘                           ║
║                                                                                ║
║  ┌─ Anomaly Log ──────────────────────────────────────────────────────────┐   ║
║  │ TIME      │ AGENT  │ SCORE │ TYPE          │ ACTION                   │   ║
║  │ 11:42:18  │ G      │ 0.83  │ tension_spike │ flagged → monitor         │   ║
║  │ 11:38:05  │ H      │ 0.71  │ drift         │ load rebalanced           │   ║
║  │ 11:15:30  │ F      │ 0.52  │ borderline    │ no action (resolved)      │   ║
║  └───────────────────────────────────────────────────────────────────────┘   ║
║                                                                                ║
║  ┌─ Expert Utilization Heatmap (MoE) ────────────────────────────────────┐    ║
║  │ Expert    │ Load  │ Confidence │ Tasks │ Status                        │    ║
║  │ code_gen  │ ████░ │ 0.92       │ 847   │ active                        │    ║
║  │ reasoning │ █████ │ 0.88       │ 1,203 │ active                        │    ║
║  │ tools     │ ██░░░ │ 0.95       │ 312   │ underutilized                 │    ║
║  │ safety    │ ███░░ │ 0.79       │ 589   │ degraded confidence           │    ║
║  └───────────────────────────────────────────────────────────────────────┘   ║
║                                                                                ║
╚════════════════════════════════════════════════════════════════════════════════╝
```

### 4.2 Dashboard Components

1. **Conservation Ratio Chart** — Time-series of `conservation` with alert thresholds marked. Annotates anomalies with root-cause hints.

2. **Fleet Topology** — Force-directed graph layout. Nodes colored by Fiedler partition (green/yellow/red). Edge width = weight, edge color = tension (cool→hot). Click any node for agent detail.

3. **Spectral Gap Chart** — Time-series showing structural clarity of the fleet.

4. **Anomaly Log** — Scrollable table of recent anomalies with severity, agent, type, and automated actions taken.

5. **Expert Utilization Heatmap** — For MoE (Mixture of Experts) models: per-expert load bars, confidence, and task throughput. Flags uneven utilization.

---

## 5. Integration with Existing Cocapn Crates

### 5.1 cuda-equipment → Confidence Propagation

```
cuda-equipment (confidence per agent)
       │
       ▼
  ┌──────────────┐
  │ confidence    │──→ Node attribute: confidence_score
  │ propagation   │──→ Tension field: T_ij = |conf_i - conf_j| · w_ij
  └──────────────┘
       │
       ▼
  Spectral Health: detects confidence divergence early
```

- Equipment publishes `confidence_score` per inference batch to the metrics bus.
- Spectral health consumes these as node attributes and computes tension from pairwise confidence differences.
- **Feedback loop:** When conservation drops, equipment adjusts routing confidence thresholds.

### 5.2 cuda-deliberation → Consensus Conservation

```
cuda-deliberation (consensus rounds)
       │
       ▼
  ┌──────────────────┐
  │ consensus         │──→ Edge weight: comm_frequency weighted by agreement
  │ conservation      │──→ Health check: conservation ≈ consensus convergence
  └──────────────────┘
       │
       ▼
  Spectral Health: conservation ratio mirrors deliberation quality
```

- Deliberation rounds produce agreement metrics (how close agents converged).
- Spectral health uses these to validate fleet coherence: low conservation + high deliberation disagreement = systemic issue.
- **Feedback loop:** RED alert triggers a forced deliberation round with full fleet participation.

### 5.3 cuda-emergence → Emergent Pattern Detection

```
cuda-emergence (pattern detection)
       │
       ▼
  ┌──────────────────┐
  │ emergent pattern  │──→ Anomaly: unexpected spectral clusters
  │ detection         │──→ Spectral gap change → new behavior emergence
  └──────────────────┘
       │
       ▼
  Spectral Health: sudden spectral changes = emergent behavior
```

- Emergence detects when agents develop novel coordination patterns.
- Spectral health cross-references: is the new pattern healthy (new cluster with high internal conservation) or pathological (fragmentation)?
- **Feedback loop:** Healthy emergence → update baseline spectral profile. Pathological → alert + quarantine.

### 5.4 cuda-necropolis → Agent Death & Conservation Impact

```
cuda-necropolis (agent lifecycle)
       │
       ▼
  ┌──────────────────┐
  │ agent death →     │──→ Node removal from graph
  │ conservation      │──→ Impact analysis: Δλ₂ after node removal
  │ impact analysis   │──→ Prioritize restarts by conservation impact
  └──────────────────┘
       │
       ▼
  Spectral Health: node removal impact scored before action
```

- When an agent dies, necropolis removes it from the graph and recomputes spectral health.
- **Impact analysis:** predicts Δconservation from removing/restarting each agent. Agents whose removal causes large conservation drops are "critical" — prioritize their recovery.
- **Feedback loop:** necropolis uses impact scores to decide restart ordering and resource allocation.

### 5.5 Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Metrics Bus (async, broadcast)                  │
│   confidence · latency · success_rate · heartbeat · comm_count     │
└──────────┬──────────┬──────────┬──────────┬──────────┬─────────────┘
           │          │          │          │          │
     equipment  deliberation  emergence  necropolis  ...
           │          │          │          │
           └──────────┴──────────┴──────────┘
                      │
              ┌───────▼────────┐
              │ Fleet Tension   │
              │ Graph Builder   │  ← builds W, T every 10s
              └───────┬────────┘
                      │
              ┌───────▼────────┐
              │ Spectral        │
              │ Analyzer        │  ← eigen-decomposition, health scoring
              └───────┬────────┘
                      │
              ┌───────▼────────┐
              │ Alert Engine    │  ← thresholds → GREEN/YELLOW/RED + actions
              └───────┬────────┘
                      │
              ┌───────▼────────┐
              │ Dashboard API   │  ← REST + WebSocket
              └────────────────┘
```

---

## 6. API Schema

### 6.1 REST Endpoints

```
GET  /api/v1/health                    → Current fleet health summary
GET  /api/v1/health/history?range=1h   → Time-series health data
GET  /api/v1/agents                    → List all agents with spectral attributes
GET  /api/v1/agents/{id}               → Single agent detail + anomaly history
GET  /api/v1/graph                     → Current tension graph (nodes + edges)
GET  /api/v1/spectrum                  → Eigenvalues + eigenvectors
GET  /api/v1/anomalies?severity=high   → Recent anomalies
GET  /api/v1/experts/utilization       → MoE expert heatmap data
POST /api/v1/actions/rebalance         → Trigger manual load rebalance
POST /api/v1/actions/restart/{id}      → Restart specific agent
POST /api/v1/actions/deliberation      → Force fleet deliberation round
```

### 6.2 Response Schemas

```jsonc
// GET /api/v1/health
{
  "status": "GREEN",
  "conservation_ratio": 0.47,
  "spectral_gap": 0.12,
  "fiedler_partitions": {
    "healthy": ["A", "B", "C", "D", "E"],
    "borderline": ["F"],
    "unhealthy": ["G", "H"]
  },
  "anomaly_count": 2,
  "last_updated": "2026-05-28T11:08:00Z"
}

// GET /api/v1/agents/{id}
{
  "agent_id": "G",
  "service_type": "deliberation",
  "confidence_score": 0.61,
  "task_success_rate": 0.72,
  "mean_latency_ms": 340,
  "load_factor": 0.88,
  "fiedler_value": -0.42,
  "anomaly_score": 0.83,
  "recent_anomalies": [
    {
      "timestamp": "2026-05-28T11:42:18Z",
      "type": "tension_spike",
      "score": 0.83,
      "action_taken": "flagged → monitor"
    }
  ]
}

// GET /api/v1/graph
{
  "nodes": [
    { "id": "A", "type": "equipment", "confidence": 0.94, "success_rate": 0.97, "fiedler": 0.31 },
    // ...
  ],
  "edges": [
    { "source": "A", "target": "B", "weight": 0.78, "tension": 0.02 },
    // ...
  ],
  "laplacian_eigenvalues": [0, 0.089, 0.204, 0.341, ...],
  "computed_at": "2026-05-28T11:08:00Z"
}
```

### 6.3 WebSocket

```
WS /api/v1/ws/health

→ Server pushes every update cycle (10s default):
{
  "type": "health_update",
  "data": { /* same as GET /api/v1/health */ }
}

→ Server pushes on alert level change:
{
  "type": "alert",
  "level": "YELLOW",
  "message": "Conservation ratio dropped to 0.35",
  "affected_agents": ["G", "H"]
}

→ Server pushes on anomaly detection:
{
  "type": "anomaly",
  "agent_id": "G",
  "score": 0.83,
  "anomaly_type": "tension_spike",
  "action": "flagged"
}
```

---

## 7. Implementation Plan

### Phase 1: Foundation (Weeks 1–2)

**Goal:** Core spectral analysis pipeline with basic dashboard.

| Task | Description | Est. |
|---|---|---|
| Metrics bus integration | Subscribe to existing metrics from equipment, deliberation, emergence, necropolis | 2d |
| Tension graph builder | Weighted adjacency matrix construction with configurable α/β | 2d |
| Spectral analyzer | Eigen-decomposition (nalgebra or ndarray-linalg), conservation ratio, Fiedler vector | 3d |
| Alert engine | Threshold-based GREEN/YELLOW/RED with hysteresis | 1d |
| REST API | Health, agents, graph, spectrum endpoints | 2d |
| Basic dashboard | Conservation chart + topology view (static) | 2d |

**Deliverables:**
- `cuda-spectral-health` crate with graph builder + analyzer
- REST API serving real-time spectral data
- Static dashboard showing conservation ratio and fleet topology

### Phase 2: Production Monitoring (Months 1–2)

**Goal:** Real-time dashboard, automated actions, full crate integration.

| Task | Description | Est. |
|---|---|---|
| WebSocket streaming | Real-time dashboard updates | 1w |
| Interactive dashboard | Clickable topology, drill-down, anomaly detail | 2w |
| Necropolis integration | Agent death impact analysis, restart prioritization | 1w |
| Deliberation integration | Conservation-triggered deliberation rounds | 1w |
| Auto-remediation | Restart, rebalance, quarantine actions | 1w |
| Emergence integration | Spectral anomaly ↔ emergent pattern correlation | 1w |
| Alerting channels | Slack/Discord/webhook notifications for YELLOW/RED | 3d |
| Persistence | Historical spectral data in SQLite/TimescaleDB | 3d |

**Deliverables:**
- Production dashboard with real-time WebSocket updates
- Automated remediation pipeline
- Historical spectral data for trend analysis
- Notifications to operator channels

### Phase 3: Intelligence (Months 3–6)

**Goal:** Predictive health, adaptive thresholds, fleet optimization.

| Task | Description | Est. |
|---|---|---|
| Predictive health | ML model predicting conservation drops 10-30 min ahead | 3w |
| Adaptive thresholds | Auto-tune alert thresholds from historical baselines | 2w |
| MoE optimization | Expert utilization → dynamic routing based on spectral health | 3w |
| Fleet topology optimization | Suggest agent deployments/redundancies based on spectral structure | 3w |
| Multi-fleet federation | Spectral health across multiple SuperInstance fleets | 3w |
| Chaos engineering | Simulate agent failures, measure Δconservation | 2w |
| Self-healing | Closed-loop: spectral health → diagnosis → action → verification | 4w |

**Deliverables:**
- Predictive fleet health model
- Self-healing fleet with closed-loop verification
- Multi-fleet federation support
- Chaos testing framework for fleet resilience

---

## Appendix: Key Formulas Reference

```
# Edge weight
w_ij = α · comm_frequency(i,j) + β · response_similarity(i,j)

# Normalized Laplacian
L_norm = I - D^{-1/2} W D^{-1/2}

# Tension-augmented Laplacian
T_ij = |attr_i - attr_j| · w_ij
L_tension = L + λ · T

# Conservation ratio
conservation = λ_2 / λ_max

# Spectral gap
spectral_gap = λ_3 - λ_2

# Fiedler partition
healthy   = { i | v₂[i] > 0 }
unhealthy = { i | v₂[i] ≤ 0 }

# Anomaly score
anomaly(i) = |v₂[i]| · (1 - conservation) · Σ_j T_ij

# Rayleigh quotient (local)
rayleigh(i) = (xᵀ L x) / (xᵀ x)    where x = 𝟙{neighbors of i}
```

---

*This document is a living design. Update as implementation reveals what works.*
