# MODEL DESCENT LIBRARY — Roadmap

**Date:** 2026-05-28
**Status:** Design Document
**Vision:** *Algorithms absorb intelligence over time, reducing inference cost to zero.*

---

## The Core Idea

Neural networks grow during training and stay big during inference. That's wasteful. Not all neurons are equally "alive" — some have converged onto stable representations, others are still exploring. Conservation-aware spectral analysis can *tell the difference*. Converged regions compress without loss. Exploring regions get more capacity. The model gets **smaller and faster over time**, not bigger.

This is the Tension-Graph Laplacian applied to neural network layers: build a dynamics-geometry compatibility operator from activation transitions × weight similarity, decompose it spectrally, and use the conservation ratio to decide when a region has "learned its role" and can be compressed.

From *THE-LATENT-ABSTRACTION.md*: *"The eigenvectors of this operator decompose the system into modes ranked by how well dynamics and structure agree."* In neural networks, the "dynamics" are activation patterns flowing through layers; the "geometry" is the weight/attribute similarity structure. When they align (low eigenvalue), the layer has converged. When they conflict (high eigenvalue), the layer is still learning.

From *SYNOPTIC-VIEW.md*: *"cuda-model-descent proves that inference can conserve intelligence."* This document is the blueprint for that proof.

---

## Mathematical Foundation

### The Neuron Conservation Operator

For a layer with N neurons, define:

```
W[i,j] = P(neuron_i → neuron_j coactivation) × similarity(w_i, w_j)
```

Where:
- **P(i→j)**: coactivation probability from batch statistics — how often neurons i and j fire together
- **similarity(w_i, w_j)**: cosine similarity of outgoing weight vectors — how similar their "downstream effect" is

The Laplacian:

```
L = D - W,  where D[i,i] = Σ_j W[i,j]
```

Eigendecomposition:

```
L φ_k = λ_k φ_k,  k = 1...N
```

### Conservation Ratio per Layer

For layer activations f (an N-dimensional attribute vector) across a spectral window of B batches:

```
CR(layer) = 1 - Var(∇²f) / Var(f)
```

Where ∇²f is the graph Laplacian applied to the activation attribute. High CR (near 1.0) = activations are smooth on the coactivation graph = the layer has converged. Low CR = the layer is still exploring.

### Why This Works (from the latent abstraction)

*"The condition for conservation detection is NOT that the system has conservation. It's that the system has ANISOTROPIC structure that the dynamics partially respects."*

Neural network layers in a trained model are deeply anisotropic — features cluster, dead neurons exist, weight matrices have low effective rank. This is exactly the condition where the Tension-Graph Laplacian reveals structure. The Laplacian doesn't just detect convergence; it tells you *which neurons* have converged (high CR eigenvectors) and *which directions* are still learning (low CR eigenvectors).

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│              MODEL DESCENT LIBRARY               │
│                                                  │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │ Conservation │  │ Compression Engine        │ │
│  │ Analyzer     │  │                          │ │
│  │              │  │ ┌────────┐ ┌───────────┐ │ │
│  │ • Graph      │──▶│ │Quantize│ │  Prune    │ │ │
│  │   Builder    │  │ └────────┘ └───────────┘ │ │
│  │ • Laplacian  │  │ ┌────────┐ ┌───────────┐ │ │
│  │ • Eigen      │  │ │Distill │ │  Fold     │ │ │
│  │ • CR Monitor │  │ └────────┘ └───────────┘ │ │
│  └──────────────┘  └──────────────────────────┘ │
│                                                  │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │ Training     │  │ Rust / CUDA Core         │ │
│  │ Loop (Python)│  │                          │ │
│  │              │  │ • GPU Laplacian           │ │
│  │ PyTorch      │  │ • Eigendecomposition      │ │
│  │ Integration  │  │ • Real-time CR tracking   │ │
│  └──────────────┘  │ • Compression kernels     │ │
│                     └──────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

---

## Component Design

### 1. Conservation-Aware Training Loop

