# C-SCHED: Consonance Scheduler Roadmap

> **Threads are voices in a choir. The scheduler is the conductor.**
> Replace round-robin with harmonic scheduling — group consonant threads together, stagger dissonant ones.

---

## 🧭 North Star

C-SCHED is a Linux kernel scheduler that schedules threads not by priority or fairness, but by **harmonic consonance** — the degree to which threads' resource-access patterns agree. Threads sharing cachelines, locks, and I/O channels are "harmonically related" and should run concurrently. Threads that conflict should be separated in time.

The long-term vision: a RISC-V extension (on caffeinix) with hardware consonance counters that feed a spectral scheduling unit.

---

## 📦 Milestone 0: The Idea — Thread as Voice

Every thread has a **pitch** determined by its resource usage signature:

| Dimension | Measured By | Pitch Analogy |
|-----------|-------------|---------------|
| **CPU** | IPC, branch mispredict rate, FPU usage | Fundamental frequency |
| **Memory** | Working set, cache miss rate, page access set | Overtone series |
| **I/O** | Block device, file descriptor, UART/channel | Timbre / instrument |
| **Locking** | Spinlock / mutex held or contended | Dissonant interval |

**Harmonic relationship**: threads accessing the same resources (same cache lines, same locks, same I/O streams) are harmonically related — they "vibrate" together. **Dissonance** = conflict.

---

## 📦 Milestone 1: Userspace Proof-of-Concept (2 weeks)

### Goal
A userspace library that schedules threads on Linux using `sched_setattr` + `SCHED_DEADLINE` or `SCHED_FIFO`, grouping them by a computed consonance metric.

### Architecture

```
                  ┌──────────────────────┐
                  │   csched daemon       │
                  │   (userspace)         │
                  └──────┬───────────────┘
                         │ perf_event_open  │
                         │ /proc/self/smaps  │
                         │ /proc/locks       │
                         ▼
                  ┌──────────────────────┐
                  │  Consonance Engine   │
                  │  - thread profiling  │
                  │  - resource graph    │
                  │  - Fiedler partition │
                  └──────┬───────────────┘
                         │ sched_setattr()
                         │ cpu affinity masks
                         ▼
                  ┌──────────────────────┐
                  │  Linux CFS / DL      │
                  │  (underlying HW sched)│
                  └──────────────────────┘
```

### Deliverables

| Artifact | Description |
|----------|-------------|
| `csched.h` | Public API header |
| `libcsched.so` | Shared library |
| `csched-daemon` | Background daemon managing partitions |
| `csched-top` | `top(1)`-style terminal UI showing consonance |
| `bench/` | Lock-heavy, mixed I/O+compute, and cache-thrashing benchmarks |

### Userspace API

```c
// Initialize the consonance scheduler
int csched_init(struct csched_config *cfg);
// cfg.mode: CSCHED_MODE_AUTO | CSCHED_MODE_MANUAL
// cfg.quantum_us: time slice per consonance group
// cfg.interval_ms: re-partition interval

// Register a thread for harmonic scheduling
int csched_add_thread(pid_t tid, struct thread_profile *profile);
// profile: optional hint of resource usage pattern

// Remove a thread from consonance tracking
int csched_remove_thread(pid_t tid);

// Query current system consonance (0.0 = war, 1.0 = perfect harmony)
int csched_get_consonance(double *out);

// Get the current Fiedler partition assignment for each tracked thread
int csched_get_partition(int *groups, int max_groups);
// groups[i] = partition number for thread i
// returns number of partitions

// Force re-partition (e.g., when a contended lock is detected)
int csched_repartition(void);

// Set scheduling strategy
int csched_set_strategy(enum csched_strategy s);
// CSCHED_STRAT_CONSONANCE  — maximize harmony (default)
// CSCHED_STRAT_FAIRNESS    — fall back to proportional fairness
// CSCHED_STRAT_BALANCED    — weighted combination
```

### Resource Profiling

The profiling engine collects per-thread metrics using:
- **`perf_event_open()`** — cache misses, branch mispredictions, instructions retired
- **`/proc/[tid]/maps`** — memory region footprints
- **`/proc/locks`** — lock contention (POSIX file locks, futexes)
- **`/proc/[tid]/sched`** — sched statistics (preemptions, wait time)

From this, build a **resource vector** for each thread:

```c
struct resource_vector {
    uint64_t *cache_lines;   // bloom filter of accessed cache lines
    uint64_t *pages;         // page access bitmap hashes
    uint64_t *lock_ids;      // locks held / contended
    uint64_t *io_channels;   // file descriptors / block devices
    size_t n_locks;
    size_t n_io;
};
```

### Consonance Metric

1. Build an **interaction graph** `G = (V, E)` where:
   - Vertices = tracked threads
   - Edge weight `w(i,j)` = interaction frequency × resource similarity

2. **Resource similarity** between threads i and j:
   ```
   J(i,j) = |R_i ∩ R_j| / |R_i ∪ R_j|
   ```
   where R_i is the set of resources accessed by thread i (page frame numbers, cache line sets, lock addresses, file descriptors).

3. **Interaction frequency** is sampled from:
   - Futex contention events (`PERF_COUNT_SW_CONTEXT_SWITCHES` with trace)
   - Lock acquisition order (inferred from strace / ltrace)
   - CPU migration patterns (threads that bounce to the same core)

4. **Laplacian matrix**:
   ```
   L_ij = Σ_k w(i,k)    if i == j
        = -w(i,j)        otherwise
   ```

5. **Conservation ratio** for a schedule:
   ```
   Conservation = 1 - (dissonant_pairs / total_pairs)
   ```
   High (> 0.7) = threads coexist peacefully. Low (< 0.4) = partition decay.

### Fiedler Partitioning Algorithm

The **Fiedler vector** is the eigenvector corresponding to the second-smallest eigenvalue of the Laplacian. It provides the optimal 2-way partition of threads (spectral bisection):

```
Fiedler(i) > 0  →  group A (consonant within)
Fiedler(i) < 0  →  group B (consonant within)
```

**Practical kernel implementation** (no eigen-decomposition needed):

```c
// Lanczos iteration: fast spectral bisection
// Input: interaction graph (n threads, m edges)
// Output: partition assignment for each thread

static int fiedler_partition(struct interaction_graph *g,
                             int *partition_out, int n_threads)
{
    double *b = calloc(n_threads, sizeof(double));
    double *r = calloc(n_threads, sizeof(double));

    // Initialize with random vector
    for (int i = 0; i < n_threads; i++) b[i] = (double)rand() / RAND_MAX;

    // 20-50 Lanczos iterations for n=1024 threads
    int n_iter = min(50, n_threads / 2);
    for (int k = 0; k < n_iter; k++) {
        // r = L * b  (sparse mat-vec, O(m))
        sparse_laplacian_multiply(g, b, r);
        // Rayleigh quotient estimate of second eigenvalue
        double theta = dot(r, b) / dot(b, b);
        // b = r - theta * b  (shift-invert)
        axpy(r, -theta, b, n_threads);
        // Orthogonalize against first eigenvector (all-ones)
        double mean = sum(r) / n_threads;
        for (int i = 0; i < n_threads; i++) r[i] -= mean;
        normalize(r, n_threads);
        // swap b and r for next iteration
        double *tmp = b; b = r; r = tmp;
    }

    // Partition by sign of Fiedler vector
    for (int i = 0; i < n_threads; i++) {
        partition_out[i] = (b[i] > 0) ? 1 : 0;
    }

    free(b); free(r);
    return 2;  // number of partitions
}
```

For k-way partitioning (> 2 groups): recursively apply spectral bisection on each partition.

**Computational cost**: O(k·m·n_iter) where k = desired groups, m = edges in interaction graph, n_iter ≈ 20–50. For 1024 threads with sparse graph (~10 edges/thread), this is ~500k mat-vec ops — fine for a repartition interval of 100ms–1s.

### Scheduling Strategy

For each scheduling round:

```
1. Profile threads → build resource vectors
2. Compute Jaccard similarity → interaction graph
3. Fiedler partition → consonant groups
4. For each group:
   a. Pick the group with lowest recent CPU time (fairness)
   b. Run all threads in that group consecutively (SCHED_FIFO within group)
   c. On context switch to a thread in a different group → first check
      if it conflicts with the current group (dissonance penalty)
5. Monitor conservation ratio in real-time
6. If conservation drops below threshold → re-partition
```

**Example: 8 threads, 3 partitions**

