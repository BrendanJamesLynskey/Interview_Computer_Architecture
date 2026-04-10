# Vector and SIMD — Interview Questions

**Subject:** Computer Architecture
**Topic:** SSE/AVX/SVE, Predication, Gather/Scatter, Vector Length Agnostic Code
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is SIMD and why is it efficient?

**Answer:**

**SIMD** (Single Instruction, Multiple Data) is a form of parallelism where one instruction operates on multiple data elements simultaneously. A single SIMD `ADD` might perform 8 parallel 32-bit additions on two 256-bit vector registers.

**Why it is efficient:**

1. **Amortised control overhead.** The fetch, decode, rename, issue, and retire cost is paid once per instruction — regardless of whether the ALU processes one element or 32. Throughput scales with vector width without growing the frontend.

2. **Simplified dependency tracking.** The renamer and scheduler treat the whole vector as one operand; no need to track 32 independent data dependencies.

3. **Fewer instructions per unit of work.** I-cache pressure and decoder bandwidth are reduced proportionally to the vector width.

4. **Data parallelism matches real workloads.** Image pixels, audio samples, array elements, neural network activations — all large collections of independent operations on small data. SIMD is a near-perfect fit.

**Cost:** Wider ALUs (bigger area, bigger power when active), longer vector load/store paths, and alignment requirements that make some loops harder to vectorise. The silicon cost is significant — an AVX-512 execution unit is several times the area of a scalar ALU — but the throughput per watt on vectorisable code is dramatic.

**Theoretical peak:** A 256-bit SIMD unit at 3 GHz processes 256/32 = 8 single-precision ops per cycle × 3 × 10⁹ = 24 GFLOPS per vector pipe per core. With FMA (fused multiply-add), double that to 48 GFLOPS. AVX-512 doubles again. Modern CPUs reach several hundred GFLOPS peak in single thread, all from SIMD.

### Q2. Compare packed (fixed width) SIMD with vector (variable length) ISAs.

**Answer:**

**Packed SIMD** (SSE, AVX, AVX-512, NEON): The instruction encoding specifies an exact operand size — "add 8 floats in a 256-bit register". Software targeting AVX-256 will only run on a machine that supports AVX-256; adding wider vectors (AVX-512) requires new instructions with new encodings. The binary hard-codes the width.

**Vector ISAs** (Cray-style, ARM SVE, RISC-V V): The instruction specifies "add vectors of floats", leaving the width to hardware. A loop counter tracks how many elements are still to process; the hardware sets the actual count for each iteration based on its physical vector length. The same binary runs on any implementation regardless of physical vector width.

**Advantages of vector ISAs:**

1. **Binary portability.** One binary runs on 128-bit, 256-bit, 512-bit, or 2048-bit hardware. No need for multiple AVX-{128,256,512} builds.

2. **Tail handling built in.** The hardware naturally processes the final partial iteration with a shorter effective vector. No need for manual scalar cleanup loops.

3. **Future-proofing.** New hardware with wider vectors speeds up old binaries for free.

4. **Predication.** SVE and RVV have first-class predicate registers that mask individual lanes, making control flow inside vectors natural.

**Advantages of packed SIMD:**

1. **Simpler hardware.** No runtime configuration of vector length; fixed-width units are easier to pipeline.
2. **Simpler compilers.** Loop unrolling and auto-vectorisation are more straightforward with known widths.
3. **Predictable performance.** Every instruction has known latency and width.

**Why SVE and RVV exist now:** The industry is returning to the vector-ISA model because packed SIMD has reached an ugly endgame — x86 has SSE, SSE2, SSE3, SSSE3, SSE4.1, SSE4.2, AVX, AVX2, AVX-512, AVX-512-VL, AVX-512-BW, AVX-512-DQ, AVX-512-VNNI, etc. Each generation adds new encodings for slightly different widths or semantics. Binary fragmentation is painful. A vector ISA with length-agnostic code solves this cleanly.

