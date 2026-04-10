# Tomasulo and Register Renaming — Interview Questions

**Subject:** Computer Architecture
**Topic:** Dynamic Scheduling, Register Renaming, Reservation Stations, ROB
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. Why does an in-order pipeline run out of performance, and what does out-of-order execution add?

**Answer:**

An in-order pipeline stalls on the first true data dependency whose producer is not yet ready. Consider:

```
LD   r1, 0(r2)     ; cache miss — 200 cycles
ADD  r3, r1, r4    ; waits 200 cycles
XOR  r5, r6, r7    ; independent — also waits 200 cycles
```

The `XOR` is structurally ready in cycle 2 but must wait until the `ADD` clears the pipe. The in-order pipe serialises work that is logically independent.

**Out-of-order (OoO) execution** relaxes the requirement that instructions execute in program order. Instead, an instruction executes as soon as its *operands* are ready and a functional unit is free. The `XOR` above can launch immediately, hiding most of the load's latency.

OoO adds three structures beyond the in-order pipeline:

1. **Instruction window** — a pool of decoded, not-yet-dispatched instructions (reservation stations or a unified issue queue).
2. **Register renaming** — breaks false (WAR, WAW) dependencies so the window can actually be reordered.
3. **Reorder buffer (ROB)** — tracks the original program order so that commit and exceptions remain precise.

The size of these structures sets the **memory-level parallelism** the core can extract: a 200-cycle miss can only be hidden if 200 cycles' worth of independent instructions fit in the window.

### Q2. What is register renaming and which dependencies does it eliminate?

**Answer:**

**Register renaming** is the translation of architectural register names (the 32 or 16 ISA registers) into a larger pool of **physical registers**. Every write to an architectural register allocates a fresh physical register. Reads are redirected to whatever physical register most recently was assigned to that architectural name (at the time of the read).

**What it eliminates:**

- **WAR (write-after-read, anti-dependency):** `ADD r1, r2, r3; ADD r2, r4, r5`. The second instruction writes `r2`, but the first needs to *read* `r2` first. Renaming gives the second `ADD` a new physical register for its result; the old `r2` is still alive for the first `ADD` to read. The second can now execute out-of-order without waiting for the first.

- **WAW (write-after-write, output dependency):** `ADD r1, r2, r3; ADD r1, r4, r5`. Both write `r1`. With renaming, each gets its own physical register; only the later one maps to the architectural `r1` at commit time.

Renaming does **not** eliminate **RAW** (read-after-write) — the true dependency where the second instruction legitimately needs the first's result. No amount of renaming can fix that.

**Why this matters:** Almost all false dependencies in real code come from register allocation pressure — the compiler re-uses the same architectural register for unrelated values in different parts of the function. Before renaming, these artificial dependencies serialised the pipeline. Renaming makes them vanish, typically exposing 3–5× more ILP.

### Q3. What is a reservation station and how does it differ from a unified issue queue?

**Answer:**

A **reservation station (RS)** is a small holding area associated with a functional unit (ALU, FPU, load/store). When an instruction is dispatched, it is placed in the RS of the appropriate unit. The RS entry carries the opcode, tags of the source operands (if not yet ready) or their values (if ready), and a destination tag. When all operands become ready, the instruction is *issued* to the functional unit and executes.

This is the original Tomasulo design: **distributed reservation stations**, one group per functional unit.

A **unified issue queue** (also called a scheduler or centralised RS) is a single pool that holds all in-flight instructions, regardless of which functional unit they target. When issued, the scheduler routes each instruction to an available unit of the right type.

**Trade-offs:**

| Property | Distributed RS | Unified issue queue |
|---|---|---|
| Utilisation | Can starve one type while another is full | Flexible — uses the whole pool |
| Wakeup/select logic | Smaller per-RS, parallel | One large CAM |
| Wire length | Local to functional unit | Global to all units |
| Scaling | Adding a unit adds an RS | Adding a unit widens issue logic |

Early Tomasulo designs used distributed RSs because CAM sizes were small. Modern wide OoO cores (Skylake, Apple M-series) use large unified issue queues (~100 entries), trading power for utilisation. The wakeup/select loop — broadcasting a completing tag to all RS entries and picking which to issue next — is on the **critical path** of the whole machine; designs often split the unified queue into banks to meet timing.

---

## Intermediate

### Q4. Describe the flow of an instruction through a modern OoO pipeline.

**Answer:**

A typical modern OoO pipeline has these stages (grouped for clarity):

