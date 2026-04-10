# Classic Pipeline and Hazards — Interview Questions

**Subject:** Computer Architecture
**Topic:** Five-Stage Pipeline, Structural/Data/Control Hazards, Forwarding, Stalls
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. Describe the classic five-stage MIPS pipeline and what each stage does.

**Answer:**

The five stages are **IF, ID, EX, MEM, WB**:

| Stage | Name | Work performed |
|---|---|---|
| IF | Instruction Fetch | Read instruction from I-cache at `PC`; compute `PC+4` |
| ID | Instruction Decode | Decode opcode; read register file; sign-extend immediate |
| EX | Execute | ALU operation; compute branch target; compute effective address for loads/stores |
| MEM | Memory Access | Access D-cache for loads and stores; non-memory instructions pass through |
| WB | Write Back | Write result to register file |

Each stage is separated by a **pipeline register** that latches the stage's outputs at the end of the clock cycle. In steady state, five instructions are in flight simultaneously — one per stage — and the pipeline completes one instruction per cycle (**CPI = 1**), though the *latency* from IF to WB for any single instruction is still five cycles.

The key insight is that **throughput is decoupled from latency**. Clock frequency is set by the slowest stage (typically MEM), not by the sum of all stage delays.

### Q2. What are the three classes of pipeline hazard?

**Answer:**

1. **Structural hazard** — two instructions need the same hardware resource in the same cycle. Example: a single-ported memory used for both instruction fetch and data access would cause IF and MEM to collide. Solved by duplicating the resource (separate I-cache and D-cache, i.e. a Harvard architecture at the cache level).

2. **Data hazard** — an instruction depends on a result that has not yet been written back by an earlier instruction. Three sub-types:
   - **RAW (Read After Write)** — true dependency. `ADD r1,r2,r3` followed by `SUB r4,r1,r5`. Most common; the only one that exists in an in-order pipeline, because writes to a register happen in program order.
   - **WAR (Write After Read)** — anti-dependency. Cannot occur in the classic 5-stage pipeline because reads happen in ID (early) and writes in WB (late), both in program order.
   - **WAW (Write After Write)** — output dependency. Also impossible in a simple in-order pipeline for the same reason.

3. **Control hazard** — the next PC is not known. On a branch, IF must fetch something in the cycle after the branch enters IF, but the branch outcome is not known until EX (one stage later if you dedicate a comparator to ID). Without prediction this costs one or more bubble cycles on every taken branch.

WAR and WAW hazards only become relevant in out-of-order machines that execute instructions in a different order than the program specifies.

### Q3. What is forwarding (bypassing) and how does it eliminate most RAW hazards?

**Answer:**

**Forwarding** is the practice of routing a result directly from where it is produced to where it is consumed, bypassing the register file. Without forwarding, a value written back in WB can only be read one cycle later in ID; the dependent instruction must stall.

Consider:

```
I1:  ADD  r1, r2, r3     ; r1 produced at end of EX (cycle 3)
I2:  SUB  r4, r1, r5     ; r1 needed at start of EX (cycle 4)
```

Timeline (cycles 1–5):

```
         1    2    3    4    5
I1:     IF   ID   EX   MEM  WB
I2:          IF   ID   EX   MEM  WB
```

`I1` has the ALU result at the end of cycle 3. `I2` needs it at the start of cycle 4. **EX-to-EX forwarding** routes the output latch of EX directly to the input mux of EX in the next cycle. No stall.

Without forwarding, `r1` would only be available after WB (end of cycle 5), so `I2` would have to wait until cycle 6 for its read in ID — a **two-cycle bubble**.

**Forwarding paths commonly implemented:**
- **EX → EX** (ALU result to next ALU op)
- **MEM → EX** (load result to subsequent use, one cycle later)
- **MEM → MEM** (load-to-store data, enabling `LD`/`ST` sequences without stall)

The inputs to the ALU thus go through a **forwarding mux** selecting from: register-file read, EX/MEM latch, MEM/WB latch, or WB value being written this cycle.

### Q4. What is the load-use hazard and why can it not be fully forwarded away?

**Answer:**

A **load-use hazard** occurs when an instruction uses a register that the immediately preceding instruction has loaded from memory:

```
I1:  LD   r1, 0(r2)      ; r1 valid at end of MEM (cycle 4)
I2:  ADD  r3, r1, r4     ; r1 needed at start of EX (cycle 4)
```

Timeline:

```
         1    2    3    4    5
I1:     IF   ID   EX   MEM  WB
I2:          IF   ID   EX   MEM  WB
                         ^
                         r1 needed here, but MEM of I1
                         is completing in the same cycle
```

The load result only emerges from the MEM stage at the *end* of cycle 4, but the dependent ADD needs it at the *start* of cycle 4. Forwarding cannot reach backward in time, so the pipeline must insert **one bubble** (stall I2 for one cycle):