**Interview insight:** RISC-V V has been argued as one of the strongest technical reasons to consider RISC-V for modern CPU designs. It starts without the legacy of SSE/AVX and ships with a clean vector model.

### Q3. What is a fused multiply-add (FMA) and why is it important?

**Answer:**

**FMA** computes `a × b + c` as a single instruction with a single rounding at the end — not "multiply then add" with two separate roundings. On vector versions, it does this per lane.

**Performance benefit:**

1. **Single-cycle throughput for the MAC operation.** A classic `MUL` followed by `ADD` would be two instructions with a dependency; FMA is one. For inner products and matrix multiplications, this doubles the effective throughput of the SIMD pipeline.

2. **Peak FLOPS counted with FMA.** "GFLOPS" in marketing is counted as 2 FLOPs per FMA (one multiply + one add). So a 256-bit SIMD with one FMA unit at 3 GHz is 8 lanes × 2 ops × 3 GHz = **48 GFLOPS per pipe**. Without FMA, the same hardware would count 24 GFLOPS.

**Accuracy benefit:**

A separate multiply followed by add rounds twice: once on the product, once on the sum. FMA rounds once, computing `round(a*b + c)` with the full unrounded product. This is **always at least as accurate** as separate ops and usually more so. It matters for:

- **Polynomial evaluation.** Horner's scheme in FMA form is both faster and more accurate.
- **Linear algebra.** Dot products with FMA maintain more precision than with separate ops, important for ill-conditioned matrices.
- **Kahan summation.** Some compensated-summation algorithms only work with single-rounding FMA.

**Downsides:**

- **Reproducibility.** A program compiled with FMA may produce slightly different results than one without — the extra precision changes the final bit of the result. Reproducible numerics across ISAs is harder.
- **Not universally supported.** Early x86 did not have it; GCC's `-mfma` is still a decision the user has to make.

**Commercial note:** Intel FMA (FMA3, from Haswell) and AMD FMA (FMA3, from Piledriver) both use the 3-operand form `d = a * b + c` that overwrites one input. Older AMD Bulldozer had FMA4 with a 4-operand form `d = a * b + c` leaving all inputs intact. FMA4 was more flexible but was dropped in favour of FMA3 for compatibility.

---

## Intermediate

### Q4. How does the compiler auto-vectorise a loop, and what prevents it?

**Answer:**

**Auto-vectorisation** is the compiler's attempt to convert a scalar loop into a SIMD loop. For a simple case:

```c
for (int i = 0; i < n; i++) a[i] = b[i] + c[i];
```

the compiler generates:

1. A **vectorised body** that loads 8 elements from `b`, 8 from `c`, adds them, and stores 8 to `a` per iteration.
2. A **scalar cleanup** that handles the final 0–7 elements.
3. Alignment handling if the pointers are not known to be 32-byte aligned at compile time.
4. Alias checks via runtime version-selection (the "multiversioning" path).

**What prevents auto-vectorisation:**

1. **Aliasing.** If the compiler cannot prove `a`, `b`, and `c` do not overlap, it must fall back to scalar. C99's `restrict` keyword helps. Most compilers generate a runtime check and a fast SIMD path when aliasing can be ruled out dynamically.

2. **Data-dependent control flow.** A loop with an `if` that early-exits (breaks) on a condition is hard to vectorise because the break cannot happen at an element boundary inside a vector. Predicated execution helps but not for breaks.

3. **Reductions with non-associative operators.** Floating-point summation is non-associative (rounding changes), so strict-precision compilers refuse to reorder. `-ffast-math` relaxes this and typically unlocks 4× or more speedup on dot products.

4. **Memory stride != 1.** SIMD loads prefer contiguous memory. Strided accesses require gather/scatter, which are much slower. Structure-of-arrays layout beats array-of-structures precisely for this reason.

5. **Function calls in the loop.** Unless the called function is inlined and also vectorisable, the loop cannot be converted.

