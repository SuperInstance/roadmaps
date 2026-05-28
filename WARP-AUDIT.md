# WARP AUDIT — FLUX-Native PLATO Adaptive Model-Agnostic Parallelism

**Date:** 2026-05-28  
**Repo:** `SuperInstance/warp` (forked from `warpdotdev/warp`)  
**Upstream:** https://github.com/warpdotdev/warp  
**License:** AGPL-3.0 (code) + MIT (warpui framework)

---

## 1. What This Repo Is

**Warp is a Rust-based agentic development environment / terminal emulator** — not a VPN, not a GPU framework, not a cloud tool. It's the Warp terminal (warp.dev), recently open-sourced with OpenAI as founding sponsor.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Warp Application                      │
│                      (app/ crate)                        │
├──────────┬──────────┬───────────┬──────────┬────────────┤
│ Terminal │ AI/Agent │  Cloud    │  Drive   │  Settings  │
│ (warp_   │  Mode    │  Sync    │          │            │
│ terminal)│ (ai/)    │ (cloud_  │          │            │
│          │          │  objects)│          │            │
├──────────┴──────────┴───────────┴──────────┴────────────┤
│                    WarpUI Framework                      │
│              (warpui_core + warpui crates)               │
│          Entity-Component-Handle pattern                 │
│          Custom GPU-rendered UI (wgpu/WGSL)             │
├─────────────────────────────────────────────────────────┤
│                    Core Libraries                        │
├──────────┬──────────┬──────────┬──────────┬────────────┤
│warp_core │  editor  │   lsp    │   mcp    │  graphql   │
│(features,│ (text    │ (language│ (tool    │ (server    │
│ platform │  editing)│ protocol)│ protocol)│  comm)     │
│  abstr.) │          │          │          │            │
├──────────┼──────────┼──────────┼──────────┼────────────┤
│warp_     │  repo_   │  warp_   │ computer_│ persistence│
│completer │ metadata │ ripgrep  │ use      │ (Diesel/   │
│(shell    │ (code    │ (search) │(screen   │  SQLite)   │
│ complete)│  index)  │          │ control) │            │
├──────────┼──────────┼──────────┼──────────┼────────────┤
│ syntax_  │  sum_    │ settings │  vim     │  watcher   │
│ tree     │  tree    │          │  (mode)  │            │
│(arborium)│          │          │          │            │
├──────────┴──────────┴──────────┴──────────┴────────────┤
│                Platform Abstractions                     │
│     macOS (Metal) │ Linux (Vulkan/X11) │ Win (DX12)    │
│                    │ WASM (WebGPU)      │               │
├─────────────────────────────────────────────────────────┤
│            Tokio async runtime + wgpu GPU rendering      │
└─────────────────────────────────────────────────────────┘
```

### Key Subsystems

| Subsystem | Location | Purpose |
|-----------|----------|---------|
| **AI/Agent** | `crates/ai/`, `app/src/ai/` | AI assistant, agent mode, tool calling |
| **Terminal** | `crates/warp_terminal/` | PTY management, terminal emulation (based on Alacritty/VTE) |
| **WarpUI** | `crates/warpui/`, `crates/warpui_core/` | Custom GPU-rendered UI framework (Flutter-inspired ECH pattern) |
| **Editor** | `crates/editor/` | Text editing, selections, multi-cursor |
| **LSP** | `crates/lsp/` | Language Server Protocol client |
| **MCP** | `crates/mcp/` | Model Context Protocol (tool integration) |
| **Syntax** | `crates/syntax_tree/` | Tree-sitter via arborium (30+ languages) |
| **Search** | `crates/warp_ripgrep/`, `crates/warp_search_core/` | Code search |
| **Repo Metadata** | `crates/repo_metadata/` | Codebase indexing, project skill discovery |
| **Computer Use** | `crates/computer_use/` | Screen control / agentic actions |
| **Persistence** | `crates/persistence/` | SQLite via Diesel ORM |
| **GraphQL** | `crates/graphql/`, `crates/warp_graphql_schema/` | Server communication |
| **Cloud Objects** | `crates/cloud_objects/` | Cloud sync, Drive |
| **IPC** | `crates/ipc/` | Inter-process communication |
| **Completer** | `crates/warp_completer/` | Shell command completion (Fig-based) |
| **Isolation** | `crates/isolation_platform/` | Sandboxed agent execution |
| **Node Runtime** | `crates/node_runtime/` | Embedded JS runtime |

### Scale

- **60+ crates** in the Cargo workspace
- **Primary language:** Rust (with WGSL shaders for GPU rendering)
- **Platform targets:** macOS, Linux, Windows, WASM (browser)
- **GPU rendering:** wgpu (supports Metal, Vulkan, DX12, WebGPU)
- **Repo size:** ~267MB
- **Dependencies:** Tokio, wgpu, winit, Alacritty/VTE, Diesel/SQLite, prost (protobuf), tree-sitter, rayon, parquet, rmcp (MCP)

### What Warp Does Today

1. **Terminal emulator** — GPU-accelerated, modern terminal with blocks, completions
2. **AI Agent Mode** — Built-in coding agent (powered by GPT), can read/write files, run commands
3. **Oz agents** — Automated issue triage, spec writing, code review, PR management
4. **Warp-on-Web** — Browser-based version compiled to WASM
5. **Cloud sync** — Drive, shared objects, team collaboration
6. **MCP support** — Model Context Protocol for tool integration
7. **Computer use** — Screen reading/control for agentic workflows

---

## 2. Fork Analysis

### Status: **VANILLA FORK — Zero modifications**

- **Fork created:** 2026-05-28T19:35:21Z (today)
- **HEAD commit:** `876b840` — identical to `warpdotdev/warp` master
- **Commits ahead:** 0
- **Commits behind:** 0
- **SuperInstance changes:** None yet

This is a clean fork with no divergence. The integration work is entirely prospective.

### Upstream Relationship

- Parent: `warpdotdev/warp` (organization, not `openai/warp`)
- OpenAI is the "founding sponsor" — not the repo owner
- The repo was originally closed-source Warp terminal, recently open-sourced
- Active development: multiple commits per day, PR-based workflow with agent-created PRs

---

## 3. THE VISION — FLUX-Native PLATO Adaptive Model-Agnostic Parallelism

### The Core Idea

Warp is already an **agentic development environment with GPU rendering, AI integration, and a plugin architecture (MCP/skills)**. We transform it into the **universal execution shell** for the entire SuperInstance ecosystem. Instead of calling OpenAI APIs, Warp runs FLUX programs locally, dispatches to PLATO rooms for context, and uses local GPUs for everything from spectral analysis to neural inference.

### 3a. Replace OpenAI Backend with FLUX VM

**Current state:** `crates/ai/` calls OpenAI/Claude/etc. APIs via HTTP. The agent generates text, runs tools, and streams responses.

**Target state:** FLUX-C bytecode runs natively inside Warp's agent loop.

```
Current:  User → Warp → HTTP → OpenAI API → Text Response → Tool Execution
Target:   User → Warp → FLUX VM → Bytecode Execution → Provable Results → Tool Execution
                ↑                                    ↓
                └── Proof Certificate ← Constraint Check ←┘
