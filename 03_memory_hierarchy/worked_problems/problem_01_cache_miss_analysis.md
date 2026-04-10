# Worked Problem 01 — Cache Miss Rate Analysis for Matrix Traversal

**Topic:** Cache behaviour on strided access
**Difficulty:** Intermediate

---

## Problem

Consider the following C code:

```c
#define N 1024
double A[N][N];

void sum_rows(double *out) {
    for (int i = 0; i < N; i++) {
        double s = 0;
        for (int j = 0; j < N; j++) {
            s += A[i][j];
        }
        out[i] = s;
    }
}

void sum_cols(double *out) {
    for (int j = 0; j < N; j++) {
        double s = 0;
        for (int i = 0; i < N; i++) {
            s += A[i][j];
        }
        out[j] = s;
    }
}
```

The system has a 32 KB L1 data cache with 64-byte lines, 8-way associative. `A` is 8 MB (1024 × 1024 × 8 bytes), 8-byte aligned.

Compute the L1 miss rates for `sum_rows` and `sum_cols`. Discuss the performance implications.

---

## Solution

**Cache geometry:**
- 32 KB L1, 64-byte lines → 512 lines total, 64 sets.
- Each cache line holds 8 doubles (64 / 8 = 8).

**Matrix layout:** C stores 2D arrays in **row-major** order. `A[i][j]` is at address `&A + (i*N + j)*8`. Elements `A[i][0]`, `A[i][1]`, ..., `A[i][7]` are in the same cache line. Consecutive rows are offset by `N × 8 = 8192 bytes`.

---

**Case 1: `sum_rows`** — row-major traversal

Each iteration of the inner loop moves to the next element along a row — contiguous in memory.

- The first access `A[i][0]` misses (cold).
- The cache line brought in holds `A[i][0..7]`. The next 7 iterations hit.
- `A[i][8]` misses again; the next 7 hit.

Miss pattern: 1 miss per 8 accesses.

**Miss rate (sum_rows)** = 1/8 = **12.5%**

Total misses per row: 1024/8 = 128 misses.
Total misses over the whole matrix: 1024 × 128 = **131,072 misses**.

The working set per outer iteration is 8 KB (one row), well below the 32 KB cache. Even if multiple rows are brought in due to prefetching, they all fit.

---

**Case 2: `sum_cols`** — column-major traversal (strided)

The inner loop advances through `i`, jumping by `N*8 = 8192 bytes` between accesses.

- `A[0][j]` misses; the cache line holds `A[0][j..j+7]` but only `A[0][j]` will be used.
- `A[1][j]` is 8192 bytes away → different cache line → miss.
- `A[2][j]` → miss. And so on.

**Miss rate (sum_cols, cold):** 100% of inner-loop accesses miss.

**But does anything stay warm across the outer iterations?** When we finish column `j` and start column `j+1`, will the lines holding column-`j` still be in cache?

- We brought in 1024 lines for column `j`, one per row.
- Total data fetched: 1024 × 64 = 64 KB.
- L1 capacity: 32 KB.

The working set for one column is 64 KB, twice the L1 size. By the time we reach `A[1023][j]`, the earlier `A[0][j]`'s line has long been evicted. When we start column `j+1`, the data we need (`A[0][j+1]`, which is the same cache line as `A[0][j]`) is no longer in cache.

**Miss rate (sum_cols):** **100%** of inner-loop accesses miss. Every access to `A` goes all the way to L2 (or worse).

Total misses over the whole matrix: 1024 × 1024 = **1,048,576 misses**.

---

**Numerical performance comparison:**

Ratio of misses: 1,048,576 / 131,072 = **8×** more misses for `sum_cols`.

**What does this mean for runtime?**

Assume:
- L1 hit cost: 4 cycles (hidden by OoO on most accesses)
- L2 hit cost: 12 cycles
- L3 hit cost: 40 cycles
- DRAM cost: 200 cycles

If `sum_cols` goes to L2 for every miss (`A` fits in L3 at 8 MB if L3 ≥ 16 MB):
- 1 M misses × 12 cycles = 12 M cycles for the whole traversal.
- Matrix has 1 M elements; so ~12 cycles per element on average.

If `sum_rows` hits L1 on 7/8 of accesses, with the remaining 1/8 hitting L2:
- 0.875 × 4 cycles + 0.125 × 12 cycles = 3.5 + 1.5 = 5.0 cycles per element effective.

Even assuming OoO hides the L1 hit latency entirely, `sum_rows` reduces the memory-stall cost to near zero, while `sum_cols` pays ~12 cycles per element on L2 traffic alone. The row-major version is therefore 8–10× faster in practice — matching the miss-rate ratio.

**What if A were much larger, overflowing L3?** `sum_cols` would become DRAM-bound, with ~200 cycles per element. At 1 M elements, the traversal would take ~200 M cycles, or ~100 ms on a 2 GHz CPU — just for a single matrix sum. The row-major version on the same data would still be ~5 cycles per element ≈ 2.5 ms.

---

**Optimisations:**

1. **Change loop order** to match storage order. This is the single biggest win.

2. **Loop blocking (tiling)** for cases where both row and column access are needed:

   ```c
   for (int ii = 0; ii < N; ii += B)
     for (int jj = 0; jj < N; jj += B)
       for (int i = ii; i < ii+B; i++)
         for (int j = jj; j < jj+B; j++)
           ...
   ```

   With block size B chosen so that a B×B tile fits in L1, the traversal reuses each loaded cache line many times before eviction.

3. **Transpose the matrix** once at the start if you need column-major access downstream. The transpose itself is expensive (O(N²) cache misses) but amortises over repeated uses.

4. **SIMD vector loads** benefit enormously from contiguous access — a single AVX-512 load reads a full cache line. Column-major access cannot use this; it needs strided loads or gather, both much slower.

---

## Discussion

This is the canonical example that every performance engineer should have internalised. The *algorithm* is identical (sum all elements of a 2D matrix); only the *access pattern* differs. Yet the performance gap is an order of magnitude.

Three lessons:

1. **Memory access pattern dominates performance** on memory-bound code. The constant factor of the algorithm is irrelevant if the cache behaviour is bad.

2. **Language-level data layout matters.** C is row-major; Fortran is column-major. Loops must match the layout.

3. **The compiler cannot always fix this.** A smart compiler might recognise the `sum_cols` loop and switch the loop order, but only if the semantics allow it — and only in simple cases. Profile-guided optimisation helps but does not replace the programmer knowing the layout.

**Interview angle:** A candidate who sketches this kind of analysis (cache geometry → miss counting → cycle estimate) demonstrates that they think about performance quantitatively, not just qualitatively. This is exactly the kind of reasoning expected in systems and HPC interviews.