6. **Data dependencies across iterations.** `a[i] = a[i-1] + b[i]` has a loop-carried dependency. The compiler cannot vectorise naively; it needs to recognise this as a prefix-sum pattern (which some compilers can, using specialised SIMD intrinsics).

7. **Pointer dereferences with unknown provenance.** A loop that walks a linked list cannot be vectorised at all — the next pointer depends on the current load.

**Compiler reports:** GCC's `-fopt-info-vec` and clang's `-Rpass=loop-vectorize` tell you which loops were vectorised and which were not (and why). This is essential for performance tuning — staring at assembly is the last resort.

### Q5. What is predication and how does it enable vectorisation of conditional code?

**Answer:**

**Predication** lets a SIMD instruction apply to only a subset of lanes, selected by a per-lane **predicate** (mask) — a vector of booleans. Lanes whose predicate is false either produce unchanged output (the "merge" policy) or zero (the "zero" policy), depending on the instruction.

**Example — the classic conditional update:**

```c
for (int i = 0; i < n; i++) {
    if (a[i] > 0) a[i] = a[i] * 2;
}
```

Naively, the `if` breaks SIMD — within a vector, some lanes take the branch and some do not. Predication rewrites the loop as:

1. Load 8 elements of `a`.
2. Compare each lane to zero, producing an 8-bit mask.
3. Multiply each lane by 2.
4. Predicated store: only write back lanes where the mask is true.

No branching. The vector still processes 8 elements per iteration, with the mask hiding the work for lanes that should not have been updated.

**Cost:** The work for "false" lanes is still done — just discarded. For a loop where the predicate is 50/50 true/false, you do twice the work per element compared to an ideal scalar version. For extreme skew (1% true), it can be net-negative compared to scalar; but even then, the cost is usually dominated by the memory loads, which had to happen anyway.

**Predicate registers:**

- **SSE / AVX:** No dedicated predicate registers. Masks are stored in the same SIMD registers as data. Operations like `VBLENDVPS` (blend based on vector mask) implement predication by selecting between two full vector results.

- **AVX-512:** First-class predicate registers `k0`..`k7`. Every vector instruction can specify a predicate. Far cleaner and enables much more predicated code.

- **ARM SVE:** Predicate registers `p0`..`p15` are fundamental. Nearly every SVE instruction is predicated; loops are written around a governing predicate that shrinks as elements are processed.

- **RISC-V V:** Similar to SVE; a mask register `v0` is used with the `vm` field of each instruction to enable predication.

**Why it matters for control flow:** Predication turns data-dependent control flow (normally a vectorisation blocker) into straightforward SIMD code. This is the single biggest semantic win of modern vector ISAs over first-generation SIMD.

### Q6. What are gather and scatter operations, and why are they slow?

**Answer:**

- **Gather:** Load N elements from N *independent* memory addresses into a single SIMD register. `vgatherdps zmm0, [zmm1 + rbase]` loads 16 floats from 16 addresses computed by adding each lane of `zmm1` to `rbase`.

- **Scatter:** Store N elements from a SIMD register to N independent addresses. Inverse of gather.

**Why they exist:** Many important workloads access memory with irregular or strided patterns — graph traversal, hash tables, particle simulations, sparse matrix operations. Without gather/scatter, the compiler must emit N separate scalar loads and insert each result into a SIMD lane — slow and expensive.

**Why they are slow compared to contiguous vector loads:**

1. **Multiple cache line accesses.** A gather of 16 floats can touch up to 16 distinct cache lines — each requires a separate L1 access (or a cache miss). A contiguous load touches 1 line.

2. **Sequential cache port arbitration.** L1 caches have 1–2 ports. A gather is decomposed internally into multiple micro-ops that time-share the ports, executing over many cycles.

3. **Potential TLB pressure.** 16 addresses may touch 16 pages, each requiring a TLB lookup. Most designs bound this by walking the gather in chunks that fit in one TLB lookup.

