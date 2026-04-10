# Coherence and Consistency — Interview Questions

**Subject:** Computer Architecture
**Topic:** MESI, Snoop vs Directory, TSO, Weak Memory Models, Fences
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is cache coherence and why is it necessary?

**Answer:**

**Cache coherence** is the property that, in a system with multiple private caches, every read of a memory location returns the most recent write to that location — regardless of which cache wrote it or which is reading it.

**Why we need it:** Without coherence, if core A writes to a shared variable in its L1 cache and core B reads from its own L1 cache, B might see the old value forever (until its line is evicted). All multithreaded code would be broken. Locks, queues, shared data structures — nothing would work.

**The formal coherence invariants (Hill, Sorin):**

1. **Single Writer, Multiple Reader (SWMR):** At any moment, a cache line either has one writer (and no other cache has a valid copy) or multiple readers (and no one is writing).

2. **Data-Value Invariant:** A reader always sees the value written by the most recent writer, in the coherence order.

Coherence is a **correctness** property: any correct parallel program must rely on it. It is weaker than the **memory consistency model**, which governs the ordering between operations on *different* addresses.

### Q2. Describe the MESI coherence protocol.

**Answer:**

**MESI** is a 4-state invalidation-based coherence protocol. Each cache line is in one of four states:

- **Modified (M):** This cache has the only valid copy, and it is dirty (different from memory). On eviction, it must be written back.

- **Exclusive (E):** This cache has the only valid copy, and it is clean (same as memory). On write, it transitions to Modified silently — no bus traffic.

- **Shared (S):** Multiple caches may have valid clean copies. All agree with memory. On write, must invalidate all other copies and transition to M (or E if only this copy exists).

- **Invalid (I):** This cache line holds no valid data. On read or write, must request the line from another cache or memory.

**State transitions:**

- **Read miss, no other cache has it:** Load from memory, state = E.
- **Read miss, one other cache has it in S or E:** Source the line, both move to S.
- **Read miss, one other cache has it in M:** Source holder writes back and transitions to S (or E in some variants); new cache goes to S.
- **Write miss, no other cache has it:** Load from memory, state = M.
- **Write miss, others have it in S:** Invalidate all, load line, state = M.
- **Write hit in S:** Send invalidate to all, transition to M (this is the cost of a write to shared data).
- **Write hit in E or M:** No bus traffic, transition to M.

**Why E exists:** The E state is a performance optimisation. A cache line in E transitions to M on write without any bus traffic — because no other cache has a copy to invalidate. Without E, every write to cached data would need a bus transaction even in single-threaded execution.

**Variations:**

- **MSI:** Drops E. Every write-after-read requires a bus transaction.
- **MOSI / MOESI:** Adds **Owned** — a line can be dirty and shared simultaneously, with one cache responsible for writing it back. Allows dirty lines to be shared without invalidation. Used by AMD.
- **MESIF:** Adds **Forward** — in a Shared state with multiple holders, one is designated as the "forwarder" that supplies the line on new reads, avoiding race conditions on who responds. Used by Intel.

### Q3. What is false sharing and how do you detect and fix it?

**Answer:**

**False sharing** occurs when two (or more) CPUs repeatedly modify different variables that happen to live on the same cache line. Even though the variables are logically independent, the coherence protocol treats the line as a unit: each write invalidates the line in the other core's cache, causing a coherence miss.

**Example:**

```c
struct Counters {
    int counter_a;   // used by thread A
    int counter_b;   // used by thread B
};
```

With `int` = 4 bytes and cache line = 64 bytes, both counters sit on the same line. Thread A incrementing `counter_a` invalidates the line in thread B's cache, and vice versa. Throughput collapses even though the two threads never touch the same variable.

**Detection:**