**Frontend:**
1. **Fetch** — read instructions from I-cache, consult branch predictor and BTB, push bytes into a fetch queue.
2. **Decode** — parse bytes into decoded micro-ops (x86) or whole instructions (ARM, RISC-V).
3. **Rename** — look up each source architectural register in the **register alias table (RAT)** to find the current physical register; allocate a fresh physical register for each destination and update the RAT.
4. **Dispatch** — allocate a ROB entry (in-order), an issue queue entry, and (for loads/stores) load-queue / store-queue entries. If any resource is exhausted, the frontend stalls.

**Backend:**
5. **Issue** — the scheduler wakes up instructions whose operands are ready and selects up to *N* per cycle for execution.
6. **Register read** — read source values from the physical register file (in some designs the values travel with the instruction from the RS).
7. **Execute** — compute the result in the functional unit.
8. **Writeback** — write the result to the physical register file; broadcast completion tag to wake dependents in the issue queue.
9. **Commit (retire)** — in program order, take the oldest ROB entry, update the architectural RAT, free the old physical register. For stores, move the entry from the store buffer into the cache.

**Key invariants:**

- **In-order at rename and at commit; out-of-order in between.**
- **The ROB entries are allocated in program order** — this is how commit order is recovered.
- **Precise exceptions** are maintained because the ROB commits in-order; an exception is deferred until its instruction reaches the head of the ROB, at which point all younger speculative work is squashed.

### Q5. What is the reorder buffer (ROB) and what information does each entry hold?

**Answer:**

The **reorder buffer** is a FIFO that tracks all in-flight instructions, allocated in fetch/rename order and freed in commit order. Its purpose is to provide a **program-order view** of the out-of-order pipeline so that commit and exception handling remain precise.

**Typical ROB entry fields:**

1. **Instruction PC** — needed for exception handling (the PC to save).
2. **Destination architectural register** — needed at commit to update the architectural RAT.
3. **Old physical register mapping** — needed at commit to free the previous physical register that this instruction shadowed.
4. **Completion bit** — set when the instruction has finished executing.
5. **Exception bit and code** — set if the instruction raised a fault; deferred to commit.
6. **Store / load queue pointer** — cross-link to the memory ordering structures.
7. **Branch mask** — tracks which speculative branch(es) this instruction is speculating past, for selective squash on misprediction.

**Sizing:** The ROB size bounds how many instructions can be in flight simultaneously. At 4 IPC with a 200-cycle L3 miss, you need ~800 ROB entries to fully hide the miss. Real designs are smaller (Intel Golden Cove: 512; Apple Firestorm: ~630) because the wakeup/select loop and rename scaling hit power walls before that.

**Commit rule:** At each cycle, the oldest ROB entries (up to the commit width, typically 4–8) are examined. Those that are marked complete and carry no exception are retired in order. On exception, the ROB is drained at the faulting entry and all younger entries are squashed.

### Q6. Walk through a Tomasulo-style cycle-by-cycle trace.

**Answer:**

Consider this code fragment running on a machine with one adder (latency 2), one multiplier (latency 4), one RS per unit (2 entries each), and register renaming:

```
I1:  MUL  F1, F2, F3
I2:  ADD  F4, F1, F5
I3:  MUL  F6, F5, F7
I4:  ADD  F8, F6, F4
```

Dependencies:
- I2 depends on I1 (RAW via F1)
- I4 depends on I2 and I3 (RAW via F4 and F6)

**Rename** (done in order, one per cycle for simplicity):
- I1: `F1 → P10`, reads F2=P3, F3=P4
- I2: `F4 → P11`, reads F1=P10 (not ready), F5=P6
- I3: `F6 → P12`, reads F5=P6, F7=P8
- I4: `F8 → P13`, reads F6=P12 (not ready), F4=P11 (not ready)

**Cycle-by-cycle trace (assume dispatch = 1 per cycle, issue = 1 per unit per cycle):**

| Cycle | Event |
|---|---|
| 1 | I1 fetched, decoded, renamed, dispatched to MUL RS. Both operands (F2, F3) ready. |
| 2 | I1 issues to MUL unit. I2 dispatched to ADD RS; F1 not ready, waits. |
| 3 | I1 executing (cycle 1/4). I3 dispatched to MUL RS; F5, F7 ready; but MUL unit busy, waits. |
| 4 | I1 executing (cycle 2/4). I4 dispatched to ADD RS; both F6 and F4 not ready, waits. |
| 5 | I1 executing (cycle 3/4). |
| 6 | I1 completes. Writeback broadcasts tag P10. I2 in ADD RS wakes up. I3 can still not issue — MUL just finished but the issue slot is used by I1's writeback. |
| 7 | I2 issues to ADD unit. I3 issues to MUL unit. |
| 8 | I2 executing (cycle 1/2). I3 executing (cycle 1/4). |
| 9 | I2 completes. Writeback broadcasts tag P11. I4 still waits (needs both F4 and F6). |
| 10 | I3 executing (cycle 3/4). |
| 11 | I3 completes. Writeback broadcasts tag P12. I4 wakes up. |
| 12 | I4 issues to ADD unit. |
| 13 | I4 executing. |
| 14 | I4 completes. |