4. **No prefetching benefit.** A stride prefetcher cannot predict gather addresses because they are computed at runtime from arbitrary indices.

5. **Exception complexity.** Any lane could fault. The hardware must maintain precise state — which elements have been successfully gathered, which are still pending.

**Typical performance:**
- Contiguous AVX-512 load: 1 cycle throughput.
- Gather of 16 elements: 10–20 cycles throughput in the best case (all hit L1), much worse on cache misses.

**Software alternatives:**

- **Restructure data (struct-of-arrays):** Often transforms gather-heavy code into contiguous loads.
- **Software gather (scalar loads + shuffle):** Sometimes faster than the hardware gather instruction because the compiler can schedule around memory latency more flexibly.
- **Specialised kernels:** For sparse matrix-vector products, hand-coded kernels using gather outperform general auto-vectorised loops.

**Interview angle:** Gather/scatter are valuable but their cost is often hidden. A candidate who recognises that "this loop is dominated by gather throughput, not compute" shows real SIMD performance intuition.

---

## Advanced

### Q7. Explain vector length agnostic (VLA) code in SVE or RISC-V V.

**Answer:**

Traditional packed SIMD code knows the vector width at compile time: AVX-256 instructions always operate on 8 floats. A loop is written as "process 8 elements per iteration" with a scalar tail.

**Vector length agnostic** code does not bake the width into the binary. The loop asks the hardware "how many elements can you process this iteration?" and the hardware answers based on its physical vector length. The same binary runs efficiently on machines with 128, 256, 512, 1024, or 2048-bit vectors.

**ARM SVE example (conceptual):**

```asm
    mov     x1, #0                // i = 0
loop:
    whilelt p0.s, x1, x2          // p0 = predicate for lanes [i, min(i+VL, n))
    b.none  done                  // if no active lanes, exit
    ld1w    z0.s, p0/z, [x3, x1, lsl #2]  // predicated load
    ld1w    z1.s, p0/z, [x4, x1, lsl #2]
    fadd    z0.s, z0.s, z1.s
    st1w    z0.s, p0, [x5, x1, lsl #2]
    incw    x1                    // i += VL (width in 32-bit lanes)
    b       loop
done:
```

**Key mechanisms:**

1. **`WHILELT` predicate generator:** Creates a predicate `p0` whose leading lanes are true up to the end of the loop range, and false after. Automatically handles the tail — the final iteration has fewer active lanes.

2. **Governing predicate on every operation:** Loads, stores, and compute are all predicated by `p0`. Inactive lanes are either zeroed or merged depending on the suffix (`/z` or `/m`).

3. **`INCW` / `INCB` / ...:** Increment by the hardware vector length — whatever it is. Code does not reference a numeric constant.

4. **`B.NONE`:** Exit when no lanes are active (last partial iteration handled; next iteration has nothing to do).

**Advantages:**

- **One binary, any hardware.** A VLA binary compiled for SVE runs on Fujitsu A64FX (512-bit), Arm Neoverse V1 (256-bit), and any future implementation regardless of width.
- **No scalar tail.** The predicate naturally shrinks for the final partial vector. No separate scalar cleanup code needed.
- **Future-proof.** A faster chip with wider vectors speeds up existing binaries automatically.

**Cost:**

- **Compiler complexity.** Auto-vectorisation is harder because loop-unrolling decisions cannot depend on a known width. GCC and clang SVE support is improving but still behind AVX-512 on some idioms.
- **Harder to reason about performance.** Running on an unknown vector width makes it harder to predict throughput for a given kernel.

**Interview insight:** VLA is the first serious break from the "know the width at compile time" paradigm that has dominated SIMD since MMX. It will likely become the default for new ISAs, and compilers are catching up. Familiarity with SVE/RVV-style code signals you are tracking the field beyond x86.

### Q8. Describe the interaction between AVX-512 and CPU frequency scaling.

