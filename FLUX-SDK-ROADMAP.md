# FLUX SDK Roadmap — Constraint-Native Developer Toolkit

**Date:** 2026-05-28  
**Status:** Design Document  
**Depends on:** flux-vm-v3, constraint-theory-core, constraint-dialect, forgemaster

---

## Overview

FLUX is a constraint-native programming ecosystem. Programs execute on a stack-based VM where constraints are first-class operations — not assertions checked after the fact, but the operational semantics themselves. Every program terminates (max 4096 cycles), every result is deterministic, and every execution produces a SHA-256 proof certificate.

The FLUX SDK makes this power accessible: a language, compiler, runtime, CLI, and standard library that let developers write constraint-native code the same way they'd write Rust or Python.

---

## 1. FLUX Language Specification

### 1.1 Design Philosophy

FLUX borrows syntax from Rust (readability), semantics from Forth (stack-oriented), and guarantees from formal methods (proof certificates). The result is a language where:

- **Constraints are syntax**, not library calls
- **Termination is guaranteed**, not hoped for
- **Correctness is proven**, not tested
- **Parallelism is explicit**, not accidental

### 1.2 Type System

#### Primitive Types

| Type | Description | Representation |
|------|-------------|----------------|
| `Int` | Arbitrary-precision integer | GMP-backed |
| `Float` | Fixed-point decimal (exact) | Not IEEE 754 — lattice-snapped |
| `Lattice` | A point on a named lattice | `(basis, coords)` pair |
| `Conservation` | A conservation condition | `(Laplacian, eigenvector, bound)` |
| `Proof` | A SHA-256 certificate chain | `(root_hash, path, leaf)` |
| `Bool` | Boolean | `true` / `false` |
| `Unit` | Empty type | `()` |
| `Bytes` | Byte sequence | For I/O interop |

#### Compound Types

```
// Tuples
type Pair = (Int, Int)

// Structs (stack frames, really)
type VoiceLead = {
    from: Lattice,
    to: Lattice,
    distance: Int,
}

// Enums (tagged unions)
type ConstraintResult =
    | Satisfied(Proof)
    | Violated(String)
    | Inconclusive

// Arrays (fixed-size, bounded by max 4096 elements)
type Spectrum = [Float; 12]
```

#### Type Modifiers

```
// A value that has been constraint-verified
let x: verified Float = 440.0.snap(pythagorean)

// A proof-bearing value
let cert: proven Int = 42.check(positive).hash()

// A bounded loop variable
for i: bounded Int in 0..256 {
    // i is provably in [0, 256)
}
```

### 1.3 Formal Grammar (EBNF)

```
program       = { item } ;
item          = fn_def | type_def | const_def | use_decl ;
fn_def        = "fn" IDENT "(" params ")" [ "->" type ] block ;
type_def      = "type" IDENT "=" type_expr ;
const_def     = "const" IDENT ":" type "=" expr ";" ;
use_decl      = "use" path "::*"? ";" ;

block         = "{" { stmt } [ expr ] "}" ;
stmt          = let_stmt | assign_stmt | expr_stmt | constraint_stmt ;
let_stmt      = "let" IDENT [ ":" type ] "=" expr ";" ;
assign_stmt   = IDENT "=" expr ";" ;
expr_stmt     = expr ";" ;

(* Constraint statements — the heart of FLUX *)
constraint_stmt
              = snap_stmt | check_stmt | conserve_stmt
              | hash_stmt | verify_stmt | chain_stmt ;
snap_stmt     = "snap" expr "to" lattice_ref ;
check_stmt    = "check" expr "against" constraint_ref ;
conserve_stmt = "conserve" expr "by" conservation_ref ;
hash_stmt     = "hash" expr ;
verify_stmt   = "verify" expr "with" proof_ref ;
chain_stmt    = "chain" expr "into" proof_ref ;

(* Expressions *)
expr          = literal | IDENT | "(" expr ")"
              | expr "." IDENT            (* field access *)
              | expr bin_op expr          (* binary ops *)
              | un_op expr                (* unary ops *)
              | call_expr                 (* function call *)
              | stack_expr                (* stack operations *)
              | control_expr ;            (* control flow *)

call_expr     = IDENT "(" [ expr { "," expr } ] ")" ;
stack_expr    = "push" expr               (* push onto data stack *)
              | "pop"                      (* pop from data stack *)
              | "dup"                      (* duplicate top *)
              | "swap"                     (* swap top two *)
              | "over" ;                   (* copy second-to-top *)

control_expr  = if_expr | loop_expr | branch_expr | parallel_expr ;
if_expr       = "if" expr block [ "else" block ] ;
loop_expr     = "loop" IDENT "in" range block ;   (* bounded *)
branch_expr   = "branch" "{" { pattern "=>" expr "," } "}" ;
parallel_expr = "parallel" "{" { expr "," } "}" ;

(* Lattices and constraints *)
lattice_ref      = IDENT | "::" path ;
constraint_ref   = IDENT | "::" path ;
conservation_ref = IDENT | "::" path ;
proof_ref        = IDENT | "::" path ;

(* Ranges — always bounded *)
range        = expr ".." expr             (* exclusive end *)
             | expr "..=" expr ;          (* inclusive end *)

(* Types *)
type         = "Int" | "Float" | "Lattice" | "Conservation"
             | "Proof" | "Bool" | "Unit" | "Bytes"
             | "[" type ";" INT "]"
             | "(" type { "," type } ")"
             | "{" { IDENT ":" type "," } "}"
             | IDENT ;

literal      = INT_LIT | FLOAT_LIT | STRING_LIT | BOOL_LIT ;
bin_op       = "+" | "-" | "*" | "/" | "==" | "!=" | "<" | ">" | "<=" | ">="
             | "and" | "or" ;
un_op        = "-" | "not" ;

path         = IDENT { "::" IDENT } ;
IDENT        = /[a-zA-Z_][a-zA-Z0-9_]*/ ;
```

