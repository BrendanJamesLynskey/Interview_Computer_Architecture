# Speculation and Recovery — Interview Questions

**Subject:** Computer Architecture
**Topic:** Speculative Execution, Precise Exceptions, Checkpointing, Flush Recovery
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What does "speculation" mean in a modern processor? What kinds of speculation are there?

**Answer:**

**Speculation** is the execution of instructions whose correctness has not yet been confirmed. The processor commits to a prediction, runs with it, and rolls back if wrong.

**Kinds of speculation, ordered by how speculative they are:**

1. **Branch speculation** — fetch and execute past a branch before its direction is known. Cleaned up by flushing on a mispredict.

2. **Value/address speculation** — execute a load before older stores have resolved their addresses (memory disambiguation). Requires a memory-ordering check when the store address resolves; replay the load if it aliased.

3. **Load-hit speculation** — wake up dependents of a load assuming L1 hit; replay on miss.

4. **Way prediction** — predict which cache way to read on a set-associative cache, saving a tag comparison cycle; replay if wrong.

5. **Indirect target speculation** — predict the target of an indirect branch or return; flush on mispredict.

6. **Memory renaming / store-to-load forwarding prediction** — predict whether a load will match an older store's address, forwarding the store's data directly.

7. **Silent store elimination** — skip a store whose value is predicted unchanged.

**Common property:** All speculation turns "wait and be correct" into "guess and be right 95%+ of the time, recover cheaply otherwise". Performance is dominated by prediction accuracy and recovery cost.

### Q2. What is a precise exception, and why must modern processors preserve them?

**Answer:**

A **precise exception** is one for which, at the moment the handler is invoked:

1. The saved PC points exactly to the faulting instruction (or to the instruction immediately after, depending on fault class).
2. All instructions *before* the faulting instruction have committed their results to architectural state.
3. No instruction *after* the faulting instruction has modified architectural state.

From the handler's perspective, the machine looks exactly as if execution had halted cleanly at the faulting instruction.

**Why we need precise exceptions:**

- **Software correctness depends on it.** Page-fault handlers, for instance, must be able to resolve the faulting page and restart the faulting instruction. If an older instruction had not yet committed, restarting the load would see an inconsistent architectural state.

- **Debugger usability.** A breakpoint or single-step must deliver control at a specific instruction with the machine in a well-defined state. Imprecise exceptions would make source-level debugging impossible.

- **Multiprogramming.** A context switch triggered by a timer interrupt must capture an instantaneously-consistent architectural snapshot. Without precise exceptions, the snapshot is ambiguous.

Early out-of-order designs (CDC 6600, IBM 360/91) had *imprecise* exceptions, and they were a maintenance nightmare. Ever since the MIPS R10000 and IBM POWER2, precise exceptions on all faults — including floating-point — have been a baseline requirement.

### Q3. How does the reorder buffer enable precise exceptions?

**Answer:**

The ROB preserves program order *at commit time*. Instructions execute out of order but retire in order. When an instruction reaches the head of the ROB:

- If it completed successfully, its result becomes architectural state (RAT update, store commit).
- If it raised an exception, the ROB's head-of-queue handling kicks in:
  1. Squash all younger ROB entries (they cannot commit — they are architecturally "after" the fault).
  2. Drop the faulting instruction without writing architectural state.
  3. Save the faulting PC to the exception-PC register.
  4. Flush the frontend and redirect fetch to the exception vector.

Crucially, the exception is **deferred until commit**. An instruction may have detected a fault 200 cycles earlier at execute time, but the fault stays latched in the ROB entry and is not reported until it reaches the head. This is what makes exceptions precise: the fault is delivered at exactly the program-order point the ISA promises, even though the underlying execution happened long before.

**Subtlety:** If a younger instruction *also* raised an exception but the older instruction is the one that reports, we must not lose the younger exception — actually we may, because a precise-exception guarantee is that only the *oldest* unresolved fault is reported. Once the older fault is handled and execution restarts, the younger instruction re-executes and may (or may not) re-raise.

---

## Intermediate

### Q4. Describe the recovery actions on a branch mispredict.

**Answer:**

When a branch executes in the backend and its direction or target turns out to disagree with the frontend's prediction, the pipeline must undo all wrong-path work. The recovery actions:

1. **Detect:** The branch unit compares actual outcome to predicted outcome. If mismatch, signal "mispredict" with the correct target.

2. **Notify the frontend:** Send the correct target PC to fetch. Fetch immediately begins reading from the new path. The predictor's state (BTB, direction table) may be updated *now* or at commit, depending on the design.

