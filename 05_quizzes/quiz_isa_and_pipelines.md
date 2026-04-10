# Quiz — ISA and Pipelines

**Subject:** Computer Architecture
**Topics covered:** ISA design, classic 5-stage pipeline, hazards, forwarding, branch prediction
**Format:** Multiple-choice and short-answer questions. Answers at the end.

---

### Q1. In a classic 5-stage MIPS pipeline, which hazard type is NOT possible?

a) RAW
b) WAR
c) Control
d) Structural (if I-cache and D-cache are separate)

---

### Q2. A pipeline with branch misprediction penalty of 15 cycles, branch frequency of 20%, and predictor accuracy of 96% has ideal CPI of 1.0. What is the effective CPI?

---

### Q3. Why does a load-use hazard require a stall even with full forwarding?

a) Because the register file write happens in WB.
b) Because the load result is only available at the end of MEM, too late for forwarding to EX of the next instruction.
c) Because forwarding paths cannot cross pipeline stages.
d) Because the load instruction is always mispredicted.

---

### Q4. What does the "reduced" in RISC refer to?

a) Reduced transistor count.
b) Reduced instruction count (fewer distinct operations).
c) Reduced decoder complexity through regular encoding and limited addressing modes.
d) Reduced clock frequency.

---

### Q5. Name two reasons modern ARM processors omit architectural delay slots while retaining most other RISC ideas.

---

### Q6. A branch target buffer (BTB) with 4096 entries is direct-mapped. What can cause it to produce the wrong target even though there is no eviction?

a) Aliasing — two branches at different PCs map to the same BTB index.
b) The BTB is cold (no prior training).
c) The branch is indirect and the target has changed.
d) All of the above.

---

### Q7. In a TAGE predictor, what is the purpose of having multiple tables with geometrically-increasing history lengths?

---

### Q8. Give two reasons why a return address stack (RAS) predicts return targets better than a general BTB.

---

### Q9. In which circumstances does a delayed-branch slot require a NOP?

a) When the compiler cannot find a safe instruction to place there.
b) When the branch is predicted not taken.
c) Whenever the branch is backward.
d) Never — all modern compilers can always fill it.

---

### Q10. A processor has 12-cycle misprediction penalty, 15% branches, and 1.0 ideal CPI. How much accuracy must the branch predictor have so that effective CPI ≤ 1.05?

---

### Q11. Which of these is NOT eliminated by forwarding?

a) RAW from an ALU op feeding another ALU op in the next cycle.
b) RAW from a load feeding an ALU op in the next cycle.
c) WAR hazards in a 5-stage in-order pipeline.
d) RAW from MEM-stage to WB-stage writes.

---

### Q12. Explain why variable-length instruction encoding (e.g., x86) makes wide-decode frontends more difficult than fixed-length encoding (e.g., RISC-V).

---

## Answers

**A1.** (b) WAR is impossible in a classic 5-stage in-order pipeline because register reads happen in ID (early) and writes in WB (late), both strictly in program order.

**A2.** CPI = 1.0 + 0.20 × 0.04 × 15 = **1.12**.

**A3.** (b). The load result emerges from MEM at the end of its MEM cycle, but the dependent ADD needs it at the *start* of its EX cycle — which is the same clock cycle. Forwarding cannot reach backward in time, so one bubble is required.

**A4.** (c). RISC refers to reduced decoder/pipeline complexity via regular, simple instructions, not strictly "fewer" instructions. Modern RISC ISAs have hundreds of instructions once SIMD and crypto extensions are added.

**A5.** (1) Delay slots leak pipeline depth into the ISA, which varies with implementation. (2) Modern branch prediction makes the bubble disappear, so the architectural exposure is pure downside. Also accepted: exception handling complications.

**A6.** (d). All three can cause a wrong target without eviction.

**A7.** Each branch naturally uses the shortest history that is useful for it. Branches that need no history use the base predictor; branches that need long correlations find a longer-history table. A single fixed history length cannot serve both cases well.

**A8.** (1) Returns form a LIFO stack structure that matches call/return nesting exactly, whereas a BTB sees returns as indirect jumps with varying targets. (2) The RAS is updated on every call (push) and return (pop), providing a perfect match for well-structured code; a BTB only learns the most recent target.

**A9.** (a). When the compiler cannot find a useful instruction to place in the slot, it must fill it with NOP. This is one of the reasons delay slots are now considered a design mistake.

**A10.** 1.05 ≥ 1.0 + 0.15 × (1 − p) × 12 → (1 − p) ≤ 0.0278 → p ≥ **97.2%**.

**A11.** (c). WAR hazards do not exist in an in-order 5-stage pipeline, so forwarding has nothing to do with eliminating them — they never occur.

**A12.** In fixed-length encoding, every N-wide decoder slot starts at a known offset (PC, PC+4, PC+8, ...), so decoders can operate fully in parallel. In variable-length encoding, decoder *i* cannot start parsing its slot until decoder *i-1* has finished, because the length of instruction *i-1* determines where instruction *i* begins. This serialisation limits wide decode throughput. Intel mitigates it with a pre-decode stage that marks instruction boundaries ahead of the main decoder, and a µop cache that caches post-decode results so steady-state hot loops bypass the decoder entirely.