### 1.4 Constraint Operations — Detailed Semantics

#### `snap` — Snap to Nearest Lattice Point

```
// Snap a frequency to the nearest Pythagorean lattice point
let raw: Float = 442.3;
let tuned: Float = snap raw to flux::lattice::pythagorean;
// tuned = 440.0 (nearest Pythagorean point)
// Deterministic. Platform-independent. Exact.

// Custom lattice with named basis
let note: Lattice = snap raw to lattice::eisenstein(12);
// Snaps to the nearest point on the 12-tone Eisenstein lattice
```

**Semantics:** Given a value `v` and a lattice `L`, compute `argmin_{p ∈ L} ||v - p||`. The snap is deterministic because the lattice is discrete and well-ordered. Ties are broken by a canonical rule (lower coordinate in the lattice basis). The result type inherits from the lattice's value type.

#### `check` — Verify a Constraint

```
// Check that a voice leading satisfies smoothness
let vl = VoiceLead { from: c, to: e, distance: 4 };
let result = check vl against flux::music::smooth_voice_leading(threshold: 3);
// result: ConstraintResult::Violated("distance 4 > threshold 3")

// Chaining checks
let safe = check value against positive
    .and(bounded(0, 100))
    .and(lattice_aligned);
```

**Semantics:** Evaluates a predicate on a value. Returns `Satisfied(Proof)` with a partial proof certificate, or `Violated(reason)` with a diagnostic. Checks can be chained with `.and()` and `.or()` combinators.

#### `conserve` — Assert Conservation Condition

```
// Assert that total tension is conserved across a transformation
let before: Float = total_tension(chord_progression);
transform(chord_progression);
let after: Float = total_tension(chord_progression);

conserve (before, after) by flux::conservation::tension_graph(
    laplacian: flux::lattice::pythagorean,
    tolerance: 0.001
);
// If conservation is violated, the program halts with a proof of violation.
// If satisfied, execution continues with an updated proof chain.
```

**Semantics:** The conservation condition is checked against the Tension-Graph Laplacian for the specified lattice. This is not a soft assertion — it's a hard constraint on execution. Violation halts the program. Satisfaction extends the proof chain.

#### `hash` / `verify` / `chain` — Proof Operations

```
// Hash a value into the proof chain
let value: Int = 42;
let cert: Proof = hash value;
// cert = SHA-256(value || previous_chain_hash)

// Verify a proof certificate
let valid: Bool = verify cert with flux::proof::merkle;
// valid = true if the certificate chain is intact

// Chain multiple proofs together (Merkle tree)
let proofs: [Proof; 4] = [hash a, hash b, hash c, hash d];
let root: Proof = chain proofs into merkle_root;
// root = SHA-256(proofs[0] || proofs[1]) || SHA-256(proofs[2] || proofs[3])
```

### 1.5 Control Flow

#### Bounded Loops

```
// loop is ALWAYS bounded — max 4096 iterations
// The compiler rejects loops that can't be statically bounded
loop i in 0..256 {
    let harmonic = snap (base_freq * (i + 1).to_float()) to pythagorean;
    push harmonic;
}
```

#### Pattern Branching

```
// Branch on constraint results
let result = check voice_lead against smooth;
branch {
    Satisfied(proof) => {
        chain proof into session_proofs;
        continue_composition();
    },
    Violated(reason) => {
        log("Voice leading too rough: " + reason);
        snap voice_lead to nearest_valid;
    },
    Inconclusive => forfeit,
}
```

#### Parallel Execution

```
// All branches run in parallel, results are joined
let [bass, tenor, alto, soprano] = parallel {
    harmonize(voice: Bass, range: bass_range),
    harmonize(voice: Tenor, range: tenor_range),
    harmonize(voice: Alto, range: alto_range),
    harmonize(voice: Soprano, range: soprano_range),
};

// After join, constraints are checked across ALL voices
conserve [bass, tenor, alto, soprano] by flux::music::counterpoint_rules;
```

### 1.6 Complete Example: Constraint-Native Music Synthesizer

