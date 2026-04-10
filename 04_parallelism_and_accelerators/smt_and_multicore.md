# SMT and Multicore — Interview Questions

**Subject:** Computer Architecture
**Topic:** Simultaneous Multithreading, Multicore, NUMA, Scaling Bottlenecks
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is Simultaneous Multithreading (SMT), and how does it differ from coarse-grained multithreading?

**Answer:**

**Simultaneous Multithreading** executes instructions from multiple threads in the same cycle, sharing almost all backend resources (functional units, caches, ROB) between them. The pipeline holds multiple architectural contexts (register files, PCs, control state) but funnels their instructions into one out-of-order backend.

**Coarse-grained multithreading** executes one thread at a time and switches to a different thread only on a long-latency event (L2 miss, TLB miss). Fine-grained multithreading switches every cycle in round-robin. SMT is the extreme: no switching at all — multiple threads coexist in every pipeline stage simultaneously.

**Why SMT wins:** Modern OoO cores rarely achieve peak IPC on a single thread. Dependency chains, memory stalls, and branch mispredicts leave functional units idle. SMT fills those idle slots with independent work from a second thread. A 4-wide core might sustain 2.0 IPC on one thread but 3.0 IPC on two threads — 50% more throughput at ~5% extra area.

**Implementation requirements:**

1. **Duplicated architectural state:** Register file, rename table, PC, flags, control registers — per thread. This is the main area cost.

2. **Tagged pipeline entries:** Every ROB, issue queue, load queue, store queue entry carries a thread ID.

3. **Thread-aware scheduler:** The issue queue must avoid starving either thread. Priority schemes ensure fairness.

4. **Partitioned or shared small structures:** Branch predictor history, TLB entries, and store buffers may be shared (cheaper but allows interference) or partitioned (fairer but smaller per-thread effective size).

**When SMT hurts:** Cache-pollution workloads where each thread has a large working set. With SMT, two threads now contend for L1/L2, and neither fits — the effective cache per thread halves, and the workload thrashes. This is why some HPC codes disable SMT on their compute nodes.

### Q2. What is the basic architectural cost of adding a second hardware thread to an OoO core?

**Answer:**

The cost breakdown, roughly:

1. **Architectural register file duplication:** For x86-64 with 16 GPRs + 16 YMM/ZMM + MMX + FP + control, the architectural state is ~2 KB per thread. Doubling = 2 KB extra SRAM.

2. **Rename table duplication:** The RAT (architectural-to-physical mapping) per thread. A few hundred bytes.

3. **PC, flags, segment regs, CR, MSR state:** Another few hundred bytes per thread.

4. **Thread ID fields:** Every pipeline register and every backend entry (ROB, IQ, LQ, SQ, store buffer) grows by a few bits.

5. **Scheduler logic:** A small amount of priority/fairness logic in the issue stage.

6. **Zero extra functional units:** The whole point of SMT is that the backend is already wider than any single thread needs. No new ALUs, no new load ports.

**Total area cost:** Typically 5–10% of the core. Intel famously claims "5% area for 30% performance" on SMT. AMD's Zen designs quote similar.