```

**Integration points:**

| Warp Component | FLUX Integration | File/Module |
|---------------|------------------|-------------|
| `crates/ai/src/agent/` | Replace LLM call dispatch with FLUX VM invocation | `agent/tool_call.rs`, `agent/message.rs` |
| `crates/ai/` | Add `FluxBackend` alongside existing `OpenAIBackend`, `AnthropicBackend` | New: `crates/ai/src/flux_backend.rs` |
| `crates/mcp/` | FLUX programs expose MCP tools automatically | `crates/mcp/src/` |
| `crates/warp_core/src/features.rs` | `FeatureFlag::FluxExecution` toggle | `features.rs` |
| `crates/isolation_platform/` | FLUX VM runs in existing sandbox | Existing sandbox + FLUX runtime |

**Key advantage:** FLUX's guaranteed termination means agent loops can't spin forever. Proof certificates mean every agent action is auditable. Constraint checking means impossible states are impossible.

### 3b. PLATO Rooms as Execution Contexts

**Current state:** `crates/repo_metadata/` indexes codebases and `crates/ai/src/agent/` uses that context for AI calls.

**Target state:** Each PLATO knowledge room becomes a Warp execution context that shapes how agents reason.

```
┌─────────────────────────────────────────┐
│           PLATO Room Layer              │
├──────────┬──────────┬──────────────────┤
│ Room A:  │ Room B:  │ Room C:          │
│ "Rust    │ "GPU     │ "Conservation    │
│  Safety" │  Kernels"│  Laws"           │
│          │          │                  │
│ LoRA     │ Training │ Constraint       │
│ adapters │ data     │ rules            │
│ + rules  │ + CUDA   │ + spectral      │
│          │ patterns │ signatures       │
├──────────┴──────────┴──────────────────┤
│         Execution Context Mixer         │
│    (blends room knowledge into FLUX     │
│     program execution parameters)       │
├─────────────────────────────────────────┤
│            FLUX VM Execution            │
└─────────────────────────────────────────┘
```

**Integration:** Create `crates/plato/` with:
- `PlatoRoom` struct mapping to existing repo metadata patterns
- Room-based context injection into agent prompts
- LoRA adapter loading for local model fine-tuning per room
- Constraint rules that FLUX programs must satisfy

### 3c. Model-Agnostic Parallelism

**Current state:** Warp calls one LLM at a time per conversation.

**Target state:** A unified dispatch layer that runs ANY compute backend through the same interface.

```rust
// Conceptual trait
trait ComputeBackend: Send + Sync {
    fn execute(&self, task: ComputeTask) -> ComputeResult;
    fn capabilities(&self) -> BackendCapabilities;
    fn cost_estimate(&self, task: &ComputeTask) -> CostEstimate;
}