3. **Squash wrong-path instructions:** Every ROB entry younger than the mispredicting branch is invalidated. This includes any in-flight execution, any reservation station entries, any store-buffer entries, and any load-queue entries.

4. **Restore rename state:** The RAT must be rolled back to the state it had *at the branch*. This is the hardest step and has two main implementations:
   - **Checkpoint-based:** At every branch, snapshot the full RAT into a checkpoint. On mispredict, restore from the nearest checkpoint before the mispredicting branch. Fast recovery, but checkpoints are expensive to store — limits how many branches can be in flight.
   - **ROB-walk (history-based):** At each ROB entry, record the "old physical register" that this instruction shadowed. On mispredict, walk the ROB backwards from the tail, undoing each rename step by step. Slow recovery (one cycle per ROB entry in the worst case), but cheap in storage.

5. **Free physical registers:** Any physical registers allocated to wrong-path instructions are returned to the free list.

6. **Restart backend:** Once rename state is restored, dispatch and execute resume with the correct path's instructions.

**Cost:** The mispredict penalty is the time from branch execute to the first correct-path instruction reaching execute — typically 12–20 cycles on modern wide OoO cores. Most of this is the frontend latency (fetch through rename) plus any rename-state restoration work.

### Q5. What is selective squash, and why is it better than full squash?

**Answer:**

**Full squash** invalidates all instructions younger than some point in the ROB — the naive recovery. **Selective squash** (also called "precise squash" or "branch-selective squash") invalidates only the instructions that depend, directly or transitively, on the bad prediction.

**Why selective squash matters:** Modern wide machines have 10+ branches in flight at once. Nested speculation means that by the time a branch mispredicts, the pipeline may be speculating 3–5 branches deeper. With full squash, the mispredict of an old branch would throw away independent work done past more-recent branches — even if those branches were correctly predicted.

**How it works:**

1. **Branch mask:** Each in-flight branch is assigned a unique bit position in a "branch ID" mask. Every dispatched instruction is tagged with the set of branch IDs that are "older but not yet resolved" at its dispatch — these are the branches it is speculating past.

2. **Mispredict:** When branch *B* mispredicts, the hardware sets a "kill mask" with bit *B* set. Every instruction whose branch mask has bit *B* set is squashed (it was speculating past *B*). Instructions whose branch mask does not have bit *B* set are independent of *B*'s outcome and survive.

3. **Branch bit release:** On correct resolution of a branch, its bit is cleared from all in-flight masks, making its ID available for reuse.

**Benefits:** On realistic code with high branch density, selective squash recovers 10–20% more throughput than full squash by preserving work that would otherwise be pointlessly re-executed.

**Cost:** Branch mask storage per ROB entry, extra comparator logic on squash. This limits how many branches can be outstanding simultaneously — typically 16 or 32 branch IDs in hardware. Exceeding this forces dispatch to stall until an older branch resolves.

### Q6. Why is speculation past stores particularly hard?

**Answer:**

Stores are different from other instructions because:

1. **They modify memory** — persistent, observable state. A speculative store cannot write the cache because another core could see the wrong value.

2. **Their address can alias other in-flight memory ops** — forwarding and ordering decisions depend on matching addresses, which may not yet be known.

3. **Cross-core ordering** is promised by the memory consistency model. Speculative stores must not leak out onto the bus even indirectly.

**Solution — the store buffer:** Stores are held in a FIFO store buffer after their address and data are computed, waiting for the corresponding ROB entry to commit. They reach the cache only on commit, in program order. Loads searching the store buffer can see and forward from in-flight stores by the same core (store-to-load forwarding), but no other core sees them.

**Memory disambiguation problem:** Consider:

```
ST  [R1], R2       ; store to unknown address
LD  R3, [R4]       ; load from unknown address
```

Can the load execute before the store? Only if we know `R1 != R4`. If we wait until both addresses are known, we serialise the load on every older store — killing ILP.

**Solution — speculative disambiguation:** The load executes speculatively assuming no alias. Its address is recorded in the load queue. When the store's address later becomes known, the hardware checks whether it matches any younger load that has already executed. If so, the load was wrong — it read stale data — and must be replayed. A memory dependence predictor learns which static load-store pairs tend to alias and gates speculation accordingly.

**Store-to-load forwarding:** A load whose address matches an older in-flight store in the store buffer can forward the store's data directly, bypassing the cache. This matches the memory model's ordering semantics and is much faster than waiting for the store to commit and then re-reading from L1. Partial-overlap cases (store to `[R1]` byte 0, load from `[R1]` bytes 0–3) complicate this and often force the load to wait.

The store buffer and load queue are among the most performance-critical structures in a modern core, and their sizing (typically 40–80 entries each) directly bounds memory-level parallelism.

---