```
Partition 0 (CPU-bound):     T1, T5
Partition 1 (I/O heavy):     T2, T3, T7
Partition 2 (lock contending): T4, T6, T8

Schedule timeline:
[T1,T5] → [T2,T3,T7] → [T4,T6,T8] → [T1,T5] → ...
```

Within each partition, threads use FIFO (no further conflict). Between partitions, threads are separated to avoid cache thrashing and lock contention.

### Test Benchmarks

| Benchmark | Description | Target Improvement vs CFS |
|-----------|-------------|--------------------------|
| `lockbench` | N threads fighting over M locks | +15% throughput |
| `pipeline` | Producer-consumer with ring buffer | +20% throughput |
| `cachethrash` | Threads stomping overlapping cache lines | +25% throughput |
| `mixed-io` | Mix of compute + I/O threads | -20% tail latency |
| `powerbench` | CXL memory bandwidth contention | -10% power draw |

---

## 📦 Milestone 2: Linux Kernel Module (2 months)

### Goal
A loadable kernel module (`csched.ko`) that hooks into the CFS scheduler and provides native consonance scheduling for SCHED_NORMAL threads.

### Kernel Integration Points

| Hook | Mechanism | File |
|------|-----------|------|
| Task wakeup | `sched_class->select_task_rq()` | `kernel/sched/fair.c` |
| Task tick | `sched_class->task_tick()` | `kernel/sched/fair.c` |
| Task fork | `sched_fork()` | `kernel/sched/core.c` |
| Context switch | `context_switch()` / `__schedule()` | `kernel/sched/core.c` |
| Perf events | `perf_event_create_kernel_counter()` | `kernel/events/core.c` |
| Tracepoints | `sched_switch`, `sched_wakeup`, `sched_migrate_task` | `include/trace/events/sched.h` |

### Module Architecture

```
                       ┌─────────────────────────┐
                       │   /proc/csched           │
                       │   Control + Visualization│
                       └──────────┬──────────────┘
                                  │
┌────────┐  ┌────────┐  ┌────────┴────────┐  ┌──────────┐
│ Perf   │  │ Trace  │  │  Consonance     │  │ Partition│
│ Events │──│ Points │──│  Engine          │──│ Scheduler│
│        │  │        │  │  - Interaction  │  │  - Per-CPU│
│        │  │        │  │    Graph Builder │  │    RunQ   │
│        │  │        │  │  - Fiedler       │  │  - Group  │
│        │  │        │  │    Partitioner   │  │    Affinity│
│        │  │        │  │  - Conservation  │  │  - Migrate│
│        │  │        │  │    Monitor       │  │    Cost   │
└────────┘  └────────┘  └──────────────────┘  └──────────┘
```

### `csched_entity` — Per-Task Extension

```c
struct csched_entity {
    struct list_head consonant_group;   // list of entities in same partition
    int             partition_id;       // current Fiedler partition
    u64             *page_signature;    // bloom filter of accessed pages
    u64             *lock_signature;    // bloom filter of contended locks
    u64             *cacheline_signature; // bloom filter of L2 cache lines
    u64             last_update;        // last profile refresh (jiffies)
    u32             cpu_affinity_hint;  // preferred CPU from partition
    atomic_t        contention_count;   // sampled lock contention events
};
```

### `/proc/csched` Interface

