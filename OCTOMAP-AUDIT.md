# OctoMap Integration Audit — SuperInstance Ecosystem

**Repo:** https://github.com/SuperInstance/octomap  
**Upstream:** https://github.com/OctoMap/octomap  
**Fork Date:** 2026-05-28  
**Auditor:** Subagent (audit-octomap)  
**Date:** 2026-05-28

---

## 1. What OctoMap Is

OctoMap is an **efficient probabilistic 3D mapping framework based on octrees**, developed at the University of Freiburg (Kai M. Wurm, Armin Hornung). It's the de facto standard for occupancy mapping in robotics, deeply integrated into ROS/ROS2.

### Core Architecture

**Data Structure:** Hierarchical octree with max depth 16 (supporting coordinates up to ±327.68m at 1cm resolution).

**Key classes (inheritance chain):**
```
AbstractOcTreeNode
  └── OcTreeDataNode<T>       — generic node holding value T, 8 children
        └── OcTreeNode         — T=float, stores log-odds occupancy

AbstractOcTree
  └── AbstractOccupancyOcTree  — adds prob_hit/miss, clamping thresholds
        └── OcTreeBaseImpl<NODE, INTERFACE>  — core tree ops (search, insert, delete, prune, ray casting, iterators)
              └── OccupancyOcTreeBase<NODE>  — pointcloud insertion, occupancy updates, BBX filtering, change detection
                    └── OcTree               — concrete occupancy octree (resolution param)
                    └── ColorOcTree          — extends with RGB color per voxel
```

**Probabilistic Model:**
- Occupancy stored as **log-odds** (`float`): `logodds(p) = log(p / (1-p))`
- Binary Bayesian update: `value += logodds_update`, clamped to `[clamping_thres_min, clamping_thres_max]`
- Threshold classification: `value > occ_prob_thres_log` → occupied
- Hit/miss probabilities: `prob_hit_log` ≈ 0.7 log-odds, `prob_miss_log` ≈ 0.4 log-odds

**Key Operations:**
- **Pointcloud insertion** (`insertPointCloud`): Bresenham-style ray casting from sensor origin through free space to occupied endpoints. OpenMP-parallelized.
- **Ray casting** (`computeRayKeys`): Discretizes 3D ray into OcTreeKey sequence
- **Pruning**: Lossless compression — collapse nodes whose 8 children have identical values
- **BBX limiting**: Bounding box filtering during insertion
- **Change detection**: Track which keys changed since last reset
- **Marching cubes** (`MCTables.h`): Surface extraction from occupancy field

**Math utilities:** Custom `Vector3`, `Quaternion`, `Pose6D` — lightweight, no external 3D math dependency.

**Submodules:**
- `octomap/` — core library (BSD license)
- `octovis/` — Qt visualization (GPL)
- `dynamicEDT3D/` — dynamic Euclidean distance transforms

### Source File Map

**Headers** (`octomap/include/octomap/`):
| File | Role |
|------|------|
| `OcTreeNode.h` | Occupancy node: log-odds storage, get/set, child aggregation |
| `OcTreeDataNode.h` | Template node: `T value`, children array, serialization |
| `OcTreeKey.h` | 16-bit key-based addressing (key[0..2] → x,y,z) |
| `OcTreeBaseImpl.h` | Core tree: search, delete, prune, expand, ray cast, iterators |
| `OccupancyOcTreeBase.h/.hxx` | Pointcloud insertion, update logic, BBX, change detection |
| `AbstractOcTree.h` | Tree type registry, serialization interface |
| `AbstractOccupancyOcTree.h` | Probabilistic params (hit/miss/thresholds), query interface |
| `OcTree.h` | Concrete `OcTree` class |
| `ColorOcTree.h` | Color-augmented occupancy tree |
| `CountingOcTree.h` | Non-probabilistic counting tree |
| `MapCollection.h/.hxx` | Multi-map management |
| `MCTables.h` | Marching cubes lookup tables |
| `ScanGraph.h` | Scan graph for SLAM-style mapping |
| `Pointcloud.h` | Basic pointcloud container |
| `octomap_types.h` | Typedefs (OcTreeKey, KeySet, KeyRay, etc.) |
| `octomap_utils.h` | `logodds()` and `probability()` conversions |
| `math/Vector3.h` | 3D vector arithmetic |
| `math/Quaternion.h` | Quaternion representation |
| `math/Pose6D.h` | 6DoF pose (3D translation + quaternion) |
| `math/Utils.h` | Coordinate transforms |