**Power cost:** Essentially zero at idle (the second thread just doesn't issue), modest at high utilisation (more of the backend is active per cycle, so dynamic power rises roughly in proportion to the IPC gain).

**Why not 4 or 8 threads?** Diminishing returns. The backend resources are finite — at some point adding more threads does not find more independent work; it just increases contention for caches and functional units. IBM POWER uses up to 8-way SMT on server designs where many threads are a good match for the backend width and memory hierarchy. Intel, AMD, and ARM mostly stick with 2-way.

### Q3. What is NUMA and why does it matter for multicore performance?

**Answer:**

**NUMA** (Non-Uniform Memory Access) is a multiprocessor memory organisation in which different CPUs have different access latencies to different memory regions. A CPU accessing its **local memory** (attached to its socket's integrated memory controller) gets fast, low-latency access. Accessing **remote memory** (attached to another socket) goes through the inter-socket interconnect (QPI/UPI on Intel, Infinity Fabric on AMD), costing 1.5–3× more latency and lower bandwidth.

**Why NUMA exists:** A single shared memory controller becomes a bottleneck once core count exceeds maybe 16. Scaling to 64 or 256 cores requires distributing memory controllers across multiple sockets/dies — which inevitably makes access non-uniform.

**Performance impact:**
- Local DRAM access: ~80 ns
- Remote DRAM (1-hop): ~130 ns
- Remote DRAM (2-hop on 4-socket systems): ~200 ns+
- Bandwidth: local memory bandwidth is the sum of all local channels; remote bandwidth is limited by the inter-socket link (typically 1/3 to 1/2 of local).

**What matters for software:**

1. **Thread and memory placement.** A thread running on socket 0 accessing memory on socket 1 pays the remote cost on every DRAM access. The OS's NUMA scheduler tries to keep threads on the node where their memory lives.

2. **First-touch allocation.** Linux's default is that a page is physically allocated on the NUMA node of the thread that first touches it (not the thread that called `malloc`). This means initialisation thread layout can accidentally pin all data to one node, leaving other nodes idle.

3. **NUMA-aware data structures.** Large parallel programs explicitly partition data across nodes, using OS APIs (`numactl`, `libnuma`, `MPI`) to control placement.

4. **Lock location matters.** A spinlock physically resident on a remote node costs double on every acquire — and the coherence traffic goes across the slow link.

**Modern complication — chiplet architectures:** AMD's Zen uses multiple CCDs (chiplets) per socket, connected via Infinity Fabric. Access to another CCD's L3 or DRAM shows some NUMA-like cost even within a single socket. Intel's Sapphire Rapids with its 4-tile architecture is similar. Software that assumed "one socket, uniform" is now wrong on modern servers.

---

## Intermediate

### Q4. Describe two common SMT resource-sharing policies.

**Answer:**

**1. Statically partitioned:** Each shared backend resource is split equally among threads. An N-entry ROB with 2 threads becomes two partitions of N/2 each. A thread cannot exceed its allocation even if the other thread is using nothing.

- **Pros:** Fairness guaranteed. No thread can starve the other. Simpler pipeline logic.
- **Cons:** Effective per-thread resource halves. Single-thread performance drops when the second thread is idle (unless the partition can be reclaimed).
- **Example:** Intel historically used this for some structures (the Skylake load buffer was partitioned between HT threads).

**2. Dynamically shared:** A single pool is used by all threads. Entries are allocated on a first-come-first-served basis, with some fairness enforcement (e.g. a cap on the fraction any one thread can hold).

- **Pros:** A single active thread gets the full pool. Utilisation is higher on mixed workloads.
- **Cons:** A misbehaving or stalled thread can starve others. Requires dynamic accounting.
- **Example:** Modern designs prefer this with adaptive thresholds — "if a thread stalls for N cycles, reduce its share cap".

**Hybrid approach (common in practice):**

- **Small, contended structures** (reservation stations, store buffer): dynamically shared with caps.
- **Per-thread state** (registers, rename map): fully duplicated.
- **Large, infrequently-contended structures** (L1, L2): shared without accounting.
- **Predictors** (branch predictor, BTB): usually shared, with the entries naturally tagged by thread (PCs differ).

**Interview insight:** The shift from static to dynamic partitioning is driven by the workload diversity — modern servers run mixed loads where one thread may be I/O-bound (using little of the backend) while another is compute-bound. Static partitioning wastes the I/O thread's unused resources. Dynamic gives the compute thread the full pool when possible.

### Q5. What is Amdahl's law and how does it bound parallel speedup?

**Answer:**

**Amdahl's law** describes the maximum speedup achievable from parallelism, given that some fraction of the work must be executed sequentially.

$$S(p) = \frac{1}{(1 - f) + f/p}$$

where:
- $S(p)$ is the speedup with $p$ parallel processors
- $f$ is the fraction of the total work that can be parallelised
- $(1 - f)$ is the sequential fraction

**Limit as p → ∞:**

$$S(\infty) = \frac{1}{1 - f}$$

**Numerical examples:**

| Parallel fraction f | 2 cores | 8 cores | 64 cores | ∞ cores |
|---|---|---|---|---|
| 0.50 | 1.33 | 1.78 | 1.97 | 2.0 |
| 0.90 | 1.82 | 4.71 | 8.77 | 10.0 |
| 0.95 | 1.90 | 5.93 | 15.42 | 20.0 |
| 0.99 | 1.98 | 7.48 | 39.25 | 100.0 |
| 0.999 | 1.998 | 7.94 | 60.77 | 1000.0 |

**The killer observation:** A mere 5% sequential fraction caps maximum speedup at 20× **no matter how many cores you throw at it**. Most real programs are lucky to be 99% parallel; the last 1% — OS calls, memory allocation, synchronisation, single-threaded initialisation — limits speedup at 100 cores to ~40×, not 100×.

**Implications for architecture:**

1. **Single-thread performance still matters.** A machine with 1000 cores but slow single-thread will lose to one with 100 fast cores on any workload with a non-trivial sequential fraction.

2. **Heterogeneous designs.** Apple's big.LITTLE approach — a few high-performance cores and many efficient cores — targets Amdahl: the sequential fraction runs on the fast cores, the parallel fraction on the efficient ones.

3. **Diminishing returns on core count.** Adding cores from 16 to 32 gives much less benefit than going from 4 to 8 at the same parallel fraction.

**Gustafson's law** is a dual formulation that assumes the problem size scales with the number of cores (the sequential fraction stays constant in absolute time but shrinks relatively as the parallel work grows). Under Gustafson, speedup scales nearly linearly with cores — but only for workloads where bigger means meaningfully bigger (scientific simulation, graph processing). For fixed-size workloads Amdahl is the binding constraint.

### Q6. What is the difference between a chip multiprocessor (CMP) and chiplets / multi-die packages?

**Answer:**

- **Chip Multiprocessor (CMP):** Multiple cores on a single monolithic die. All cores share L3 cache and the memory controllers on the same piece of silicon. Examples: Intel Skylake, AMD Zen 1 single-CCX.

- **Chiplets / Multi-Chip Module (MCM):** Multiple smaller dies in the same package, connected via an inter-die interconnect. Examples: AMD Zen 2+ (multiple CCDs + an IOD), Intel Sapphire Rapids (EMIB-connected tiles), Apple M1 Ultra (UltraFusion-connected M1 Max dies).

**Why chiplets:**

1. **Yield.** Defect rates are roughly proportional to die area. A single 600 mm² die has far lower yield than six 100 mm² dies. Chiplets let you throw away only the bad pieces.

2. **Process mixing.** The compute cores benefit from the most advanced (and expensive) process node. The IO, analogue, and memory interfaces gain nothing from 3 nm and can stay on cheap 7 nm. Chiplets allow mixing: 3 nm compute chiplets with a 7 nm IOD.

3. **Modularity.** One chiplet design serves many SKUs. AMD's CCD is reused across consumer, workstation, and server products.

4. **Reticle limit.** Monolithic dies are capped at around 800 mm² (the EUV lithography reticle size). Chiplets let you build systems beyond this limit.

**Costs:**

1. **Inter-die latency.** Going off-die to another chiplet is slower than staying on-die. AMD's Infinity Fabric adds maybe 30 ns of latency between CCDs. For cache-to-cache transfers this is measurable.

2. **Power per bit.** Driving a signal off-die costs more energy than on-die. Modern die-to-die interfaces (UCIe, Infinity Fabric) are ~1 pJ/bit, vs ~0.1 pJ/bit on-die.

3. **Complex floor-planning and thermal management.** Multiple dies in one package are harder to cool uniformly.

4. **NUMA-like behaviour within a socket.** Software must now consider not just "is this memory on the right socket" but "is this memory on the right chiplet within the socket". AMD's behaviour on Zen 2+ is famously complicated here.

**Where this is going:** 2024–2026 leading-edge designs are pushing toward even more aggressive disaggregation — separate compute, IO, memory-subsystem, and accelerator chiplets. NVIDIA's Grace Hopper, AMD's MI300, and Intel Ponte Vecchio are extreme examples. The interconnect between chiplets is rapidly becoming the dominant design constraint.

---

## Advanced

### Q7. What is the "dark silicon" problem?

**Answer:**

**Dark silicon** refers to the portion of a chip that cannot be powered on simultaneously because the power budget is exceeded. As transistor density continues to grow but power density does not scale proportionally, a given thermal envelope allows fewer and fewer transistors to be active at once.

**The root cause:**

Historically, **Dennard scaling** held: as feature size shrank, voltage and current scaled with it, so power density stayed constant. Doubling transistor count kept power the same. This broke around 2005: leakage current dominated at sub-65 nm nodes, and voltage stopped scaling because of noise margins. From then on, transistor count grew much faster than power budget.

**Consequence:** By ~2010, it became impossible to run all the transistors on a modern chip at full speed simultaneously within a reasonable thermal envelope. Estimates in the literature suggested that by 22 nm, 50% or more of a die could be "dark" — powered off or clocked down — at any given time under typical workloads.

**Architectural responses:**

1. **Clock gating everywhere.** Fine-grained shutoff of unused functional units, cache ways, decoders, etc.

2. **Power gating.** Beyond clock gating — physically cut the power supply to idle blocks, eliminating leakage.

3. **Dynamic Voltage and Frequency Scaling (DVFS).** Lower V and F when the load is light, exchanging performance for energy.

4. **Heterogeneous cores.** Big cores for burst performance, small cores for steady-state. Apple big.LITTLE runs efficient cores for light loads and only lights up performance cores when needed. ARM DynamIQ is the standardised form.

5. **Specialisation.** Specialised units (matrix multiply, video decode, crypto) are orders of magnitude more efficient at their task than a general CPU doing the same work. Using a specialised unit when the workload permits and keeping the general CPU dark is a net energy win.

6. **Dim silicon.** A compromise — run more of the chip at a lower clock frequency. IPC×frequency product is worse but more of the chip contributes.

**Consequences for programming and ISA design:** "Free" single-thread performance is over. ISAs and compilers need to expose specialisation (matrix extensions, tensor cores, domain-specific instructions) so software can target the efficient units. Programmers need to understand that using the CPU vs using the accelerator is increasingly a throughput vs efficiency choice, and most mobile workloads choose efficiency.

**The dark-silicon trajectory is why accelerators have proliferated so dramatically since 2015.** When adding more general-purpose cores hits a thermal wall, adding specialised units that sit dark most of the time and activate only when the workload asks is the only way to extract more work from the same silicon budget.

### Q8. Describe the "scalability of shared-memory coherence" problem and common solutions.

**Answer:**

As core count grows, maintaining cache coherence imposes costs that grow worse than linearly in the number of cores. The problem has several facets:

**1. Snoop traffic scales as O(N) per transaction.** In a snoop-based system, every core snoops every transaction. Total snoop work is O(N²) for a system where all cores are active.

**2. Directory storage scales as O(N) per line.** A simple directory with a bit-vector of sharers needs N bits per line. For millions of cache lines and hundreds of cores, this is many MB of directory storage — bigger than the L3 itself.

**3. Coherence bandwidth becomes a bottleneck.** Cross-core cache-line transfers compete for finite interconnect bandwidth. Hot lines (spinlocks, shared counters) can saturate the on-chip network.

**4. Worst-case latency grows.** A cache miss that ends up needing a coherence operation across the widest span of the chip takes time proportional to chip dimensions. As chips grow, this grows.

**Solutions:**

**a) Directory compression.** Instead of a full bit-vector per line, use:
- **Coarse-grained directories:** Track sharers at the granularity of groups of cores (e.g. one bit per 4 cores). On invalidation, all cores in a marked group get the message. Wastes some bandwidth but saves storage.
- **Limited pointers:** Track up to k sharers exactly; if more, fall back to "broadcast". Works because most lines have few sharers.
- **Sparse directories:** Store directory only for lines currently cached somewhere (not for all lines in memory).

**b) Region coherence.** Track coherence at larger granularity than a single cache line — e.g. per 4 KB region. Cuts directory size but reduces precision (may force unnecessary invalidations).

