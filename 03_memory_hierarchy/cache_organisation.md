# Cache Organisation — Interview Questions

**Subject:** Computer Architecture
**Topic:** Direct-Mapped, Set-Associative, Replacement, Write Policies, Inclusion
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is the difference between direct-mapped, set-associative, and fully-associative caches?

**Answer:**

Cache associativity controls how many distinct locations a given memory address can occupy:

- **Direct-mapped:** Every memory address maps to exactly one cache line. Index = (address / line size) mod (number of lines). Fast and cheap to look up — only one tag comparison needed — but prone to conflict misses when two frequently-used addresses happen to map to the same line.

- **N-way set-associative:** The cache is divided into sets of N lines each. An address's set is determined by its index bits, but within that set, the line may be in any of the N slots. Lookup reads all N tags in parallel and selects the matching way. Conflict misses only occur when more than N hot addresses map to the same set.

- **Fully associative:** Any address can go in any cache line. Lookup requires comparing against every tag simultaneously (CAM-style). No conflict misses — misses are purely capacity or compulsory. Expensive in silicon; only used for very small caches (TLBs, victim buffers).

**Typical modern choices:**
- L1 caches: 4–8 way (balancing latency and miss rate)
- L2 caches: 8–16 way
- L3 caches: 16–32 way
- TLBs: fully associative or very high associativity

**Why not always fully associative?** Cost. A fully-associative 32 KB L1 with 64-byte lines would need 512 parallel tag comparators — impractical at high clock speeds. Set-associative caches get most of the benefit with a small fraction of the hardware.

### Q2. Explain the address decomposition for a set-associative cache.

**Answer:**

An address is split into three fields: **tag**, **index**, and **offset**. The offset selects a byte within a cache line; the index selects a set; the tag distinguishes different addresses that map to the same set.

**Example:** 32 KB L1, 8-way, 64-byte lines, 48-bit virtual address.

- Line size: 64 bytes → **offset** = 6 bits.
- Total lines: 32 KB / 64 B = 512 lines.
- Number of sets: 512 / 8 = 64 sets → **index** = 6 bits.
- Tag = 48 − 6 − 6 = **36 bits**.

On a lookup:
1. Bits [11:6] of the address select the set (64 entries).
2. Bits [47:12] are compared in parallel against all 8 tag entries in that set.
3. If any matches and the line is valid, that way is selected and bits [5:0] select the requested byte(s).
4. If no match, it is a miss.

**Common point of confusion:** The index bits must come from the portion of the address *below* the tag bits — specifically, the bits immediately above the offset. Using other bits for the index is possible ("skewed-associative" caches) but complicates coherence and prefetching.

### Q3. What are compulsory, capacity, and conflict misses?

**Answer:**

The "three Cs" classification of cache misses, due to Hill (1987):

- **Compulsory miss** (cold miss): the first access to a line — it has never been in the cache. Unavoidable without prefetching.

- **Capacity miss:** the line was in the cache but got evicted because the working set exceeds the cache size. Would not occur in an infinitely-large cache.

- **Conflict miss:** the line was in the cache but got evicted because too many other lines mapped to the same set. Would not occur in a fully-associative cache of the same size.

**Measurement technique:** Run the program through three cache simulations:

1. Infinite cache → only compulsory misses.
2. Fully-associative cache of actual size → compulsory + capacity misses.
3. Actual cache organisation → compulsory + capacity + conflict.

Subtracting gives the breakdown.

**Why it matters:**

- High **compulsory** rate → prefetching (software or hardware) is the fix.
- High **capacity** rate → need a larger cache or tighter working set (loop blocking, data layout).
- High **conflict** rate → need more associativity or better index mapping (padding arrays to avoid pathological strides).

**Fourth C — coherence miss:** Added later for multicore — a miss caused by another core's write invalidating this core's copy. Critical in parallel code where false sharing can dominate.

### Q4. What is a write-through vs write-back cache?

**Answer:**

The write policy determines what happens on a store that hits the cache:

- **Write-through:** Every store writes both the cache and the next level of the hierarchy. The cache line is always consistent with the next level — no dirty data.

- **Write-back:** Stores write only the cache line and mark it dirty. The dirty data is written to the next level only when the line is evicted.

**Trade-offs:**

| Property | Write-through | Write-back |
|---|---|---|
| Write bandwidth to next level | High — every store | Low — only on eviction |
| Complexity | Simpler — no dirty state | More complex — dirty bits, write-back queue |
| Crash recovery | Easier — memory always up to date | Harder — dirty data can be lost |
| Coherence | Simpler | More complex — dirty lines are the cache's "private" state |