**Critical path:** I1 → I2 → I4 = 4 + 2 + 2 = 8 cycles of useful execution, plus one rename/dispatch cycle per instruction. The total of 14 cycles reflects the dispatch serialisation and single-issue constraint.

**Key observations:**
- I3 runs in parallel with I2 — both are ready simultaneously (cycle 7) because I3 does not depend on I1 or I2.
- Without renaming, I3's `F5` write would have been serialised against I2's `F5` read even though there was no true dependency — WAR hazard eliminated.
- Increasing issue width or adding a second multiplier would shorten the trace only up to the true-dependency limit (8 cycles here).

This is the essence of dynamic scheduling: the hardware finds all the parallelism the true dataflow graph permits.

---

## Advanced

### Q7. Describe the physical register file (PRF) rename scheme vs the "reservation-station holds values" scheme.

**Answer:**

There are two distinct microarchitectural styles for implementing register renaming:

**Classic Tomasulo (value-in-RS):** The reservation station entry holds the *value* of each source operand, or a **tag** indicating it is still being computed. When a functional unit completes, it broadcasts (tag, value) on a **common data bus (CDB)**. Every RS entry with a matching tag captures the value into its slot. When all slots are values (no tags), the entry can issue. The register file only holds architectural state.

**Physical register file (PRF-based):** There is a single large physical register file with more entries than the architectural register count (e.g. 200+ on a modern core). Rename assigns a physical register number to each destination; the RS / issue queue only holds *tags* (physical register numbers), never values. When a producer writes the PRF, the scheduler wakes up consumers; they then read the PRF (either in a dedicated register-read stage or as operand forwarding) and execute.

**Trade-offs:**

| Property | Value-in-RS | PRF-based |
|---|---|---|
| RS area | Large (holds 64-bit values) | Small (holds tags only) |
| Number of register reads | All in the RS — implicit | Explicit reads before execute |
| Broadcast network | CDB broadcasts (tag, value) — wide wires | Broadcasts (tag) only, values via PRF |
| Scalability | Poor — RS becomes huge for wide machines | Good — tags stay compact |
| Power | Broadcasts large values | Broadcasts only tags |

**Why modern designs use PRF:** Value-in-RS scales badly. A 128-entry unified issue queue holding two 64-bit source values per entry is 128×128 bits = 2 KB of latches, and every CDB broadcast drives 128 entries with 64-bit data — unacceptable for wire delay and power. PRF-based rename moves the value storage to one centralised, banked register file (efficient) and keeps the issue queue compact (fast). All modern high-performance cores (x86, ARM Cortex A-class, Apple M-series, Zen) use PRF-based rename.

The term "Tomasulo" is still used loosely to describe PRF-based OoO, but strictly speaking Tomasulo's original 1967 paper describes the value-in-RS scheme; PRF-based rename was pioneered by the MIPS R10000 in 1996.

### Q8. What is a "register file read port" and why does it constrain OoO issue width?

**Answer:**

A **register file read port** is a path from the register file to a functional unit that can read one register per cycle. An N-wide OoO issue stage that can launch two-operand ALU instructions needs **2N read ports** on the register file in the worst case.

This is **not** cheap. A register file is a 2D array of flip-flops or latches with one address decoder per port. Read ports add silicon area **quadratically** (roughly): each port needs its own wordline driver, pre-charge circuitry, and bitline sensing; and the wires between ports dominate. A 200-entry PRF with 10 read ports is a significant fraction of the core area.

**Consequences:**

1. **Issue width is bounded by register file bandwidth.** An 8-wide machine might need 16 read ports — impractical. Designs typically pick 4 ALU + 2 FPU × 2 operands = ~12 ports, then bank the PRF to cheat.

2. **Banking.** Split the PRF into several banks (say, 4 banks of 50 entries each), each with fewer ports. Rename assigns physical registers to banks to balance utilisation. The scheduler must be **bank-aware**: it cannot issue two instructions in the same cycle if both need reads from the same bank's full port count. When conflicts occur, one instruction is delayed by a cycle.

3. **Operand forwarding reduces pressure.** Most consumer instructions need values that were just produced in the previous cycle or two. Bypassing those directly from the ALU output — without ever going through the PRF — eliminates a read port access. Modern designs track which values are "hot" (recently produced) and prefer bypass over PRF read.