**c) Hierarchical coherence.** Partition the system into clusters; maintain coherence within a cluster with one protocol (e.g. snoop) and across clusters with another (e.g. directory). Each hop only sees a fraction of the total traffic.

**d) Relaxed coherence.** Give up on line-level coherence for some data. GPUs historically did this — programmer must explicitly move data to/from the GPU. NVIDIA's unified memory is a step back toward CPU-like coherence, but with coarse granularity.

**e) Interconnect topology optimisation.** Ring, mesh, torus, fat-tree. Each has different bandwidth and latency characteristics. Intel Skylake-SP moved from ring (which scaled poorly past ~12 cores) to mesh (scales to 50+). AMD uses Infinity Fabric with a hub-and-spoke topology. Apple's M1 Ultra uses a wide "UltraFusion" bridge.

**f) Private per-core caches with shared LLC.** The L1 and L2 are private; coherence is only on the LLC. This reduces per-line coherence state and lets the private caches be optimised for latency.

**Realistic outcome:** Most designs shipping in 2026 support 64–128 cores per socket with acceptable coherence performance, using a combination of the above. Beyond that, coherence efficiency degrades noticeably, and some HPC designs (Cray, early Fugaku) explicitly abandon coherence at the top level in favour of message passing between nodes.

### Q9. What is the "memory wall" and how have modern designs attacked it?

