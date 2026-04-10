# Wide Issue and Scheduling — Interview Questions

**Subject:** Computer Architecture
**Topic:** Multi-Issue, Wakeup/Select, Load/Store Queues, Memory Disambiguation
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What does "issue width" mean, and how does it differ from "dispatch width" and "commit width"?

**Answer:**

These three terms describe different points in the OoO pipeline, each with its own bandwidth:

- **Dispatch width** — how many decoded instructions can enter the backend (ROB + issue queue) per cycle. Set by the frontend's decode and rename throughput.

- **Issue width** — how many instructions the scheduler can launch into execution per cycle. Set by the number of functional units and the read-port count of the physical register file.

- **Commit (retire) width** — how many instructions can leave the ROB per cycle. Set by the rate at which the architectural state (RAT, store buffer drain) can be updated.

**Typical modern ratios:**
- Apple Firestorm: ~8 wide decode, 8 wide issue, 8 wide commit
- Intel Golden Cove: 6 wide decode (from legacy decoders) + 8 wide µop cache, 12 wide issue, 8 wide commit
- ARM Cortex-X3: 6 wide decode, 8 wide issue, 8 wide commit

**Why these can differ:** The dispatch side is bound by decoder throughput (especially on x86's variable-length decode). The issue side can be wider because multiple functional units are cheap and issues are bursty. The commit side must be wide enough to avoid ROB backpressure, but on average only needs to match dispatch. Imbalance leads to silicon waste (one part blocked behind another) or throughput bottlenecks.

A candidate who talks only about "N-wide" without distinguishing these is missing the real design trade-offs.

### Q2. What does the wakeup-select loop do and why is it critical path?

**Answer:**

The wakeup-select loop is the part of the scheduler responsible for keeping the execution units fed each cycle. Its job:

1. **Wakeup:** when a producing instruction completes, broadcast its destination tag to every issue-queue entry. Any entry waiting on that tag marks the operand as ready.

2. **Select:** among all entries now ready to issue, pick up to N for this cycle (respecting functional-unit availability and priority).

3. **Issue:** dispatch the selected instructions into the execute pipeline.

All three must happen **within a single clock cycle** for the pipeline to sustain back-to-back dependent execution — otherwise, a `ADD` feeding the next `ADD` would incur a bubble each cycle.

**Why this is the critical path:**

- Wakeup is a broadcast over all issue queue entries (~100+ CAMs per tag).
- Select is a priority encoder over all ready entries — combinational logic with log-depth.
- The result must be latched and the selected instructions must have their operands read from the PRF before the next cycle.

All of this in one clock period sets a hard upper bound on clock frequency. Cutting the loop across two cycles (pipelining wakeup) requires speculative wakeup — predict that a dependent is still safe to wake one cycle early — and complicates replay on cache-miss misprediction.

### Q3. What is an "issue queue entry" and how does it differ from an "ROB entry"?

**Answer:**

- An **ROB entry** tracks an instruction's existence in program order, from dispatch to commit. It stores destination register, exception status, completion bit, and all the metadata needed for in-order retire.

- An **issue queue entry** tracks an instruction's readiness to execute. It stores the source operand tags (physical register numbers), the ready bits, the opcode, the destination tag, and (for value-holding schemes) the operand values.

**Lifecycle difference:**

- An instruction occupies an ROB entry from dispatch until commit — potentially hundreds of cycles.
- An instruction occupies an issue queue entry only from dispatch until issue — once it is selected and sent to the execution unit, the IQ entry is freed and reusable.

This is why issue queues are typically smaller than ROBs (e.g. 100-entry IQ with 500-entry ROB): the IQ only holds instructions that have not yet executed, while the ROB holds both executing and completed-but-uncommitted.

**Consequence:** An instruction that has completed execution but cannot commit yet (because an older instruction is still running) sits in the ROB but not the IQ. It is waiting for the ROB head pointer to reach it.

---

## Intermediate

### Q4. What is memory disambiguation and why is it necessary?

**Answer:**

**Memory disambiguation** is the process of determining whether a load and a store to dynamically-computed addresses are accessing the same memory location. Only when this is known can the scheduler correctly order them.

**Why it matters:** Consider:

```
I1: ST  [R1+8], R2
I2: LD  R3, [R4]
I3: ADD R5, R3, R6     ; depends on I2
```

If `R1+8 == R4`, the load `I2` must wait until the store `I1` commits (or forward from the store buffer). If they differ, the load is independent and can execute immediately — unlocking ILP.

**The naive conservative choice** — always wait until the store address resolves before issuing the load — kills performance. Typical code has many stores with long dependency chains feeding their address computations; serialising every load on every older store would starve the backend.

**The speculative choice** — let the load execute as soon as its own address is ready, assuming no alias — unlocks most of the potential ILP but creates a correctness problem: if the store's address later turns out to match, the load was wrong.

**Solution — memory-order violation detection:**

1. Loads that execute speculatively are kept in the **load queue** with their address and the data they returned.
2. When each older store's address resolves, it is checked against all younger loads in the load queue. A match indicates a violation.
3. On violation, the load (and everything dependent on it) is squashed and replayed. This is a memory-ordering machine clear — expensive, like a branch mispredict.

**Memory dependence predictor:** A small history table tracks which static loads have violated before. When a load is about to issue speculatively, the predictor consults its PC; if previously violated, the scheduler delays the load until older stores resolve. This cuts violation rates dramatically.

### Q5. What is the store queue, and how does it interact with store-to-load forwarding?

**Answer:**

The **store queue** is a FIFO between dispatch and cache commit, holding all in-flight stores until their ROB entries retire. Each entry holds:

- Virtual address (when computed)
- Physical address (after TLB lookup)
- Data (when computed)
- Size (byte, half, word, double, etc.)
- ROB entry pointer
- Ready bits: address-ready, data-ready, and "committed"

A store only writes to the cache (and thus becomes globally visible) when it commits. Before that, it lives in the store queue.

**Store-to-load forwarding:** When a load executes, it searches the store queue for any older store whose address matches. If found, the load forwards the store's data directly, bypassing the cache. This satisfies the uniprocessor ordering semantics (a load sees the most recent store from its own core) without waiting for the store to commit.

**Complications:**

1. **Partial overlap.** If the store writes one byte at `[R1]` and the load reads four bytes at `[R1]`, only one byte can be forwarded — the load cannot complete without cache access for the remaining bytes. Modern designs typically stall the load in this case.

2. **Size mismatch.** A store of a word followed by a load of a byte at the same base address can forward (byte-select the stored word). A byte store followed by a word load cannot (the other three bytes are unknown).

3. **Address not yet computed.** If an older store has a not-yet-resolved address, the load might alias it. The scheduler can either wait or use a dependence predictor to speculate.

4. **Alignment and sub-cache-line granularity.** Cross-cache-line loads that forward from multiple stores are particularly expensive; some designs forbid forwarding in this case, forcing the load to wait for commit.

**Performance cost of forwarding failure:** A failed forward (partial overlap, alignment, or just-missed) forces the load to wait for the store to commit, then reissue. This can cost 10+ cycles and is visible in performance counters as `ld_blocks.store_forward`. A major target of performance tuning is to arrange data so that forwarding succeeds — for instance, by storing and loading at the same natural width.

### Q6. How does an indirect-branch predictor (ITTAGE, indirect predictor) work?

**Answer:**

A plain BTB predicts branch targets assuming each branch PC has a stable target. Indirect branches (virtual dispatch, function pointers, computed jumps) violate this: the same branch PC can jump to different targets over time.

An **indirect-branch target predictor** extends TAGE-style history indexing to *targets*. Rather than predicting taken/not-taken, it predicts a target PC from the history of recent branches and the current branch PC.

**Structure (ITTAGE):**

1. **Base predictor:** A PC-indexed BTB, returning the most-recently-used target — correct when the indirect branch is monomorphic.

2. **Tagged tables:** Multiple tables indexed by PC XOR hash-of-history, with geometrically-increasing history lengths. Each entry stores a target PC and a confidence counter.

3. **Longest-match rule:** On lookup, the target comes from the longest-history table whose tag matches. Updates allocate and train entries only on mispredictions.

**Why it works:**

- **Polymorphic call sites:** A virtual dispatch through a vtable might resolve to class `A` in one path and class `B` in another. The history leading up to the call is typically different in the two cases, so the longer-history index disambiguates them.

- **Switch statements:** `switch(x)` on variable `x` often correlates with earlier computations on `x` that show up in branch history; the predictor learns the correlation.

- **Function pointers in callbacks:** Callbacks called from a dispatch loop with data-dependent patterns benefit similarly.

**Accuracy:** Modern ITTAGE predictors achieve 90–95% accuracy on indirect branches in object-oriented code, a huge improvement over the 50–60% of plain last-target predictors. This is one reason JIT-compiled dynamic languages (V8, HotSpot) have closed much of their performance gap with static languages — the hardware predictor learns their dispatch patterns.

**Security consideration:** Indirect predictors are the attack surface of **Spectre v2** (branch target injection). Mitigations (IBRS, IBPB, eIBRS, retpoline) deliberately flush or bypass the indirect predictor at privilege transitions, sacrificing some accuracy for isolation.

---

## Advanced

### Q7. Describe the trade-offs in sizing the ROB, issue queue, load queue, and store queue.

**Answer:**

These four structures together determine the memory-level and instruction-level parallelism a core can extract. Their sizing is a joint optimisation, not independent choices.

**ROB size:** Upper bound on in-flight instructions. Set by the rule $\text{ROB} \geq \text{IPC}_{\text{target}} \times \text{max latency}$. For 8 IPC and a 300-cycle L3 miss, you need 2400 entries — far more than any real design. Actual designs (300–600) balance against power, rename cost, and the point of diminishing returns on real workloads.

**Issue queue size:** Determines how many instructions can be "waiting to execute". Smaller than the ROB because many ROB entries are for completed-but-uncommitted instructions. Sized at maybe 1/4 to 1/3 of the ROB. Larger IQ allows the scheduler to look further ahead for ready work, improving utilisation on dependency-bound workloads.

**Load queue size:** Caps the number of in-flight loads. Every load allocates a load-queue entry at dispatch and frees it at commit. Undersizing stalls dispatch on memory-heavy code. Oversizing is expensive: load-queue entries must be CAM-searched on every store address resolution.

**Store queue size:** Caps in-flight stores. Each entry holds address + data + size. Undersizing stalls dispatch when stores cannot drain fast enough (e.g. streaming writes to a cold cache). Oversizing hurts because every load searches the store queue — doubling its size roughly doubles the CAM cost.

**Joint constraints:**

1. **Balance.** If ROB is 500 but load queue is 64, memory-heavy loops stall on load queue even though ROB is empty. If ROB is 500 but issue queue is only 60, the scheduler cannot find ready work among 500 in-flight instructions and IPC drops.

2. **Power and area.** Large structures are the biggest OoO core's biggest power consumers after the SRAM caches. Wakeup CAMs scale super-linearly in power with entry count.

3. **Workload sensitivity.** SPEC int workloads rarely need more than ~200 ROB entries to reach peak IPC. HPC / memory-bound workloads benefit from 500+. Mobile vs server designs often size structures differently for this reason.

**Modern examples:**

| Core | ROB | IQ | LQ | SQ |
|---|---|---|---|---|
| Intel Golden Cove | 512 | 205 | 192 | 114 |
| AMD Zen 4 | 320 | 188 | 136 | 64 |
| Apple Firestorm | ~630 | ~354 | ~188 | ~110 |
| ARM Cortex-X3 | 320 | ~168 | 112 | 72 |

### Q8. What is speculative wakeup and how does it interact with load replay?

**Answer:**

**Speculative wakeup** addresses the problem that the wakeup-select loop cannot fit in a single cycle at high clock frequencies. The fix is to wake up dependents *before* the producer has confirmed its result.

**Normal flow:**
1. Producer P issues.
2. Producer P executes (N cycles).
3. Producer P writes back; completion tag broadcast.
4. Consumer C with matching source tag wakes up.
5. Consumer C is selected and issues in the next cycle.

**Speculative flow (for fixed-latency ops):**
1. Producer P issues.
2. At the moment P issues, the scheduler immediately marks C's source ready, assuming P's result will arrive in N cycles.
3. C is scheduled to issue N cycles later, aligned with P's writeback.
4. When C's cycle arrives, the PRF value written by P in the same cycle is bypassed directly into C's ALU input.

**Benefit:** Back-to-back dependent ALU ops run at one cycle apart — matching the ideal pipeline throughput. Without speculative wakeup, there would be a bubble waiting for the wakeup-select loop to propagate.

**Problem: variable-latency producers.** Loads are the canonical case. The scheduler assumes L1 hit (typically 4 cycles) and wakes dependents 4 cycles after the load issues. If the load actually misses:

1. The consumer woke up on time, issued, and started executing.
2. At execute, the operand from the load is not yet valid (the load is still waiting for the cache fill).
3. The consumer's execution is invalid. Its result is garbage.

**Replay:** The scheduler detects the latency miss and **replays** the consumer (and any of *its* dependents that also woke speculatively). The consumer is re-inserted into the issue queue and re-scheduled when the load actually completes.

**Replay storm:** If one load miss triggers a chain of replays (C1 replays, then its consumer C2 had already started on speculative wakeup from C1, must also replay, etc.), the same instruction can execute many times. Pentium 4 suffered badly from this; subsequent cores bounded it by disabling speculative wakeup for "far dependents" of loads.

**Design trade-off:** More aggressive speculative wakeup → higher peak IPC on hits, higher replay cost on misses. Modern cores tune this based on load hit-rate prediction — a small structure that learns which loads habitually miss and suppresses speculative wakeup for them.

### Q9. Explain register file banking and how it affects wide issue.

**Answer:**

The physical register file in a wide OoO core needs many read and write ports. A naive single-bank design with, say, 12 read ports is nearly impossible to close timing on — port count scales silicon area super-linearly, and the required wire routing becomes a nightmare.

**Banking:** Split the PRF into several smaller banks. Each bank has fewer ports (say, 3 read + 2 write). The total port count across all banks is still large, but each individual bank is small and fast.

**How banks are assigned:** Each physical register number maps to exactly one bank (e.g. by low-order bits). When an instruction is renamed, its destination's bank is determined by the free-list allocation. The scheduler must know which bank each source register lives in when selecting instructions to issue.

**Bank conflicts:** Two instructions issued in the same cycle with source operands in the same bank may compete for ports. If the bank has 3 read ports and 4 sources target it simultaneously, one instruction is delayed.

**Mitigations:**

1. **Free-list bank balancing:** Rename allocates physical registers from banks in round-robin, so that frequently-used values are evenly distributed.

2. **Bank-conflict detection at select:** The scheduler pre-checks bank availability when choosing which instructions to issue. In the event of a conflict, one instruction is held back.

3. **Operand bypass networks:** Much of the time, a consumer reads a value that was just produced in the last 1–2 cycles. Bypass networks forward values directly from the ALU output without touching the PRF, reducing port pressure.

4. **Duplication:** Some cores duplicate part of the PRF, giving the effect of more read ports at the cost of consistency (write ports must broadcast to all copies).

**Performance impact:** Bank conflicts are usually a small percentage of issue slots lost (1–3%) on typical code. The trade-off is worth it because the single-bank alternative would force a lower clock frequency or a narrower machine.

**Port count budget example:** A core with 4 ALU + 2 FPU + 2 LD + 1 ST pipelines, each needing up to 3 source reads, would naively need 3×(4+2+2+1) = 27 read ports on the integer/FP PRFs combined. Banked to 4 banks with 4 read ports each = 16 ports effective, accepting some bank conflicts. Duplicated register files (integer vs FP) further split the load.

### Q10. What is "memory renaming" and how does it relate to register renaming?

**Answer:**

**Memory renaming** is a speculative optimisation that treats a load as if it were reading from a register that a producer store wrote to — bypassing the load queue, cache, and store buffer altogether.

**Motivation:** Many stores are immediately followed by a load of the same address (stack spills and reloads, register save/restore around function calls). The canonical sequence:

```
ST  [rsp+8], rax        ; spill
...                     ; some other code
LD  rbx, [rsp+8]        ; reload
```

A naive OoO core treats these as independent memory ops. The load waits for address resolution, searches the store queue, forwards the data. Even with forwarding, the search and the serialisation with the store cost cycles.

**Memory renaming:** If the hardware recognises that this load's address matches the store's, it renames the load's destination directly to the **physical register that held the store's source** — no memory path at all.

**How it works:**

1. A history table (indexed by load PC) records which recent store (identified by PC + distance) tends to feed this load.
2. At dispatch, the load's PC is looked up; if a prediction exists, the load is renamed to point directly at the producer store's source physical register.
3. The load becomes a zero-cycle register copy, identical to move elimination.
4. A verification still happens in the background: when both addresses resolve, the hardware checks that they do indeed match. If not, the load is squashed and replayed as a normal memory access.

**Relationship to register renaming:** Both are speculative identity substitutions that exploit the fact that many operations are "moving a value from A to B". Register renaming does this for explicit register-to-register moves; memory renaming does it for memory round-trips through stack slots. Both save critical-path latency by eliminating the operation entirely at the rename stage.

**Performance:** On stack-heavy code (ABI register save/restore, spill-heavy inner loops), memory renaming can eliminate 5–10% of load ops entirely. It is present in some form on Intel (since Sandy Bridge, in limited form), Apple M-series, and high-end AMD Zen variants.

**Complication:** Memory renaming must correctly handle cache coherence. If another core writes to the stack slot (extremely unusual but possible), the renamed load would return stale data. The solution is to only memory-rename loads from **private stack regions** that are provably not shared — the same PC-based locality that makes the prediction accurate also makes the sharing rare.