## Advanced

### Q7. How do Meltdown and Spectre exploit speculative execution?

**Answer:**

Both attacks exploit the fact that **speculative execution leaves microarchitectural traces** even when architecturally squashed. The microarchitectural state (cache lines, TLB entries, branch predictor state) is not protected by the ISA's access-control checks, so it can be used as a side channel to leak architecturally-inaccessible information.

**Meltdown (CVE-2017-5754):** Targets the memory permission check in the speculative load path. On affected Intel CPUs, a load from a kernel address in user mode *would* eventually raise a page fault — but the fault check is enforced at commit, not at execute. In the speculative window before the fault fires, the load returns the kernel data, and a dependent operation can use that data to index into a user-controlled array:

```
mov  rax, [kernel_addr]      ; speculative read; value loaded into rax
and  rax, 0xff               ; isolate one byte
shl  rax, 12                 ; scale by page size (4 KB)
mov  rbx, [user_array + rax] ; touches one cache line out of 256
```

After the fault squashes, the attacker measures access time to `user_array[i*4096]` for each `i`; the fastest one is the byte that was read from kernel memory.

**Mitigation:** KPTI (Kernel Page Table Isolation) — unmap kernel pages from the user page table entirely, so the speculative load has no TLB entry and cannot even get as far as reading data. Expensive: context switches now require TLB flushes.

**Spectre v1 (bounds-check bypass):** Trains a branch predictor to predict the wrong way through a bounds check, then induces a wrong-path load that reads out-of-bounds. The wrong-path load is eventually squashed but leaves a cache trace.

```c
if (x < array1_len) {          // branch mispredicts "taken"
    y = array2[array1[x] * 256];
}
```

When the branch is mispredicted as taken with `x` out of bounds, the inner load reads `array1[x]` (garbage, including secret data) and uses it to index `array2`, leaving a cache-line trace.

**Mitigation:** `lfence` after bounds checks (serialising barrier that blocks speculation); compiler patterns like Speculative Load Hardening; data-flow-level fixes that mask the indirect load's index with the branch condition.

**Spectre v2 (branch target injection):** Trains the indirect-branch predictor from a user process to mispredict into attacker-chosen gadgets in the kernel, executing wrong-path instructions that leak data.

**Mitigation:** IBRS/IBPB/STIBP microcode barriers; retpoline (replace indirect jumps with a return-based construct the predictor cannot train on); eIBRS hardware fix in later Intel parts.

**Lesson for architects:** Speculation-driven side channels are not bugs in individual implementations — they are an inevitable consequence of the "speculate freely, clean up on squash" performance model. Defending against them has required new hardware mechanisms to constrain *which* microarchitectural effects speculative instructions may cause, and software mitigations that sprinkle barriers around sensitive boundaries.

### Q8. Describe checkpoint-based recovery and its scaling limitations.

**Answer:**

**Checkpoint-based recovery** saves a snapshot of the rename state (and other relevant structures) at each speculation point, so that on mispredict the machine can restore in O(1) time by copying the checkpoint back.

**What is checkpointed:**
- The rename alias table (RAT) — the architectural-to-physical mapping.
- The free list pointer (so freed physical registers from squashed instructions can be reclaimed).
- The loop-level GHR and branch predictor state (sometimes).
- The load/store queue head pointers.

**Advantages over ROB-walk recovery:**
- Recovery latency is one cycle (snapshot restore) regardless of how many instructions must be squashed.
- No sequential "undo" through the ROB entries.

**Scaling limitations:**

1. **Storage cost.** Each checkpoint is a full copy of the RAT. On a machine with 32 architectural registers and 8-bit physical register tags, a RAT snapshot is 256 bits. Checkpointing every in-flight branch (say 20) costs 20 × 256 = 5 Kb of storage, plus the muxing to restore it quickly. Wider registers and more architectural state (vector, predicate) scale this linearly.

2. **Checkpoint allocation.** The set of active checkpoints is a scarce resource. If the frontend encounters more branches than there are checkpoint slots, dispatch must stall. This caps the speculative depth — typically 16–32 branches deep, limiting ILP on branch-dense code.

3. **Interaction with precise exceptions.** A fault raised by an older instruction must logically restore to the state *at* the fault, not at the nearest checkpoint. If checkpoints are only at branches, fault recovery still needs some kind of ROB walk or fine-grained tracking.

**Hybrid approach:** Most modern cores use **coarse-grained checkpoints at branches plus fine-grained ROB walk for exceptions**. This keeps the checkpoint count small while handling the common case (branch mispredict) in O(1) and the rare case (exception) acceptably.