**Answer:**

The **memory wall** (Wulf and McKee, 1994) is the observation that CPU speed and DRAM latency have diverged exponentially over decades. CPU single-thread performance doubled every 18 months for a long time; DRAM access latency has improved only a few percent per year. The result: a cache miss today takes hundreds of CPU cycles of stall, and this gap keeps widening.

**Current state:**
- L1 hit: ~4 cycles
- L2 hit: ~12 cycles
- L3 hit: ~40–80 cycles
- Local DRAM: ~200–300 cycles
- Remote DRAM (NUMA): ~400–500 cycles
- NVMe SSD: ~100,000+ cycles
- Disk: millions of cycles

Each step is an order of magnitude or more. A workload where 5% of accesses miss all caches spends half its time waiting for DRAM.

**Architectural attacks on the memory wall:**

**1. Deep and wide out-of-order execution.** Hide latency with parallelism. A 500-entry ROB can hold hundreds of in-flight instructions, enabling 10+ outstanding memory accesses simultaneously (memory-level parallelism). This converts latency-bound to bandwidth-bound, which is a more tractable problem.

**2. Hardware prefetchers.** Pull lines into the cache before the CPU demands them. Correct on ~90% of streaming workloads; can bridge 50–70% of the DRAM latency gap on regular access patterns.

