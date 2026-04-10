# Branch Prediction — Interview Questions

**Subject:** Computer Architecture
**Topic:** Static/Dynamic Prediction, BTB, RAS, Bimodal/Gshare/TAGE Predictors
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. Why is branch prediction necessary in modern processors?

**Answer:**

A modern processor has a pipeline depth of 15–25 stages and issues 4–8 instructions per cycle. Branches appear roughly every 5–6 instructions in typical code. Without prediction, every branch would stall the frontend until the condition was evaluated, idling the entire pipe for the branch resolution latency — typically 10–15 cycles on a deep out-of-order machine. That would cut IPC by a factor of 3–5×.

Prediction lets the frontend **speculatively fetch past the branch**, keeping the backend fed. If the prediction is correct, the speculation commits normally. If wrong, the pipeline is flushed from the branch onwards and refetched. The performance of a modern CPU is dominated by how often this speculation is right: at 95% accuracy the misprediction penalty is manageable; at 85% it is catastrophic (see `classic_pipeline_and_hazards.md` Q8).

### Q2. What is the difference between static and dynamic branch prediction?

**Answer:**

**Static prediction** uses a fixed rule encoded at compile time or wired into the hardware. No runtime state is needed.

Common static rules:
- **Always predict not taken** — simplest; correct ~50% on unbiased branches.
- **Always predict taken** — correct ~65% because most branches (especially loops) are taken.
- **Backwards taken, forwards not taken (BTFNT)** — exploits the fact that backward branches are usually loop closures (taken) and forward branches are often if-statement skips (often not taken). Correct ~70%.
- **Compiler hint bits** — the ISA reserves a hint bit in the branch encoding that the compiler sets using profile data. x86 once had such hints, now ignored.

**Dynamic prediction** uses hardware structures that record recent branch behaviour and predict based on it. This is essential for deep pipelines because static accuracy (~70%) implies a 30% misprediction rate, which is ruinous at modern pipeline depths.

A dynamic predictor can achieve 95–99% accuracy on typical code because most branches are **strongly biased** — they go the same way nearly every time they execute — and the remaining branches show recognisable patterns (loop counters, alternating, correlated with nearby branches) that history-based predictors can learn.

### Q3. What is a Branch Target Buffer (BTB) and why is it separate from the direction predictor?

**Answer:**

A **BTB** is a small cache, indexed by the branch PC, that stores the **target address** of recently-taken branches. When IF presents a PC to the BTB, a hit returns the predicted target for the next fetch immediately — before the instruction has even been decoded.

The BTB is separate from the **direction predictor** (which answers "taken or not taken?") because the two have different information needs:

- **Direction** needs history-based correlation and is often globally indexed (gshare, TAGE).
- **Target** needs a straightforward PC-indexed cache, because the target for a given branch is (usually) constant.

On a correct BTB hit and a correct direction prediction, the fetch can redirect **zero cycles** after the branch is fetched — the next cycle's IF reads from the predicted target. Without a BTB, even a correctly predicted "taken" branch would cost at least one bubble because the target is not known until decode.

**BTB misses** happen on:
1. A branch seen for the first time — cold BTB.
2. A branch that was evicted.
3. An **indirect branch** whose target has changed (calls through a function pointer, virtual method dispatch).

The BTB also carries a **type tag** (conditional, call, return, indirect) so the frontend knows to invoke the return-address stack, indirect predictor, or direction predictor appropriately.

### Q4. What is the return address stack (RAS)?

**Answer:**

A **return address stack** is a small hardware stack that predicts the target of `RET` instructions. Whenever a `CALL` is fetched, its return address (PC + call-instruction size) is pushed onto the RAS. When a `RET` is fetched, the RAS is popped and the popped value is used as the predicted target.

**Why this is important:** A `RET` is an indirect jump whose target changes at every invocation of the callee (a function called from two call sites returns to two different places). A plain BTB sees the `RET` as one branch with two different historical targets and cannot distinguish them.

The RAS exploits the structural fact that function calls nest like a stack: each `CALL` will (almost always) be matched by a `RET` that returns to the matching push. A 16–32 entry RAS is sufficient for almost all real programs.

**Failure modes:**
- **Deep recursion** overflows the RAS; the oldest entries are lost and those returns mispredict.
- **`setjmp`/`longjmp`** or exception unwinding skips returns, leaving stale RAS entries that cause mispredictions on subsequent returns.
- **Tail calls** (a function ends with `JMP callee` instead of `CALL callee; RET`) omit the return, causing a RAS underflow eventually.
- **Return-oriented-programming** attacks deliberately desynchronise the RAS from the actual call stack.

Modern CPUs shadow the RAS or keep a second "overflow" structure to recover gracefully.

---

## Intermediate

### Q5. Describe the two-bit saturating counter (bimodal) predictor.

**Answer:**

A **bimodal predictor** is a table of 2-bit saturating counters, indexed by low-order bits of the branch PC. Each counter has four states:

```
        00          01          10          11
    Strongly    Weakly      Weakly      Strongly
    Not Taken   Not Taken   Taken       Taken
```

