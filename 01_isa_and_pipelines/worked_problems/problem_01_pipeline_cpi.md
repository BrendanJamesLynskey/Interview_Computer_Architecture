# Worked Problem 01 — Effective CPI from Pipeline Stalls

**Topic:** Classic pipeline performance analysis
**Difficulty:** Intermediate

---

## Problem

A classic 5-stage in-order pipeline has the following characteristics:

- Ideal CPI = 1.0 (one instruction per cycle)
- 25% of instructions are loads; 10% of loads are followed immediately by a dependent instruction (load-use hazard), each causing a 1-cycle stall.
- 18% of instructions are branches; the branch predictor is 92% accurate, and the misprediction penalty is 2 cycles.
- 1% of instructions incur a 10-cycle cache miss stall.

Compute the effective CPI.

---

## Solution

Each stall source contributes independently to the effective CPI, following:

$$\text{CPI}_{\text{eff}} = \text{CPI}_{\text{ideal}} + \sum_i (\text{fraction}_i) \times (\text{stall cycles}_i)$$

**Load-use hazard contribution:**

Load-use hazards affect 25% × 10% = 2.5% of instructions, each costing 1 cycle:

$$0.25 \times 0.10 \times 1 = 0.025 \text{ CPI}$$

**Branch misprediction contribution:**

Branches are 18% of instructions; 8% of them mispredict at 2 cycles each:

$$0.18 \times (1 - 0.92) \times 2 = 0.18 \times 0.08 \times 2 = 0.0288 \text{ CPI}$$

**Cache miss contribution:**

Cache misses affect 1% of instructions at 10 cycles each:

$$0.01 \times 10 = 0.100 \text{ CPI}$$

**Total effective CPI:**

$$\text{CPI}_{\text{eff}} = 1.0 + 0.025 + 0.0288 + 0.100 = \mathbf{1.154}$$

---

## Discussion

This example shows how the contributions stack up. Each stall source looks small individually, but they aggregate to a ~15% throughput loss on otherwise-ideal code.

**Which is the biggest culprit?** The cache miss stall dominates at 0.10 CPI — more than 3× the next-largest source. Even at a 1% miss rate, 10-cycle stalls add up. This is why modern memory hierarchies invest so heavily in reducing miss rates: shaving the 1% to 0.5% would save ~4% of runtime on this workload, more than any achievable improvement to branch prediction.

**What if the cache-miss penalty grows to 200 cycles (a realistic L3 miss)?**

$$0.01 \times 200 = 2.0 \text{ CPI from cache misses alone}$$

The new effective CPI would be 1.0 + 0.025 + 0.0288 + 2.0 = **3.05** — triple the ideal, with the machine spending two-thirds of its time waiting for memory. This is why out-of-order execution, which can overlap independent work with the miss, is essential for any CPU competing on real workloads.

**Branch prediction sensitivity:** If the predictor accuracy drops from 92% to 85%, the branch contribution becomes 0.18 × 0.15 × 2 = 0.054, a 0.025 increase — comparable to the load-use cost. A predictor improvement from 92% to 97% saves the same amount.

**Interview takeaway:** Practice computing these CPI breakdowns by hand. The ability to quickly estimate which stall source matters most in a given scenario is exactly what's being tested in performance-oriented interviews.