**Cache of recently-used checkpoints:** Some designs release checkpoint slots as branches resolve successfully, freeing them for reuse by younger branches. This allows an effectively unbounded number of branches in flight as long as the old ones resolve before the new ones allocate.

### Q9. What is "memory consistency" in a speculative OoO core, and how does it interact with the memory model?

**Answer:**

Memory consistency is the promise a processor makes about the order in which its memory operations become visible to other cores. In a speculative OoO core there is a tension:

- **OoO execution reorders everything** for ILP.
- **The memory model constrains what reorderings are observable.**

The rule is: **reorderings within a single core are allowed as long as they cannot be distinguished from in-order execution by any other core.** The hardware must enforce this via the load/store queues and commit discipline.

**TSO (x86) example:** TSO allows loads to pass older stores to different addresses, but disallows every other reordering. The microarchitecture can freely issue loads before older stores — but it must snoop the store buffer on each load to provide forwarding if they alias. It must also guarantee that store commits appear in program order and that loads retire in program order. The store buffer is effectively a private queue that hides cross-core store-order violations.

**Weak memory (ARM, RISC-V) example:** These models allow almost all reorderings. The hardware can speculatively execute and commit loads in any order. Software must use explicit fences (`DMB ISH`, `DMB SY`) to enforce ordering at synchronisation points.

**Speculative load issue problem:** Consider this across two cores in a TSO machine:

```
Core 0:          Core 1:
ST  [X], 1       ST  [Y], 1
LD  R1, [Y]      LD  R2, [X]
```

TSO forbids the outcome `R1 = R2 = 0`. But if each core speculatively executes its load *before* its older store, both cores can observe the old value of the other's variable, violating TSO.

**The fix:** Each core's load queue tracks in-flight speculative loads. If, after the load executes, the hardware observes any event that could make the load's result visible to another core in a way that contradicts TSO (e.g. an invalidation message arrives for the line before the load's store has committed), the load is replayed. This is called **load-load ordering replay** or **memory-ordering machine clear**.

On Intel and AMD, these replays show up in performance counters as `machine_clears.memory_ordering`, and they are a known hazard in highly-contended lock-free code.

**Weak-memory advantage:** Machines with weaker memory models pay nothing for this — they are allowed to produce the "forbidden" outcome. They only enforce ordering when software inserts fences. This is one reason ARM servers and mobile chips can achieve high single-thread performance at low power.

### Q10. Explain "load-load reordering" and a code pattern that breaks under it.

**Answer:**

**Load-load reordering** is when a processor speculatively executes (or commits) a younger load before an older load, even though the program issues them in a specific order. Most weak memory models allow this; TSO and SC do not.

**Why it happens:** OoO cores issue memory ops as soon as operands are ready. If the younger load's address is ready first (perhaps because the older load is waiting on a dependency), the hardware can issue the younger one and start a cache access, speculatively assuming the older load will not change the outcome.

**Classic breaking pattern — initialisation race:**

```c
// Producer
data = 42;                   // store 1
init_flag = 1;               // store 2

// Consumer
while (init_flag == 0) { }   // load 2
int x = data;                // load 1
```

The producer writes the data and then sets a flag. The consumer waits for the flag, then reads the data. Intuitively, after the flag is observed as 1, the data should also be observed as 42.

**What can go wrong under weak memory (e.g. ARM without barriers):**

1. The consumer's `load 1` (for `data`) executes out of order *before* `load 2` (for `init_flag`), speculatively. At this moment `data` is still its old value.
2. `load 2` then executes and sees `init_flag = 1`.
3. The consumer exits the loop and uses `x` — which is the pre-initialisation value of `data`, because the speculative load ran too early.

**Why the compiler can also break this:** Even without hardware reordering, a C compiler can reorder the two loads because they touch different variables. `volatile` does not help — `volatile` prevents compiler reordering but not hardware reordering.

**Correct code:**

```c
// Producer
data = 42;
atomic_store_explicit(&init_flag, 1, memory_order_release);

// Consumer
while (atomic_load_explicit(&init_flag, memory_order_acquire) == 0) { }
int x = data;
```

The **release-acquire pair** prevents both compiler and hardware from reordering `data` and `init_flag`. On x86 this compiles to plain `mov` instructions (TSO is naturally acquire/release). On ARM it compiles to `LDAR` / `STLR` (load-acquire, store-release) which carry one-sided fences at no extra cost. On older ARM, it requires an explicit `DMB` barrier.

**Why this matters for interviews:** Anyone claiming to write lock-free code must understand that the "obvious" flag-then-data pattern is broken on weak-memory systems without fences. Most real-world lock-free bugs boil down to missing acquire/release semantics — and they manifest only under rare timing, making them nightmares to debug.