```flux
use flux::lattice::{pythagorean, eisenstein};
use flux::music::{voice_leading, counterpoint, plr_group};
use flux::proof::{merkle, chain};
use flux::conservation::tension_graph;

// Define a chord as a lattice-aligned structure
type Chord = [Lattice; 4];

// Generate a four-voice chord progression
fn progression(root: Lattice, scale: lattice, voices: Int) -> [Chord; 8] {
    let mut chords: [Chord; 8] = [];
    let mut current = root;
    let mut proofs: [Proof; 0] = [];
    
    loop i in 0..8 {
        // Snap each voice to the scale lattice
        let chord = parallel {
            snap (current + interval(0)) to scale,
            snap (current + interval(2)) to scale,
            snap (current + interval(4)) to scale,
            snap (current + interval(7)) to scale,
        };
        
        // Check smooth voice leading from previous chord
        if i > 0 {
            let vl = voice_leading::smooth(
                from: chords[i - 1],
                to: chord,
                max_distance: 3
            );
            let result = check vl against smooth_voice_leading;
            branch {
                Satisfied(p) => push p to proofs,
                Violated(_) => {
                    // Re-snap to nearest valid voice leading
                    chord = snap chord to voice_leading::valid(chords[i - 1]);
                },
            }
        }
        
        // Conserve total spectral tension
        conserve (chord_tension(chord), baseline_tension) 
            by tension_graph(laplacian: scale);
        
        chords[i] = chord;
        current = plr_group::next(current, scale);
    }
    
    // Produce proof certificate for the entire progression
    let root_proof = chain proofs into merkle;
    verify root_proof with flux::proof::merkle;
    
    chords
}

// Usage
fn main() -> Unit {
    let c_major = lattice::pythagorean(root: 261.63, octaves: 3);
    let prog = progression(root: c_major[0], scale: c_major, voices: 4);
    
    // Export as MIDI with embedded proof certificates
    flux::music::export_midi(prog, with_proofs: true);
}
```

---

## 2. FLUX Compiler

### 2.1 Architecture

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  FLUX       │     │  Typed AST  │     │  Optimized   │     │  FLUX-C      │
│  Source     │────▶│  (with      │────▶│  IR          │────▶│  Bytecode    │
│  (.flux)    │     │  constraint │     │  (constraint-│     │  (.fc)       │
│             │     │  types)     │     │  aware)      │     │  60 opcodes  │
└─────────────┘     └─────────────┘     └──────────────┘     └──────────────┘
     Parser              Type               Optimizer          Code Generator
                    Checker +                            (60-opcode target)
                    Constraint
                    Inference
```

### 2.2 Parser

**Input:** FLUX source (`.flux`)  
**Output:** Untyped AST  
**Implementation:** Recursive descent, hand-written (no parser generator — constraint on complexity)

```rust
// Simplified parser architecture
pub struct Parser {
    tokens: Vec<Token>,
    pos: usize,
    loop_depth: u16,  // Track nesting for bounded loop enforcement
}

impl Parser {
    pub fn parse(&mut self) -> Result<Program, ParseError> {
        let mut items = Vec::new();
        while !self.at_end() {
            items.push(self.parse_item()?);
        }
        // Validate: total loop iterations across all loops ≤ 4096
        self.validate_loop_bounds()?;
        Ok(Program { items })
    }
    
    fn parse_constraint_stmt(&mut self) -> Result<Stmt, ParseError> {
        match self.peek() {
            Token::Snap => self.parse_snap(),
            Token::Check => self.parse_check(),
            Token::Conserve => self.parse_conserve(),
            Token::Hash => self.parse_hash(),
            Token::Verify => self.parse_verify(),
            Token::Chain => self.parse_chain(),
            _ => Err(ParseError::ExpectedConstraint),
        }
    }
}
```

### 2.3 Type Checker + Constraint Inference

**Input:** Untyped AST  
**Output:** Typed AST with inferred constraint annotations

The type checker does more than standard Hindley-Milner:

1. **Standard type inference** — unification of expression types
2. **Constraint propagation** — track which values have been `snap`-ed, `check`-ed, `conserve`-d
3. **Proof tracking** — ensure every code path maintains a valid proof chain
4. **Termination analysis** — verify all loops are statically bounded

```rust
pub struct TypeChecker {
    env: TypeEnv,
    constraint_graph: ConstraintGraph,
    proof_state: ProofState,
    loop_bound: u16,  // Cumulative loop bound, max 4096
}

impl TypeChecker {
    pub fn check_program(&mut self, prog: &Program) -> Result<TypedProgram, TypeError> {
        // Phase 1: Standard type checking
        // Phase 2: Constraint flow analysis (which values are constrained)
        // Phase 3: Proof chain completeness (every path produces valid proofs)
        // Phase 4: Termination guarantee (all loops bounded)
    }
    