```
$ cat /proc/csched

C-SCHED Consonance Scheduler v0.2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

System Consonance: 0.73  (scale 0.0–1.0)
Conservation:      0.68  (threshold: 0.40)
Last Partition:    2.34s ago
Graph Size:        47 threads, 312 edges

Partition 0 (12 threads) — Consonance: 0.91
  PID  COMM             CPU%  MEM%  PAGESIG     LOCKS
  1234 worker_0         12.3   4.2  0xab1c...   (none)
  1241 worker_3          9.8   3.9  0xab1d...   (none)
  1250 gc_thread         1.2   1.1  0xab1e...   (none)
  ...

Partition 1 (18 threads) — Consonance: 0.85
  PID  COMM             CPU%  MEM%  PAGESIG     LOCKS
  3101 db_reader        34.1  12.3  0xde41...   mutex:0xffff...
  3105 db_writer        28.7  11.8  0xde42...   mutex:0xffff...
  3110 log_flusher       3.2   0.9  0xde43...   (none)
  ...

Partition 2 (17 threads) — Consonance: 0.72
  PID  COMM             CPU%  MEM%  PAGESIG     LOCKS
  5102 http_accept      22.1   8.7  0x441f...   spinlock:0x...
  5110 http_worker_0    15.4   6.2  0x4420...   spinlock:0x...
  ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Consonance Heatmap (thread × thread):
    │  T0  T1  T2  T3  T4  T5  T6  T7
 ───┼──────────────────────────────────
 T0 │ ██  ██  ░░  ░░  ██  ██  ░░  ░░
 T1 │ ██  ██  ░░  ░░  ██  ██  ░░  ░░
 T2 │ ░░  ░░  ██  ██  ░░  ░░  ██  ██
 T3 │ ░░  ░░  ██  ██  ░░  ░░  ██  ██
 T4 │ ██  ██  ░░  ░░  ██  ██  ░░  ░░
 T5 │ ██  ██  ░░  ░░  ██  ██  ░░  ░░
 T6 │ ░░  ░░  ██  ██  ░░  ░░  ██  ██
 T7 │ ░░  ░░  ██  ██  ░░  ░░  ██  ██
Legend: ██ = consonant, ░░ = dissonant, ▒▒ = unknown
```

### `csched` DebugFS Controls

```
/sys/kernel/debug/csched/
├── consonance              (ro)  current system consonance value
├── conservation            (ro)  current conservation ratio
├── repartition_interval_ms (rw)  milliseconds between re-partitions (default: 500)
├── conservation_threshold  (rw)  min conservation before forced repartition (0.4)
├── max_partitions          (rw)  max Fiedler groups (default: 8)
├── profile_interval_ms     (rw)  ms between resource profiling (default: 100)
├── forcerepartition        (wo)  write 1 to force immediate re-partition
├── trace_enabled           (rw)  enable perf trace collection
├── stats/
│   ├── total_partitions    (ro)  number of re-partitions triggered
│   ├── avg_consonance      (ro)  rolling average of consonance
│   ├── sched_overhead_ns   (ro)  C-SCHED overhead per schedule
│   └── migrations_avoided (ro)  how many cross-partition migrations were blocked
└── profiles/
    ├── pid_1234            (ro)  thread resource profile (JSON-ish)
    ├── pid_1241            (ro)
    └── ...
```

### Scheduling Policy Modification

#### `select_task_rq` (wakeup path)

```c
// In fair.c select_task_rq_fair:
#ifdef CONFIG_CSCHED
static int
csched_select_task_rq(struct task_struct *p, int prev_cpu, int sd_flags, int wake_flags)
{
    struct csched_entity *ce = &p->csched;

    if (!ce->partition_id)
        return prev_cpu;  // not tracked; fall back to default

    // Prefer CPUs running same partition
    for_each_online_cpu(cpu) {
        struct csched_rq *crq = per_cpu_ptr(&csched_rq, cpu);
        if (crq->active_partition == ce->partition_id)
            return cpu;  // same partition → minimal cross-group interference
    }

    // No same-partition CPU available: find least-dissonant CPU
    int best_cpu = prev_cpu;
    double min_dissonance = INFINITY;
    for_each_online_cpu(cpu) {
        int cpu_part = per_cpu(csched_rq, cpu).active_partition;
        double d = compute_dissonance(ce->partition_id, cpu_part);
        if (d < min_dissonance) {
            min_dissonance = d;
            best_cpu = cpu;
        }
    }
    return best_cpu;
}
#endif
```

#### `task_tick` (time accounting)

```c
#ifdef CONFIG_CSCHED
static void csched_task_tick(struct rq *rq, struct task_struct *curr, int queued)
{
    struct csched_rq *crq = per_cpu_ptr(&csched_rq, cpu_of(rq));
    struct csched_entity *ce = &curr->csched;
    unsigned long now = jiffies;

    // Update partition quantum tracking
    crq->partition_ticks[ce->partition_id]++;

    // Check if current partition has exceeded its quantum
    unsigned long quota = crq->partition_quota[ce->partition_id];
    if (crq->partition_ticks[ce->partition_id] > quota) {
        // Preempt — schedule next partition
        curr->se.avg.vruntime = curr->se.avg.vruntime +
            (crq->partition_ticks[ce->partition_id] - quota) * 1000000;
        resched_curr(rq);
    }

    // Periodic profiling: sample page signatures
    if (time_after(now, ce->last_update + profile_interval)) {
        sample_resource_signature(curr);
        ce->last_update = now;
    }

    // Periodic re-partitioning: if conservation is low or interval elapsed
    if (time_after(now, crq->last_repartition + repartition_interval)) {
        queue_work(system_unbound_wq, &crq->repartition_work);
    }
}
#endif
```