**Sources** (`octomap/src/`):
| File | Role |
|------|------|
| `OcTree.cpp` | Minimal — just constructor with resolution |
| `OcTreeNode.cpp` | `addValue`, `getMeanChildLogOdds`, `getMaxChildLogOdds` |
| `OcTreeDataNode.cpp` | Serialization, copy, equality |
| `AbstractOcTree.cpp` | Tree factory, file I/O |
| `AbstractOccupancyOcTree.cpp` | Parameter getters/setters, query helpers |
| `ColorOcTree.cpp` | Color query methods |
| `CountingOcTree.cpp` | Counting operations |
| `OcTreeBaseImpl.cpp` | Ray casting (`computeRayKeys`, `computeRay`), tree traversal, pruning, I/O |

---

## 2. Fork Analysis

**Status: Clean fork, zero modifications.**

- Created: 2026-05-28T19:09:21Z (today)
- Single branch: `devel` (matches upstream default)
- 0 stars, 0 forks, 0 issues, 0 PRs
- Issues disabled (`has_issues: false`)
- No additional branches
- `pushed_at` matches upstream last push (2026-02-08)

**Conclusion:** This is a pristine fork — no SuperInstance-specific commits yet. Clean slate for integration work.

---

## 3. Integration Points — Where Our Science Plugs In

### 3a. Constraint-Theory-Core → Octree Node Representation

**The Problem:** OctoMap stores log-odds as IEEE 754 `float` (32-bit). After millions of Bayesian updates, floating-point rounding errors accumulate:
- `logodds(p) = log(p/(1-p))` uses `log()` and `exp()` — both introduce rounding
- `value += log_odds_update` repeated millions of times → drift
- Two robots with identical sensor data can produce subtly different maps due to FPU ordering
- `std::min/std::max` clamping against float thresholds is non-associative

**Our Fix: Lattice-Snapped Probabilities**

Replace the `float value` in `OcTreeDataNode<T>` with a constraint-lattice representation:
- Map log-odds space onto a discrete lattice (e.g., rational approximations)
- Updates become lattice-snap operations: deterministic, no rounding
- Same sensor data → identical map, guaranteed (deterministic maps across all robots)
- The octree decomposition itself IS a constraint lattice: each node at depth d corresponds to a lattice point in position-occupancy space

**Key files to modify:**
| File | Change | Effort |
|------|--------|--------|
| `OcTreeDataNode.h` | Template `T` → lattice value type (or specialize `T=LatticeValue`) | Low |
| `OcTreeNode.h` | Replace `float` log-odds with lattice-snapped log-odds | Low |
| `octomap_utils.h` | Lattice-aware `logodds()` / `probability()` | Low |
| `OccupancyOcTreeBase.hxx` | `updateNodeRecurs` — use lattice update instead of float add | Medium |
| `AbstractOccupancyOcTree.h` | Clamping thresholds become lattice points | Low |

**Effort:** ~2 weeks for core, ~4 weeks for full validation

---

### 3b. Conservation Spectral → Map Quality / Anomaly Detection

**The Idea:** Build a tension graph from octree update patterns.

- Nodes that are updated together (same scan, same ray) → edges in a "co-update graph"
- Apply conservation spectral analysis to this graph
- **High conservation** = smooth, coherent updates (normal operation)
- **Conservation drops** = anomalies — sensor errors, dynamic objects, SLAM drift

**Specific Applications:**
- **Loop closure validation:** Conservation should spike if the "revisited" area doesn't match prior map
- **Dynamic object detection:** Regions with persistently low conservation are dynamic
- **Sensor fault detection:** Sudden conservation drop across the whole map = sensor issue
- **Fiedler vector partitioning:** Automatically separates "static regions" from "dynamic regions"