    fn infer_snap(&mut self, value: Expr, lattice: LatticeRef) -> Result<TypedExpr, TypeError> {
        let value_type = self.infer(value)?;
        let lattice_type = self.resolve_lattice(lattice)?;
        // Snap produces a lattice-aligned type
        let result_type = Type::LatticeAligned(value_type, lattice_type);
        // Mark this value as constraint-verified
        self.constraint_graph.mark_verified(result_type);
        Ok(TypedExpr::Snap { value, lattice, result_type })
    }
}
```

### 2.4 Optimizer

**Input:** Typed AST  
**Output:** Optimized IR

Optimization passes that are aware of constraint semantics:

| Pass | Description |
|------|-------------|
| **Constant Folding** | Evaluate `snap` at compile time when the input is constant |
| **Dead Constraint Elimination** | Remove `check` operations whose result is unused |
| **Proof Chain Compaction** | Merge adjacent `hash` + `chain` into a single Merkle step |
| **Loop Unrolling** | Unroll small bounded loops (≤ 16 iterations) |
| **Parallel Fusion** | Merge adjacent `parallel` blocks with compatible constraints |
| **Lattice Specialization** | Monomorphize generic lattice operations for known lattices |

```rust
pub fn optimize(ir: TypedIR) -> OptimizedIR {
    let ir = constant_folding(ir);
    let ir = dead_constraint_elimination(ir);
    let ir = proof_chain_compaction(ir);
    let ir = loop_unrolling(ir, max_unroll: 16);
    let ir = parallel_fusion(ir);
    let ir = lattice_specialization(ir);
    ir
}
```

### 2.5 Code Generator

**Input:** Optimized IR  
**Output:** FLUX-C bytecode (60 opcodes)

The 60 opcodes of the FLUX VM:

```
// Stack operations (10)
PUSH, POP, DUP, SWAP, OVER, ROT, DROP, PICK, ROLL, DEPTH

// Arithmetic (8)
ADD, SUB, MUL, DIV, MOD, NEG, ABS, CMP

// Constraint operations (6)
SNAP, CHECK, CONSERVE, HASH, VERIFY, CHAIN

// Control flow (8)
JMP, JZ, JNZ, CALL, RET, LOOP, BREAK, CONTINUE

// Parallel (4)
PAR_FORK, PAR_JOIN, PAR_MAP, PAR_REDUCE

// Memory (6)
LOAD, STORE, LOAD_IDX, STORE_IDX, ALLOC, FREE

// Type operations (4)
CAST, TYPEOF, IS, AS

// Lattice operations (4)
LATTICE_NEW, LATTICE_SNAP, LATTICE_NEAREST, LATTICE_DISTANCE

// Proof operations (4)
PROOF_NEW, PROOF_EXTEND, PROOF_VERIFY, PROOF_ROOT

// I/O (4)
READ, WRITE, FLUSH, DEBUG_PRINT

// Special (2)
NOP, HALT
```

```rust
pub struct CodeGenerator {
    bytecode: Vec<u8>,
    constants: ConstantPool,
    labels: HashMap<String, usize>,
}

impl CodeGenerator {
    pub fn generate(ir: OptimizedIR) -> FluxBytecode {
        let mut gen = CodeGenerator::new();
        for item in ir.items {
            gen.gen_item(item);
        }
        gen.emit(Op::Halt);
        FluxBytecode {
            version: 3,
            constants: gen.constants,
            code: gen.bytecode,
            metadata: ir.metadata,  // Source maps for debugging
        }
    }
}
```

---

## 3. FLUX Runtime

### 3.1 Stack-Based Interpreter

The reference interpreter. Runs anywhere — embedded systems, WASM, kernels.

```rust
pub struct FluxVM {
    data_stack: Vec<Value>,          // Main data stack
    call_stack: Vec<Frame>,          // Call frames
    proof_chain: ProofChain,         // Running proof certificate
    cycle_count: u16,                // Current cycle (max 4096)
    heap: Heap,                      // Dynamic memory
    lattice_cache: LatticeCache,     // Cached lattice structures
}

impl FluxVM {
    pub fn execute(&mut self, bytecode: &FluxBytecode) -> Result<ExecResult, VMError> {
        loop {
            // Termination guarantee
            self.cycle_count += 1;
            if self.cycle_count > 4096 {
                return Err(VMError::CycleLimitExceeded {
                    cycles: self.cycle_count,
                    proof: self.proof_chain.current_hash(),
                });
            }
            
            let op = self.fetch_op(bytecode)?;
            match op {
                Op::Snap => {
                    let lattice = self.pop_lattice_ref()?;
                    let value = self.pop()?;
                    let snapped = lattice.snap(value)?;
                    self.push(snapped);
                    self.proof_chain.extend(snapped);
                }
                Op::Check => {
                    let constraint = self.pop_constraint_ref()?;
                    let value = self.pop()?;
                    match constraint.check(&value) {
                        Ok(proof) => {
                            self.push(Value::Bool(true));
                            self.proof_chain.extend(proof);
                        }
                        Err(violation) => {
                            self.push(Value::Bool(false));
                            // Don't halt — let the program handle violation
                        }
                    }
                }
                Op::Conserve => {
                    let conservation = self.pop_conservation_ref()?;
                    let values = self.pop_n(2)?;  // before, after
                    match conservation.check(values[0], values[1]) {
                        Ok(proof) => self.proof_chain.extend(proof),
                        Err(violation) => {
                            // Conservation violation is fatal
                            return Err(VMError::ConservationViolated {
                                before: values[0],
                                after: values[1],
                                violation,
                                proof: self.proof_chain.finalize(),
                            });
                        }
                    }
                }
                Op::Halt => {
                    let proof = self.proof_chain.finalize();
                    return Ok(ExecResult {
                        stack: self.data_stack.clone(),
                        proof,
                        cycles: self.cycle_count,
                    });
                }
                // ... remaining 56 opcodes
                _ => self.execute_standard_op(op)?,
            }
        }
    }
}
```

### 3.2 JIT via Cranelift

For hot paths that need native speed. The JIT compiles FLUX-C bytecode to native machine code.

```rust
pub struct FluxJIT {
    cranelift: JITBuilder,
    compiled_cache: HashMap<FunctionId, *const u8>,
    target_arch: Arch,  // x86_64, AArch64, RISC-V
}