**What modern CPUs use:** L1 caches are usually **write-back** because the store rate is high and writing through would saturate the L1/L2 bus. L2 and L3 are also write-back. The memory controller is the only point where writes finally reach DRAM.

**Write miss policies:**
- **Write-allocate:** On a store that misses, bring the line into the cache first, then write it. Standard with write-back.
- **No-write-allocate:** On a store miss, write directly to the next level without pulling the line in. Sometimes paired with write-through for streaming writes that are unlikely to be re-read.

**Store buffers as hybrids:** Most modern cores have a **store buffer** between the store pipeline and the L1 cache. Stores accumulate in the buffer and are merged or coalesced before committing. This gives write-back-like efficiency even on a write-through L1.

---

## Intermediate

### Q5. Describe LRU and pseudo-LRU replacement. Why is true LRU rarely implemented?

**Answer:**

**True LRU** (Least Recently Used) evicts the line in the set that has been unused for the longest. It requires per-line timestamps or a total order over all N ways of the set. For N-way associativity:

- **Timestamp method:** log₂(access count) bits per line, updated on every access. Complex and space-hungry.
- **Total-order method:** Maintain a rank field of log₂(N!) bits per line encoding the full order. For N=8, that is ~16 bits per line — not cheap — and updating the order on each access is non-trivial.

**True LRU is rarely implemented** beyond N=4 because the cost scales poorly. Instead, **pseudo-LRU** is used:

**Tree-PLRU (for N-way, N = power of 2):** Use a binary tree of N−1 bits. Each internal node's bit indicates which subtree was less recently used. On access, update the bits along the root-to-leaf path to "point away" from the accessed leaf. On eviction, follow the tree bits to find the victim.

- For 8-way: 7 tree bits per set. Very cheap.
- Approximates LRU: the evicted line is guaranteed to be in the "older half" (and in practice is often the true LRU).

**Other practical policies:**

- **NRU** (Not Recently Used): one bit per line, set on access, cleared periodically. Evict any line whose bit is clear.
- **Random:** Pick a random way. Surprisingly effective and simplest to implement; Intel L2 used random replacement in some generations.
- **RRIP** (Re-Reference Interval Prediction): A modern generalisation. Lines are classified by their predicted time to next reuse; scan for "distant re-reference" candidates. Widely used in L3 caches. Particularly good at resisting streaming workloads that would evict the entire cache under LRU.

### Q6. What is an inclusive vs exclusive vs NINE cache hierarchy?

**Answer:**

These terms describe the relationship between levels of the cache hierarchy (say, L2 and L3).

- **Inclusive:** Every line present in L1 (or L2) is also present in L3. The outer cache is a strict superset of the inner caches.

- **Exclusive:** A line is present in exactly one level. When a line is promoted from L3 to L1 on a hit, it is removed from L3. When evicted from L1, it is written back to L3.

- **NINE (Non-Inclusive Non-Exclusive):** Weaker than inclusive — a line present in L1 *might* be in L3 or not. No strict invariant is enforced.

**Trade-offs:**

| Property | Inclusive | Exclusive | NINE |
|---|---|---|---|
| Effective capacity | L3 size (L1 contents duplicated) | L1 + L2 + L3 | Between the two |
| Snoop filtering | L3 tags are sufficient to check if any cache has the line | L3 tags are insufficient — must snoop inner caches | Partial filtering |
| Complexity | Inclusion must be enforced on eviction | Promotion/demotion on every hit | Simpler |
| Intel example | Nehalem, Sandy Bridge L3 | Broadwell L3 (changed) | Skylake-X L3 |

**Why Intel switched from inclusive to non-inclusive L3 (Skylake-X):** Core counts grew large (10+), and the L3 capacity was no longer dominant over the sum of L1/L2 sizes. An inclusive L3 wastes space on redundant L1/L2 copies. Dropping inclusion freed up ~25% of effective cache.

**Cost of dropping inclusion:** Snoop filtering is now harder. Inclusive L3 lets the directory check "is this line in any core's cache?" by looking only at L3 tags. Without inclusion, a separate **snoop filter** structure is needed — which is effectively a tag-only cache that tracks which cores have which lines.

**Inclusion enforcement:** In an inclusive hierarchy, evicting a line from L3 forces invalidation of any copies in L1/L2 (a "back-invalidate"). This ensures the invariant holds. The cost is that a poor L3 replacement decision can evict a hot line and cause cascading invalidations.

### Q7. What is a victim cache and when is it useful?

**Answer:**

A **victim cache** is a small fully-associative cache that holds lines recently evicted from a larger direct-mapped or low-associativity cache. On a miss in the main cache, the victim cache is checked; if the line is there, it is swapped back in with zero external memory traffic.