**Answer:**

Early AVX-512 implementations (Skylake-SP, Cascade Lake) had a feature notoriously known as the "**AVX-512 frequency penalty**". When AVX-512 instructions were executed, the core transitioned to a lower clock frequency — sometimes 20–30% below the all-core turbo — to stay within power and thermal limits. This affected *all* code running on the core during and after the AVX-512 burst, not just the SIMD instructions themselves.

**Why it happened:**

1. **Wider SIMD units consume more power when active.** AVX-512 functional units are roughly 2× the AVX-256 area and power. Running them at full clock within the TDP envelope would exceed the thermal limit.

2. **Voltage guardbanding.** Wide SIMD generates more power noise (Vdroop), requiring more voltage margin for reliable operation. Higher voltage means lower frequency for the same thermal budget.

3. **Coarse-grained transitions.** The frequency change took many microseconds to stabilise. Once you dropped, you stayed dropped for a while — so a brief AVX-512 burst could hurt minutes of subsequent scalar code.

**Consequences:**

- Workloads with occasional AVX-512 and mostly scalar/AVX-2 often ran *slower* overall than pure AVX-2 versions, because the frequency penalty dominated the compute gain.
- Many performance engineers explicitly disabled AVX-512 via environment variables or compile flags on server workloads.
- Glibc's memcpy was famously reverted from AVX-512 to AVX-256 on affected Intel parts.

**Modern situation (Ice Lake and later):**

- **Power delivery improvements:** Better voltage regulation and finer-grained FIVR/DLVR allow AVX-512 to run closer to the all-core turbo.
- **Per-core transitions:** Frequency transitions happen faster and are localised; other cores do not pay the price.
- **"Light" vs "heavy" AVX-512 instructions:** The penalty is now tiered — simple AVX-512 arithmetic runs near full speed, heavy AVX-512 (like VNNI, multi-cycle FMAs, many-lane shifts) still incurs some reduction.
- **Alder Lake/Raptor Lake controversy:** Intel disabled AVX-512 on consumer parts (E-cores do not support it; asymmetric scheduling is hard) — a separate issue from the frequency penalty.

**Lesson for performance engineers:** You cannot just turn on AVX-512 and assume speedup. You must profile the *whole* workload including the effect on non-AVX-512 threads running on the same core or socket. The lesson generalises: powerful SIMD extensions couple compute and power in ways that scalar code does not, and the interaction must be measured, not assumed.

### Q9. What is a systolic array and why is it a different paradigm from general-purpose SIMD?

**Answer:**

A **systolic array** is a regular grid of small processing elements (PEs) in which data flows through the array in a choreographed rhythm — inputs enter on one side, results emerge on another, and each PE does a small fixed operation on whatever data is currently passing through it.

**Canonical example — matrix multiplication C = A × B on a 2D systolic array:**

- Rows of A stream in from the left.
- Columns of B stream in from the top.
- Each PE receives one element of A (from its left neighbour) and one element of B (from its above neighbour), multiplies them, adds to its local accumulator, and passes the values on to its right and below neighbours.
- After the data has streamed through, the accumulators hold the result matrix C.

**Key properties:**

1. **Data reuse is maximised.** Each matrix element is loaded from memory once and then propagates through N PEs as it travels across the array. An N×N systolic array reaches O(N²) compute per memory access, compared to O(N) for a naive CPU implementation.

2. **Local communication.** PEs only talk to their immediate neighbours. No long wires, no global buses, no coherence traffic. This is friendly to layout and scales well.

3. **Deterministic dataflow.** There are no caches, no branch prediction, no dynamic scheduling. The entire execution is a planned pipeline. No wasted work on speculation.

4. **High arithmetic intensity.** A systolic multiply-accumulate is two ops per data element — but each element participates in many MACs as it flows through. Effective ops per byte of memory traffic is enormous.

**Why it differs from SIMD:**