1. **Performance counters:** Look for abnormally high L1 miss rates on shared data structures, or explicit coherence-miss counters (`MEM_LOAD_RETIRED.L3_HIT_HITM` on Intel indicates a load that hit in another core's L3 dirty state — classic coherence traffic).

2. **Profilers:** Tools like Intel VTune, AMD uProf, Linux perf c2c can identify cache lines with high cross-core traffic and pin down the responsible data.

3. **Code review:** Look for closely-packed data accessed by different threads (thread-local counters, queue heads and tails, lock variables adjacent to protected data).

**Fixes:**

1. **Padding to cache line size:**
   ```c
   struct Counters {
       alignas(64) int counter_a;
       alignas(64) int counter_b;
   };
   ```
   Wastes bytes but guarantees independence.

2. **Per-thread copies:** Each thread has its own counter; sum them at the end.

3. **Data structure redesign:** Group cold data together, hot data per-thread.

**Numerical impact:** False sharing can turn a 10 ns uncontended access into a 200 ns contested one — a **20× slowdown**. For high-rate counters (e.g. statistics in a hot loop), this dominates total runtime.

---

## Intermediate

### Q4. What is the difference between snoop-based and directory-based coherence?

**Answer:**

These are two implementations of cache coherence with very different scaling properties.

**Snoop-based (snooping):**

Every coherence transaction is broadcast on a shared bus (or its logical equivalent). Every cache monitors (snoops) the bus and checks each transaction against its own tag array. On a match, it responds appropriately (e.g. supplies dirty data, invalidates its copy).

- **Pros:** Simple, low-latency for small systems, fast common cases.
- **Cons:** Bus bandwidth is a hard ceiling. Every cache pays the snoop cost on every transaction, whether it holds the line or not. Scales poorly beyond ~16 cores.

**Directory-based:**

A centralised **directory** tracks, for each cache line in memory, which caches have copies. On a miss, the requesting cache sends a point-to-point message to the directory. The directory forwards the request only to the caches that actually hold the line.

- **Pros:** Scales to hundreds of cores — only relevant caches are contacted per request. No broadcast bottleneck.
- **Cons:** Higher latency (extra directory hop), directory storage overhead (bit vector per line identifying sharers — grows with core count).

**Hybrid approaches in modern systems:**

- **Intel Skylake+ server:** Mesh-based interconnect with embedded snoop filter (a form of directory) at every LLC slice. Small systems behave like snoop; large systems scale like directory.
- **AMD Infinity Fabric:** Directory-style routing between CCDs (chiplets), snoop within a CCD.
- **ARM CHI (Coherent Hub Interface):** Directory-based at the system level, used in multi-cluster designs.

**The key scaling fact:** Snoop traffic is O(N) per transaction; directory traffic is O(k) where k is the number of sharers of that specific line. For lines that are private or held by few caches, directory wins dramatically.

### Q5. What is a memory consistency model and how does it differ from coherence?

**Answer:**

- **Coherence** is about *a single memory location*: when multiple caches hold copies of the same line, they must agree on the value. Coherence answers "what is the value of location X?".

- **Consistency** is about *multiple memory locations*: given a set of reads and writes to different addresses by different cores, what global ordering is observable? Consistency answers "what orderings of X and Y are possible?".

A system can be coherent but have a weak consistency model: each location is individually consistent, but the relative order of operations to different locations is loose.

**Example illustrating the difference:**

```
Initial: X = 0, Y = 0
Core 0:                Core 1:
ST [X], 1              r1 = LD [Y]
ST [Y], 1              r2 = LD [X]
```

**Under sequential consistency** (strongest): The only possible outcomes are those consistent with some interleaving of the four instructions in program order. `(r1=1, r2=0)` is forbidden because it would require Core 1's load of Y to have happened after Core 0's store of Y, but the load of X (happening after that load of Y on Core 1) to have happened before Core 0's store of X — violating program order on Core 0.

**Under TSO:** Same outcomes as SC for this example.

**Under a weak model (ARM without barriers):** Core 0's two stores can complete in reverse order from another core's perspective, because independent stores may be reordered. The outcome `(r1=1, r2=0)` becomes possible. Correct code would need a release fence between the two stores on Core 0 and an acquire fence between the two loads on Core 1.

**Standard consistency models, strongest to weakest:**

- **SC (Sequential Consistency):** All ops appear in some global order consistent with each core's program order.
- **TSO (Total Store Order):** x86, SPARC TSO. Like SC but allows loads to pass older stores to different addresses.
- **PSO (Partial Store Order):** Older SPARC. Allows stores to different addresses to be reordered.
- **RMO (Relaxed Memory Order):** ARMv7, POWER. Most orderings allowed; explicit fences required.
- **ARMv8, RISC-V RVWMO:** Release Consistency. Loads and stores can be freely reordered except around acquire/release annotations.

### Q6. What is release consistency and why does it help performance?

**Answer:**

**Release consistency (RC)** classifies synchronisation operations into two types:

- **Acquire:** A load that announces "I am entering a critical region". No subsequent memory op may be reordered before it.
- **Release:** A store that announces "I am leaving a critical region". No prior memory op may be reordered after it.

Non-synchronisation memory ops (normal loads and stores) can be freely reordered with each other — the hardware enforces no implicit ordering between them.

**Why this is weaker than SC or TSO:** In SC/TSO, every load and every store is implicitly ordered with every other one. The hardware must track and enforce this. In RC, only the acquire/release annotated ops carry ordering; everything else is free.

**Why it helps performance:**

1. **More out-of-order reordering.** The core can issue loads out of order, commit stores out of order, and generally schedule freely between acquire and release boundaries.

2. **Fewer fences in common code.** Code that does not use synchronisation primitives pays zero ordering cost. Code that does use them pays only at the acquire/release points.

3. **Cheaper synchronisation primitives.** An acquire load (ARMv8 `LDAR`, RISC-V `lr` with aq bit) enforces only the ordering it needs, not a full memory barrier. This is much cheaper than a full `DMB` on ARMv7.

**Language-level mapping:**

- C11/C++11 `memory_order_acquire`, `memory_order_release` map directly.
- C11/C++11 `memory_order_seq_cst` on ARMv8 compiles to `LDAR`/`STLR` plus (on some architectures) an additional fence to enforce the total order, making it slightly more expensive than acquire/release.
- On x86, seq_cst loads are free (TSO already gives acquire semantics on loads), but seq_cst stores require `MFENCE` or `XCHG` to prevent the store from sitting in the store buffer across a later load.

**Interview insight:** The fact that `seq_cst` is free-ish on x86 and costly on ARM explains why many lock-free algorithms "just work" on x86 and break on ARM. Code that relies on implicit TSO behaviour will look correct in testing but race on ARM.

---

## Advanced

### Q7. How does the MOESI "Owned" state reduce coherence traffic?

**Answer:**

In MESI, if one cache has a line in Modified (dirty), another cache requests a read-only copy. The owner must either:

1. **Write back to memory** and then both caches transition to Shared. The memory now has the latest copy. But this costs a write-back to DRAM — two bus transactions (dirty line to memory, memory to new reader) and the write-back bandwidth.

2. **Source the dirty line directly to the reader** without writing to memory. But now memory is stale, and the coherence protocol in MESI has no state that says "shared, but not clean with respect to memory". The dirty source must therefore write back.

**MOESI adds the Owned state.** A line in Owned (O) is:
- Shared by multiple caches (reads are permitted)
- Dirty relative to memory
- *One specific cache* — the owner — is responsible for writing it back on eviction

**How it reduces traffic:**

- The owner can source the dirty line to new readers via cache-to-cache transfer, without writing back to memory. The requester receives the line in Shared; the owner transitions from M to O.
- Subsequent readers also get cache-to-cache transfers from the owner.
- Only when the owner evicts the line does a memory write happen — possibly never, if the owner is also the last holder to lose the line.

**Net effect:** Dirty shared data (producer-consumer patterns, common in producer/consumer queues and workload distribution) is shared at cache-to-cache latency, not DRAM latency. Write-backs are deferred or avoided.

**Who uses it:** AMD (Opteron and all subsequent Zen designs) use MOESI. Intel uses MESIF, which takes a different approach (a "Forwarder" state in which one of the Shared holders is the designated source for new read requests, eliminating the race where multiple Shared holders would all try to respond).

**Trade-off:** MOESI requires extra state bit(s) per cache line and more complex logic. The performance benefit is real but workload-dependent — most on codes with frequent producer-consumer patterns.

### Q8. Describe a store buffer and how it interacts with TSO.

**Answer:**

A **store buffer** is a FIFO between the core's pipeline and the L1 cache. Stores are committed to the store buffer in program order but only drain to the L1 cache at some later time (typically when the corresponding ROB entry commits, in a write-back ordering).

**Why store buffers exist:**

1. **Hide store latency.** A store that misses in L1 would otherwise stall subsequent ops. Buffering lets execution continue.
2. **Coalesce writes.** Multiple stores to the same cache line can be combined in the buffer before a single bus transaction commits them.
3. **Enable load-store forwarding.** A younger load can peek into the store buffer and forward a matching older store's value directly.

**TSO and the store buffer:** TSO's defining relaxation is that a younger load may pass an older store to a different address. This is exactly what a store buffer naturally does:

1. Core issues `ST [X], 1` → enters store buffer, not yet in cache.
2. Core issues `LD R1, [Y]` → goes to cache immediately, sees whatever is currently there.
3. The store of X eventually drains to cache.

From another core's perspective, the load happened "before" the store even though the load was program-order-later. This is the only TSO relaxation, and it is free to implement.

**TSO and correctness — the hazard:** The classic pattern:

```
Core 0:              Core 1:
ST [X], 1            ST [Y], 1
LD R1, [Y]           LD R2, [X]
```

TSO allows `(R1=0, R2=0)`! Both cores' stores sit in their own store buffers while both loads complete using the pre-store values. The outcome is not forbidden by TSO because TSO only promises that each core's own stores appear in order; it does not promise that another core's stores are visible before one's own loads.

**If you need this to be forbidden:** Insert a full fence (`MFENCE` on x86) between the store and the load on each core. The fence drains the store buffer before allowing the load to proceed. This is the canonical store-load fence, and it is the **only** type of fence that TSO requires (all other orderings are free).

**Why TSO code "just works" most of the time:** The store-load reordering is the only relaxation, and it is uncommon for it to matter outside of lock-free algorithms (Dekker's algorithm and its relatives). Most application code does not depend on this ordering, which is why x86 developers rarely think about memory barriers.

### Q9. What is memory disambiguation in the context of coherence, and how does it differ from intra-core disambiguation?

**Answer:**

**Intra-core memory disambiguation** (covered in `02_out_of_order_execution/wide_issue_and_scheduling.md`) is about ordering within a single core's pipeline — deciding whether a speculative load can pass older stores in the same core's store buffer.

**Coherence-level memory disambiguation** is a different problem: given a load that has already executed, can the coherence system guarantee its value is still correct in the context of the global memory model? Specifically, has any *other core* invalidated the line between the load's execute and its commit?

**The TSO load-ordering invariant:** Under TSO, loads must retire in program order from each core's perspective. If Core 0 executes loads L1 and L2 speculatively (L2 before L1), and between L2's execute and L1's execute, Core 1 writes the line that L2 reads — making L2's value stale — then committing L2 before L1 would violate TSO.

**Solution — load-ordering machine clear:**

1. Each speculatively-executed load is tracked in the load queue with its physical address and the data it read.
2. When any **coherence invalidate** arrives, it is searched against the load queue. A matching entry — a load that has read the line that is now being invalidated — indicates a potential TSO violation.
3. If the matching load has not yet committed, it (and everything speculatively after it) is squashed and replayed. This is a **memory-ordering machine clear**.

**Cost:** Similar to a branch mispredict — full pipeline flush from the offending load. Intel calls these `machine_clears.memory_ordering` in performance counters, and they are visible on highly-contended lock-free code.

**When it happens in practice:**
- Threads spinning on a flag held by another core.
- High-rate updates to a lock-protected counter where multiple cores repeatedly read-modify-write.
- False sharing even without actual data race.

**Weak memory architectures avoid this entirely:** Because they don't promise TSO load ordering, there is nothing to violate; speculative loads can commit in any order. This is one of the fundamental reasons weak-memory cores have an easier time at high clock frequencies — they skip the whole load-ordering check.

### Q10. Explain "MWAIT" and "UMWAIT" (user-space wait) instructions and their role in efficient waiting.

**Answer:**

These instructions let software tell the hardware "I am about to spin-wait on a memory location; suspend me until the line changes or a timer expires." They trade polling CPU cycles for wake-on-invalidation efficiency.

**MWAIT (kernel-only, post-Prescott):**

- `MONITOR rax` marks the cache line containing `[rax]` for monitoring. The hardware will snoop coherence traffic for this line.
- `MWAIT` puts the core into a low-power state (C-state C1 or deeper, configurable). The core exits the wait state when:
  1. A coherence write to the monitored line arrives (cache line invalidated or updated).
  2. An interrupt fires.
  3. A timeout elapses (optional).

**Use case:** Idle CPUs in the kernel scheduler. Rather than spinning on the run queue, the CPU enters `MWAIT` monitoring a wake-up flag. A producer waking a thread writes the flag; the hardware snoops the invalidate, `MWAIT` returns, and the scheduler resumes. Saves enormous amounts of power on idle machines.

**UMWAIT (Tiger Lake and later, user-mode):**

- Same idea but executable from user mode.
- `UMONITOR` marks the line.
- `UMWAIT` waits with a maximum duration (in TSC cycles).
- Wakes on coherence event, timeout, or external interrupt.

**Use case:** User-space spinlocks. A lock contender calls `UMONITOR` + `UMWAIT` on the lock word. The releasing thread writes the lock word, which generates a coherence invalidate, waking the waiter. No syscall, no context switch, but also no wasted spinning.

**Limitations:**

- The timeout is bounded (a few thousand TSC cycles). Not a sleep replacement.
- Waking on *any* coherence event, not specifically the value of interest. The waiter must re-check and potentially wait again if the value is not what it expected.
- Not all hypervisors expose `UMWAIT` to guests.
- On high-frequency contention, the cost of entering/exiting the monitor state may exceed pure spinning.

**Why this matters for modern concurrent code:** Well-written lock implementations (Intel TBB, Java's LockSupport, Linux futex internals) can use MWAIT-class instructions to bridge the gap between pure spinning (wastes CPU) and kernel blocking (expensive context switch). This is the "adaptive" part of an adaptive lock: try a short spin, then `UMWAIT`, then if that times out, fall back to a futex-based sleep.

The same idea appears in ARMv8.3+ as `WFE` (Wait For Event) and the associated `SEVL` (Send Event Local), though with slightly different semantics.