**3. Bigger caches.** L3 sizes have grown from ~8 MB (circa 2010) to 96 MB+ on some 2024 server parts, and recent consumer CPUs (AMD X3D) add 96 MB L3 via 3D stacking. Hit rate goes up, but diminishing returns set in quickly past the working set.

**4. Higher-bandwidth DRAM.** DDR5 to HBM3 to CXL-attached memory pools. Bandwidth has scaled even as latency has not, so wide SIMD / vector workloads can keep busy.

**5. NVDIMMs and CXL memory.** Treat persistent memory as a slow DRAM tier, with the OS managing data placement. Another level in the hierarchy, increasing the number of meaningful placement decisions.

**6. Moving compute closer to memory.** Processing-in-memory (PIM) research puts small processors inside DRAM dies. Samsung's HBM-PIM products are early examples. Reduces data movement, which is the dominant energy cost.

**7. Accelerators with their own memory.** GPUs, TPUs, and NPUs come with fast local memory (HBM) and handle specific workloads (matrix multiply) where the dense compute-to-memory ratio justifies the dedicated bandwidth.

**Software perspective:** The memory wall has shifted the programmer's mental model. Optimising for cache behaviour (data layout, blocking, prefetching) now matters more than algorithmic constant factors in most performance-critical code. A "3× slower algorithm" that fits in cache often beats a "theoretically faster algorithm" that misses cache.