```
         1    2    3    4    5    6
I1:     IF   ID   EX   MEM  WB
I2:          IF   ID   --   EX   MEM  WB
```

Now MEM-to-EX forwarding (cycle 4 → cycle 5) delivers the value. This one-cycle stall is called the **load-delay slot** and is the single most common stall cause on in-order RISC pipelines.

**Compiler mitigation:** the instruction scheduler tries to place an independent instruction between the load and its first use, filling the load-delay slot with useful work. On heavily memory-bound code this is rarely possible, which motivates out-of-order execution.

---

## Intermediate

### Q5. What is the branch penalty in the classic 5-stage pipeline, and how is it reduced?

**Answer:**

In the textbook pipeline, a branch is decoded in ID and resolved in EX (the ALU computes the condition and target). By the time the branch outcome is known at the end of cycle 3, two instructions have already entered IF and ID — both wrong-path if the branch is taken. The **branch penalty** is therefore 2 cycles if predicted "not taken" and the branch is actually taken.

**Mitigations, in order of increasing complexity:**

1. **Early branch resolution** — move the branch comparator and target adder into ID. Now the outcome is known at the end of cycle 2, leaving only one wrong-path instruction in IF. Cost: a dedicated comparator in ID, plus extra forwarding (since the source register might still be in the pipe). Penalty: 1 cycle.

2. **Predict not taken** — always fetch the fall-through. Correct for forward conditional branches about 50% of the time; correct for backward branches (loops) only on loop exit. Penalty: 1 cycle on mispredict.

3. **Predict taken with a BTB** — a **Branch Target Buffer** caches the PC and target of recently-taken branches. On a hit, the fetch engine immediately redirects to the cached target. Penalty: 0 cycles on a correct prediction; 1+ cycles on a mispredict.

4. **Dynamic prediction** — bimodal, gshare, TAGE predictors achieve 95%+ accuracy on typical code. This is essential for deep modern pipelines (15–20 stages) where a misprediction can cost 15+ cycles.

5. **Delayed branch** — historical workaround (see the ISA Design Principles document). Exposes the bubble as an architectural delay slot.

### Q6. What is the difference between "kill" (flush) and "stall" as pipeline control responses?

**Answer:**

- **Stall** (bubble insertion): freeze the upstream stages for one or more cycles while the downstream stages continue. The instruction that caused the hazard stays in place; a NOP is injected into the next stage. Used for data hazards where the correct value will be available shortly — no work is wasted.

- **Kill** (flush): convert one or more in-flight instructions into NOPs by squashing their pipeline register contents. Used when speculative work turns out to be incorrect — most commonly after a branch misprediction, and also after a trap or exception.

**Control difference:**

- Stalling needs a **hold signal** to the pipeline registers upstream of the hazard and an **enable** signal to inject NOPs downstream.
- Killing needs a **clear signal** to the pipeline registers holding the wrong-path instructions, plus a PC redirect in IF.

**Why the distinction matters for performance:** Stalls are bounded and predictable — you lose exactly the cycles you have to. Flushes cost the entire pipeline depth from fetch to the point of flush, so they scale linearly with pipeline length. This is the fundamental reason deeply pipelined processors invest so heavily in branch prediction: the cost of a flush is catastrophic.

### Q7. Describe how precise exceptions are maintained in an in-order pipeline.

**Answer:**

A **precise exception** requires that:

1. The PC saved to the exception handler points to exactly the faulting instruction.
2. All instructions before the faulting one have committed their results.
3. No instruction after the faulting one has modified architectural state.

In an in-order pipeline, achieving this is straightforward because instructions enter WB in program order. The rules are:

1. **Detect exceptions at the latest possible stage that is still in-order.** Most exceptions are detected in EX (arithmetic overflow, divide by zero, invalid opcode after decode) or MEM (page fault, alignment, permission). These are all before WB.

2. **Do not write back on an exception.** When stage *k* detects an exception, the pipeline register feeding WB is cleared for that instruction and all younger ones. Architectural state remains as though the faulting instruction had never entered WB.

3. **Save the faulting PC.** The PC is carried as part of the pipeline register through every stage (it must be anyway, for branches to compute targets). On exception, the current stage's PC is latched into the exception PC register (`EPC`).

4. **Flush younger instructions.** All stages upstream of the faulting instruction are converted to NOPs, because those instructions have not yet committed anything.

5. **Redirect IF to the exception vector.**

**Complication — late-detecting exceptions:** A TLB miss on a load is detected in MEM. By that time, a younger instruction is already in EX. If EX has no side effects other than its WB, killing it is fine. But what if that younger instruction is a store? In an in-order pipeline, the store's commit (to the store buffer or to memory) happens in MEM or later, by which point the load in front has already left MEM without issue. So the store's commit is strictly ordered after the load's exception detection. Good.

In an out-of-order pipeline, this becomes much harder and is solved by the reorder buffer — see `02_out_of_order_execution/speculation_and_recovery.md`.