**Motivation:** Small direct-mapped caches have bad conflict-miss behaviour. Two frequently-accessed lines that map to the same index thrash each other. A 4–16 entry fully-associative victim cache catches exactly those thrashing victims.

**Origin:** Jouppi, 1990, in the context of early RISC processors with tiny (4–8 KB) direct-mapped L1 caches. A 4-entry victim cache reduced L1 miss rates by 20–50% on typical workloads — a massive improvement for almost no silicon.

**Modern relevance:** L1 caches are now 8-way associative, so conflict misses are much less common. Victim caches have largely fallen out of favour. However, the same *idea* lives on:

- **AMD Bulldozer's L3 "victim cache"** behaviour: L3 held lines evicted from L2, effectively functioning as a large victim layer (the cache was marketed as exclusive).
- **TLBs** sometimes use a small victim structure to catch thrashing between a fully-associative L1 TLB and a set-associative L2 TLB.

**Interview value:** Knowing about victim caches signals awareness of the historical progression of cache design and the economic reasoning (a small fully-associative structure can fix the pathological behaviour of a much larger but less flexible one).

---

## Advanced

### Q8. Given a 64 KB, 4-way set-associative L1 with 64-byte lines, what is the impact of adding a 256-entry 4-way L1 TLB? Compute the page reach and discuss limits.

**Answer:**

**Cache geometry:**
- Size: 64 KB
- Associativity: 4-way
- Line size: 64 B
- Number of sets = 64 KB / (4 × 64 B) = **256 sets**
- Index bits = log₂(256) = 8

**Cache indexing constraint:** Cache index bits come from address bits above the offset (bits [13:6] here). With a 4 KB page (12 offset bits), the cache index bits [13:6] include bits [13:12] which are *above* the page offset — meaning they come from the **physical page number** and are not known until after TLB translation. This is the classic **aliasing problem** for virtually-indexed physically-tagged (VIPT) caches.

- If (cache size / associativity) ≤ page size, the index is entirely within the page offset and VIPT is safe without page-colouring constraints.
- Here, 64/4 = 16 KB > 4 KB, so there *are* two index bits above the page offset. The hardware must either page-colour (OS allocates pages so these bits match) or do additional aliasing checks.

**TLB reach calculation:**
- 256 entries × 4 KB page = **1 MB** of virtual address space covered by the L1 TLB.

**Is 1 MB enough?** For small working sets (hot loops, stacks), yes. For anything touching large arrays, no — a single 10 MB dataset scan incurs TLB misses on almost every cache line fetch, and each miss triggers a page walk (typically 4 memory accesses on x86-64, or a 4-level walk on AArch64).

**Mitigations:**

1. **Huge pages (2 MB or 1 GB):** One TLB entry covers 2 MB of virtual address. A 256-entry TLB with 2 MB pages reaches 512 MB — enough for most workloads. OS support for transparent huge pages is essential.

2. **Second-level TLB:** A larger unified L2 TLB (e.g. 2048 entries) catches L1 TLB misses at lower cost than a full page walk.

3. **Page-walk cache:** Caches the intermediate levels of the page table, reducing the walk cost from 4 memory accesses to 1–2 on most misses.

4. **Prefetching TLB entries:** Some designs prefetch adjacent TLB entries, especially on sequential access patterns.

**TLB reach as an ISA-level concern:** This is why both x86 and ARM have committed to supporting multiple page sizes at the hardware level — the fundamental tension between fine-grained memory protection (small pages) and TLB efficiency (large pages) cannot be solved by a single size.

### Q9. Explain the difference between VIPT, PIPT, and VIVT caches.

**Answer:**

These refer to how virtual vs physical addresses are used for cache **indexing** and **tagging**.

| Style | Index | Tag | Lookup order |
|---|---|---|---|
| **VIVT** (Virtually-Indexed, Virtually-Tagged) | virtual | virtual | Fast — no TLB lookup |
| **VIPT** (Virtually-Indexed, Physically-Tagged) | virtual | physical | Cache lookup and TLB lookup in parallel |
| **PIPT** (Physically-Indexed, Physically-Tagged) | physical | physical | TLB first, then cache |

**VIVT (rarely used today):** Fastest lookup because the virtual address directly selects the line and tag. But every context switch requires flushing the cache (or tagging entries with ASIDs) because the same virtual address means different things in different processes. Also, **synonyms** (two virtual addresses mapping to the same physical address, common in shared memory) can produce multiple cached copies of the same data, violating coherence.