impl FluxJIT {
    pub fn compile(&mut self, bytecode: &FluxBytecode, func_id: FunctionId) -> Result<CompiledFn, JITError> {
        // Translate FLUX-C bytecode to Cranelift IR
        let mut builder = FunctionBuilder::new(&mut self.cranelift);
        
        for op in bytecode.iter() {
            match op {
                Op::Snap => {
                    // Snap becomes: call to lattice_snap_native (C ABI)
                    let val = builder.stack_load(Type::F64, 0);
                    let lattice = builder.stack_load(Type::I64, 8);
                    let snapped = builder.ins().call(
                        self.snap_native_fn(),
                        &[val, lattice]
                    );
                    builder.stack_store(snapped, 0);
                }
                Op::Conserve => {
                    // Conservation check with inline proof extension
                    // Falls back to interpreter on violation
                }
                // ... standard ops compile directly
                Op::Add => {
                    let b = builder.stack_pop(Type::F64);
                    let a = builder.stack_pop(Type::F64);
                    builder.stack_push(builder.ins().fadd(a, b));
                }
                _ => { /* ... */ }
            }
        }
        
        let code_ptr = builder.finalize();
        self.compiled_cache.insert(func_id, code_ptr);
        Ok(CompiledFn { ptr: code_ptr })
    }
}
```

**Tiered compilation strategy:**

| Tier | Threshold | Strategy |
|------|-----------|----------|
| 0 (Interpreter) | First execution | Interpret bytecode directly |
| 1 (Baseline JIT) | 10 calls | Simple Cranelift translation, no optimization |
| 2 (Optimized JIT) | 100 calls | Full optimization, constraint-specialized |
| 3 (GPU Offload) | 1000 calls + parallel ops | Move to CUDA kernel |

### 3.3 GPU Backend (CUDA)

Parallel constraint checking on NVIDIA GPUs. For `parallel` blocks and batch lattice snaps.

```cuda
// CUDA kernel for parallel Pythagorean lattice snap
__global__ void snap_pythagorean_kernel(
    const float* __restrict__ values,
    float* __restrict__ output,
    const int* __restrict__ basis,
    int n
) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx >= n) return;
    
    float v = values[idx];
    
    // Find nearest point on Pythagorean lattice
    // Lattice basis: (1, 0) and (0, 1) scaled by fundamental
    int a = roundf(v / basis[0]);
    int b = 0;
    float snapped = a * basis[0];
    float best_dist = fabsf(v - snapped);
    
    // Check nearby lattice points (3x3 neighborhood)
    for (int da = -1; da <= 1; da++) {
        for (int db = -1; db <= 1; db++) {
            float candidate = (a + da) * basis[0] + (b + db) * basis[1];
            float dist = fabsf(v - candidate);
            if (dist < best_dist) {
                best_dist = dist;
                snapped = candidate;
            }
        }
    }
    
    output[idx] = snapped;
}
```

**When GPU kicks in:**

- `parallel` blocks with > 256 elements
- Batch `snap` operations on arrays
- Spectral analysis (FFT-based conservation checking)
- Merkle tree construction for large proof chains

---

## 4. FLUX CLI

### 4.1 Commands

```bash
# Compile FLUX source to bytecode
$ flux build composition.flux -o composition.fc
# Output: composition.fc (FLUX-C bytecode, ~4KB)
#         composition.fc.proof (proof metadata)

# Execute bytecode
$ flux run composition.fc
# Output: execution result + SHA-256 certificate
# CERT: a3f8c2d1e4b5...

# Verify a proof certificate
$ flux verify composition.fc.proof --root a3f8c2d1e4b5...
# ✓ Proof chain valid (47 steps, 0 violations)
# ✓ Conservation conditions: 12/12 satisfied
# ✓ Termination: 847/4096 cycles

# Debug mode — step through with constraint inspection
$ flux debug composition.fc
flux> step 1
  [0] PUSH 440.0
  Stack: [440.0]
  Proof: root
  
flux> step 2
  [1] SNAP pythagorean
  Stack: [440.0] (snapped to Pythagorean lattice)
  Proof: sha256(440.0 || root)
  
flux> constraints
  Active: none
  Pending: smooth_voice_leading (will check at step 14)
  
flux> continue
  Execution completed in 847 cycles
  Proof: a3f8c2d1e4b5...

# Export bindings for other languages
$ flux export --lang python composition.fc -o composition_flux.py
$ flux export --lang rust composition.fc -o composition_flux.rs
$ flux export --lang js composition.fc -o composition_flux.js
$ flux export --lang c composition.fc -o composition_flux.h

# Inspect bytecode
$ flux disasm composition.fc
  0x0000: PUSH_CONST 0      ; 440.0
  0x0002: LOAD_LATTICE 1    ; pythagorean
  0x0004: SNAP
  0x0005: STORE_LOCAL 0     ; tuned_freq
  0x0007: PUSH_CONST 2      ; 256
  ...

