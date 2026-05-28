# Conservation Spectral SDK — Multi-Language Schema

**Version:** 0.1.0-draft
**Date:** 2026-05-28
**Status:** Implementation-ready specification

This document defines the core library that every Conservation Spectral product builds on. A developer should be able to start implementing from this document alone.

---

## Table of Contents

1. [Architecture Decision: FFI vs Pure](#1-architecture-decision-ffi-vs-pure)
2. [Canonical Serialization Schema](#2-canonical-serialization-schema)
3. [Core Types (per language)](#3-core-types-per-language)
4. [Core Operations (per language)](#4-core-operations-per-language)
5. [Cross-Language Conformance Tests](#5-cross-language-conformance-tests)
6. [Version Compatibility Strategy](#6-version-compatibility-strategy)
7. [Package Layout](#7-package-layout)

---

## 1. Architecture Decision: FFI vs Pure

**Recommendation: Rust core + C FFI + language bindings.**

| Strategy | Pros | Cons |
|---|---|---|
| **Rust core + C FFI + PyO3/WASM/header** (CHOSEN) | Single source of truth for numerics; guaranteed cross-language parity; SIMD/eigendecomposition optimized once; minimal binary sizes via WASM | Build complexity; Rust toolchain required; FFI overhead (~10ns/call, negligible for matrix ops) |
| Pure per-language | No FFI complexity; idiomatic APIs | 4 implementations to keep in sync; numerical divergence; duplicated testing; eigendecomposition reimplemented 4 times |

**Binding strategy:**

```
conservation-spectral-core (Rust crate)
├── conservation-spectral-sys    → C header (cbindgen)
├── conservation-spectral-python → PyO3 wheel
├── conservation-spectral-js     → WASM (wasm-pack)
└── conservation-spectral.h      → drop-in C header
```

Rust crate `conservation-spectral-core` contains all numerics. Language bindings are thin wrappers that:
1. Convert language-native types to/from C FFI types
2. Expose idiomatic APIs (generics in Rust, classes in Python/JS, structs in C)
3. Handle memory ownership

**Eigendecomposition backend:** Rust uses `nalgebra` + `rust-ndarray` for dense, `sprs` for sparse. For large graphs (>10k nodes), optional LAPACK linkage via `lapack-src`.

---

## 2. Canonical Serialization Schema

All languages read/write the same format. Binary by default (MessagePack), with JSON as a human-readable alternative.

### 2.1 MessagePack Schema (binary, preferred)

```rust
// In Rust — defines the canonical wire format.
// All languages serialize to/from this shape.

// Header: magic bytes + version
// [0xCA, 0xFE, 0x53, 0x50] = "CAFÉ SP" (Conservation SPectral)
// u16 major_version, u16 minor_version

use rmp_serde::{Serializer, Deserializer};
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
pub struct CanonicalGraph {
    pub version: SchemaVersion,
    pub vertices: Vec<VertexData>,
    pub edges: Vec<EdgeData>,
    pub metadata: HashMap<String, Value>,
}

#[derive(Serialize, Deserialize)]
pub struct SchemaVersion {
    pub major: u16,
    pub minor: u16,
}

#[derive(Serialize, Deserialize)]
pub struct VertexData {
    pub id: u64,
    pub label: Option<String>,
    pub attributes: HashMap<String, AttributeValue>,
}

#[derive(Serialize, Deserialize)]
pub struct EdgeData {
    pub source: u64,
    pub target: u64,
    pub weight: f64,
    pub attributes: HashMap<String, AttributeValue>,
}

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum AttributeValue {
    F64 { value: f64 },
    I64 { value: i64 },
    String { value: String },
    Bool { value: bool },
    F64Array { values: Vec<f64> },
}

#[derive(Serialize, Deserialize)]
pub struct CanonicalDecomposition {
    pub version: SchemaVersion,
    pub graph_hash: [u8; 32],           // BLAKE3 of the graph, for integrity
    pub eigenvalues: Vec<f64>,
    pub eigenvectors: Vec<Vec<f64>>,    // eigenvectors[i] = i-th eigenvector
    pub laplacian_type: LaplacianType,
    pub num_vertices: usize,
}

#[derive(Serialize, Deserialize)]
pub enum LaplacianType {
    Unnormalized,
    SymmetricNormalized,
    RandomWalkNormalized,
}

#[derive(Serialize, Deserialize)]
pub struct CanonicalReport {
    pub version: SchemaVersion,
    pub conservation_ratios: Vec<ConservationRatio>,
    pub anomalies: Vec<Anomaly>,
    pub spectral_gap: f64,
    pub cheeger_constant: f64,
    pub fingerprint: SpectralFingerprint,
}

#[derive(Serialize, Deserialize)]
pub struct ConservationRatio {
    pub eigenvector_index: usize,
    pub eigenvalue: f64,
    pub ratio: f64,
    pub attribute_name: String,
}

#[derive(Serialize, Deserialize)]
pub struct Anomaly {
    pub vertex_id: u64,
    pub eigenvector_index: usize,
    pub deviation: f64,
    pub anomaly_type: AnomalyType,
    pub description: String,
}

#[derive(Serialize, Deserialize)]
pub enum AnomalyType {
    ConservationViolation,
    StructuralBreak,
    SpectralOutlier,
    TransitionAnomaly,
}

#[derive(Serialize, Deserialize)]
pub struct SpectralFingerprint {
    pub eigenvalue_histogram: Vec<f64>,    // binned eigenvalue distribution
    pub spectral_entropy: f64,
    pub effective_dimension: f64,           // number of "significant" eigenvectors
    pub gap_profile: Vec<f64>,              // consecutive eigenvalue gaps
    pub conservation_profile: Vec<f64>,     // conservation ratio per eigenvector
}
```

### 2.2 JSON Schema (human-readable, debug mode)

Same structure, serde_json serialized. Recognizable by `{"version": {"major": 0, "minor": 1}, ...}`.

### 2.3 File Extension Convention

| Format | Extension |
|---|---|
| MessagePack | `.csp` |
| JSON | `.csp.json` |

All loaders detect format from magic bytes (MessagePack) vs `{` (JSON).

---

## 3. Core Types (per language)

### 3.1 Rust (`conservation-spectral-core`)

```rust
use ndarray::{Array1, Array2};
use std::collections::HashMap;

/// Generic graph with vertices of type V and edge attributes of type A.
/// In practice, V is typically `u64` and A is `f64` (edge weight).
pub struct TensionGraph<V, A>
where
    V: Clone + Eq + std::hash::Hash,
    A: Clone,
{
    vertices: Vec<V>,
    vertex_index: HashMap<V, usize>,
    adjacency: Vec<Vec<(usize, A)>>,
    vertex_attributes: HashMap<String, Array1<f64>>,
    directed: bool,
}

impl<V, A> TensionGraph<V, A>
where
    V: Clone + Eq + std::hash::Hash + Send + Sync,
    A: Clone + Into<f64> + Send + Sync,
{
    pub fn new() -> Self;
    pub fn with_capacity(n_vertices: usize) -> Self;
    pub fn add_vertex(&mut self, v: V) -> usize;
    pub fn add_edge(&mut self, source: V, target: V, weight: A);
    pub fn add_attribute(&mut self, name: &str, values: Array1<f64>);
    pub fn vertex_count(&self) -> usize;
    pub fn edge_count(&self) -> usize;
    pub fn adjacency_matrix(&self) -> Array2<f64>;
}

pub struct Laplacian {
    pub matrix: Array2<f64>,
    pub degree_matrix: Array2<f64>,
    pub weight_matrix: Array2<f64>,
    pub normalized: bool,
    pub num_vertices: usize,
}

pub struct EigenDecomposition {
    pub eigenvalues: Array1<f64>,
    pub eigenvectors: Array2<f64>,   // columns = eigenvectors
    pub graph_hash: [u8; 32],
    pub laplacian_type: LaplacianType,
}

pub struct ConservationReport {
    pub ratios: Vec<ConservationRatio>,
    pub anomalies: Vec<Anomaly>,
    pub spectral_gap: f64,
    pub cheeger_constant: f64,
    pub fingerprint: SpectralFingerprint,
}

pub struct ConservationTracker {
    window_size: usize,
    history: Vec<f64>,
    eigen_decomposition: Option<EigenDecomposition>,
    baseline_ratios: Vec<f64>,
}

pub struct SpectralFingerprint {
    pub eigenvalue_histogram: Vec<f64>,
    pub spectral_entropy: f64,
    pub effective_dimension: f64,
    pub gap_profile: Vec<f64>,
    pub conservation_profile: Vec<f64>,
}

pub struct Fix {
    pub edge: Option<(u64, u64)>,
    pub vertex: Option<u64>,
    pub suggested_weight: Option<f64>,
    pub description: String,
    pub confidence: f64,
}
```

### 3.2 Python (`conservation-spectral`)

```python
"""conservation_spectral — Python bindings via PyO3."""

from __future__ import annotations
from typing import Generic, TypeVar, Optional
import numpy as np

V = TypeVar("V")
A = TypeVar("A")

class TensionGraph(Generic[V, A]):
    """Weighted directed graph with named vertex attributes."""

    def __init__(self, directed: bool = True) -> None: ...
    def add_vertex(self, vertex: V) -> int: ...
    def add_edge(self, source: V, target: V, weight: A) -> None: ...
    def add_attribute(self, name: str, values: np.ndarray) -> None:
        """Attach a named float array to vertices (indexed by vertex order)."""
        ...
    @property
    def vertex_count(self) -> int: ...
    @property
    def edge_count(self) -> int: ...
    def adjacency_matrix(self) -> np.ndarray: ...


class Laplacian:
    """Computed from TensionGraph via build_laplacian()."""

    matrix: np.ndarray          # (n, n) float64
    degree_matrix: np.ndarray
    weight_matrix: np.ndarray
    normalized: bool
    num_vertices: int


class EigenDecomposition:
    """Result of eigendecompose()."""

    eigenvalues: np.ndarray     # (n,) float64, sorted ascending
    eigenvectors: np.ndarray    # (n, n) float64, columns = eigenvectors
    laplacian_type: str         # "unnormalized" | "symmetric_normalized" | "random_walk"

    def save(self, path: str) -> None:
        """Serialize to .csp format."""
        ...

    @classmethod
    def load(cls, path: str) -> "EigenDecomposition":
        """Deserialize from .csp or .csp.json format."""
        ...


class ConservationReport:
    """Full analysis output."""

    ratios: list[ConservationRatio]
    anomalies: list[Anomaly]
    spectral_gap: float
    cheeger_constant: float
    fingerprint: SpectralFingerprint

    def to_dict(self) -> dict: ...
    def to_json(self, indent: int = 2) -> str: ...
    def save(self, path: str) -> None: ...


class ConservationRatio:
    eigenvector_index: int
    eigenvalue: float
    ratio: float           # gradient variance in this eigen-direction
    attribute_name: str


class Anomaly:
    vertex_id: int
    eigenvector_index: int
    deviation: float
    anomaly_type: str      # "conservation_violation" | "structural_break" | "spectral_outlier" | "transition_anomaly"
    description: str


class SpectralFingerprint:
    eigenvalue_histogram: list[float]
    spectral_entropy: float
    effective_dimension: float
    gap_profile: list[float]
    conservation_profile: list[float]


class ConservationTracker:
    """Sliding-window tracker for real-time conservation monitoring."""

    def __init__(self, window_size: int = 100) -> None: ...
    def update(self, new_data: np.ndarray) -> Optional[ConservationReport]:
        """Feed new observations. Returns report if window is full and anomaly detected, else None."""
        ...
    def reset(self) -> None: ...
    @property
    def current_ratios(self) -> list[float]: ...
    @property
    def baseline(self) -> list[float]: ...


class Fix:
    edge: Optional[tuple[int, int]]
    vertex: Optional[int]
    suggested_weight: Optional[float]
    description: str
    confidence: float
```

### 3.3 JavaScript/TypeScript (`conservation-spectral`)

```typescript
// conservation-spectral — WASM bindings via wasm-pack

/** Weighted graph with vertex attributes. */
export class TensionGraph {
  constructor(directed?: boolean);

  addVertex(label?: string): number;                    // returns vertex index
  addEdge(source: number, target: number, weight: number): void;
  addAttribute(name: string, values: Float64Array): void;

  get vertexCount(): number;
  get edgeCount(): number;
  adjacencyMatrix(): Float64Array;                      // flat, row-major, n*n

  /** Serialize to CSP binary format. */
  toBytes(): Uint8Array;

  /** Deserialize from CSP binary. */
  static fromBytes(data: Uint8Array): TensionGraph;
}

/** Laplacian matrix from build_laplacian(). */
export class Laplacian {
  readonly matrix: Float64Array;        // n*n flat row-major
  readonly degreeMatrix: Float64Array;
  readonly weightMatrix: Float64Array;
  readonly normalized: boolean;
  readonly numVertices: number;
}

/** Eigendecomposition result. */
export class EigenDecomposition {
  readonly eigenvalues: Float64Array;
  readonly eigenvectors: Float64Array;  // n*n flat, column-major
  readonly laplacianType: "unnormalized" | "symmetric_normalized" | "random_walk_normalized";

  save(): Uint8Array;
  static load(data: Uint8Array): EigenDecomposition;
  static loadJSON(json: string): EigenDecomposition;
}

export interface ConservationRatio {
  eigenvectorIndex: number;
  eigenvalue: number;
  ratio: number;
  attributeName: string;
}

export interface Anomaly {
  vertexId: number;
  eigenvectorIndex: number;
  deviation: number;
  anomalyType: "conservation_violation" | "structural_break" | "spectral_outlier" | "transition_anomaly";
  description: string;
}

export interface SpectralFingerprint {
  eigenvalueHistogram: number[];
  spectralEntropy: number;
  effectiveDimension: number;
  gapProfile: number[];
  conservationProfile: number[];
}

export interface Fix {
  edge?: [number, number];
  vertex?: number;
  suggestedWeight?: number;
  description: string;
  confidence: number;
}

export class ConservationReport {
  readonly ratios: ConservationRatio[];
  readonly anomalies: Anomaly[];
  readonly spectralGap: number;
  readonly cheegerConstant: number;
  readonly fingerprint: SpectralFingerprint;

  toJSON(): string;
  static fromJSON(json: string): ConservationReport;
}

export class ConservationTracker {
  constructor(windowSize?: number);

  /** Feed new observation. Returns report if anomaly detected, null otherwise. */
  update(newData: Float64Array): ConservationReport | null;
  reset(): void;

  get currentRatios(): number[];
  get baseline(): number[];
}
```

### 3.4 C (`conservation_spectral.h` — header-only via static lib)

```c
// conservation_spectral.h — C API
// Build: link against libconservation_spectral.a (or .so/.dylib)

#ifndef CONSERVATION_SPECTRAL_H
#define CONSERVATION_SPECTRAL_H

#include <stddef.h>
#include <stdint.h>
#include <stdbool.h>

#ifdef __cplusplus
extern "C" {
#endif

// ============================================================
// Error handling
// ============================================================

typedef enum {
    CS_OK = 0,
    CS_ERR_NULL_POINTER = 1,
    CS_ERR_OUT_OF_BOUNDS = 2,
    CS_ERR_INVALID_STATE = 3,
    CS_ERR_ALLOC_FAILED = 4,
    CS_ERR_IO = 5,
    CS_ERR_SERIALIZATION = 6,
    CS_ERR_DIMENSION_MISMATCH = 7,
} CsError;

// ============================================================
// Opaque handles — use create/destroy functions
// ============================================================

typedef struct CsGraph CsGraph;
typedef struct CsLaplacian CsLaplacian;
typedef struct CsEigenDecomposition CsEigenDecomposition;
typedef struct CsConservationReport CsConservationReport;
typedef struct CsConservationTracker CsConservationTracker;

// ============================================================
// TensionGraph
// ============================================================

CsGraph* cs_graph_create(bool directed);
void     cs_graph_destroy(CsGraph* graph);

CsError  cs_graph_add_vertex(CsGraph* graph, uint64_t* out_index);
CsError  cs_graph_add_vertex_labeled(CsGraph* graph, const char* label, uint64_t* out_index);
CsError  cs_graph_add_edge(CsGraph* graph, uint64_t source, uint64_t target, double weight);

/// Attach a named attribute array. `values` must have `cs_graph_vertex_count()` elements.
CsError  cs_graph_add_attribute(CsGraph* graph, const char* name,
                                 const double* values, size_t len);

size_t   cs_graph_vertex_count(const CsGraph* graph);
size_t   cs_graph_edge_count(const CsGraph* graph);

/// Returns flat row-major adjacency matrix. Caller must free() the returned pointer.
CsError  cs_graph_adjacency_matrix(const CsGraph* graph, double** out, size_t* out_len);

// ============================================================
// Laplacian
// ============================================================

typedef enum {
    CS_LAPLACIAN_UNNORMALIZED = 0,
    CS_LAPLACIAN_SYMMETRIC_NORMALIZED = 1,
    CS_LAPLACIAN_RANDOM_WALK_NORMALIZED = 2,
} CsLaplacianType;

void     cs_laplacian_destroy(CsLaplacian* lap);

/// Returns flat row-major matrix. Caller must free().
CsError  cs_laplacian_matrix(const CsLaplacian* lap, double** out, size_t* out_len);
size_t   cs_laplacian_num_vertices(const CsLaplacian* lap);
bool     cs_laplacian_is_normalized(const CsLaplacian* lap);

// ============================================================
// EigenDecomposition
// ============================================================

void     cs_eigen_destroy(CsEigenDecomposition* eigen);

/// Eigenvalues, sorted ascending. Caller must free().
CsError  cs_eigen_eigenvalues(const CsEigenDecomposition* eigen,
                               double** out, size_t* out_len);

/// Eigenvectors as flat column-major matrix (n*n). Caller must free().
CsError  cs_eigen_eigenvectors(const CsEigenDecomposition* eigen,
                                double** out, size_t* out_rows, size_t* out_cols);

CsLaplacianType cs_eigen_laplacian_type(const CsEigenDecomposition* eigen);

// Serialization
CsError  cs_eigen_save(const CsEigenDecomposition* eigen, const char* path);
CsError  cs_eigen_load(const char* path, CsEigenDecomposition** out);
CsError  cs_eigen_save_json(const CsEigenDecomposition* eigen, const char* path);
CsError  cs_eigen_load_json(const char* path, CsEigenDecomposition** out);

// ============================================================
// ConservationReport
// ============================================================

typedef struct {
    size_t  eigenvector_index;
    double  eigenvalue;
    double  ratio;
    char    attribute_name[256];
} CsConservationRatio;

typedef enum {
    CS_ANOMALY_CONSERVATION_VIOLATION = 0,
    CS_ANOMALY_STRUCTURAL_BREAK = 1,
    CS_ANOMALY_SPECTRAL_OUTLIER = 2,
    CS_ANOMALY_TRANSITION_ANOMALY = 3,
} CsAnomalyType;

typedef struct {
    uint64_t       vertex_id;
    size_t         eigenvector_index;
    double         deviation;
    CsAnomalyType  anomaly_type;
    char           description[512];
} CsAnomaly;

typedef struct {
    double* eigenvalue_histogram;
    size_t  eigenvalue_histogram_len;
    double  spectral_entropy;
    double  effective_dimension;
    double* gap_profile;
    size_t  gap_profile_len;
    double* conservation_profile;
    size_t  conservation_profile_len;
} CsSpectralFingerprint;

typedef struct {
    CsConservationRatio* ratios;
    size_t               ratios_len;
    CsAnomaly*           anomalies;
    size_t               anomalies_len;
    double               spectral_gap;
    double               cheeger_constant;
    CsSpectralFingerprint fingerprint;
} CsConservationReportData;

void     cs_report_destroy(CsConservationReport* report);
CsError  cs_report_data(const CsConservationReport* report, CsConservationReportData* out);
void     cs_report_data_free(CsConservationReportData* data);  // frees inner pointers

// ============================================================
// ConservationTracker
// ============================================================

CsConservationTracker* cs_tracker_create(size_t window_size);
void                   cs_tracker_destroy(CsConservationTracker* tracker);

/// Feed new observations. Sets `out_report` to non-NULL if anomaly detected.
/// Caller must destroy `*out_report` via cs_report_destroy().
CsError  cs_tracker_update(CsConservationTracker* tracker,
                            const double* data, size_t data_len,
                            CsConservationReport** out_report);
void     cs_tracker_reset(CsConservationTracker* tracker);

/// Current conservation ratios. Caller must free() the returned pointer.
CsError  cs_tracker_current_ratios(const CsConservationTracker* tracker,
                                    double** out, size_t* out_len);

#ifdef __cplusplus
}
#endif

#endif // CONSERVATION_SPECTRAL_H
```

---

## 4. Core Operations (per language)

### 4.1 Rust

```rust
// conservation_spectral_core::ops

use crate::types::*;

/// Build a Tension-Graph Laplacian from transition probabilities and a similarity kernel.
///
/// W[i,j] = transitions[i,j] * similarity_kernel(i, j)
/// L = D - W  (or normalized variant)
///
/// `similarity_kernel` is a closure: Fn(usize, usize) -> f64
pub fn build_laplacian<F>(
    transitions: &Array2<f64>,
    similarity_kernel: F,
    laplacian_type: LaplacianType,
) -> Result<Laplacian, CsError>
where
    F: Fn(usize, usize) -> f64,
{
    let n = transitions.nrows();
    let mut w = Array2::<f64>::zeros((n, n));
    for i in 0..n {
        for j in 0..n {
            w[[i, j]] = transitions[[i, j]] * similarity_kernel(i, j);
        }
    }
    // D = diag(W · 1)
    let degree: Array1<f64> = w.sum_axis(Axis(1));
    let d_matrix = Array2::from_diag(&degree);
    let l = match laplacian_type {
        LaplacianType::Unnormalized => &d_matrix - &w,
        LaplacianType::SymmetricNormalized => {
            let d_inv_sqrt = degree.mapv(|d| if d > 0.0 { 1.0 / d.sqrt() } else { 0.0 });
            let d_inv_sqrt_mat = Array2::from_diag(&d_inv_sqrt);
            let eye = Array2::eye(n);
            &eye - d_inv_sqrt_mat.dot(&w).dot(&d_inv_sqrt_mat)
        }
        LaplacianType::RandomWalkNormalized => {
            let d_inv = degree.mapv(|d| if d > 0.0 { 1.0 / d } else { 0.0 });
            let d_inv_mat = Array2::from_diag(&d_inv);
            Array2::eye(n) - d_inv_mat.dot(&w)
        }
    };
    Ok(Laplacian {
        matrix: l,
        degree_matrix: d_matrix,
        weight_matrix: w,
        normalized: laplacian_type != LaplacianType::Unnormalized,
        num_vertices: n,
    })
}

/// Compute eigendecomposition. `num_vectors` = number of eigenvectors to compute.
/// If num_vectors == 0 or >= n, compute full decomposition.
/// Uses nalgebra symmetric eigendecomposition for dense, LOBPCG for large sparse.
pub fn eigendecompose(
    laplacian: &Laplacian,
    num_vectors: usize,
) -> Result<EigenDecomposition, CsError> {
    // ... delegates to backend
}

/// Conservation ratio for an attribute along the k-th eigenvector.
/// CR(k) = Var(∇(attribute projected onto eigenvector_k))
/// Low CR = attribute is conserved in this mode.
pub fn conservation_ratio(
    eigen: &EigenDecomposition,
    attribute: &Array1<f64>,
    eigenvector_index: usize,
) -> f64 {
    let phi = eigen.eigenvectors.column(eigenvector_index);
    let projection = phi.dot(attribute);
    let gradient = projection.iter()
        .zip(projection.iter().skip(1))
        .map(|(a, b)| b - a);
    // variance of gradient
    let n = gradient.len() as f64;
    if n == 0.0 { return f64::INFINITY; }
    let mean = gradient.clone().sum::<f64>() / n;
    gradient.map(|g| (g - mean).powi(2)).sum::<f64>() / n
}

/// Batch conservation ratios for all eigenvectors.
pub fn conservation_ratios(
    eigen: &EigenDecomposition,
    attribute: &Array1<f64>,
    attribute_name: &str,
) -> Vec<ConservationRatio> {
    (0..eigen.eigenvalues.len())
        .map(|k| ConservationRatio {
            eigenvector_index: k,
            eigenvalue: eigen.eigenvalues[k],
            ratio: conservation_ratio(eigen, attribute, k),
            attribute_name: attribute_name.to_string(),
        })
        .collect()
}

/// Detect anomalies: vertices where local conservation deviates from global.
pub fn detect_anomalies(
    tracker: &ConservationTracker,
    threshold: f64,  // e.g., 3.0 sigma
) -> Vec<Anomaly> {
    // ... z-score based detection on tracked ratios
}

/// Compute spectral fingerprint: summary statistics of the eigenspectrum.
pub fn spectral_fingerprint(
    eigen: &EigenDecomposition,
) -> SpectralFingerprint {
    let evals = &eigen.eigenvalues;

    // Eigenvalue histogram (binned into sqrt(n) bins)
    let n_bins = (evals.len() as f64).sqrt().ceil() as usize;
    let emin = evals[evals.argmin()];
    let emax = evals[evals.argmax()];
    let bin_width = (emax - emin) / n_bins as f64;
    let histogram: Vec<f64> = (0..n_bins)
        .map(|b| evals.iter().filter(|&&e| {
            e >= emin + b as f64 * bin_width && e < emin + (b + 1) as f64 * bin_width
        }).count() as f64)
        .collect();

    // Spectral entropy: H = -Σ p_i log(p_i) where p_i = λ_i / Σ λ_i
    let total: f64 = evals.sum();
    let entropy = if total > 0.0 {
        evals.iter()
            .map(|&e| { let p = e / total; if p > 0.0 { -p * p.ln() } else { 0.0 } })
            .sum()
    } else { 0.0 };

    // Effective dimension: exp(entropy) (perplexity of eigenvalue distribution)
    let effective_dim = entropy.exp();

    // Gap profile
    let gaps: Vec<f64> = evals.windows(2).map(|w| w[1] - w[0]).collect();

    SpectralFingerprint {
        eigenvalue_histogram: histogram,
        spectral_entropy: entropy,
        effective_dimension: effective_dim,
        gap_profile: gaps,
        conservation_profile: vec![], // filled by caller via conservation_ratios
    }
}

/// Suggest corrections for detected anomalies.
pub fn suggest_correction(
    graph: &TensionGraph<u64, f64>,
    anomaly: &Anomaly,
) -> Vec<Fix> {
    // Heuristic: look at edges incident to the anomalous vertex
    // Suggest weight adjustments that reduce gradient variance
    // ... implementation TBD
}

/// Convenience: full pipeline from graph to report.
pub fn analyze(
    graph: &TensionGraph<u64, f64>,
    attribute_name: &str,
    laplacian_type: LaplacianType,
) -> Result<ConservationReport, CsError> {
    let trans = graph.adjacency_matrix();
    let lap = build_laplacian(&trans, |i, j| 1.0, laplacian_type)?; // identity kernel default
    let eigen = eigendecompose(&lap, 0)?;
    let attr = graph.vertex_attributes.get(attribute_name)
        .ok_or(CsError::InvalidState)?;
    let ratios = conservation_ratios(&eigen, attr, attribute_name);

    // Spectral gap = largest gap between consecutive eigenvalues
    let gaps: Vec<f64> = eigen.eigenvalues.windows(2).map(|w| w[1] - w[0]).collect();
    let spectral_gap = gaps.iter().cloned().fold(f64::NEG_INFINITY, f64::max);

    // Cheeger constant approximation from spectral gap
    let cheeger = spectral_gap / 2.0;  // Cheeger inequality: λ₂/2 ≤ h ≤ √(2λ₂)

    let mut fingerprint = spectral_fingerprint(&eigen);
    fingerprint.conservation_profile = ratios.iter().map(|r| r.ratio).collect();

    Ok(ConservationReport {
        ratios,
        anomalies: vec![],
        spectral_gap,
        cheeger_constant: cheeger,
        fingerprint,
    })
}
```

### 4.2 Python

```python
"""conservation_spectral — public API."""

import numpy as np
from . import _bindings  # PyO3 auto-generated


def build_laplacian(
    transitions: np.ndarray,
    similarity_kernel=None,
    laplacian_type: str = "unnormalized",
) -> Laplacian:
    """
    Build Tension-Graph Laplacian from transition matrix and similarity kernel.

    Args:
        transitions: (n, n) transition probability matrix.
        similarity_kernel: callable(i, j) -> float, or None for identity kernel.
        laplacian_type: "unnormalized" | "symmetric_normalized" | "random_walk_normalized"

    Returns:
        Laplacian object.
    """
    if similarity_kernel is None:
        similarity_kernel = lambda i, j: 1.0
    return _bindings.build_laplacian(transitions, similarity_kernel, laplacian_type)


def eigendecompose(
    laplacian: Laplacian,
    num_vectors: int = 0,
) -> EigenDecomposition:
    """
    Compute eigendecomposition of Laplacian.

    Args:
        laplacian: Laplacian object from build_laplacian().
        num_vectors: Number of eigenvectors to compute. 0 = full decomposition.

    Returns:
        EigenDecomposition with eigenvalues and eigenvectors.
    """
    return _bindings.eigendecompose(laplacian, num_vectors)


def conservation_ratio(
    eigen: EigenDecomposition,
    attribute: np.ndarray,
    eigenvector_index: int,
) -> float:
    """
    Compute conservation ratio of an attribute along the k-th eigenvector.
    Low ratio = high conservation.
    """
    return _bindings.conservation_ratio(eigen, attribute, eigenvector_index)


def conservation_ratios(
    eigen: EigenDecomposition,
    attribute: np.ndarray,
    attribute_name: str = "default",
) -> list[ConservationRatio]:
    """Conservation ratios for all eigenvectors."""
    return _bindings.conservation_ratios(eigen, attribute, attribute_name)


def detect_anomalies(
    tracker: ConservationTracker,
    threshold: float = 3.0,
) -> list[Anomaly]:
    """
    Detect anomalies from tracker state.
    threshold = number of standard deviations for flagging.
    """
    return _bindings.detect_anomalies(tracker, threshold)


def spectral_fingerprint(eigen: EigenDecomposition) -> SpectralFingerprint:
    """Compute spectral fingerprint from eigendecomposition."""
    return _bindings.spectral_fingerprint(eigen)


def suggest_correction(
    graph: TensionGraph,
    anomaly: Anomaly,
) -> list[Fix]:
    """Suggest corrections for a detected anomaly."""
    return _bindings.suggest_correction(graph, anomaly)


def analyze(
    transitions: np.ndarray,
    attribute: np.ndarray,
    attribute_name: str = "default",
    laplacian_type: str = "unnormalized",
) -> ConservationReport:
    """
    Full pipeline: transitions → Laplacian → eigendecomposition → report.

    One-function entry point for quick analysis.
    """
    lap = build_laplacian(transitions, laplacian_type=laplacian_type)
    eigen = eigendecompose(lap)
    ratios = conservation_ratios(eigen, attribute, attribute_name)
    fp = spectral_fingerprint(eigen)
    # ... assemble report
    return _bindings.analyze(transitions, attribute, attribute_name, laplacian_type)
```

### 4.3 JavaScript/TypeScript

```typescript
// conservation_spectral — public API (WASM-backed)

import {
  TensionGraph as WasmGraph,
  Laplacian as WasmLaplacian,
  EigenDecomposition as WasmEigen,
  ConservationReport as WasmReport,
  ConservationTracker as WasmTracker,
  build_laplacian as wasmBuildLaplacian,
  eigendecompose as wasmEigendecompose,
  conservation_ratio as wasmConservationRatio,
  spectral_fingerprint as wasmSpectralFingerprint,
  detect_anomalies as wasmDetectAnomalies,
  suggest_correction as wasmSuggestCorrection,
  analyze as wasmAnalyze,
} from "./pkg/conservation_spectral";

export function buildLaplacian(
  transitions: Float64Array,
  n: number,
  similarityKernel?: (i: number, j: number) => number,
  laplacianType: "unnormalized" | "symmetric_normalized" | "random_walk_normalized" = "unnormalized",
): Laplacian {
  // If custom kernel, compute W in JS, pass to WASM
  if (similarityKernel) {
    const w = new Float64Array(n * n);
    for (let i = 0; i < n; i++)
      for (let j = 0; j < n; j++)
        w[i * n + j] = transitions[i * n + j] * similarityKernel(i, j);
    return wasmBuildLaplacian(w, n, laplacianType);
  }
  return wasmBuildLaplacian(transitions, n, laplacianType);
}

export function eigendecompose(
  laplacian: Laplacian,
  numVectors: number = 0,
): EigenDecomposition {
  return wasmEigendecompose(laplacian, numVectors);
}

export function conservationRatio(
  eigen: EigenDecomposition,
  attribute: Float64Array,
  eigenvectorIndex: number,
): number {
  return wasmConservationRatio(eigen, attribute, eigenvectorIndex);
}

export function spectralFingerprint(eigen: EigenDecomposition): SpectralFingerprint {
  return wasmSpectralFingerprint(eigen);
}

export function detectAnomalies(
  tracker: ConservationTracker,
  threshold: number = 3.0,
): Anomaly[] {
  return wasmDetectAnomalies(tracker, threshold);
}

export function suggestCorrection(
  graph: TensionGraph,
  anomaly: Anomaly,
): Fix[] {
  return wasmSuggestCorrection(graph, anomaly);
}

export function analyze(
  transitions: Float64Array,
  n: number,
  attribute: Float64Array,
  attributeName: string = "default",
  laplacianType?: string,
): ConservationReport {
  return wasmAnalyze(transitions, n, attribute, attributeName, laplacianType);
}
```

### 4.4 C

```c
// Core operations exposed through the C header.
// These are the functions a C user calls directly.

// ---- Build Laplacian ----

/// Build Laplacian from a transition matrix with an optional similarity kernel.
/// If `similarity_kernel` is NULL, identity kernel (all 1.0) is used.
/// `transitions` is a flat row-major n×n matrix.
/// `similarity_kernel(i, j, user_data)` returns the similarity between vertices i and j.
/// Caller must destroy the returned Laplacian via cs_laplacian_destroy().
typedef double (*CsSimilarityKernel)(size_t i, size_t j, void* user_data);

CsError cs_build_laplacian(
    const double* transitions, size_t n,
    CsLaplacianType type,
    CsSimilarityKernel kernel, void* kernel_user_data,
    CsLaplacian** out
);

// ---- Eigendecompose ----

/// Compute eigendecomposition of a Laplacian.
/// `num_vectors` = 0 for full decomposition.
/// Caller must destroy via cs_eigen_destroy().
CsError cs_eigendecompose(
    const CsLaplacian* laplacian,
    size_t num_vectors,
    CsEigenDecomposition** out
);

// ---- Conservation Ratio ----

/// Compute conservation ratio for one attribute along one eigenvector.
CsError cs_conservation_ratio(
    const CsEigenDecomposition* eigen,
    const double* attribute, size_t attr_len,
    size_t eigenvector_index,
    double* out_ratio
);

// ---- Detect Anomalies ----

/// Detect anomalies from tracker. Returns array of anomalies.
/// Caller must free via cs_anomalies_free().
CsError cs_detect_anomalies(
    const CsConservationTracker* tracker,
    double threshold,
    CsAnomaly** out, size_t* out_len
);
void cs_anomalies_free(CsAnomaly* anomalies);

// ---- Spectral Fingerprint ----

/// Compute spectral fingerprint. Caller must free via cs_fingerprint_free().
CsError cs_spectral_fingerprint(
    const CsEigenDecomposition* eigen,
    CsSpectralFingerprint* out
);
void cs_fingerprint_free(CsSpectralFingerprint* fp);

// ---- Suggest Correction ----

/// Suggest corrections for an anomaly. Caller must free the returned array.
typedef struct {
    bool    has_edge;
    uint64_t edge_source;
    uint64_t edge_target;
    bool    has_vertex;
    uint64_t vertex;
    bool    has_suggested_weight;
    double  suggested_weight;
    char    description[512];
    double  confidence;
} CsFix;

CsError cs_suggest_correction(
    const CsGraph* graph,
    const CsAnomaly* anomaly,
    CsFix** out, size_t* out_len
);
void cs_fixes_free(CsFix* fixes);

// ---- Convenience: Full Pipeline ----

/// One-shot: transitions + attribute → full report.
CsError cs_analyze(
    const double* transitions, size_t n,
    const double* attribute, size_t attr_len,
    const char* attribute_name,
    CsLaplacianType laplacian_type,
    CsConservationReport** out
);
```

---

## 5. Cross-Language Conformance Tests

### 5.1 Shared Test Vectors

A canonical set of test graphs and expected outputs, stored in `.csp.json` in the repo:

```
tests/
├── vectors/
│   ├── simple_chain_4.json          # 4-node chain graph
│   ├── cycle_8.json                 # 8-node cycle
│   ├── star_5.json                  # 5-node star
│   ├── bipartite_3_3.json           # 3+3 bipartite
│   ├── music_ii_v_i.json            # ii-V-I transition graph
│   ├── random_20_seed42.json        # 20-node random graph (seeded)
│   └── known_eigenvalues/           # Pre-computed expected results
│       ├── simple_chain_4_expected.json
│       ├── cycle_8_expected.json
│       └── ...
├── conformance/
│   ├── test_laplacian.py            # Python conformance runner
│   ├── test_laplacian.rs            # Rust conformance runner
│   ├── test_laplacian.ts            # JS conformance runner
│   └── test_laplacian.c             # C conformance runner
└── property/
    ├── test_properties.py           # Property-based (hypothesis)
    ├── test_properties.rs           # Property-based (proptest)
    └── test_properties.ts           # Property-based (fast-check)
```

### 5.2 Conformance Test Protocol

Each language must:

1. **Load** a test vector (graph + transitions + attributes)
2. **Build Laplacian** and compare matrix values (tolerance: 1e-10)
3. **Eigendecompose** and compare eigenvalues/eigenvectors (tolerance: 1e-8, eigenvectors compared up to sign flip)
4. **Compute conservation ratios** and compare (tolerance: 1e-8)
5. **Compute spectral fingerprint** and compare (tolerance: 1e-6)
6. **Round-trip serialize** → deserialize and verify byte-identical output

### 5.3 Property-Based Tests (run in all languages)

```python
# Example: Python property tests using hypothesis
from hypothesis import given, settings
from hypothesis.strategies import integers, floats, lists
import numpy as np
from conservation_spectral import build_laplacian, eigendecompose, conservation_ratio

@given(
    n=integers(min_value=3, max_value=20),
    seed=integers(min_value=0, max_value=2**31),
)
@settings(max_examples=200)
def test_eigenvalues_non_negative(n, seed):
    """All Laplacian eigenvalues must be >= 0 (semidefinite)."""
    rng = np.random.default_rng(seed)
    T = rng.random((n, n))
    T /= T.sum(axis=1, keepdims=True)  # row-stochastic
    lap = build_laplacian(T)
    eigen = eigendecompose(lap)
    assert all(ev >= -1e-10 for ev in eigen.eigenvalues), f"Negative eigenvalue: {eigen.eigenvalues}"

@given(
    n=integers(min_value=3, max_value=10),
    seed=integers(min_value=0, max_value=2**31),
)
def test_cheeger_bound(n, seed):
    """Cheeger inequality: λ₂/2 ≤ h ≤ √(2λ₂)."""
    # ... build graph, compute, verify
    pass

@given(
    n=integers(min_value=4, max_value=15),
    seed=integers(min_value=0, max_value=2**31),
)
def test_conservation_ratio_bounded(n, seed):
    """Conservation ratio is always >= 0."""
    rng = np.random.default_rng(seed)
    T = rng.random((n, n))
    T /= T.sum(axis=1, keepdims=True)
    attr = rng.standard_normal(n)
    lap = build_laplacian(T)
    eigen = eigendecompose(lap)
    for k in range(n):
        cr = conservation_ratio(eigen, attr, k)
        assert cr >= -1e-10, f"Negative conservation ratio at k={k}: {cr}"

@given(n=integers(min_value=3, max_value=8), seed=integers(0, 1000))
def test_roundtrip_serialization(n, seed):
    """Serialize → deserialize must produce identical eigenvalues."""
    rng = np.random.default_rng(seed)
    T = rng.random((n, n)); T /= T.sum(axis=1, keepdims=True)
    eigen = eigendecompose(build_laplacian(T))
    eigen.save("/tmp/test_roundtrip.csp")
    loaded = EigenDecomposition.load("/tmp/test_roundtrip.csp")
    np.testing.assert_allclose(eigen.eigenvalues, loaded.eigenvalues, atol=1e-12)
```

```rust
// Rust equivalent using proptest
use proptest::prelude::*;

proptest! {
    #[test]
    fn eigenvalues_non_negative(n in 3usize..20, seed in 0u64..10000) {
        let mut rng = rand::rngs::StdRng::seed_from_u64(seed);
        let mut t = Array2::<f64>::zeros((n, n));
        for i in 0..n {
            let mut row: Vec<f64> = (0..n).map(|_| rng.gen::<f64>()).collect();
            let sum: f64 = row.iter().sum();
            for j in 0..n { t[[i,j]] = row[j] / sum; }
        }
        let lap = build_laplacian(&t, |_,_| 1.0, LaplacianType::Unnormalized).unwrap();
        let eigen = eigendecompose(&lap, 0).unwrap();
        for &ev in eigen.eigenvalues.iter() {
            assert!(ev >= -1e-10, "Negative eigenvalue: {}", ev);
        }
    }
}
```

### 5.4 CI Matrix

Every PR runs conformance tests across all 4 languages:

```
┌─────────────┬──────────┬──────────┬──────────┬──────────┐
│ Test        │ Rust     │ Python   │ JS/WASM  │ C        │
├─────────────┼──────────┼──────────┼──────────┼──────────┤
│ vectors     │ ✓        │ ✓        │ ✓        │ ✓        │
│ properties  │ ✓        │ ✓        │ ✓        │ ✗ (TBD)  │
│ roundtrip   │ ✓        │ ✓        │ ✓        │ ✓        │
│ eigenvalue  │ ✓        │ ✓        │ ✓        │ ✓        │
│ ratio       │ ✓        │ ✓        │ ✓        │ ✓        │
└─────────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## 6. Version Compatibility Strategy

### 6.1 Semantic Versioning (all languages)

All packages share the same version number. `conservation-spectral v0.5.2` means the same API in Rust, Python, JS, and C.

```
MAJOR.MINOR.PATCH

MAJOR: Breaking API change in ANY language
MINOR: New feature, backward compatible in ALL languages
PATCH: Bug fix, no API change
```

### 6.2 Serialization Version Field

Every `.csp` file includes a version header:
```
[magic: 4 bytes] [major: u16] [minor: u16] [payload...]
```

Loaders MUST:
- Reject files with a newer major version
- Warn on newer minor version (may have unknown fields — ignore them)
- Accept same or older versions with backward-compat deserialization

### 6.3 API Evolution Rules

| Change | Allowed in | Bumps |
|---|---|---|
| Add new function | MINOR | 0.x+1.0 |
| Add new field to struct | MINOR (with default) | 0.x+1.0 |
| Remove function | MAJOR | x+1.0.0 |
| Change function signature | MAJOR | x+1.0.0 |
| Change struct field type | MAJOR | x+1.0.0 |
| Add new enum variant | MINOR (must have default) | 0.x+1.0 |
| Change serialization format | MAJOR | x+1.0.0 |

### 6.4 Feature Flags (Rust)

```toml
[features]
default = ["dense"]
dense = ["nalgebra"]
sparse = ["sprs"]
lapack = ["lapack-src"]
serde-support = ["serde", "rmp-serde"]
python = ["pyo3"]
wasm = ["wasm-bindgen"]
```

### 6.5 Release Process

1. Update version in `Cargo.toml`, `package.json`, `setup.py`, `conservation_spectral.h`
2. Run full conformance matrix
3. Tag: `v{MAJOR}.{MINOR}.{PATCH}`
4. Publish: `cargo publish`, `twine upload`, `npm publish`, GitHub release with `.h` + static lib artifacts

---

## 7. Package Layout

```
conservation-spectral/
├── core/                           # Rust crate
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs                  # Re-exports
│   │   ├── types.rs                # TensionGraph, Laplacian, EigenDecomposition, etc.
│   │   ├── ops.rs                  # build_laplacian, eigendecompose, etc.
│   │   ├── laplacian.rs            # Laplacian construction
│   │   ├── eigen.rs                # Eigendecomposition backend
│   │   ├── conservation.rs         # Conservation ratio computation
│   │   ├── anomaly.rs              # Anomaly detection
│   │   ├── fingerprint.rs          # Spectral fingerprint
│   │   ├── tracker.rs              # Sliding window tracker
│   │   ├── fix.rs                  # Correction suggestions
│   │   └── serde.rs                # Canonical serialization
│   └── tests/
│       ├── conformance.rs
│       └── property.rs
├── sys/                            # C FFI (cbindgen)
│   ├── Cargo.toml
│   ├── src/
│   │   └── lib.rs                  # #[no_mangle] extern "C" wrappers
│   └── cbindgen.toml
├── python/                         # PyO3 bindings
│   ├── Cargo.toml
│   ├── pyproject.toml
│   ├── src/
│   │   └── lib.rs                  # #[pymodule] + #[pyclass] + #[pyfunction]
│   ├── conservation_spectral/      # Python package
│   │   ├── __init__.py
│   │   ├── _bindings.pyi           # Type stubs
│   │   └── api.py                  # High-level Python API
│   └── tests/
│       ├── conformance/
│       └── property/
├── js/                             # WASM bindings (wasm-pack)
│   ├── Cargo.toml
│   ├── src/
│   │   └── lib.rs                  # #[wasm_bindgen] wrappers
│   ├── pkg/                        # wasm-pack output
│   ├── src/                        # TypeScript wrapper
│   │   ├── index.ts
│   │   └── types.ts
│   └── tests/
│       └── conformance/
├── c/                              # C distribution
│   ├── include/
│   │   └── conservation_spectral.h
│   ├── lib/                        # Pre-built static/dynamic libs
│   │   ├── linux-x64/libconservation_spectral.a
│   │   ├── macos-arm64/libconservation_spectral.a
│   │   └── windows-x64/conservation_spectral.lib
│   └── examples/
│       └── basic_analysis.c
├── tests/                          # Shared test infrastructure
│   ├── vectors/
│   │   ├── generate_vectors.py
│   │   └── *.json
│   └── conformance/
│       └── runner.py               # Cross-language runner
├── docs/
│   └── api/                        # Auto-generated API docs
├── Cargo.toml                      # Workspace root
└── README.md
```

---

## Appendix A: Implementation Priority

| Phase | Scope | Est. LOC (Rust) |
|---|---|---|
| **P0** | TensionGraph + Laplacian + eigendecompose + conservation_ratio + serialization | ~1,500 |
| **P1** | C FFI + Python bindings + conformance tests | ~800 |
| **P2** | JS/WASM bindings + spectral_fingerprint | ~500 |
| **P3** | ConservationTracker + detect_anomalies + suggest_correction | ~600 |
| **P4** | Sparse backend + LAPACK + performance tuning | ~400 |

**Total estimated Rust LOC: ~3,800**. Language bindings add ~200-400 LOC each.

---

## Appendix B: Numerical Notes

1. **Eigenvector sign ambiguity:** Eigenvectors are defined up to a scalar multiple. Conformance tests compare eigenvectors by checking `|dot(u,v)| ≈ 1` rather than exact element equality.

2. **Degenerate eigenvalues:** When eigenvalues are nearly equal, eigenvectors are unstable. Conservation ratios for degenerate modes should be computed on the *subspace* spanned by the degenerate eigenvectors, not individual vectors. Flagged in the report via `degenerate: true`.

3. **Sparse vs dense:** For graphs with >1000 vertices, use sparse representation. The Laplacian of a graph with max degree d has at most O(nd) nonzeros. Sparse eigendecomposition uses LOBPCG or Lanczos.

4. **Thread safety:** Rust core is `Send + Sync`. Python releases the GIL during computation. WASM runs single-threaded (Web Workers for parallelism). C is thread-safe for distinct objects; shared objects need external synchronization.