4. **Read ports are in the clock-period budget.** The path from "schedule picks instruction" → "operand bank selected" → "wordline asserted" → "data on bitline" → "into ALU input" is often the longest combinational path in the core. Cutting it with an extra pipeline stage (register-read stage) widens the issue/execute gap and hurts replay cost.

The register file is therefore one of the classic **fundamental scaling bottlenecks** of wide OoO. It is why the shift to specialised accelerators (GPUs, tensor cores) wins at width: they trade general programmability for a structured dataflow that fits a simpler register model.

### Q9. Explain "move elimination" and "zero idiom" optimisations in modern x86 renamers.

**Answer:**

These are renamer-level tricks that eliminate common instructions before they ever reach the execution units.

**Move elimination** (also called "zero-latency mov"): A `MOV rax, rbx` (register-to-register copy) is semantically "make `rax` hold the same value as `rbx`". In a PRF rename scheme, this can be achieved by simply pointing the RAT entry for `rax` to the *same physical register* as `rbx`. No execute-stage work is required — the rename stage is enough.

**Implementation details:**
- The old physical register mapped to `rax` is freed (or is pushed to the free list at commit).
- Both `rax` and `rbx` now point to the same physical register. A subsequent write to either allocates a new one, re-establishing independence (this is copy-on-write at the register level).
- Reference-counting or a "multi-owner" bit in the PRF is needed to prevent premature free.

Intel Ivy Bridge (2012) introduced this; AMD Zen (2017) followed. Well-optimised code (including memcpy loops, ABI glue, register clobber prefixes) becomes materially faster.

**Zero idiom:** `XOR rax, rax` is the canonical way to zero a register on x86 (smaller encoding than `MOV rax, 0`). The renamer recognises this specific idiom and treats it as "produce the constant 0 in a fresh physical register" with no dependency on the source `rax`. The benefits are:

1. **Breaks false dependencies.** A naïve implementation would treat `XOR rax, rax` as reading `rax` — creating a false dependency on whoever wrote `rax` last. Recognising the idiom makes the instruction truly independent, so it can issue immediately.

2. **Zero latency.** Similar to move elimination, the renamer can allocate a physical register already tagged as holding 0 — no need for the ALU to compute it. Intel, AMD, and Apple all do this.

3. **SIMD version:** `PXOR xmm0, xmm0`, `VPXOR ymm0, ymm0`, and `VPXOR zmm0, zmm0` are all recognised. The last one is important because AVX-512 zmm clobbers are expensive; idiom recognition avoids the physical port.

**Interview insight:** A candidate who understands these optimisations understands that the renamer is not a mechanical lookup — it is a small, fast program that pattern-matches the instruction stream and elides common patterns. The same machinery supports **sign-extension idioms**, **stack adjustment idioms**, and (on some cores) **return-address prediction corrections**.

### Q10. What is a "scheduler replay" and how does it differ from a branch mispredict flush?

**Answer:**

A **scheduler replay** (often just "replay") is the re-execution of an instruction and its dependents because the scheduler's assumption about timing turned out to be wrong — *without* a branch misprediction.

The classic case: the scheduler predicts a load will hit L1 (4-cycle latency) and wakes up the load's dependents four cycles after the load issues. If the load actually misses in L1, its result is not ready when the dependents execute. The dependents' wakeup has already happened and they have consumed stale or zero operands. The scheduler detects this (by comparing the load's actual latency to the scheduled timing) and **replays** the affected instructions.

**Difference from branch mispredict flush:**

| Property | Replay | Flush |
|---|---|---|
| Cause | Latency misprediction, memory disambiguation | Branch / target misprediction, fault |
| Scope | Only dependents of the mispredicted event | All instructions after the squashed point |
| Frontend | Unaffected — no refetch | Full refetch from corrected PC |
| Cost | One extra round through the scheduler | Full pipeline depth |
| Observable | Usually only as extra µops issued | Large IPC drop, counters show mispredict |

**Pentium 4's replay disaster:** Pentium 4 used a "replay queue" that could cascade replays indefinitely. A load-use chain could enter a "replay storm" where the same instruction executed dozens of times before the memory hierarchy caught up. The effective latency of a cache miss became enormous on some workloads. Subsequent Intel designs (Core 2, Sandy Bridge) bound the replay chain depth and made replay much rarer.

**Why replay exists at all:** Without speculative scheduling, every load would freeze its dependents for the worst-case memory latency. That would destroy ILP on hot code. Speculative scheduling assumes the best case and accepts replay as the cost of being wrong occasionally. On modern cores with 95%+ L1 hit rates, the gamble pays off on balance, but memory-bound workloads can still suffer — one reason why prefetching and cache-aware coding matter so much in practice.