# REPL mode for interactive exploration
$ flux repl
flux> let x = snap 443.2 to pythagorean
440.0
flux> check x > 0
Satisfied(Proof(sha256:7f2a...))
flux> hash x
Proof(sha256:b3c1...)
```

### 4.2 Generated Bindings Example (Python)

```python
# composition_flux.py — auto-generated by flux export
import flux

class Composition:
    """FLUX-compiled constraint-native composition with proof certificates."""
    
    def __init__(self):
        self._vm = flux.VM()
        self._bytecode = flux.load_bytecode("composition.fc")
    
    def run(self) -> flux.ExecResult:
        """Execute the composition. Returns result + proof certificate."""
        return self._vm.execute(self._bytecode)
    
    def progression(self, root: float, scale: str = "pythagorean") -> list[flux.Chord]:
        """Call the progression function from FLUX bytecode."""
        return self._vm.call("progression", [root, scale])

# Usage
comp = Composition()
result = comp.run()
print(f"Cycles: {result.cycles}/4096")
print(f"Proof:  {result.proof.root_hash}")
print(f"Valid:  {flux.verify(result.proof)}")
```

---

## 5. FLUX Standard Library

### 5.1 `flux::lattice` — Lattice Primitives

```flux
use flux::lattice::{Pythagorean, Eisenstein, Custom};

// Built-in Pythagorean lattice (a² + b² = c²)
let pyth = Pythagorean::new(fundamental: 1.0, octaves: 5);

// Built-in Eisenstein lattice (hexagonal, circle of fifths)
let eis = Eisenstein::new subdivisions: 12;

// Custom lattice from basis vectors
let my_lattice = Custom::new(
    basis: [(1, 0), (0.5, 0.866)],  // Hexagonal basis
    bounds: (-100, 100),
);

// Operations
let nearest = pyth.nearest(3.7);        // 3.605... (= √13)
let distance = pyth.distance(3.7);       // 0.095...
let index = pyth.index(nearest);         // O(log N) lookup
let point = pyth.at_index(42);           // Reverse lookup
let range = pyth.in_range(3.0, 4.0);    // All lattice points in [3, 4]
```

### 5.2 `flux::conservation` — Tension-Graph Laplacian

```flux
use flux::conservation::{TensionGraph, Spectral};

// Build a Tension-Graph Laplacian for a chord progression
let tg = TensionGraph::new(
    nodes: [c_major, f_major, g_major, c_major],
    edges: [(0,1), (1,2), (2,3)],  // Adjacency
    weight_fn: spectral_tension,    // Edge weight = spectral tension
);

// Compute eigenvalues (conservation = alignment with eigenvectors)
let eigen = tg.eigenvalues();
let conservation_score = eigen.dominant_ratio();  // 112× or not

// Check conservation condition
conserve (before_tension, after_tension) by tg;

// Spectral analysis of a system
let spectrum = Spectral::analyze(tg);
let modes = spectrum.dominant_modes(n: 3);  // Top 3 eigenmodes
let innovation_potential = spectrum.gap_fraction();  // Fraction of "empty" space
```

### 5.3 `flux::proof` — SHA-256 Certificates

```flux
use flux::proof::{Merkle, Certificate, Chain};

// Build a Merkle tree from execution steps
let tree = Merkle::new();
loop i in 0..256 {
    let step_result = compute_step(i);
    let leaf = hash step_result;
    tree.insert(leaf);
}

// Get root hash (commitment to entire execution)
let root: Proof = tree.root();
// root = SHA-256(SHA-256(leaf_0 || leaf_1) || SHA-256(leaf_2 || leaf_3) || ...)

// Verify inclusion of a specific step
let proof_path = tree.proof(index: 42);
let valid = verify proof_path with root;
// valid = true (Merkle proof that step 42 is in the tree)

// Chain multiple Merkle roots (cross-session proofs)
let session_chain = Chain::new();
session_chain.append(root);
session_chain.append(next_root);
let chain_proof = session_chain.finalize();
// chain_proof proves the entire sequence of sessions
```

### 5.4 `flux::music` — PLR Group, Voice Leading, Counterpoint

```flux
use flux::music::{PLRGroup, VoiceLeading, Counterpoint};

// PLR group operations (Parallel, Leading-tone exchange, Relative)
let plr = PLRGroup::new(lattice: pythagorean);
let c_major = Chord::new([C, E, G]);
let a_minor = plr.relative(c_major);     // R(C) = Am
let f_major = plr.parallel(a_minor);     // P(Am) = F
let e_minor = plr.leading(c_major);      // L(C) = Em

// Voice leading with constraint checking
let vl = VoiceLeading::new(
    from: [C4, E4, G4],
    to: [F4, A4, C5],
    rules: [smoothness(2), no_parallels, contrary_motion_preferred],
);
let result = check vl against smooth_voice_leading;

// Species counterpoint generator
let cp = Counterpoint::species_1(
    cantus_firmus: [D4, F4, E4, D4, C4, D4, E4, F4, E4, D4],
    lattice: pythagorean,
    constraints: [no_tritone, smooth_voice_leading, conservation],
);
let counterpoint = cp.generate();
// Each note is lattice-aligned, each interval is constraint-verified
// The entire counterpoint has a proof certificate
```

### 5.5 `flux::fleet` — Agent Deliberation, Consensus, Emergence

```flux
use flux::fleet::{Agent, Deliberation, Consensus, Emergence};