### Q10. Describe false-sharing-induced performance collapse. How would you diagnose it in production?

**Answer:**

**False sharing** (covered at basics in `cache_organisation.md`) is the killer of naive multi-threaded code. Unlike contention on an explicit shared resource, it is invisible in the source — two threads that appear to touch independent variables happen to share a cache line and pay full coherence costs on every write.

**Why "performance collapse":**

- Uncontended cache hit: ~4 cycles.
- Same line being bounced between cores in a modified state: ~100–300 cycles per transfer.
- A tight loop with one increment per iteration: **75× slowdown** if the line is bouncing.

At scale, a 16-thread benchmark that should show linear speedup over one thread instead shows **sublinear or even inverse scaling** — more threads mean worse performance as the bounce rate on the shared line increases.

**Diagnostic approach in production:**

**1. Look at the scaling curve.** Run the workload with 1, 2, 4, 8, 16 threads. If throughput peaks at a low thread count and then degrades, suspect contention — false sharing is a strong candidate.

**2. Check coherence performance counters.**
- Intel: `MEM_LOAD_RETIRED.L3_HIT`, especially `HITM` (hit modified — means the line was dirty in another cache). A high HITM rate is the fingerprint.
- Intel: `MEM_LOAD_L3_HIT_RETIRED.XSNP_MISS` (clean snoop from another core).
- AMD: `L3_ACCESS.COHERENT_READ_HIT_XO` and friends.
- Linux `perf c2c`: **the** purpose-built tool. It samples coherence events and produces a report listing cache lines with heavy cross-core traffic, including the source code locations accessing them.

**3. Inspect flagged cache lines.**
`perf c2c record` + `perf c2c report` yields output like:

```
   Cacheline        LLCL-Miss-Loads    LLCL-Miss-Stores    Readers     Writers
   0xfff000001234       3450                  5670           2             2
```

For each flagged line, it shows the source code locations and thread IDs responsible. This usually makes the false sharing obvious.

**4. Verify with a fix.** Add `alignas(64)` (or the equivalent cacheline padding) to the suspect structure and re-run. If throughput scales linearly, confirmed.

**Common culprits:**

1. **Thread-local statistics counters** packed into a struct shared by all threads.
2. **Per-CPU variables** without cache-line alignment (Linux kernel has `DEFINE_PER_CPU_ALIGNED` for exactly this).
3. **Lock + protected data** on the same line: after acquiring the lock, the first read of the data fetches the same line the lock was on, which is fine — but other threads polling the lock during contention bounce the line.
4. **Shared queue head and tail pointers** on the same line.
5. **Array of small structures** (e.g. `struct { int x; } arr[16]` accessed by different threads on different elements).

**Fix patterns:**

- `alignas(64) struct` to round structure size up to a cache line.
- Split hot and cold fields into separate structures.
- For arrays of per-thread data, ensure each element is at least 64 bytes and 64-byte aligned.
- On Linux, `__cacheline_aligned_in_smp` does this per the active architecture's cache line size.

**Interview signal:** A candidate who immediately reaches for `perf c2c` (or its equivalent) shows they have done production performance work on multicore. Candidates who only talk about "use a profiler" without naming specific tools or counters are guessing.