**Prediction rule:** Predict "taken" if MSB of the counter is 1, else "not taken".

**Update rule:** On a taken branch, increment (saturating at 11). On a not-taken branch, decrement (saturating at 00).

**Why two bits, not one?** With a single bit, a single anomalous outcome would flip the prediction. The two-bit version requires *two consecutive* opposite outcomes to flip the prediction, giving hysteresis. This matters because real branches have occasional exceptional iterations (e.g. a loop exit is the last iteration of a mostly-taken branch) that a one-bit predictor would be destabilised by.

**Example: loop with 100 iterations**

- One-bit predictor: mispredicts the first iteration (learns from cold), then correct for 99, then mispredicts the last (loop exit), then mispredicts the *first* iteration of the next loop because the state was flipped. That's **3 mispredicts per 100**.
- Two-bit predictor: mispredicts the first iteration, correct for 99, mispredicts the last, but the state only moved from "strongly taken" to "weakly taken" — next loop iteration it predicts taken again correctly. That's **2 mispredicts per 100**, a 33% improvement.

**Table size trade-off:** Too small and unrelated branches alias into the same counter ("destructive aliasing"). Too large and more silicon is wasted than the accuracy improvement justifies. 4096–16384 entries is typical for the bimodal stage of a hybrid predictor.

### Q6. What is the correlating (two-level, global history) predictor? How does gshare work?

**Answer:**

A **two-level predictor** separates the history (what branches recently did) from the prediction table (what that history maps to). The idea is that branches are not independent — the outcome of one branch is often correlated with recent branches.

**Example:**
```c
if (x == 0)  { ... }      // branch A
if (x != 0)  { ... }      // branch B
```
Here branch B always takes the opposite direction of A. A bimodal predictor sees each branch individually; a correlating predictor records the history of recent outcomes and uses it to disambiguate.

**Structure:** A **global history register (GHR)** is an *n*-bit shift register that records the outcomes of the last *n* conditional branches (1 = taken, 0 = not taken). A **pattern history table (PHT)** of 2-bit saturating counters is then indexed by some function of the GHR and the branch PC.

**gshare** uses the XOR of the GHR with the branch PC as the PHT index:

$$\text{index} = \text{PC}_{\text{low}} \oplus \text{GHR}$$

XOR mixes PC and history, so branches at different PCs with the same GHR (or the same PC with different GHRs) map to different PHT entries. This spreads entries across the table and reduces aliasing compared to concatenation.

**Why it works:** The predictor learns per-(PC, history) patterns. A loop counter with a repeating exit pattern creates one PHT entry per phase of the pattern. Branches with data-dependent behaviour that correlates with recent history (a very common case) are now distinguishable.

**Cost:** A single PHT indexed by *k* bits has $2^k$ entries. A gshare predictor with 12 bits of GHR and 14 bits of PC (14-bit index) has 16384 entries × 2 bits = **4 KB**. This typically achieves 93–96% accuracy on SPEC.

### Q7. What is a tournament (hybrid) predictor?

**Answer:**

A **tournament predictor** combines two (or more) predictors and uses a meta-predictor to choose which one to trust for each branch. The classic combination is bimodal + gshare:

- **Bimodal** is excellent for strongly-biased branches that do not need history (it has no aliasing across branches on the PC-indexed table).
- **gshare** is better for branches whose behaviour correlates with recent history.

**Meta-predictor:** A second table of 2-bit counters, indexed by PC, tracking *which* of the two predictors has been more accurate for this branch recently. Incremented when gshare is right and bimodal wrong; decremented when the opposite. The MSB selects which predictor's output to use.

**Why this works:** Different branches have different "temperaments". Loop iteration count branches are strongly biased and do not need history. Complex data-dependent branches benefit strongly from history. Letting each branch find the predictor that suits it adds ~1–2% accuracy over either alone, which is significant at the top of the curve where every percentage point matters.

The Alpha 21264 famously introduced the tournament predictor and its design has been widely influential.

---

## Advanced

### Q8. Describe TAGE. What makes it more accurate than gshare?

**Answer:**

**TAGE** (TAgged GEometric history length) is the state of the art in branch direction prediction, used in nearly every high-end CPU shipped after about 2010. It was introduced by Seznec and Michaud in 2006.

**Key ideas:**

1. **Multiple tables with geometrically-increasing history lengths.** Rather than picking a single history length (as gshare does), TAGE maintains several tables $T_0, T_1, ..., T_n$, where table $T_i$ is indexed by a hash of the PC and the most recent $L_i$ bits of global history, with $L_i$ growing geometrically (e.g. 0, 5, 14, 44, 134 bits).

2. **Tagged entries.** Each non-base table entry carries a partial tag (hash of PC and history). A lookup only succeeds if the tag matches, avoiding the aliasing problems of PC-XOR-history indexing. The base table $T_0$ is untagged (a bimodal predictor).

3. **Longest-match rule.** On a lookup, all tables are probed in parallel. The predictor's output comes from the **tagged table with the longest history** that matches; the base table is used only if no tagged table matches.

