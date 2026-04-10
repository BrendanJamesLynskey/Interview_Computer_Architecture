# Quiz — Memory Hierarchy

**Subject:** Computer Architecture
**Topics covered:** Caches, TLBs, virtual memory, coherence, consistency
**Format:** Multiple-choice and short-answer questions. Answers at the end.

---

### Q1. A 32 KB 8-way set-associative cache with 64-byte lines has how many sets?

---

### Q2. Which of the following is NOT one of the "three Cs" cache miss types?

a) Compulsory
b) Capacity
c) Conflict
d) Coherence

---

### Q3. A write-back cache with write-allocate policy does what on a store that misses?

a) Writes directly to the next level without touching the cache.
b) Fetches the line into the cache, then performs the write; marks the line dirty.
c) Writes to both the cache and the next level simultaneously.
d) Allocates a new line but does not fetch it.

---

### Q4. Explain why tree-PLRU is used instead of true LRU on high-associativity caches.

---

### Q5. In an inclusive cache hierarchy, evicting a line from L3:

a) Has no effect on L1/L2.
b) Forces a back-invalidation of any copies in L1/L2.
c) Promotes the line to L4.
d) Invalidates all cache lines in the system.

---

### Q6. A VIPT L1 cache with 16 KB size, 4-way associativity, 64-byte lines, on a system with 4 KB pages. Does this cache suffer from the VIPT aliasing problem? Justify.

---

### Q7. Name two causes of a TLB shootdown and explain why shootdowns are expensive.

---

### Q8. Under the MESI protocol, a cache line in Exclusive state that receives a write from its own core transitions to which state, and how much bus traffic is generated?

---

### Q9. In TSO, which of the following reorderings is permitted?

a) Store → Store
b) Load → Load
c) Load → Store (later store)
d) Load passing an older store to a different address

---

### Q10. What is false sharing and how would you detect it in production using Linux tooling?

---

### Q11. A workload has 0.5 memory accesses per instruction, a 2% TLB miss rate on the L2 TLB, and an average page-walk cost of 25 cycles. Compute the CPI contribution from page walks.

---

### Q12. Explain the "memory wall" in one or two sentences and name two architectural techniques used to mitigate it.

---

### Q13. Why does release consistency allow higher performance than sequential consistency, despite being weaker?

---

### Q14. What is the MOESI "Owned" state, and when does it outperform plain MESI?

---

### Q15. A multi-level page table on x86-64 with 4 KB pages has how many levels, and what is the worst-case number of memory accesses for a single TLB-miss walk (ignoring page-walk caches)?

---

## Answers

**A1.** Total size / (ways × line size) = 32 KB / (8 × 64) = 32768 / 512 = **64 sets**.

**A2.** (d). Coherence is a "fourth C" sometimes added, but it is not part of Hill's original three.

**A3.** (b). Write-allocate means the missing line is fetched into the cache, and the write updates the cached copy marking it dirty. Write-back means the dirty data stays in the cache until eviction.

**A4.** True LRU requires a total ordering of the ways in each set, which needs log(N!) bits and complex update logic — O(N²) for some schemes. Tree-PLRU uses only N−1 bits per set and updates in O(log N). It approximates LRU closely enough (the evicted line is always in the older half of the set) for negligible miss-rate difference.

**A5.** (b). Inclusion invariant requires that any line in the inner caches also be in the outer; evicting from outer forces invalidation of inner copies.

**A6.** Cache size / associativity = 16 KB / 4 = 4 KB = page size. The index bits are entirely within the page offset, so VIPT is safe without aliasing concerns. Answer: **No VIPT aliasing problem**.

**A7.** Causes: unmapping a page (munmap), permission change on a mapped page (mprotect), COW fault duplication, page migration. Expensive because: the OS must send an IPI to every CPU that might have the translation cached, each CPU must take an interrupt and invalidate its TLB, and the originator must wait for all acknowledgements before proceeding — thousands to tens of thousands of cycles including the cross-core synchronisation.

**A8.** Transitions to **Modified** with **zero** bus traffic. The Exclusive state's reason for existing is exactly this case — a silent E→M transition that avoids the coherence broadcast that would be needed from Shared.

**A9.** (d). TSO allows loads to pass older stores to different addresses (the natural behaviour of a store buffer). Store→Store, Load→Load, and Load→later-Store are all forbidden under TSO.

**A10.** False sharing: two or more threads repeatedly modifying different variables that happen to lie on the same cache line, causing coherence traffic even though the accesses are logically independent. Detect with `perf c2c record && perf c2c report`, which samples HITM coherence events and identifies cache lines with heavy cross-core traffic along with source locations.

**A11.** Walks per instruction = 0.5 × 0.02 = 0.01. CPI contribution = 0.01 × 25 = **0.25 CPI**. Substantial — roughly a quarter of a cycle per instruction going to page walks.

**A12.** The memory wall is the growing gap between CPU speed and DRAM latency, where a cache miss can stall hundreds of cycles. Mitigations include: large out-of-order windows with many in-flight memory ops (hiding latency with parallelism), prefetching (hardware and software), bigger caches, huge pages, HBM and 3D-stacked memory, processing-in-memory.

**A13.** Release consistency only requires ordering around explicitly-marked acquire/release operations. Between those points, the hardware is free to reorder memory ops arbitrarily, exposing more ILP and allowing simpler (and faster) store-buffer and commit logic. Sequential consistency requires preserving program order everywhere, which forces costly store-buffer drainage and load-ordering checks on every op.

**A14.** Owned is "dirty and shared" — one cache has the authoritative dirty copy but others can have read-only copies, with the owner responsible for writing back on eviction. It outperforms MESI on producer-consumer patterns because the producer can share dirty data cache-to-cache without writing through to memory; MESI would require a write-back to memory before sharing.

**A15.** Four levels (PML4, PDPT, PD, PT) for 48-bit addresses. Worst-case walk is **4 memory accesses** (one per level). Five-level paging (Ice Lake and later) for 57-bit addresses adds a fifth level and one more access. Page-walk caches reduce the typical cost to 1–2 accesses by caching upper-level entries.