**Integration points:**
| Component | File | Role |
|-----------|------|------|
| Co-update tracker | New: `ConservationTracker.h` | Tracks which OcTreeKeys update together |
| Spectral analysis | New: `ConservationSpectral.h` | Builds Laplacian, computes conservation from update graph |
| Anomaly reporter | New: `MapQualityMonitor.h` | Thresholds on conservation, reports anomalies |
| Integration hook | `OccupancyOcTreeBase.hxx` → `insertPointCloud` | Feed updates to tracker |
| Change detection | `OccupancyOcTreeBase.h` → `changed_keys` | Already tracked! Natural data source |

**Key insight:** OctoMap already has `use_change_detection` and `changed_keys`. We can build on this directly — the infrastructure for tracking "what changed" is already there.

**Effort:** ~3-4 weeks

---

### 3c. Geometric Algebra (ga-core) → 3D Spatial Operations

**Current State:** OctoMap implements its own 3D math:
- `Vector3.h`: dot, cross, norm, rotate
- `Quaternion.h`: quaternion multiplication, normalization
- `Pose6D.h`: 6DoF transform (translation + quaternion)
- Ray casting: Bresenham-style 3D line discretization in `OcTreeBaseImpl.cpp`
- Collision detection: node bounding box intersection
- Nearest neighbor: search with radius

**GA Replacement (Cl(3,1) Conformal Geometric Algebra):**

| Current Operation | GA Equivalent | Advantage |
|---|---|---|
| Ray casting (Bresenham) | Meet operation (line ∧ plane) | Exact intersection, no discretization artifacts |
| Collision detection | Join operation (sphere ∧ box) | Unified for all primitives |
| Nearest neighbor | Project (point onto surface) | Geometrically exact |
| Pose transform (Pose6D) | Motor (rotor + translator) | Single operation for rotation+translation |
| BBX test | Outer product test | Simpler, unified |

**This is NOT just "swap Vector3 for multivector."** The gains come from:
1. **Eliminating the ray discretization** — `computeRayKeys` currently uses Bresenham which can miss voxels. GA meet gives exact ray-voxel intersections.
2. **Unified collision primitives** — currently only supports axis-aligned boxes. GA supports spheres, cylinders, arbitrary convex shapes as first-class objects.
3. **Rigid body transforms** — `Pose6D` → conformal motor. More numerically stable for chained transforms (SLAM).

**Files to modify:**
| File | Change | Effort |
|------|--------|--------|
| `math/Vector3.h` | Replace with GA multivector wrapper (or keep for compat, add GA backend) | Medium |
| `math/Quaternion.h` | Map to rotor in Cl(3,0) or Cl(3,1) | Low |
| `math/Pose6D.h` | Map to conformal motor | Medium |
| `OcTreeBaseImpl.cpp` | `computeRayKeys` → GA-based ray-voxel intersection | High |
| `OcTreeBaseImpl.cpp` | `computeRay` → same | Medium |
| New: `GASpatialOps.h` | Unified meet/join/project operations | Medium |

**Migration strategy:** Keep existing API, add GA backend that produces same OcTreeKey outputs but via exact geometric computation rather than Bresenham approximation.

**Effort:** ~6-8 weeks for full GA spatial ops; ~3 weeks for ray casting alone

---

### 3d. Sheaf Theory → Multi-Robot Map Fusion

**The Problem:** Multiple robots mapping the same space need to merge octrees. Current approach: naive overlay, which breaks when maps disagree.

**Sheaf-Theoretic Solution:**

Define a sheaf where:
- **Base space:** The physical environment (topological space)
- **Sections over open sets:** Each robot's local map (octree subtree)
- **Restriction maps:** How local maps project to smaller regions
- **Compatibility condition:** Two robots' maps agree on their overlap

**Cohomological Analysis:**
- **H⁰(X, F) = Globally consistent merged map** — if this is the only non-zero group, all maps agree
- **H¹(X, F) ≠ 0 → Obstructions exist** — maps disagree somewhere
- The support of H¹ tells you **WHERE** they disagree — automatic conflict detection
- **Sheaf Laplacian:** Gives optimal merging weights — weighted average of overlapping maps