// Implementations:
// - FluxBackend (FLUX bytecode)
// - LocalLlmBackend (candle/ONNX on GPU)
// - RemoteApiBackend (OpenAI, Anthropic, etc.)
// - CudaKernelBackend (custom CUDA via cuda-* crates)
// - RustCrateBackend (native Rust functions)
// - SpectralBackend (conservation-spectral analysis)
```

**Where it lives:** New crate `crates/compute_dispatch/` with a scheduler that picks the optimal backend based on:
- Task type (inference, analysis, compilation, search)
- Available hardware (GPU memory, compute capability)
- Cost constraints
- Latency requirements
- Accuracy guarantees needed

### 3d. Local GPU Scaling (RTX 4050)

**Current state:** Warp uses wgpu for UI rendering only.

**Target state:** GPU becomes the primary compute engine for everything beyond UI.

| Task | GPU Library | Integration Point |
|------|------------|-------------------|
| FLUX bytecode JIT | Cranelift → PTX → CUDA | `crates/flux_jit/` (new) |
| Spectral analysis | conservation-spectral CUDA kernels | `crates/spectral/` (new) |
| Geometric algebra | ga-core GPU kernels | `crates/geometric/` (new) |
| Neural inference | ONNX Runtime / candle | `crates/neural_inference/` (new) |
| UI rendering | wgpu (existing) | `crates/warpui/` (extend) |

**GPU Scheduler:** Add `crates/gpu_scheduler/` that:
- Allocates GPU memory between UI rendering and compute tasks
- Prioritizes UI responsiveness (Warp is interactive)
- Batches compute tasks during idle frames
- Falls back to CPU when GPU is overloaded
- Uses wgpu's existing device/queue architecture

### 3e. The Killer App: Spectral Development Environment

A developer tool that:
1. **Understands code through spectral analysis** — conservation-spectral-js computes spectral signatures of codebases, detecting structural patterns, anomalies, and conservation law violations
2. **Reasons through PLATO rooms** — constraint-aware knowledge rooms ensure every suggestion respects project-specific rules
3. **Executes through FLUX** — every action is provably correct and terminating
4. **Dispatches intelligently** — GPU for compute-heavy tasks, CPU for logic, remote for large models
5. **Self-monitors** — fleet-spectral-health watches the system's own spectral fingerprint
6. **Self-improves** — model descent compresses learned patterns into better heuristics

This isn't just an IDE. It's a **self-improving, formally verified, spectrally-aware development environment**.

---

## 4. Specific Integration Points — SuperInstance Repos → Warp

### 4.1 conservation-spectral (Rust/Python/JS/C) → Spectral Analysis Engine

**What it does:** Computes spectral signatures, conservation laws, Laplacian analysis, structural patterns.

**Integration:** 
- New crate: `crates/spectral/`
- Rust FFI bindings to conservation-spectral C library
- GPU-accelerated Laplacian computation via CUDA
- Exposes spectral analysis as:
  - MCP tool (for agent use)
  - Background indexer (like `repo_metadata` but spectral)
  - Real-time code health dashboard in WarpUI
- **Files to create:** `crates/spectral/src/lib.rs`, `crates/spectral/src/laplacian.rs`, `crates/spectral/src/signature.rs`
- **Files to modify:** `Cargo.toml` (add workspace member), `crates/repo_metadata/` (extend with spectral data)
- **Estimated LOC:** 3,000-5,000

### 4.2 flux-vm-v3 → Constraint Execution Backend

**What it does:** Virtual machine with guaranteed termination, proof certificates, constraint checking.

**Integration:**
- New crate: `crates/flux_runtime/`
- Embed FLUX VM as a compute backend alongside LLM APIs
- Agent mode: FLUX programs replace/augment LLM tool calls for deterministic operations
- Isolation platform: FLUX programs run in existing sandbox
- **Key files:** `crates/ai/src/agent/tool_call.rs` (add FLUX tool type), `crates/isolation_platform/` (FLUX runtime)
- **Estimated LOC:** 5,000-8,000

### 4.3 constraint-theory-core → Exact Arithmetic Foundation

**What it does:** Exact arithmetic, constraint solving, mathematical foundations.

**Integration:**
- New crate: `crates/constraint_engine/`
- Powers constraint checking for FLUX programs
- Exact arithmetic for spectral computations (no floating point drift)
- Used by PLATO rooms for rule enforcement
- **Estimated LOC:** 2,000-3,000

### 4.4 ga-core → Spatial/Geometric Operations

**What it does:** Geometric algebra operations, multivectors, GPU-accelerated spatial computation.

**Integration:**
- New crate: `crates/geometric/`
- GPU kernel integration for geometric operations
- Used for spatial reasoning in code analysis (AST as geometric structure)
- Potential for novel code visualization (geometric algebra of code structure)
- **Estimated LOC:** 2,000-4,000

### 4.5 symplectic-opt → Energy-Conserving Optimization

**What it does:** Symplectic optimization that conserves energy-like quantities during iterative processes.

**Integration:**
- Use for model descent (self-improvement that conserves "knowledge energy")
- Optimization of GPU task scheduling (symplectic integrator for resource allocation)
- Training LoRA adapters with conservation guarantees
- **Estimated LOC:** 1,500-2,500

### 4.6 cuda-* Crates → GPU Acceleration

**What they do:** CUDA kernel wrappers for various compute tasks.

**Integration:**
- New crate: `crates/cuda_compute/`
- Unified GPU compute layer using existing wgpu device
- When wgpu's compute shaders aren't enough, drop to CUDA directly
- Runtime detection: CUDA available → use native kernels; fallback → wgpu compute
- **Estimated LOC:** 3,000-5,000

### 4.7 plato-* → Knowledge Rooms

**What they do:** Structured knowledge management with constraints, LoRA adapters, training data.

**Integration:**
- New crate: `crates/plato/`
- Room-based context system for agents
- Each room = a Warp "workspace" with associated knowledge
- LoRA adapter management for per-room model customization
- Room sharing via Warp's existing cloud sync infrastructure
- **Files to create:** `crates/plato/src/room.rs`, `crates/plato/src/context.rs`, `crates/plato/src/adapter.rs`
- **Estimated LOC:** 4,000-6,000

### 4.8 cocapn-* → Fleet Coordination

**What they do:** Multi-agent fleet coordination, task distribution, consensus.

**Integration:**
- Extend Warp's existing agent system to coordinate multiple Warp instances
- Fleet mode: multiple Warp terminals collaborating on a task
- Agent-to-agent communication via cocapn protocols
- **Estimated LOC:** 3,000-4,000

### 4.9 fleet-spectral-health → Self-Monitoring

**What it does:** Spectral analysis of fleet health, anomaly detection.

**Integration:**
- New crate: `crates/fleet_health/`
- Monitors Warp's own performance metrics as spectral data
- Detects degradation, memory leaks, performance anomalies via conservation law violations
- Dashboard in WarpUI showing system health
- **Estimated LOC:** 2,000-3,000

---

## 5. Full System Architecture

```
                    ┌─────────────────────────────────┐
                    │         USER INTERFACE           │
                    │   WarpUI (GPU-rendered via wgpu) │
                    │   ┌─────────┐  ┌──────────────┐ │
                    │   │Terminal │  │ Spectral     │ │
                    │   │Blocks   │  │ Dashboard    │ │
                    │   └────┬────┘  └──────┬───────┘ │
                    │   ┌────┴────┐  ┌──────┴───────┐ │
                    │   │ Agent   │  │ Fleet Health │ │
                    │   │ Chat    │  │ Monitor      │ │
                    │   └────┬────┘  └──────┬───────┘ │
                    └────────┼──────────────┼─────────┘
                             │              │
                    ┌────────┴──────────────┴─────────┐
                    │       COMPUTE DISPATCH LAYER      │
                    │   (Model-Agnostic Parallelism)    │
                    │  ┌─────────────────────────────┐ │
                    │  │       Task Scheduler         │ │
                    │  │  (cost, latency, accuracy)   │ │
                    │  └──┬──────┬──────┬──────┬─────┘ │
                    │     │      │      │      │       │
                    │  ┌──┴──┐┌──┴──┐┌──┴──┐┌──┴──┐   │
                    │  │FLUX ││Local││CUDA ││API  │   │
                    │  │ VM  ││ LLM ││Kern.││Remot│   │
                    │  └──┬──┘└──┬──┘└──┬──┘└──┬──┘   │
                    └─────┼──────┼──────┼──────┼───────┘
                          │      │      │      │
              ┌───────────┼──────┼──────┼──────┼───────────┐
              │           │      │      │      │           │
              │  ┌────────┴──────┴──────┴──────┴──────┐    │
              │  │         GPU SCHEDULER               │    │
              │  │  ┌──────────────────────────────┐   │    │
              │  │  │  RTX 4050 (6GB VRAM)          │   │    │
              │  │  │  ┌──────┐ ┌────────┐ ┌─────┐ │   │    │
              │  │  │  │ UI   │ │Compute │ │Neural│ │   │    │
              │  │  │  │Render│ │Shaders │ │Infer.│ │   │    │
              │  │  │  │(wgpu)│ │(CUDA)  │ │(candl)│   │    │
              │  │  │  └──────┘ └────────┘ └─────┘ │   │    │
              │  │  └──────────────────────────────┘   │    │
              │  └─────────────────────────────────────┘    │
              │              GPU COMPUTE LAYER               │
              └──────────────────┬──────────────────────────┘
                              │
              ┌───────────────┴───────────────────────┐
              │        KNOWLEDGE / CONTEXT LAYER       │
              │  ┌─────────┐  ┌──────────┐  ┌───────┐ │
              │  │ PLATO   │  │ Spectral │  │Constr.│ │
              │  │ Rooms   │  │ Index    │  │Engine │ │
              │  │ ┌─────┐ │  │          │  │       │ │
              │  │ │LoRA │ │  │ Codebase │  │Exact  │ │
              │  │ │Adapt│ │  │ spectral │  │arith. │ │
              │  │ │loadr│ │  │ signature│  │       │ │
              │  │ └─────┘ │  │          │  │       │ │
              │  └─────────┘  └──────────┘  └───────┘ │
              └───────────────┬───────────────────────┘
                              │
              ┌───────────────┴───────────────────────┐
              │        FLEET COORDINATION LAYER        │
              │  ┌──────────┐  ┌────────────────────┐ │
              │  │ cocapn   │  │ fleet-spectral-    │ │
              │  │ protocol │  │ health monitor     │ │
              │  └──────────┘  └────────────────────┘ │
              │  ┌──────────────────────────────────┐ │
              │  │  Multi-Warp coordination          │ │
              │  │  Agent-to-agent communication     │ │
              │  │  Distributed task execution       │ │
              │  └──────────────────────────────────┘ │
              └───────────────────────────────────────┘