### CFS Integration

C-SCHED sits **above** CFS — it doesn't replace the fair scheduler but guides its decisions:

```
Normal CFS path:
  pick_next_task_fair() → rb_first() → return next task by vruntime

C-SCHED modified path:
  pick_next_task_fair()
    → if NO_CSCHED: goto normal
    → get current rq's active partition
    → look up this partition's vruntime tree (partitioned CFS rbtrees)
    → rb_first(partition_tree) → return next task in same partition
    → if empty: advance to next partition, return its first task
```

This is implemented by splitting the CFS runqueue into **per-partition rbtrees**:

```c
struct csched_rq {
    struct rb_root_cached  partition_trees[CSCHED_MAX_PARTITIONS];
    int                    active_partition;
    unsigned long          partition_ticks[CSCHED_MAX_PARTITIONS];
    unsigned long          partition_quota[CSCHED_MAX_PARTITIONS];
    unsigned long          last_repartition;
    struct work_struct     repartition_work;
    struct interaction_graph *graph;
};
```

**Quantization across partitions**: Each partition gets a fair share of CPU time over epoch (sum of quotas = SCHED_LATENCY * nr_partitions). Within a partition, CFS provides fairness.

### Repartition Workqueue

```c
static void csched_repartition_work(struct work_struct *work)
{
    struct csched_rq *crq = container_of(work, struct csched_rq, repartition_work);

    // 1. Build interaction graph from current profiles
    build_interaction_graph(crq->graph);

    // 2. Compute Fiedler partition
    int new_partitions[NR_TASKS];
    int n_parts = fiedler_partition(crq->graph, new_partitions, nr_threads());

    // 3. Check dissonance delta: don't repartition if change is small
    double delta = compute_partition_delta(crq->current_partitions, new_partitions);
    if (delta < MIN_PARTITION_DELTA && crq->conservation > crq->conservation_threshold)
        return;  // stability > churn

    // 4. Migrate tasks to new partitions (lazy migration)
    for_each_tracked_task(p) {
        int old_part = p->csched.partition_id;
        int new_part = new_partitions[p->pid % NR_TASKS];

        if (old_part != new_part) {
            // Remove from old partition tree, insert into new
            dequeue_csched_entity(p, old_part);
            p->csched.partition_id = new_part;
            enqueue_csched_entity(p, new_part);
        }
    }

    // 5. Recompute quotas
    recompute_partition_quotas(crq);

    crq->last_repartition = jiffies;
    crq->conservation = compute_conservation(crq->graph, new_partitions);
}
```

### Key Implementation Considerations

1. **Spinlock hierarchy**: `csched_rq` lock nests inside (or outside) rq lock depending on path. Document carefully to avoid deadlock.
2. **Memory pressure**: Bloom filters for page/lock signatures should be small (64 bytes per signature).
3. **Real-time compatibility**: Disable C-SCHED for `SCHED_FIFO`/`SCHED_RR`/`SCHED_DEADLINE` tasks.
4. **CPU hotplug**: Drain and redistribute partitions on CPU offline.
5. **Task migration cost**: Large tasks (heavy page working sets) pay high migration cost. Factor into partition decisions.

### Self-Hosting / Eating Your Own Dogfood

Use C-SCHED to schedule its own kernel workqueues:

```c
// C-SCHED workqueue threads get partitioned with the threads they represent
static int csched_kworker_init(void)
{
    struct task_struct *kworker = kthread_run(csched_partition_kworker, crq,
                                              "csched/%d", cpu);
    csched_add_thread(kworker->pid, &(struct thread_profile){
        .type = THREAD_KWORKER,
        .affinity_partition = -1,  // auto-partition
    });
    return 0;
}
```

---

## 📦 Milestone 3: RISC-V Caffeinix Extension (6 months)

### Goal
A hardware-accelerated Consonance Scheduler on the caffeinix RISC-V platform, with a **CSR-based consonance counter** and a simple **spectral scheduling unit**.

### Hardware Additions