// Create a fleet of constraint-aware agents
let fleet = Fleet::new(n_agents: 16);

// Each agent has a deliberation cycle
loop round in 0..128 {
    parallel {
        // Agents consider proposals
        fleet.consider(proposal(round)),
    }
    
    // Resolve: agents that have enough confidence commit
    let resolutions = fleet.resolve(confidence_threshold: 0.95);
    
    // Agents that can't decide forfeit
    let forfeits = fleet.forfeit(timeout: round * 2);
    
    // Check fleet-wide conservation
    conserve (fleet.tension_before, fleet.tension_after) 
        by fleet_laplacian;
}

// Detect emergent patterns
let patterns = Emergence::detect(fleet.history);
branch {
    Some(pattern) => {
        log("Fleet discovered: " + pattern.description);
        let proof = hash pattern;
        chain proof into fleet_knowledge;
    },
    None => continue,
}
```

---

## 6. Integration Examples

### 6.1 Web Server Middleware — Constraint-Checking API Gateway

```rust
// In your Actix-web / Axum / Rocket app
use flux::{VM, Bytecode, Proof};

async fn constraint_middleware(req: Request, next: Next) -> Response {
    // Load the constraint program for this endpoint
    let bytecode = Bytecode::load("api_constraints.fc").unwrap();
    let mut vm = VM::new();
    
    // Build constraint input from the request
    let input = flux::value!({
        "user_id": req.headers["x-user-id"],
        "action": req.method.to_string(),
        "resource": req.path(),
        "rate_limit_remaining": get_rate_limit(&req),
    });
    
    // Execute constraint program
    match vm.execute_with_input(&bytecode, input) {
        Ok(result) if result.stack_top() == Value::Bool(true) => {
            // Constraints satisfied — attach proof certificate to response
            let mut response = next.run(req).await;
            response.headers.insert(
                "X-Flux-Proof",
                result.proof.root_hash().to_string(),
            );
            response
        }
        Ok(result) => {
            // Constraint violated — reject with proof of why
            Response::new(403).json(json!({
                "error": "constraint_violated",
                "proof": result.proof.root_hash(),
                "cycles": result.cycles,
            }))
        }
        Err(VMError::CycleLimitExceeded { .. }) => {
            // The constraint program itself ran too long — deny by default
            Response::new(500).json(json!({"error": "constraint_timeout"}))
        }
        Err(VMError::ConservationViolated { violation, .. }) => {
            Response::new(500).json(json!({
                "error": "conservation_violated",
                "detail": violation.to_string(),
            }))
        }
    }
}
```

### 6.2 Database Constraint Layer

```sql
-- Replace PostgreSQL CHECK constraints with FLUX verification
CREATE TABLE transactions (
    id SERIAL PRIMARY KEY,
    from_account INT,
    to_account INT,
    amount DECIMAL(19, 4),
    -- Instead of: CHECK (amount > 0)
    -- Use: FLUX constraint with proof certificate
    flux_proof BYTEA  -- SHA-256 proof that the transaction is valid
);

-- Trigger: run FLUX verification before INSERT
CREATE FUNCTION verify_transaction() RETURNS TRIGGER AS $$
    import flux
    def verify():
        vm = flux.VM()
        result = vm.execute("transaction_constraints.fc", {
            "from": NEW.from_account,
            "to": NEW.to_account,
            "amount": NEW.amount,
        })
        if not result.valid:
            raise Exception(f"Constraint violated: {result.proof}")
        NEW.flux_proof = result.proof_bytes
        return NEW
$$ LANGUAGE plpython3u;
```

### 6.3 Audio Plugin (VST/AU) — Real-Time Constraint Synthesis

```flux
// harmonizer.flux — runs inside a VST plugin
// Called per audio buffer (128 samples at 48kHz ≈ every 2.67ms)

fn process_buffer(input: [Float; 128], chords: [Chord; 4]) -> [Float; 128] {
    let mut output: [Float; 128] = [];
    let mut last_harmony: Chord = chords[0];
    
    loop i in 0..128 {
        let freq = input[i];
        
        // Snap input to nearest chord tone (real-time lattice alignment)
        let harmony = snap freq to flux::lattice::pythagorean(root: chords[0].root);
        
        // Check voice leading from previous frame
        if i > 0 {
            let vl = check voice_leading(last_harmony, harmony) 
                against smooth(threshold: 2);
            branch {
                Satisfied(_) => (),  // Good, keep it
                Violated(_) => {
                    // Re-snap to nearest valid voice leading
                    harmony = snap harmony to valid_voice_leading(last_harmony);
                },
            }
        }
        
        output[i] = harmony.to_frequency();
        last_harmony = harmony;
    }
    
    // Proof certificate covers the entire buffer
    hash output;
    output
}
```

### 6.4 CI/CD Pipeline — FLUX as Verification Step

```yaml
# .github/workflows/verify.yml
name: FLUX Verification