```

---

## 6. What Would Make the Original Creators' Jaws Drop

### 6.1 Self-Proving Terminal

Warp's terminal already executes commands. Imagine a terminal where:
- Every command's output is **spectrally analyzed** in real-time
- Conservation laws detect anomalies in system behavior
- The terminal can **predict** command failures before execution using FLUX proof certificates
- It's not just executing — it's **reasoning about what it's executing**

This would blow their minds because they built a terminal that uses AI to *suggest* commands. We'd build a terminal that **proves** commands will work before running them.

### 6.2 Living Documentation

Warp has "blocks" (output grouped with input). Imagine:
- Each block carries a **spectral signature** — a mathematical fingerprint of what happened
- Blocks are indexed in PLATO rooms with full semantic understanding
- You can query your terminal history with natural language and get **provably correct** answers
- The terminal **learns** your patterns and adapts (model descent compresses experience)

### 6.3 Agent With a Conscience

Warp's agents use GPT to write code. Our agent would:
- **Formally verify** every change it proposes (FLUX proof certificates)
- **Conserve knowledge** during self-improvement (symplectic optimization)
- **Understand the geometry** of code (geometric algebra on ASTs)
- **Know its own limits** (constraint theory defines what it can't do)
- **Coordinate** with other agents mathematically (cocapn consensus)

### 6.4 Spectral Code Review

When you open a PR, Warp currently shows AI-generated summaries. Our version would:
- Compute the **spectral delta** between branches
- Identify **conservation law violations** (e.g., error handling added but never triggered)
- Visualize code changes as **geometric transformations** in a PLATO room
- Provide **proof certificates** that the change preserves invariants

### 6.5 The Terminal That Improves Itself

Warp is built in Rust. Imagine it:
- Spectrally analyzes its **own codebase** in real-time
- Detects performance degradation through **conservation law violations**
- Suggests optimizations that are **provably correct** (FLUX-verified patches)
- Self-tunes GPU scheduling through **symplectic optimization**
- Maintains its health through **fleet spectral monitoring**

This isn't a feature — it's a **new category of software**. A self-aware, self-improving, mathematically grounded development environment.

---

## 7. Implementation Roadmap

### Phase 0: Foundation (Weeks 1-2)

**Goal:** Set up the fork, understand the codebase deeply, establish build system.

| Task | Files | LOC | Notes |
|------|-------|-----|-------|
| Clone and build Warp locally | — | 0 | `./script/bootstrap && ./script/run` |
| Map all 60+ crates and their dependencies | — | 0 | Documentation only |
| Add `crates/superinstance/` workspace members | `Cargo.toml` | ~50 | New workspace section |
| Create `crates/si_core/` for shared types | New crate | ~500 | Common types for SI integration |
| Add `FeatureFlag::FluxExecution`, `FeatureFlag::SpectralAnalysis` | `crates/warp_core/src/features.rs` | ~20 | Runtime feature toggles |

### Phase 1: FLUX Runtime Integration (Weeks 3-6)

**Goal:** Get FLUX programs running inside Warp's agent loop.

| Task | Files | LOC | Notes |
|------|-------|-----|-------|
| Create `crates/flux_runtime/` | New crate | ~2,000 | FLUX VM bindings |
| Add `FluxBackend` to AI dispatch | `crates/ai/src/` | ~1,500 | Alongside OpenAI/Anthropic |
| FLUX tool type for agent | `crates/ai/src/agent/tool_call.rs` | ~500 | FLUX as a tool provider |
| Isolation platform integration | `crates/isolation_platform/` | ~1,000 | Sandbox FLUX execution |
| MCP server for FLUX tools | `crates/mcp/` extension | ~800 | Auto-expose FLUX programs as MCP tools |
| Tests and documentation | — | ~1,000 | |

**Phase 1 Total:** ~6,800 LOC

### Phase 2: Spectral Analysis Engine (Weeks 5-8)

**Goal:** Spectral analysis of codebases running in Warp.

| Task | Files | LOC | Notes |
|------|-------|-----|-------|
| Create `crates/spectral/` | New crate | ~3,000 | conservation-spectral Rust bindings |
| GPU Laplacian computation | `crates/spectral/src/gpu.rs` | ~1,000 | wgpu compute shaders |
| Extend `repo_metadata` with spectral data | `crates/repo_metadata/` | ~500 | Add spectral signatures to index |
| Spectral health dashboard UI | `app/src/` | ~1,500 | WarpUI components |
| MCP tool for spectral queries | `crates/mcp/` extension | ~400 | |
| Tests | — | ~500 | |

**Phase 2 Total:** ~6,900 LOC

### Phase 3: PLATO Rooms (Weeks 7-10)

**Goal:** Knowledge room system integrated with agent execution.

| Task | Files | LOC | Notes |
|------|-------|-----|-------|
| Create `crates/plato/` | New crate | ~3,000 | Room management, context blending |
| Room UI in Warp | `app/src/` | ~2,000 | Room browser, room settings |
| Context injection into agent | `crates/ai/src/` | ~800 | Room-aware agent execution |
| Room persistence | `crates/persistence/` | ~500 | Store rooms in SQLite |
| Cloud sync for rooms | `crates/cloud_objects/` | ~500 | Share rooms via Drive |
| Tests | — | ~800 | |

**Phase 3 Total:** ~7,600 LOC

### Phase 4: Model-Agnostic Dispatch (Weeks 9-12)

**Goal:** Unified compute backend with intelligent scheduling.

| Task | Files | LOC | Notes |
|------|-------|-----|-------|
| Create `crates/compute_dispatch/` | New crate | ~2,500 | Trait definitions, scheduler |
| Local LLM backend (candle/ONNX) | New crate: `crates/neural_inference/` | ~2,000 | GPU-accelerated inference |
| GPU scheduler | New crate: `crates/gpu_scheduler/` | ~1,500 | Manage GPU memory/compute |
| Backend cost estimation | `crates/compute_dispatch/` | ~500 | |
| Dispatch UI (backend selection) | `app/src/` | ~1,000 | |
| Tests | — | ~1,000 | |

**Phase 4 Total:** ~8,500 LOC

### Phase 5: GPU Compute Integration (Weeks 11-14)

**Goal:** CUDA/geometric algebra/symplectic optimization running on GPU.

| Task | Files | LOC | Notes |
|------|-------|-----|-------|
| Create `crates/cuda_compute/` | New crate | ~2,000 | CUDA kernel wrappers |
| Create `crates/geometric/` | New crate | ~2,000 | ga-core GPU integration |
| Create `crates/symplectic/` | New crate | ~1,500 | symplectic-opt integration |
| Constraint engine | `crates/constraint_engine/` | ~1,500 | constraint-theory-core bindings |
| FLUX JIT compilation | `crates/flux_runtime/` extension | ~2,000 | Cranelift → GPU bytecode |
| Tests | — | ~1,000 | |

**Phase 5 Total:** ~10,000 LOC

### Phase 6: Fleet Coordination (Weeks 13-16)

**Goal:** Multi-Warp coordination and self-monitoring.

| Task | Files | LOC | Notes |
|------|-------|-----|-------|
| Create `crates/fleet/` | New crate | ~2,500 | cocapn protocol integration |
| Fleet health monitoring | `crates/fleet_health/` | ~2,000 | fleet-spectral-health |
| Multi-instance UI | `app/src/` | ~1,500 | Fleet dashboard |
| Agent-to-agent communication | `crates/ai/src/` extension | ~1,000 | |
| Tests | — | ~800 | |

**Phase 6 Total:** ~7,800 LOC

### Phase 7: Polish & Killer Features (Weeks 15-20)

**Goal:** Spectral development environment, self-improvement, jaw-dropping features.

| Task | Files | LOC | Notes |
|------|-------|-----|-------|
| Spectral code review | `app/src/` | ~2,000 | PR analysis with spectral deltas |
| Living documentation | `app/src/` | ~1,500 | Semantic terminal history |
| Model descent integration | `crates/compute_dispatch/` | ~1,500 | Self-improvement loop |
| Proving terminal | `crates/ai/src/agent/` | ~2,000 | Pre-execution proof |
| Self-analysis dashboard | `app/src/` | ~1,500 | Warp analyzing itself |
| Documentation and demos | — | ~2,000 | |

**Phase 7 Total:** ~10,500 LOC

---

### Summary

| Phase | Timeline | LOC | Focus |
|-------|----------|-----|-------|
| 0: Foundation | Weeks 1-2 | ~570 | Setup, understand codebase |
| 1: FLUX Runtime | Weeks 3-6 | ~6,800 | FLUX VM in agent loop |
| 2: Spectral Analysis | Weeks 5-8 | ~6,900 | Code spectral signatures |
| 3: PLATO Rooms | Weeks 7-10 | ~7,600 | Knowledge rooms |
| 4: Model-Agnostic Dispatch | Weeks 9-12 | ~8,500 | Unified compute layer |
| 5: GPU Compute | Weeks 11-14 | ~10,000 | CUDA/GA/symplectic on GPU |
| 6: Fleet Coordination | Weeks 13-16 | ~7,800 | Multi-instance |
| 7: Killer Features | Weeks 15-20 | ~10,500 | Spectral dev env, self-improve |
| **TOTAL** | **~20 weeks** | **~58,670** | |

---

## Key Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| AGPL-3.0 license requires open-sourcing modifications | Fine for ecosystem tools; keep proprietary logic in FLUX programs (not Warp code) |
| Upstream diverges fast (multiple commits/day) | Rebase strategy; keep SI code in separate crates under `crates/si_*` |
| GPU memory contention (UI + compute) | GPU scheduler prioritizes UI; compute runs in idle frames |
| FLUX VM performance vs native LLM calls | Use FLUX for deterministic tasks; keep LLMs for creative tasks |
| 60+ crate workspace = slow builds | Use sccache; only rebuild changed crates |
| WarpUI learning curve (custom framework) | Start with MCP tools (no UI changes needed for Phase 1-2) |

---

## Why Warp Is The Perfect Shell

1. **Already an execution environment** — terminal + agent + tools
2. **Already GPU-accelerated** — wgpu rendering pipeline we can extend for compute
3. **Already has AI integration** — agent mode, MCP, tool calling — we just add backends
4. **Already has indexing** — repo_metadata gives us code understanding to layer spectral analysis on top of
5. **Already has plugin architecture** — MCP tools and skills let us integrate without forking core
6. **Already cross-platform** — macOS, Linux, Windows, WASM
7. **Rust** — same language as conservation-spectral, flux-vm, ga-core, constraint-theory-core
8. **Active community** — well-maintained upstream, contribution workflow exists
9. **AGPL** — aligns with open ecosystem philosophy
10. **OpenAI sponsorship** — ironic: we're replacing their backend with FLUX

**The terminal is the most natural home for everything we're building.** Every developer already lives in one. We're not asking them to adopt a new tool — we're making their existing tool mathematically aware, provably correct, and self-improving.