**Implementation Architecture:**
```
Robot A octree ──┐
                 ├── Sheaf Fusion Layer ──→ Global OctoMap
Robot B octree ──┤         │
                 │    H¹ computation
Robot C octree ──┘    (conflict detection)
```

**Files to create:**
| New File | Role |
|----------|------|
| `SheafFusion.h` | Define the sheaf on octree overlap regions |
| `SheafCohomology.h` | Compute H⁰, H¹ for conflict detection |
| `SheafLaplacian.h` | Optimal merge weights |
| `MultiRobotMapServer.h` | Coordinate fusion across robot maps |

**Files to modify:**
| File | Change |
|------|--------|
| `MapCollection.h/.hxx` | Add sheaf-aware merge method |
| `OcTreeBaseImpl.h` | Add `computeOverlap()` method for detecting overlapping regions |
| `OccupancyOcTreeBase.hxx` | Add `fuseWith()` method |

**Effort:** ~8-12 weeks (this is research-grade work)

---

### 3e. Persistent Homology → Topological Map Analysis

**This is the most natural integration — the octree IS a filtration.**

**Key Insight:** An octree at various depths forms a natural filtration:
- Depth 0: Single node (the whole space)
- Depth 1: 8 octants
- ... increasing resolution ...
- Depth 16: Full resolution voxels

**Applying Persistent Homology:**
- Filter by occupancy threshold: as you sweep from "definitely free" to "definitely occupied", topology changes
- **β₀ (connected components):** How many separate occupied regions exist?
- **β₁ (1-cycles/tunnels):** Doorways, corridors, archways
- **β₂ (2-cycles/voids):** Enclosed rooms, cavities

**Practical Output:**
- **Persistence diagram:** Shows which topological features are "real" (long bars) vs noise (short bars)
- **Topological skeleton:** The navigationally-relevant structure (doorways connect rooms, corridors connect areas)
- This is **far more useful for navigation** than raw occupancy — a robot doesn't need to know every voxel, just the topology

**Integration:**
| Component | Implementation |
|-----------|---------------|
| Filtration builder | Traverse octree at each depth level, build simplicial complex from occupied voxels |
| Boundary matrix | From simplicial complex → chain complex |
| Persistence computation | Reduce boundary matrix (standard PH algorithm) |
| Betti number tracker | Track β₀, β₁, β₂ as function of occupancy threshold |
| Topological skeleton | Extract from persistent 1-cycles (doorways, corridors) |

**Files to create:**
| New File | Role |
|----------|------|
| `TopologicalAnalysis.h` | PH computation on octree filtration |
| `PersistenceDiagram.h` | Data structure for persistence pairs |
| `TopologicalSkeleton.h` | Extract navigable structure from persistent features |
| `OctreeFiltration.h` | Build simplicial complex from octree at various thresholds |

**Effort:** ~4-6 weeks for core PH; ~8-10 weeks for full topological skeleton extraction

---

### 3f. Tropical Algebra → Efficient Map Compression

**Current Compression (Pruning):** OctoMap prunes nodes where all 8 children have identical values. Simple but limited.

**Tropical (max-plus) Compression:**

The occupancy field over 3D space can be viewed as a tropical polynomial:
- `f(x,y,z) = max_i(aᵢ + wᵢ·[x,y,z])` where `aᵢ` are offsets and `wᵢ` are weights
- Fit tropical polynomial pieces to the occupancy surface
- Each piece covers a region at the resolution that piece needs
- **Result:** Lossless compression at much smaller memory — keep "most informative" resolution at each point

**Tropical Approach vs Current Pruning:**
- Current: Binary decision — prune or don't (all-or-nothing at each node)
- Tropical: Graduated — can represent a region at ANY resolution in the hierarchy
- Tropical polynomial fitting gives the OPTIMAL resolution at each point
- Regions with flat occupancy → very coarse (single tropical term)
- Regions near surfaces → fine (many tropical terms)

