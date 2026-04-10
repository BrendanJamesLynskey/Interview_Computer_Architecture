# Quiz — Out-of-Order Execution

**Subject:** Computer Architecture
**Topics covered:** Tomasulo, register renaming, ROB, reservation stations, speculation, scheduling
**Format:** Multiple-choice and short-answer questions. Answers at the end.

---

### Q1. Register renaming eliminates which dependency types?

a) RAW only
b) WAR only
c) WAW only
d) WAR and WAW

---

### Q2. What is the purpose of a Reorder Buffer?

a) Reorder instructions before they enter the frontend.
b) Provide a program-order view for in-order commit, enabling precise exceptions.
c) Cache recently-executed instructions.
d) Merge writes to the same register into a single commit.

---

### Q3. A wakeup-select loop must complete in one clock cycle to sustain back-to-back dependent ALU ops. Name two techniques used to meet this constraint at high frequencies.

---

### Q4. What distinguishes a physical register file (PRF) rename scheme from a value-in-reservation-station (classical Tomasulo) scheme?

---

### Q5. Which of the following is NOT typically stored in a ROB entry?

a) Destination architectural register.
b) Completion bit.
c) Source operand values.
d) Exception code and pending fault bit.

---

### Q6. A core has 500 ROB entries, 4 IPC target, and a 300-cycle last-level miss. What is the maximum theoretical memory-level parallelism (MLP) it can expose for a streaming miss pattern? Briefly justify.

---

### Q7. Why is a scheduler replay cheaper than a branch mispredict flush?

---

### Q8. Move elimination in modern x86 renamers exploits what observation?

a) Most MOV instructions are redundant and can be removed by the compiler.
b) A register-to-register MOV can be implemented by pointing two RAT entries at the same physical register.
c) MOV instructions have predictable cache behaviour.
d) MOV instructions can always be zero-latency.

---

### Q9. Explain what "selective squash" (branch-selective squash) does and why it outperforms full squash on branch-dense code.

---

### Q10. A memory-ordering machine clear happens when:

a) A branch is mispredicted.
b) A speculative load is invalidated by a coherence message before retirement.
c) The L2 cache is full.
d) The store queue overflows.

---

### Q11. Give one reason why a physical register file is banked in a wide OoO core, and one cost of doing so.

---

### Q12. Explain how a zero-idiom (e.g., `XOR rax, rax`) is optimised in the rename stage of a modern x86 CPU.

---

## Answers

**A1.** (d). Register renaming eliminates WAR and WAW by giving each destination a fresh physical register. RAW (true dependency) cannot be eliminated by any renaming — it is a real dataflow requirement.

**A2.** (b). The ROB enables in-order commit, which is how precise exceptions are preserved across out-of-order execution.

**A3.** Any two of: (1) speculative wakeup based on predicted producer latency, (2) banking the PRF so register reads fit inside the loop, (3) operand forwarding that bypasses the PRF for recently-produced values, (4) distributing the scheduler into smaller banks, (5) using a staged wakeup-select with early warning signals.

**A4.** In PRF-based rename, physical registers are a single large pool that stores all values; reservation stations hold only tags, not values. In value-in-RS rename, each RS entry holds the operand values when they become available, and the CDB broadcasts (tag, value) pairs. PRF-based scales better because RS storage stays small and CDB broadcasts only tags.

**A5.** (c). Source operand values live in the physical register file (or in the RS in the classic scheme), not in the ROB. The ROB tracks ordering metadata, not operand data.

**A6.** At 4 IPC and 300-cycle latency, the ROB is the limiting structure: 500 entries / 300 cycles ≈ 1.67 cache-miss-equivalents at steady state. More importantly, the number of outstanding *memory* ops is bounded separately by the load and store queue sizes — typically ~100 loads in flight. So practical MLP is ~100, not 500.

**A7.** Because a replay only re-executes the specific instruction whose timing assumption was wrong (and its affected dependents), while a flush invalidates everything younger than the mispredicted branch — potentially hundreds of correctly-predicted instructions — and requires refetching from the I-cache.

**A8.** (b). The renamer points both the source and destination RAT entries at the same physical register, turning the MOV into a zero-cycle rename operation with no execution-unit work.

**A9.** Selective squash marks each in-flight instruction with a mask of the branches it speculates past. On a mispredict of branch B, only instructions whose mask includes B are squashed; independent work past correctly-predicted branches survives. On branch-dense code with multiple branches in flight, this preserves 10–20% more useful work than full squash.

**A10.** (b). A memory-ordering machine clear is triggered when the coherence system detects that a speculatively-executed load has read a line that was subsequently invalidated by another core before the load retired, potentially violating the memory model.

**A11.** Reason: to reduce the number of read/write ports per SRAM, allowing higher clock frequency and smaller area. Cost: bank conflicts — two instructions that need reads from the same bank in the same cycle force one of them to be delayed.

**A12.** The renamer pattern-matches `XOR reg, reg` and treats it as a zero-producer rather than a genuine XOR operation. It allocates a fresh physical register tagged as holding zero, without any dependency on the source `reg`. The instruction bypasses the ALU and completes at rename, both breaking any false dependency on the source and avoiding execution-unit work entirely.