#### CSR: `mconsonance` — Machine-Level Consonance Counter

| Field | Bits | Description |
|-------|------|-------------|
| CONS | [63:0] | Running consonance measurement (scaled fixed-point) |

The counter tracks hardware-level resource conflicts:

```
mconsonance = Σ (cache_miss_overlap + lock_busy_cycles + memory_channel_stalls)
```

Updated every cycle by the coherence controller + memory scheduler + lock monitor.

#### CSR: `mcons_config` — Consonance Configuration

| Field | Bits | Description |
|-------|------|-------------|
| EN | [0] | Enable consonance tracking |
| PARTITION_SZ | [3:1] | Log2 of number of partitions (0–7 → 1–128) |
| PROFILE_INTERVAL | [11:4] | Profile interval in cycles (2^N) |
| AUTO_REPART | [12] | Auto re-partition on conservation threshold |
| CONS_THRESH | [19:13] | Conservation threshold (0–127 / 127) |

#### CSR: `mcons_partition` — Current Partition Map

| Field | Bits | Description |
|-------|------|-------------|
| PART_INFO | [63:0] | Up to 8 partition IDs packed (8 bits each) |

#### CSR: `mcons_seed` — Random Seed for Fiedler Initialization

```
Software writes a random seed → hardware Fiedler engine uses for initialization.
```

### Spectral Scheduling Unit (SSU)

A hardware block that accelerates the Fiedler partition in parallel:

```
                      ┌─────────────────────────────────────┐
                      │       Spectral Scheduling Unit       │
                      │                                     │
                      │  ┌─────────────────────────────┐    │
  CSR conns ─────────▶│  │  Interaction Graph Memory   │    │
                      │  │  (SRAM, 1024×1024 sp matrix)│    │
                      │  └──────────┬──────────────────┘    │
                      │             │                        │
                      │  ┌──────────▼──────────────────┐    │
                      │  │  Lanczos Iteration Engine    │    │
                      │  │  (4-stage pipelined mat-vec) │    │
                      │  ├── FP16 MAC array (64-wide)   │    │
                      │  └──────────┬──────────────────┘    │
                      │             │                        │
                      │  ┌──────────▼──────────────────┐    │
                      │  │  Partition Register File    │    │
                      │  │  (1024 × 8-bit partitions)  │    │
                      │  └─────────────────────────────┘    │
                      │                                     │
                      │  Interrupt: IRQ_CONSONANCE_UPDATE   │
                      └─────────────────────────────────────┘
```

**Performance**: 64-wide FP16 MAC at 1 GHz → 64 GFLOP/s spectral processing. Fiedler partition of 1024 threads completes in ~16 µs (vs ~500 µs in software).

### RISC-V ISA Extension

#### New Instructions

```
# Load thread profile into SSU
csched.profile rs1, rs2
# rs1 = hart ID, rs2 = pointer to thread profile (physical address)

# Trigger re-partition on SSU
csched.repartition

# Read partition assignment for a hart
csched.partition rd, rs1
# rd = partition ID, rs1 = hart ID

# Write conservation threshold
csched.threshold rs1
# rs1 = new conservation threshold

# Barrier: wait for SSU to finish re-partition
csched.barrier
```

#### Encoding (R-type, custom-3 opcode space)

```
31      27 26   25 24     20 19     15 14     12 11       7 6     0
┌─────────┬─────┬────────┬────────┬────────┬──────────┬──────────┐
│ funct5  │ 0   │  rs2   │  rs1   │ funct3 │   rd     │ opcode   │
│ 00000-  │     │ (src)  │ (src)  │ 000    │  (dst)   │ 1011011  │
│ 00100   │     │        │        │        │          │          │
└─────────┴─────┴────────┴────────┴────────┴──────────┴──────────┘

funct5 encoding:
  00000 → csched.profile
  00001 → csched.repartition
  00010 → csched.partition (rd = partition, rs1 = hart)
  00011 → csched.threshold
  00100 → csched.barrier
```

### Caffeinix Integration