**Files to modify/create:**
| File | Change |
|------|--------|
| New: `TropicalCompression.h` | Fit tropical polynomials to occupancy surfaces |
| `OcTreeBaseImpl.h` | Add tropical-compressed serialization |
| `OcTreeBaseImpl.cpp` | `prune()` → tropical-enhanced pruning |
| New: `TropicalCodec.h` | Encode/decode octree as tropical polynomial |

**Effort:** ~4-6 weeks

---

### 3g. Wasserstein Distance → Map Comparison

**Current State:** OctoMap has no built-in map comparison metric. `operator==` does exact equality — useless for real maps.

**Wasserstein (Earth Mover's) Distance:**
- Treat each map as a measure (probability distribution over occupied/free)
- Wasserstein-1 = minimum "work" to transform one map into another
- Respects the geometry of the space (unlike voxel-by-voxel comparison)
- Optimal transport formulation

**Applications:**
1. **Loop closure detection:** Compare current local map against stored maps — low Wasserstein = loop closure
2. **Map quality assessment:** Compare against ground truth
3. **Map fusion confidence:** Wasserstein distance between two robots' maps of overlapping region
4. **Change detection:** Wasserstein between map at t and map at t+Δt

**Implementation:**
- Octree → discrete measure: occupied voxels carry weight proportional to occupancy probability
- Use Sinkhorn algorithm for approximate Wasserstein (O(n log n) instead of O(n³))
- Exploit octree structure for acceleration: coarser levels give lower bounds

**Files to create:**
| New File | Role |
|----------|------|
| `WassersteinMetric.h` | Compute W₁ between two octrees |
| `SinkhornTransport.h` | Approximate optimal transport via Sinkhorn iterations |
| `OctreeMeasure.h` | Convert octree to discrete measure for transport |
| `MapComparison.h` | High-level: loop closure, change detection via Wasserstein |

**Effort:** ~3-4 weeks

---

## 4. Killer App: "Constraint-Map"

### Vision

**Constraint-Map** is a next-generation occupancy mapping system that:
1. **NEVER drifts** — lattice-snapped probabilities guarantee deterministic maps
2. **DETECTS its own errors** — conservation spectral analysis flags anomalies in real-time
3. **Gives you topology for free** — persistent homology extracts navigable structure (rooms, doorways, corridors) directly from the map
4. **Fuses multi-robot maps correctly** — sheaf cohomology identifies conflicts and sheaf Laplacian resolves them optimally

### Architecture

```
                    ┌─────────────────────┐
                    │   Constraint-Map    │
                    │   (Top-level API)   │
                    └─────────┬───────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                   │
   ┌────────▼────────┐ ┌─────▼──────┐ ┌─────────▼──────────┐
   │ Lattice Octree  │ │ Topological│ │ Sheaf Fusion Layer  │
   │ (constraint-    │ │ Analysis   │ │ (multi-robot map    │
   │  theory-core)   │ │ (persist.  │ │  merge with H¹     │
   │                 │ │  homology) │ │  conflict detect)   │
   └────────┬────────┘ └─────┬──────┘ └─────────┬──────────┘
            │                 │                   │
   ┌────────▼────────┐ ┌─────▼──────┐ ┌─────────▼──────────┐
   │ GA Spatial Ops  │ │ Tropical   │ │ Conservation       │
   │ (exact ray      │ │ Compress   │ │ Spectral           │
   │  casting,       │ │ (optimal   │ │ (anomaly detect,   │
   │  collision)     │ │  compress) │ │  quality monitor)  │
   └─────────────────┘ └────────────┘ └────────────────────┘
            │                 │                   │
            └─────────────────┼───────────────────┘
                              │
                    ┌─────────▼───────────┐
                    │   Wasserstein       │
                    │   (map comparison,  │
                    │    loop closure)    │
                    └─────────────────────┘
```

### What ROS/ROS2 Doesn't Have

Current ROS2 occupancy mapping:
- **Float-probability maps** that drift and aren't reproducible
- **No topological analysis** — you need separate topological mapping
- **Naive map merging** — just overlay and hope
- **No self-diagnosis** — the map is whatever the sensors say
- **Bresenham ray casting** — misses voxels, approximate

Constraint-Map solves ALL of these. It's the "constraint-native 3D mapping" that doesn't exist in the ROS ecosystem.

### Pitch

> "Constraint-Map: The first 3D occupancy mapper that guarantees deterministic maps, detects its own errors, and gives you the topological structure of your environment for free. Built on OctoMap, powered by SuperInstance mathematics."

---

## 5. Specific Files/Modules to Modify — Implementation Plan

### Phase 1: Foundation (4-6 weeks)

**Priority: Constraint-theory-core integration (lattice probabilities)**

| File | Change | Lines Est. | Effort |
|------|--------|------------|--------|
| `include/octomap/LatticeValue.h` | **NEW** — lattice-snapped float wrapper | ~200 | 2 days |
| `OcTreeDataNode.h` | Template specialization for `LatticeValue` | ~50 | 1 day |
| `OcTreeNode.h` | Use `LatticeValue` instead of raw `float` | ~30 | 1 day |
| `octomap_utils.h` | `logodds()` / `probability()` for lattice values | ~40 | 1 day |
| `OccupancyOcTreeBase.hxx` | `updateNodeRecurs` with lattice update | ~80 | 3 days |
| `AbstractOccupancyOcTree.h` | Thresholds as lattice points | ~20 | 0.5 day |
| `AbstractOccupancyOcTree.cpp` | Threshold setters | ~30 | 0.5 day |
| Tests | Verify determinism across platforms | ~300 | 3 days |

### Phase 2: Self-Diagnosis (3-4 weeks)

**Priority: Conservation spectral anomaly detection**

| File | Change | Lines Est. | Effort |
|------|--------|------------|--------|
| `include/octomap/ConservationTracker.h` | **NEW** — co-update graph builder | ~300 | 3 days |
| `include/octomap/MapQualityMonitor.h` | **NEW** — spectral analysis + anomaly reporting | ~400 | 5 days |
| `OccupancyOcTreeBase.hxx` | Hook tracker into `insertPointCloud` | ~30 | 1 day |
| `OccupancyOcTreeBase.h` | Add `getQualityReport()` | ~20 | 0.5 day |
| Tests | Simulate sensor faults, verify detection | ~200 | 3 days |

### Phase 3: Topological Intelligence (6-8 weeks)

**Priority: Persistent homology for topological skeleton**

| File | Change | Lines Est. | Effort |
|------|--------|------------|--------|
| `include/octomap/OctreeFiltration.h` | **NEW** — build simplicial complex from octree | ~400 | 5 days |
| `include/octomap/TopologicalAnalysis.h` | **NEW** — PH computation | ~600 | 10 days |
| `include/octomap/PersistenceDiagram.h` | **NEW** — data structure | ~200 | 2 days |
| `include/octomap/TopologicalSkeleton.h` | **NEW** — extract navigable structure | ~500 | 8 days |
| `OcTreeBaseImpl.h` | `toSimplicialComplex()` | ~100 | 3 days |
| Tests | Known environments, verify Betti numbers | ~300 | 5 days |

### Phase 4: Multi-Robot (8-12 weeks)

**Priority: Sheaf-theoretic map fusion**

| File | Change | Lines Est. | Effort |
|------|--------|------------|--------|
| `include/octomap/SheafFusion.h` | **NEW** — define sheaf on octree overlaps | ~500 | 8 days |
| `include/octomap/SheafCohomology.h` | **NEW** — H⁰/H¹ computation | ~600 | 12 days |
| `include/octomap/SheafLaplacian.h` | **NEW** — optimal merge weights | ~400 | 6 days |
| `include/octomap/MultiRobotMapServer.h` | **NEW** — fusion coordinator | ~500 | 8 days |
| `MapCollection.h/.hxx` | Sheaf-aware merge | ~100 | 3 days |
| `OcTreeBaseImpl.h` | `computeOverlap()` | ~80 | 2 days |
| Tests | Multi-robot scenarios with known ground truth | ~400 | 7 days |

### Phase 5: Spatial Ops Upgrade (6-8 weeks)

**Priority: GA-based ray casting**

| File | Change | Lines Est. | Effort |
|------|--------|------------|--------|
| `include/octomap/ga/GAMultivector.h` | **NEW** — Cl(3,1) multivector | ~400 | 5 days |
| `include/octomap/ga/GASpatialOps.h` | **NEW** — meet, join, project | ~500 | 8 days |
| `OcTreeBaseImpl.cpp` | `computeRayKeys` → GA-based | ~200 | 5 days |
| `math/Vector3.h` | GA backend (keep API compat) | ~100 | 2 days |
| `math/Pose6D.h` | Map to conformal motor | ~80 | 2 days |
| Tests | Compare GA vs Bresenham ray casting | ~200 | 3 days |

### Phase 6: Compression & Comparison (5-8 weeks)

**Priority: Tropical compression + Wasserstein comparison**

| File | Change | Lines Est. | Effort |
|------|--------|------------|--------|
| `include/octomap/TropicalCompression.h` | **NEW** — tropical polynomial fitting | ~400 | 6 days |
| `include/octomap/WassersteinMetric.h` | **NEW** — W₁ between octrees | ~500 | 8 days |
| `include/octomap/SinkhornTransport.h` | **NEW** — approximate OT | ~300 | 5 days |
| `include/octomap/MapComparison.h` | **NEW** — loop closure via Wasserstein | ~300 | 4 days |
| `OcTreeBaseImpl.cpp` | `prune()` → tropical-enhanced | ~100 | 3 days |
| Tests | Compression ratios, loop closure accuracy | ~300 | 5 days |

---

## 6. Dependency Map

```
Phase 1 (Lattice) ───────────────────────────────┐
                                                   │
Phase 2 (Conservation) ─── depends on Phase 1 ───┤
                                                   │
Phase 3 (Topology/PH) ──── independent ───────────┤
                                                   │
Phase 4 (Sheaf Fusion) ─── depends on Phase 1 ───┤
                                                   │
Phase 5 (GA Spatial) ───── independent ───────────┤
                                                   │
Phase 6 (Tropical + Wasserstein) ── depends on 1 ─┘
```

Phases 3 and 5 can proceed in parallel with Phase 1 (they don't require lattice values). Phase 2, 4, and 6 should wait for Phase 1.

---

## 7. Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Performance regression from lattice ops | Medium | Benchmark against float baseline; lattice ops are integer arithmetic — likely faster |
| PH computation too slow for real-time | High | Only compute on compressed/pruned octree; use cohomology persistence for speed |
| Sheaf cohomology implementation complexity | High | Start with H⁰/H¹ only; defer higher cohomology |
| GA ray casting different results from Bresenham | Medium | Validate that same voxels are marked; GA should be strictly better |
| Breaking ROS2 compatibility | High | Keep existing API; add constraint-map as optional layer |
| Memory overhead from co-update tracking | Medium | Periodic pruning of conservation graph; sliding window |

---

## 8. Summary Verdict

**OctoMap is an excellent integration target for SuperInstance.** Here's why:

1. **Mature, widely deployed** — thousands of robotics projects depend on it. Any improvement has massive impact.
2. **Clean C++ codebase** — well-structured template hierarchy makes surgical modifications possible.
3. **Already has hooks** — `changed_keys`, `use_change_detection`, `MapCollection`, `CountingOcTree` provide natural attachment points.
4. **Mathematical alignment is stunning:**
   - The octree IS a lattice → constraint theory fits naturally
   - The octree IS a filtration → persistent homology fits naturally
   - Multiple robots = multiple sections → sheaf theory fits naturally
   - Log-odds updates = algebraic operations → tropical algebra fits naturally
5. **BSD license** — no GPL contamination concerns for the core library.
6. **Clean fork** — zero modifications yet, we can build exactly what we need.

**Recommended first move:** Phase 1 (lattice probabilities) + Phase 3 (persistent homology) in parallel. Lattice gives us the "never drifts" story. PH gives us the "topology for free" story. Together, that's the killer demo.

**Total estimated effort for full Constraint-Map:** 30-46 weeks of focused engineering work.

**Minimum viable "Constraint-Map" (Phases 1+3):** 10-14 weeks.