---

## Advanced

### Q8. A pipeline has a hit rate of 98% in the branch predictor, a misprediction penalty of 12 cycles, 18% branches in the instruction mix, and an otherwise ideal CPI of 1.0. What is the effective CPI?

**Answer:**

The effective CPI accounts for the average penalty contributed by mispredictions:

$$\text{CPI}_{\text{eff}} = \text{CPI}_{\text{ideal}} + (\text{branch fraction}) \times (\text{mispredict rate}) \times (\text{penalty})$$

Substituting:

$$\text{CPI}_{\text{eff}} = 1.0 + 0.18 \times 0.02 \times 12 = 1.0 + 0.0432 \approx 1.04$$

So a 98% predictor loses only 4.3% of peak throughput on this workload.

**Sensitivity analysis** (the part that wins the interview):

- Dropping to 95% accuracy: $1.0 + 0.18 \times 0.05 \times 12 = 1.108$, or a **11% slowdown**.
- Dropping to 90%: $1.0 + 0.18 \times 0.10 \times 12 = 1.216$, or **22% slowdown**.

Prediction accuracy is **hyper-linear** in performance impact when the pipeline is long. This is why a 2-point accuracy improvement on the predictor is worth more silicon than almost any other single frontend change.

Conversely, if you halve the misprediction penalty (shorter pipeline or faster branch resolution) from 12 to 6, even at 95% accuracy you get $1.0 + 0.054 = 1.054$, a 5% slowdown — better than a 98% predictor on the deep pipe. The two knobs trade against each other.

### Q9. Explain "commit" vs "complete" vs "retire" in a pipeline that supports precise exceptions.

**Answer:**

These terms are often used loosely, but architects distinguish them carefully:

- **Complete** (also "finish"): the instruction has produced its result. In an in-order pipeline, this happens at the end of EX (or MEM for loads). The result exists but has not yet been written to architectural state.

- **Commit** (also "writeback" in in-order machines, or "retire" in out-of-order machines): the instruction's result is written to architectural state — the register file or the memory system. After commit, the instruction is architecturally visible and cannot be undone.

- **Retire**: in out-of-order designs, synonymous with commit. The instruction is removed from the reorder buffer and its physical register is marked free for reuse.

**Why the distinction matters:**

In an in-order 5-stage pipeline, complete happens at EX and commit happens at WB — separated by one stage. The pipeline register between them holds the "complete but not yet committed" value.

In an out-of-order pipeline, instructions can **complete in any order** (a fast integer op finishes in 1 cycle; a cache-missing load takes 200) but they must **commit in program order** to preserve precise exceptions. The gap between completion and commit can span hundreds of cycles, and the reorder buffer holds all the in-flight results during that window.

An instruction that has completed but not committed is **speculative** from the architectural point of view — even if the branch that led to it was correctly predicted, it might still be killed by an exception in an older in-flight instruction. Stores illustrate this clearly: a store that has "completed" (address and data computed) is held in the store buffer until every older instruction has committed, and only then does it become globally visible.

### Q10. What is a "replay" and why is it used in modern pipelines?

**Answer:**

A **replay** is the re-execution of an instruction inside the pipeline because a speculative assumption about it proved wrong, even though no branch was mispredicted. The instruction is not flushed and refetched from I-cache; it is re-issued from within the pipeline.

**Why this exists:** Modern wide out-of-order pipelines schedule instructions based on *predicted* latencies, not known ones. For example, the scheduler assumes a load hits the L1 cache (4-cycle latency) and wakes up dependent instructions on that basis. If the load actually misses in L1, its result is not ready when the dependent instruction enters EX. Rather than flushing the whole pipeline (expensive) or stalling all instructions waiting for the load (loses ILP on independent work), the scheduler **replays** only the affected dependents.

**Common replay causes:**

1. **Load latency mispredict** — L1 miss when the scheduler assumed a hit.
2. **Memory disambiguation failure** — a load was speculatively executed ahead of an older store, and the store turns out to alias. The load must replay after the store commits.
3. **TLB miss retry** — load proceeds speculatively assuming TLB hit; on miss, the walker runs, then the load replays.
4. **Cache port conflict** — two loads issued to the same bank in the same cycle; one replays.

**Cost model:** A replay is cheaper than a flush because it touches only the dependent chain of the affected instruction. In the best case only one dependent replays; in the worst case ("replay storm"), a long chain cascades and performance collapses. Intel Sandy Bridge reworked its scheduler specifically to bound replay storm depth after Pentium 4 suffered badly on memory-bound workloads.

**Interview insight:** The presence of replays is why in real processors, the advertised "load-use latency" is actually a minimum. Under load-latency misprediction, dependents pay the full memory hierarchy cost — which the performance counters report as a load miss, not a scheduling event, making the underlying cause hard to diagnose without microarchitectural knowledge.