on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: flux-lang/setup-flux@v1
        with:
          version: '0.3.0'
      
      # Compile and verify all FLUX constraint programs
      - name: Build
        run: flux build constraints/*.flux -o build/
      
      # Verify proof certificates from previous run
      - name: Verify Proofs
        run: |
          for proof in artifacts/*.proof; do
            flux verify "$proof" --strict
          done
      
      # Run constraint tests
      - name: Test
        run: flux test constraints/
      
      # Export bindings for downstream consumers
      - name: Export Bindings
        run: |
          flux export --lang rust build/ -o bindings/rust/
          flux export --lang python build/ -o bindings/python/
      
      # Upload proof artifacts
      - uses: actions/upload-artifact@v4
        with:
          name: flux-proofs
          path: build/*.proof
```

---

## 7. Implementation Timeline

### Phase 1: Foundation (Weeks 1-4)

| Week | Deliverable |
|------|-------------|
| 1 | Language spec finalized, parser implemented |
| 2 | Type checker with constraint inference |
| 3 | Code generator targeting 60-opcode FLUX-C |
| 4 | Stack-based interpreter, `flux build` and `flux run` |

### Phase 2: Developer Experience (Weeks 5-8)

| Week | Deliverable |
|------|-------------|
| 5 | `flux verify`, `flux debug`, REPL mode |
| 6 | `flux export` for Python and Rust |
| 7 | Standard library: `flux::lattice`, `flux::proof` |
| 8 | Standard library: `flux::music`, `flux::conservation` |

### Phase 3: Performance (Weeks 9-12)

| Week | Deliverable |
|------|-------------|
| 9 | Cranelift JIT (x86_64) |
| 10 | Cranelift JIT (AArch64, RISC-V) |
| 11 | CUDA backend for parallel ops |
| 12 | Tiered compilation, benchmarks |

### Phase 4: Ecosystem (Weeks 13-16)

| Week | Deliverable |
|------|-------------|
| 13 | `flux::fleet` library, agent deliberation |
| 14 | Web middleware integration, database constraint layer |
| 15 | Audio plugin (VST/AU) integration |
| 16 | CI/CD integration, documentation, examples

---

## 8. Repository Structure

```
flux-sdk/
├── crates/
│   ├── flux-syntax/          # Parser, AST, lexer
│   ├── flux-types/           # Type system, constraint types
│   ├── flux-check/           # Type checker, constraint inference
│   ├── flux-optimize/        # Optimizer passes
│   ├── flux-codegen/         # Code generator (FLUX-C bytecode)
│   ├── flux-vm/              # Stack-based interpreter
│   ├── flux-jit/             # Cranelift JIT compiler
│   ├── flux-cuda/            # GPU backend
│   └── flux-cli/             # CLI tool
├── std/                      # Standard library
│   ├── flux-lattice/
│   ├── flux-conservation/
│   ├── flux-proof/
│   ├── flux-music/
│   └── flux-fleet/
├── bindings/                 # Exported bindings
│   ├── python/
│   ├── rust/
│   ├── js/
│   └── c/
├── examples/                 # Integration examples
│   ├── web-middleware/
│   ├── database-constraints/
│   ├── audio-plugin/
│   └── ci-cd/
├── tests/                    # Test suite
│   ├── conformance/          # Language spec conformance
│   ├── constraint/           # Constraint operation tests
│   ├── proof/                # Proof certificate tests
│   └── performance/          # Benchmarks
├── docs/                     # Documentation
│   ├── spec/                 # Language specification
│   ├── opcodes/              # Opcode reference
│   └── tutorials/
└── Cargo.toml
```

---

## 9. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Stack-based VM | Matches FLUX's Forth heritage, simpler to prove correct, 60-opcode ceiling is natural |
| 4096 cycle limit | Guarantees termination, forces developers to think about algorithmic complexity |
| SHA-256 proofs (not SNARKs) | Simpler, auditable, no trusted setup, sufficient for the constraint domain |
| No IEEE 754 floats | Floats are approximate; FLUX values snap to exact lattice points |
| `snap` as first-class op | The snap IS the constraint — it's not a library call, it's the language |
| Cranelift (not LLVM) | Faster compilation, better for JIT, sufficient optimization for FLUX's bounded programs |
| Bounded arrays (max 4096) | Consistent with cycle limit, prevents unbounded memory in a bounded-execution system |
| Conservation is fatal | If conservation is violated, the program is wrong — there's no "close enough" in constraint-native computing |

---

## 10. Open Questions

1. **Foreign function interface** — How should FLUX call into C/Rust? Direct FFI undermines the termination guarantee. Possible solution: FFI calls count toward cycle limit, FFI results must be `snap`-ed before use.

2. **Async/I/O model** — FLUX programs terminate in ≤ 4096 cycles, but I/O might not complete. Possible solution: I/O returns a `Future` that must be `await`-ed (counts as a cycle).

3. **Persistent proof chains** — Should proof certificates survive across sessions? The `chain` op supports this, but the storage and verification of cross-session proofs needs a standard.

4. **Lattice extensibility** — Can users define arbitrary lattices at runtime, or must they be compile-time? Runtime lattices would need to be included in the proof certificate.

5. **GPU memory model** — How does the CUDA backend interact with the proof chain? GPU-side proofs need to be brought back to the host for verification.

---

*The constraint IS the computation. This SDK makes that a developer experience, not a research paper.*