The training loop builds a spectral picture of each layer's convergence state every N batches, then applies compression or expansion decisions.

```python
for epoch in epochs:
    for batch in dataloader:
        y = model(x)
        loss = criterion(y, target)
        
        # Accumulate coactivation statistics
        for layer in model.layers:
            analyzer.observe(layer.activations, layer.weights)
        
        loss.backward()
        optimizer.step()
    
    # Spectral analysis (once per epoch for efficiency)
    for layer in model.layers:
        graph = analyzer.build_graph(layer)          # coactivation × weight_similarity
        laplacian = analyzer.build_laplacian(graph)
        eigenvalues, eigenvectors = analyzer.decompose(laplacian, top_k=32)
        cr[layer] = analyzer.conservation_ratio(eigenvalues, layer.activations)
    
    # Compression / expansion decisions
    for layer in model.layers:
        if cr[layer] > compress_threshold:
            engine.compress(layer, cr[layer])
        elif cr[layer] < expand_threshold:
            engine.expand(layer)  # add neurons for exploration
    
    log(f"Epoch {epoch}: {trainer.param_count()} params, "
        f"avg CR = {trainer.avg_conservation():.3f}")
```

**Key insight from the latent abstraction:** Conservation should be computed *per mode*, not just per layer. A layer can have high CR in some eigenvector directions (those subspaces are converged) and low CR in others (still learning). Fine-grained compression targets the high-CR subspaces within a layer, not the whole layer.

### 2. Compression Strategies

Four levels, ranked by aggressiveness. The conservation ratio determines which level applies:

| Level | Strategy | CR Threshold | Mechanism |
|-------|----------|-------------|-----------|
| **1** | Quantization | 0.70 – 0.85 | FP32 → FP16 → INT8 for converged weights |
| **2** | Pruning | 0.85 – 0.92 | Remove neurons in perfectly conserved subspaces (near-zero eigenvalue modes) |
| **3** | Distillation | 0.92 – 0.97 | Replace converged sublayer with lookup table / SVD factorization |
| **4** | Folding | 0.97+ | Merge consecutive layers if CR is near 1.0 (they're computing the same thing) |

**Level 1 — Quantization:**

High-conservation layers have stable activation distributions. Quantize them progressively:

```python
def quantize_layer(layer, cr, current_dtype):
    if cr > 0.70 and current_dtype == torch.float32:
        return layer.half()  # FP32 → FP16
    if cr > 0.80 and current_dtype == torch.float16:
        return quantize_to_int8(layer)  # FP16 → INT8
    return layer
```

**Level 2 — Pruning:**

Neurons corresponding to low-eigenvalue eigenvectors are in perfectly conserved subspaces — their activation patterns are predictable from the rest. Remove them:

```python
def prune_layer(layer, eigenvectors, eigenvalues, cr):
    # Identify neurons in conserved subspaces
    conserved_mask = eigenvalues < pruning_threshold
    redundant_neurons = identify_redundant(layer, eigenvectors[:, conserved_mask])
    
    # Prune and reconnect
    pruned_layer = remove_neurons(layer, redundant_neurons)
    return finetune_reconnections(pruned_layer, dataloader, steps=100)
```

**Level 3 — Distillation:**

When CR > 0.92, the layer's function is nearly deterministic in its conserved subspaces. Replace it:

```python
def distill_layer(layer, analyzer):
    # Build input-output map from batch statistics
    io_map = analyzer.build_io_map(layer)
    
    # Factorize via SVD or fit a lookup table
    if effective_rank(io_map) < max_rank:
        return SVDFactorization.from_layer(layer, rank=effective_rank)
    else:
        return LookupTable.from_io_map(io_map)
```

**Level 4 — Folding:**

When consecutive layers both have CR > 0.97, they're computing nearly the same transformation. Merge them:

```python
def fold_layers(layer_a, layer_b, cr_a, cr_b):
    if cr_a > 0.97 and cr_b > 0.97:
        # layer_b ∘ layer_a ≈ single linear map
        merged_weight = layer_b.weight @ layer_a.weight
        merged_bias = layer_b.weight @ layer_a.bias + layer_b.bias
        return nn.Linear(merged_weight.shape[1], merged_weight.shape[0])
    return layer_a, layer_b
```

### 3. API Design

```python
from model_descent import ConservationTrainer, CompressionConfig

# Basic usage
trainer = ConservationTrainer(
    model=my_model,
    compression=CompressionConfig(
        threshold=0.85,          # compress when conservation > 85%
        strategy='progressive',  # gradually increase compression level
        min_params=0.1,          # never compress below 10% of original params
        max_compression_rate=0.1, # compress at most 10% of params per epoch
        finetune_steps=100,      # finetune after each compression step
    ),
    spectral= SpectralConfig(
        window=100,              # analyze last 100 batches
        top_k=32,                # top-k eigenvectors to compute
        eigen_backend='gpu',     # use GPU for eigendecomposition
    ),
)

# Train with automatic descent
for epoch in range(100):
    metrics = trainer.train_one_epoch(train_loader)
    print(f"Params: {metrics.param_count:,} / "
          f"CR: {metrics.avg_conservation:.3f} / "
          f"Accuracy: {metrics.accuracy:.2%}")

# Export the compressed model
compressed_model = trainer.export_model()
compressed_model.eval()
# Inference is now faster — fewer params, quantized weights, folded layers
```

**Advanced: per-layer inspection**

```python
# Inspect conservation state of each layer
report = trainer.conservation_report()
for layer_name, info in report.layers.items():
    print(f"{layer_name}: CR={info.conservation:.3f}, "
          f"compression={info.level}, "
          f"params={info.current_params}/{info.original_params}")

# Export spectral analysis for visualization
eigen_data = trainer.export_spectral_data()
# Contains eigenvalues, eigenvectors, CR(k) per mode for each layer
```

**Advanced: manual control**

```python
# Override automatic compression for specific layers
trainer.set_layer_policy('layer3', policy='freeze')     # never compress
trainer.set_layer_policy('layer7', policy='aggressive')  # compress aggressively

# Manual compression trigger
trainer.compress_now(threshold=0.90)
```

### 4. Rust / CUDA Core

Python handles the training loop and PyTorch integration. The heavy linear algebra lives in Rust + CUDA for production speed.

**Rust library (`model-descent-core`):**

```rust
/// Conservation analyzer — GPU-accelerated
pub struct ConservationAnalyzer {
    spectral_window: usize,
    top_k: usize,
    gpu_context: CudaContext,
}

impl ConservationAnalyzer {
    /// Build coactivation graph from batch of activations
    pub fn build_graph(&self, activations: &Tensor, weights: &Tensor) -> Graph;
    
    /// Compute graph Laplacian (sparse, on GPU)
    pub fn build_laplacian(&self, graph: &Graph) -> SparseLaplacian;
    
    /// Top-k eigendecomposition (LOBPCG or Lanczos on GPU)
    pub fn decompose(&self, laplacian: &SparseLaplacian, k: usize) -> EigenDecomposition;
    
    /// Conservation ratio from eigenvalues + activation variance
    pub fn conservation_ratio(&self, eigen: &EigenDecomposition, activations: &Tensor) -> f64;
}

/// Real-time conservation monitor for inference
pub struct ConservationMonitor {
    analyzer: ConservationAnalyzer,
    thresholds: CompressionThresholds,
    buffer: RingBuffer<ActivationSnapshot>,
}

impl ConservationMonitor {
    /// Call during inference — lightweight, amortized O(1) per batch
    pub fn observe(&mut self, activations: &[Tensor]);
    
    /// Returns true if conservation has crossed the compression threshold
    pub fn should_compress(&self) -> Option<Vec<LayerAction>>;
}
```

**CUDA kernels:**

```
cuda_model_descent/
├── kernels/
│   ├── coactivation.cu          # Coactivation matrix from activation tensors
│   ├── laplacian.cu             # Sparse Laplacian construction
│   ├── eigendecompose.cu        # Top-k LOBPCG eigendecomposition
│   ├── conservation_ratio.cu    # CR computation from eigenvalues
│   ├── quantize.cu              # FP32→FP16→INT8 quantization
│   └── prune.cu                 # Structured pruning by eigenvector mask
├── rust/
│   ├── src/
│   │   ├── lib.rs               # FFI bridge to CUDA kernels
│   │   ├── analyzer.rs          # ConservationAnalyzer
│   │   ├── monitor.rs           # Real-time ConservationMonitor
│   │   └── compression.rs       # Compression engine
│   └── Cargo.toml
└── python/
    ├── model_descent/
    │   ├── __init__.py
    │   ├── trainer.py            # ConservationTrainer
    │   ├── config.py             # CompressionConfig, SpectralConfig
    │   ├── strategies/
    │   │   ├── quantize.py
    │   │   ├── prune.py
    │   │   ├── distill.py
    │   │   └── fold.py
    │   └── pybind.rs             # PyO3 bindings to Rust core
    └── setup.py
```

### 5. Killer Demo: ResNet-18 on CIFAR-10

The proof-of-concept that makes people pay attention.

**Setup:**
- ResNet-18 (11.7M params) trained on CIFAR-10
- Standard training for 50 epochs to establish baseline accuracy
- Then enable Model Descent for epochs 51–100
- Target: compress to ~1M params while maintaining >93% accuracy

**Expected trajectory:**

```
Epoch  50: 11,689,512 params | CR=0.62 | Acc=94.2%  ← baseline
Epoch  55: 11,689,512 params | CR=0.71 | Acc=94.1%  ← quantizing early layers
Epoch  60:  9,412,864 params | CR=0.78 | Acc=94.0%  ← pruning begins
Epoch  70:  5,234,176 params | CR=0.86 | Acc=93.8%  ← heavy pruning
Epoch  80:  2,847,232 params | CR=0.91 | Acc=93.5%  ← distillation kicks in
Epoch  90:  1,523,456 params | CR=0.94 | Acc=93.1%  ← folding
Epoch 100:    934,816 params | CR=0.97 | Acc=92.8%  ← 12.5× compression
```

**Visualization outputs:**
1. Param count vs accuracy curve (the "descent" — params go down, accuracy stays flat)
2. Conservation ratio heatmap per layer over time (watch layers go from blue→red)
3. Eigenvalue spectrum animation per layer (watch eigenvalues collapse as layers converge)
4. Inference latency chart (should drop proportionally to param count)

**Hardware:**
- Training: single GPU (RTX 3090 or better)
- Inference benchmark: CPU + GPU timing
- Edge deployment: export to ONNX, benchmark on Raspberry Pi

### 6. Applications Beyond Image Classification

**LLM Inference Optimization:**

The most commercially relevant application. LLM inference is expensive because every layer runs for every token. But not every layer "learns" equally:

```python
# Apply model descent to a transformer
from model_descent import ConservationTrainer

trainer = ConservationTrainer(llm_model)

# After a pretraining/fine-tuning run:
# - Early attention layers (tokenization) converge fast → fold/prune
# - Middle layers (context) are moderately conserved → quantize
# - Late layers (output projection) are still exploring → leave alone
```

Expected: 2-4× inference speedup with <1% perplexity increase for well-trained LLMs.

**Mixture-of-Experts (MoE) Compression:**

MoE models have underused experts. Conservation analysis identifies them:

```python
# Expert-level conservation analysis
for expert_id in range(num_experts):
    expert = moe_model.get_expert(expert_id)
    cr = analyzer.compute_conservation(expert, recent_batches)
    if cr > 0.90:
        print(f"Expert {expert_id} is converged → candidate for compression")
```

Expected: compress or remove 30-50% of experts in trained MoE models.

**Reinforcement Learning (Policy Compression):**

RL policies converge onto repeated behaviors. The converged portion of the policy network is compressible:

```python
# Online policy compression during training
trainer = ConservationTrainer(policy_network)
for step in environment:
    action = trainer.act(observation)
    trainer.observe(conservation_update_interval=1000)
    if step % 10000 == 0:
        trainer.apply_compression()
```

Expected: 5-10× policy compression for deployment to edge devices (robots, drones).

**Edge Deployment (Automatic Model-to-Hardware Fitting):**

Specify the hardware, let model descent figure out how much to compress:

```python
from model_descent import EdgeDeploy

deployer = EdgeDeploy(
    model=trained_model,
    target=EdgeDevice.RASPBERRY_PI_4,  # 4GB RAM, ARM Cortex-A72
    max_latency_ms=50,
    min_accuracy=0.90,
)
compressed = deployer.fit()
# Automatically compresses until the model fits the hardware budget
```

---

## Roadmap

### Phase 1: PyTorch Plugin (Weeks 1-2)

**Goal:** Working Python library that compresses a model during training.

- [ ] Conservation analyzer: graph builder, Laplacian, eigendecomposition (NumPy/SciPy)
- [ ] Compression strategies: quantize, prune, distill, fold
- [ ] `ConservationTrainer` class with PyTorch integration
- [ ] Unit tests on small models (MLP on MNIST)
- [ ] CI pipeline (pytest, mypy)

**Deliverable:** `pip install model-descent` that works with any PyTorch model.

### Phase 2: Killer Demo + Validation (Weeks 3-4)

**Goal:** The ResNet-18 on CIFAR-10 demo that proves the concept.

- [ ] Run baseline training (no compression)
- [ ] Run model descent training
- [ ] Generate comparison plots (params, accuracy, latency)
- [ ] Benchmark on CPU and edge device
- [ ] Write technical blog post / demo notebook

**Deliverable:** Reproducible notebook + plots that show 10× compression with <2% accuracy loss.

### Phase 3: Rust Core (Weeks 5-12)

**Goal:** Production-speed conservation analysis.

- [ ] Rust library with CUDA backend
- [ ] GPU-accelerated coactivation graph construction
- [ ] Sparse Laplacian on GPU
- [ ] Top-k LOBPCG eigendecomposition (CUDA)
- [ ] PyO3 bindings for Python
- [ ] Benchmark: 100× speedup over NumPy for large layers

**Deliverable:** `model-descent-core` crate on crates.io, Python bindings via pip.

### Phase 4: CUDA Kernels (Weeks 13-24)

**Goal:** Full GPU pipeline — analysis, compression, and inference in CUDA.

- [ ] Custom CUDA kernels for each compression strategy
- [ ] Fused kernel: observe → analyze → compress in one GPU pass
- [ ] Real-time `ConservationMonitor` for production inference
- [ ] Memory-efficient streaming eigendecomposition (no full Laplacian in memory)
- [ ] Multi-GPU support for large models

**Deliverable:** CUDA library callable from PyTorch, TensorFlow, and standalone C/Rust.

### Phase 5: Production & Applications (Weeks 25-28)

**Goal:** Real-world validation and domain-specific applications.

- [ ] LLM inference optimization (LLaMA/Mistral)
- [ ] MoE expert compression
- [ ] Edge deployment pipeline
- [ ] Integration with ONNX Runtime and TensorRT
- [ ] Production monitoring dashboard
- [ ] Performance benchmarks and case studies

**Deliverable:** Production-ready library with documentation, benchmarks, and at least two real-world case studies.

---

## Open Research Questions

1. **Conservation as training signal.** From the latent abstraction: *"Conservation as an objective function, not just a diagnostic."* Can we add a conservation loss term that *guides* the network toward compressible representations? This would be a regularizer: `loss_total = loss_task + λ · (1 - CR)`. If this works, the model learns representations that are both accurate and compressible by construction.

2. **Multi-scale conservation.** The latent abstraction identifies this as a frontier: compute conservation at micro (neuron), meso (layer block), and macro (whole model) scales. Does multi-scale analysis reveal hierarchical structure in neural networks? Can we recover which layers form functional groups?

3. **Temporal conservation dynamics.** CR(t) is a time series. Its derivative detects phase transitions in training (analogous to modulation detection in music). Can we predict when a model is about to converge or collapse by watching dCR/dt?

4. **Conservation transfer.** If we compute the Laplacian on a well-trained model, can we transfer it to a new model being trained on a related task? This would be "structural pre-training" — the new model gets a head start on which representations should be conserved.

5. **The Ising warning.** From the latent abstraction: *"Random walks on symmetric graphs WILL NOT show conservation."* This means model descent won't work on perfectly random or highly symmetric architectures. We need a diagnostic for "does this architecture have enough anisotropy for conservation detection?" before promising compression results.

6. **Optimal compression schedule.** How aggressively should we compress? Too fast → accuracy collapses. Too slow → wasted compute. The conservation ratio gives us a signal, but the mapping from CR to optimal compression rate is an open problem.

---

## Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Compression destroys rare-but-important features | Medium | High | Per-mode CR analysis — only compress modes, not whole layers |
| Eigendecomposition too slow for large layers | Medium | Medium | Top-k methods (LOBPCG), streaming approximation, amortize over epochs |
| Conservation doesn't correlate with compressibility | Low | Fatal | Kill condition — if the demo fails, the hypothesis fails. Honest negative result. |
| Works on CNNs but not transformers | Medium | High | Transformer-specific graph construction (attention heads as subgraphs) |
| Finetuning after compression is expensive | Low | Medium | Progressive compression with small finetune steps between epochs |

---

## The Connection to the SuperInstance Ecosystem

Model Descent is the *cuda-model-descent* component from the Synoptic View. It sits in Layer 5 (The Fleet), connecting to:

- **constraint-theory-core**: The conservation ratio is a constraint — it measures how tightly a layer's activations are constrained to a low-dimensional manifold. The compression step snaps the layer to that manifold (like a Pythagorean lattice snap, but for neural weights).
- **cuda-emergence**: Fleet-wide spectral analysis detects conservation in collective behavior. Model descent detects conservation in individual model layers. Same math, different scale.
- **cuda-deliberation**: The Consider/Resolve/Forfeit protocol reaches consensus through confidence propagation. Model descent's compression decisions are a form of "resolved" consensus — the conservation ratio IS the confidence that a layer has converged.
- **cuda-necropolis**: Agents die and their knowledge is harvested. Compressed neurons are "dead" — their function is harvested into a smaller representation. The Innovation Cycle applies: neurons are born (expansion), codify (convergence), become ubiquitous (high CR), get compressed (death), and their harvested function seeds the compressed model.
- **cuda-dream-cycle**: Idle agents consolidate memories. Model descent's finetuning step after compression is the same pattern — the model "dreams" (finetunes) to consolidate its compressed representation.
- **The Innovation Cycle**: Discovery → Codification → Ubiquity → Boredom → Rebellion → Discovery. In model descent: new neurons (discovery) → learn stable features (codification) → become redundant (ubiquity) → CR exceeds threshold (boredom) → get pruned (rebellion) → new capacity allocated (discovery).

Model descent IS the Innovation Cycle applied to neural network architecture. The model doesn't just learn — it evolves, compresses, and reinvests its saved capacity into new exploration. This is the same cycle that governs music, languages, galaxies, and agent fleets.

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Compression ratio (ResNet-18 / CIFAR-10) | >10× param reduction |
| Accuracy retention | >95% of baseline |
| Inference speedup | >5× on GPU, >10× on CPU |
| Conservation correlation | CR should predict compressibility with r > 0.9 |
| Overhead | <15% training time increase for conservation analysis |
| Phase 1 delivery | 2 weeks |
| Phase 3 delivery | 3 months |

---

*"Algorithms absorb intelligence over time, reducing inference cost to zero."*

*Not because we force them to shrink, but because conservation detects what has already converged. The intelligence is already compressed — we just need to measure it.*