**PIPT (common for L2/L3):** Both index and tag come from the physical address. No aliasing or synonym problems. Context switches do not invalidate the cache. But the TLB must be consulted first, serialising TLB latency before cache access — slow for L1.

**VIPT (common for L1):** The clever compromise. Index bits come from the **page offset portion** of the virtual address, which is the same as the physical address (the lower 12 bits of both). Tag bits come from the physical address after TLB translation. Cache set lookup and TLB lookup happen in parallel — the cache reads all 4 (or 8) ways of the set while the TLB translates the upper bits; once the TLB finishes, the resulting physical tag is compared to the cache tags to select the hit way.

**VIPT constraint:** Index bits must fit within the page offset. If `cache_size / associativity > page_size`, some index bits come from the virtual page number — causing potential aliasing. The workarounds are:

1. **Cap the VIPT cache size:** keep `cache_size/associativity ≤ page_size`. On x86 with 4 KB pages, this means an L1 is limited to `4 KB × associativity`. A 32 KB 8-way cache just fits. A 64 KB 4-way cache does not.

2. **Page colouring:** The OS allocates physical pages such that the high index bits match the virtual ones. Fragile and constrains the page allocator.

3. **Aliasing detection:** Add hardware to detect when two virtual addresses with different "extra" index bits refer to the same physical line. Complex.

**Modern reality:** Apple M-series uses 16 KB pages, which relaxes the VIPT constraint and allows much larger L1 caches. Intel and AMD stick with 4 KB and accept the ~32 KB L1 ceiling — or use other tricks (MICRO-TAGs, partial PIPT).

### Q10. What is a prefetcher? Describe two common hardware prefetch algorithms and when they fail.

**Answer:**

A **hardware prefetcher** observes memory access patterns and issues speculative cache fills for lines the CPU has not yet requested, hoping they will be used soon. Successful prefetches hide memory latency. Wasted prefetches waste memory bandwidth and pollute the cache.

**Two common algorithms:**

**1. Stream prefetcher (next-line / stride detector):**

- **Next-line:** On a miss for line X, prefetch line X+1. Exploits spatial locality. Very cheap to implement.
- **Stride detector:** Watches for consistent stride patterns. For a sequence of accesses A, A+8, A+16, ..., detect stride = 8 and prefetch A+24, A+32 ahead of the demand miss. Requires per-stream history: maintain a table indexed by PC (the load instruction) with "last address" and "predicted stride" fields.

- **Strength:** Catches the common cases — sequential array traversal, strided matrix rows. Hides most of the latency on memory-bound numerical kernels.
- **Failure modes:** Non-strided access (hash tables, linked lists, graph traversal). Polluting non-working-set lines. Training on wrong-path loads (most designs now ignore wrong-path).

**2. Temporal / correlation prefetcher:**

- Records pairs of miss addresses: "after missing on A, we often miss on B next". Build a history table mapping one miss to its typical successors. On a miss for A, look up B and prefetch it.

- Handles irregular but repeatable access patterns (linked data structures traversed repeatedly).
- **Strength:** Works where stream prefetching cannot — e.g. hash chain walks, tree lookups.
- **Weakness:** Large history table required. First traversal of a new structure is not prefetched. Cold startup cost.

**When prefetchers fail:**

1. **Streaming workloads that exceed cache capacity.** The prefetcher fills the cache with lines that are used once and then evicted; it just races the demand stream, never getting ahead.

2. **Adversarial access patterns** (hash tables, random pointer chase). No detectable pattern.

3. **Cross-page boundaries.** Hardware prefetchers typically stop at page boundaries (otherwise they might cross into unmapped regions, wasting TLB entries and bandwidth). Long sequential scans suffer a bubble at every 4 KB page.

4. **Wrong-path pollution.** If wrong-path loads train the prefetcher, the prefetcher learns garbage and pollutes the cache with wrong-path data. Modern designs filter out wrong-path events.

5. **Bandwidth saturation.** A prefetcher that issues too aggressively saturates the memory controller, slowing down demand misses and net-losing. Good prefetchers throttle themselves based on observed miss rate and queue depth.

**Software prefetch (`__builtin_prefetch`, `PREFETCHT0`):** The programmer inserts explicit prefetch hints where they know the access pattern better than the hardware. Most useful on graph traversal and pointer-chasing code. Often inserted by profile-guided optimisation in HPC kernels.

**Interview angle:** The best answer notes that prefetching is a **confidence-weighted speculation**, with the same accuracy/cost trade-off as branch prediction. Aggressive prefetching wins on regular workloads and loses on irregular ones; the key microarchitectural work is in the training filters that decide *which* streams to prefetch and *how far* ahead.