```c
// In caffeinix's trap handler (trap.c):
void handle_csched_interrupt(void) {
    uint64_t consonance = read_csr(mconsonance);
    uint64_t config     = read_csr(mcons_config);
    uint64_t partition  = read_csr(mcons_partition);

    if (consonance < (config & CONS_THRESH_MASK)) {
        // Conservation too low — SSU should have auto-repartitioned
        // If AUTO_REPART is disabled, force it in software
        if (!(config & AUTO_REPART)) {
            csched_repartition();
        }
    }

    // Update per-thread consonance tracking in thread struct
    current_thread->csched_consonance = consonance;
}

// In scheduler.c (caffeinix):
void sched(void) {
    // If SSU has a new partition, use it
    if (csched_have_new_partition()) {
        int part = csched_get_partition(this_hart_id());
        // Update run queue with partition-aware ordering
        csched_update_runqueue(part);
    }

    // Normal schedule loop with partition awareness
    for (t = &thread[0]; t <= &thread[NTHREAD - 1]; t++) {
        if (t->csched_partition != current_partition)
            continue;  // only schedule threads from same group
        // ... dispatch
    }
}
```

### Hardware Counter Monitoring (`/sys/firmware/csched`)

```
/sys/firmware/csched/
├── hw_consonance          (ro)  raw mconsonance CSR value
├── hw_config              (rw)  raw mcons_config CSR value
├── hw_partitions          (ro)  current partition assignments
├── hw_lanczos_iter        (rw)  SSU Lanczos iterations (default: 32)
├── hw_ssu_temperature     (ro)  SSU thermal sensor
└── hw_fp16_mac_perf       (ro)  MAC unit utilization %

# Example:
$ cat /sys/firmware/csched/hw_consonance
0.87
$ echo "64" > /sys/firmware/csched/hw_lanczos_iter
$ cat /sys/firmware/csched/hw_partitions
hart 0: partition 2
hart 1: partition 2
hart 2: partition 0
hart 3: partition 0
hart 4: partition 1
...
```

---

## 📊 Benchmark Targets

| Metric | Current (CFS) | C-SCHED Target | Benefit |
|--------|---------------|----------------|---------|
| Lock-heavy throughput | 1.0× baseline | **1.15×** | Fewer lock conflicts (dissonant threads staggered) |
| Mixed I/O+compute tail latency (p99) | 1.0× baseline | **0.80×** | Compute threads don't starve I/O threads (partitioned) |
| Power (cache conflicts) | 1.0× baseline | **0.90×** | Fewer cache misses from conflicting threads (grouped) |
| Scheduler overhead (context switch) | ~2µs | **~3µs** (+50%) | Fiedler partition overhead amortized over 500ms window |
| Fairness (max/min runtime ratio) | 1.05× | **1.10×** | Partition quanta introduce slight fairness degradation |
| Re-partition cost (1024 threads) | N/A | **500µs** (SW) / **16µs** (HW) | Acceptable at 500ms intervals |

---

## 🧪 Verification & Correctness

### Safety Invariants

1. **No thread is starved**: Partition quanta have a hard minimum floor (configurable, default 10% of epoch).
2. **Conservation invariants don't cause livelock**: Re-partition is triggered by timer, not by conservation events alone. If Fiedler partition oscillates, stickiness factor (delta threshold) prevents churn.
3. **Lock ordering is respected**: C-SCHED never modifies lock acquisition order; it only changes which threads run when.
4. **Deterministic under edge cases**: When all threads have identical resource signatures (consonance = 1.0), C-SCHED degrades to CFS with one partition. When all threads conflict (consonance = 0.0), each thread gets its own partition → effectively round-robin.

### Testing Strategy

| Test | Scope | Method |
|------|-------|--------|
| Unit: Fiedler partitioning | Userspace | Known graphs, verify partition output |
| Unit: Conservation ratio | Userspace | Simulated schedules, verify monotonicity |
| Integration: Lockbench | Kernel | `lockbench -t 64 -l 8`, compare throughput |
| Integration: Tail latency | Kernel | `cyclictest` mixed with `stress -c 4` |
| Integration: Power | Kernel + HW | RAPL power readings |
| Stress: 1024 threads | Kernel | Fork bomb + benchmark |
| Stress: CPU hotplug | Kernel | On/offline CPUs while scheduling |
| Stress: OOM | Kernel | Run with minimal memory |
| Regression: Full LTP | Kernel | Linux Test Project suite |

---

## 🔭 Stretch Goals

### C-SCHED + SCHED_DEADLINE Hybrid
Use C-SCHED for best-effort threads and SCHED_DEADLINE for real-time threads. The partition structure provides isolation boundaries