**Why this is better:**

- Branches that benefit from long history automatically use it. Branches that do not (strongly biased loop branches) use a short-history table or the base and do not suffer long-history aliasing.
- Each branch is **implicitly allocated the history length that suits it**, without any explicit meta-prediction.
- Updates only allocate entries on mispredictions, with a "useful" bit marking entries that have actually helped. This keeps scarce table space filled with productive entries.

**Performance:** TAGE achieves 97–99% accuracy on SPEC CPU, compared to 94–95% for gshare. On branch-heavy workloads (compilers, SPEC int) the difference is multiple percent IPC.

**Modern variants:** ITTAGE predicts indirect-branch targets using the same principle; TAGE-SC-L adds a statistical corrector and a loop predictor as additional components.

### Q9. How does a perceptron predictor work, and when is it better than TAGE?

**Answer:**

A **perceptron branch predictor** (Jimenez, 2001) treats branch prediction as a classification problem. Each branch PC indexes a weight vector $w = (w_0, w_1, ..., w_n)$ where $w_0$ is a bias weight and $w_i$ corresponds to the *i*-th most recent branch outcome (from the global history). The prediction is:

$$y = w_0 + \sum_{i=1}^{n} w_i \cdot h_i$$

where $h_i \in \{-1, +1\}$ is the *i*-th history bit (encoded as $+1$ for taken, $-1$ for not taken). Prediction is taken iff $y \geq 0$.

**Training:** on misprediction (or when $|y|$ is small, indicating low confidence), update weights:

$$w_i \leftarrow w_i + t \cdot h_i$$

where $t = +1$ if the branch was taken, $-1$ otherwise. Weights are saturated to a bounded range.

**What this buys:**

- **Linear storage in history length.** To use *n* bits of history, TAGE needs tables proportional to how many distinct patterns exist (which can grow exponentially). A perceptron needs only *n* weights per branch, so it can use **much longer histories** — hundreds or even thousands of bits.

- Handles **linearly separable** patterns natively. Any branch outcome that is a linear function of recent history (including long-range correlations) is learned efficiently.

**Limitations:**

- Cannot learn XOR-like patterns — the classic failure mode of a single-layer perceptron. "Branch B is taken iff exactly one of A, C is taken" is not linearly separable.
- The summation is on the critical path of the predictor and becomes timing-critical at high clock speeds. Solutions use **piecewise-linear** arrangements (separate weight table per history position) that can be precomputed.

**Hybrid reality:** Modern processors use neither pure TAGE nor pure perceptron but combinations. Intel's Sandy Bridge used a perceptron; AMD Zen uses a hashed perceptron + TAGE hybrid. The dominance of one technique over the other depends on workload mix and cycle time budget.

### Q10. Wrong-path execution: explain the subtle correctness issues it creates.

**Answer:**

**Wrong-path execution** is the execution of instructions fetched along a predicted path that turns out to be incorrect. These instructions are eventually squashed by the flush on branch mispredict, so architecturally they "never happened" — but they can still cause observable effects that must be handled carefully.

**Correctness-critical issues:**

1. **Memory accesses must not fault.** A wrong-path load can present a bogus address (e.g. dereferencing a pointer that was only valid on the correct path). If that address causes a page fault, the handler would run and alter architectural state. The rule is that **speculative loads are allowed to silently fail**: the TLB miss is squashed, the page walker is cancelled, and no exception is raised until the instruction actually reaches commit. This is why page faults on wrong-path loads are invisible to software.

2. **Stores must not reach memory.** Wrong-path stores write to the **store buffer** with a speculative tag but do not merge into the cache. On squash, the store-buffer entries are discarded.

3. **Cache-line fills are still initiated.** This is deliberate: a wrong-path load that triggers a line fill warms the cache for the correct path, if the correct path eventually references the same line. This turns out to be a net win on real workloads. It is also the observable side channel exploited by **Spectre v1** — the attacker controls a wrong-path load to pull a secret-dependent line into the cache, then measures cache state from a non-speculative context.

4. **Prefetchers must ignore wrong-path signals.** If a hardware stride prefetcher learns from wrong-path load addresses, it will train on garbage and pollute the cache.

5. **Performance counters must distinguish.** Most modern ISAs provide counters that measure "speculative" vs "retired" events separately, because a "retired load" is architecturally visible while a "speculative load" may be wrong-path noise.

6. **Store-to-load forwarding must be careful.** A younger wrong-path load cannot forward from a younger wrong-path store (fine, both squashed together) but must also not forward *to* an older correct-path load — achieved by tagging each load with the speculative state of the branch it is speculating past.

**The wider lesson:** wrong-path execution is architecturally invisible but microarchitecturally real. All the Spectre-class vulnerabilities stem from the fact that the microarchitecture was *designed* to make wrong-path execution fast (cache fills, branch-predictor updates) without realising that those same side effects could be observed by a careful attacker. Fixing this without giving up the performance has been one of the central challenges of the last decade of processor design.