- **SIMD** is a general-purpose vector pipeline. Each instruction does one thing to many lanes. Lanes do not communicate with each other directly. Data reuse comes from register file and caches.

- **Systolic** is a specialised dataflow pipeline. Each PE does one thing repeatedly, and data flows continuously between PEs. Data reuse is built into the movement pattern, not stored in any cache.

**Real-world instances:**

- **Google TPU:** A 256×256 systolic array is the core of each Tensor Processing Unit. Designed specifically for matrix multiply, which dominates neural network compute.
- **Groq:** Dataflow accelerator with systolic-array-like compute lanes.
- **NVIDIA Tensor Cores:** Small 4×4×4 mixed-precision systolic-style arrays embedded in GPU SMs, composable into larger matrix ops.

**Trade-off:** Systolic arrays are extraordinarily efficient for matrix multiply and convolution but useless for anything else. A CPU's SIMD unit at 48 GFLOPS is fully general; a systolic array at 10 TFLOPS is locked to a specific operation. The right design depends on whether your workload has enough matrix compute to justify the specialisation — which is exactly what deep learning inference and training do.

### Q10. A 512-bit SIMD ALU running at 2.5 GHz computes 32 int16 MACs per cycle. What is its peak ops/sec and what determines whether you can reach it?

**Answer:**

**Peak:**

32 lanes × 2 ops/MAC × 2.5 × 10⁹ cycles/sec = **160 GOPS**

(Each MAC counts as 2 operations: one multiply and one add. This is the standard industry convention.)

**Reaching peak requires all of the following to hold simultaneously:**

1. **Enough data parallelism.** The workload must offer 32 independent lanes of work per cycle with no cross-lane dependency. Loops with loop-carried state or scalar reductions do not qualify.

2. **Memory bandwidth matches compute.** 32 int16s = 64 bytes per cycle. At 2.5 GHz, this is 160 GB/s of input bandwidth per operand × 2 operands = 320 GB/s just to feed the ALU. DDR5-5600 provides maybe 90 GB/s per channel — four channels is the typical consumer limit. You cannot stream in 320 GB/s from DRAM; the data must come from cache or registers.

3. **Data reuse.** To make the bandwidth problem tractable, each loaded element must be used multiple times. A matrix multiply with block size B reuses each element ~B times; B must be chosen so the working block fits in L1 or L2 and the compute-to-memory ratio exceeds the bandwidth ratio.

4. **Correct alignment.** Unaligned loads can be slower or split across cache lines. Peak requires data aligned to 64 bytes (cache line) for 512-bit loads.

5. **No pipeline stalls.** Loop overhead (branch, counter update, pointer increment) must be inlined or handled by separate execution ports so it does not eat vector pipe cycles. Modern OoO cores handle this well.

6. **No cache misses.** A single L2 miss (~40 cycles) wastes ~1280 vector ops worth of compute. Working sets must be resident.

7. **No frequency throttling.** As discussed in Q8, heavy SIMD can reduce clock frequency, which directly reduces the 2.5 GHz figure.

**Realistic achievable:**

For a well-tuned GEMM (matrix multiply) kernel on cached data: **70–85%** of peak is reachable.

For a typical un-tuned loop: **10–30%** of peak.

For a memory-bound streaming loop: **< 5%** of peak, because memory bandwidth not compute is the bottleneck.

**The key insight:** Peak GOPS is an upper bound, not a spec to optimise toward blindly. The *effective* compute is determined by whichever of compute, memory bandwidth, and cache hit rate is the narrowest bottleneck. Good HPC and machine-learning code is dominated by arithmetic intensity tuning — making each memory access carry enough compute to saturate the pipe.

**Roofline model** (Williams et al.) formalises this: plot achievable throughput as min(peak_compute, bandwidth × arithmetic_intensity). Your workload lies somewhere on this roofline, and the optimisation task is to push it along the intensity axis until it touches the peak compute line.
