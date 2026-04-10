# ISA Design Principles — Interview Questions

**Subject:** Computer Architecture
**Topic:** Instruction Set Design, RISC vs CISC, Addressing Modes, Calling Conventions
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. What is an instruction set architecture (ISA) and what does it define?

**Answer:**

An **instruction set architecture** is the contract between hardware and software. It specifies everything a programmer (or compiler) can observe and rely on, while leaving the microarchitectural implementation free to change.

An ISA defines:

1. **Instruction set** — the operations the processor can perform (arithmetic, memory, control flow, system).
2. **Instruction encoding** — how opcodes and operands are packed into bits.
3. **Architectural registers** — general-purpose and special-purpose registers visible to software.
4. **Memory model** — addressing, endianness, alignment rules, and the ordering guarantees between loads and stores (consistency model).
5. **Exception and interrupt behaviour** — what causes a trap, which state is saved, and which handlers run.
6. **Privilege levels and system state** — user vs supervisor mode, control and status registers.

A key property is that **two chips implementing the same ISA are binary compatible**: a binary compiled for one runs unchanged on the other, even if the internal pipeline, cache sizes, and issue width differ by an order of magnitude.

### Q2. What is the difference between RISC and CISC?

**Answer:**

**RISC (Reduced Instruction Set Computer)** and **CISC (Complex Instruction Set Computer)** describe two historical design philosophies.

| Property | RISC | CISC |
|---|---|---|
| Instruction count | Few, simple instructions | Many, often specialised |
| Encoding | Fixed length (e.g. 4 bytes) | Variable length (1–15 bytes on x86) |
| Operands | Load/store architecture | Memory operands on most ops |
| Instructions per cycle | Target 1 (original RISC) | Often multi-cycle microcode |
| Compiler role | Does scheduling and optimisation | Hardware hides some complexity |

**Canonical examples:**
- RISC: MIPS, SPARC, ARM, POWER, RISC-V
- CISC: x86, VAX, System/360

**Modern reality:** The distinction has blurred. x86 processors decode CISC instructions into RISC-like micro-ops and schedule them out of order, giving them the frontend flexibility of CISC with the backend efficiency of RISC. Meanwhile ARM has acquired SIMD, crypto, and atomic extensions that rival x86 in instruction count. What matters today is the **decoder complexity**, the **instruction encoding density**, and the **microarchitectural freedom** the ISA grants.

### Q3. What are addressing modes and why do they matter?

**Answer:**

An **addressing mode** is the method by which an instruction computes an operand address. Common modes include:

| Mode | Syntax (example) | Effective address |
|---|---|---|
| Immediate | `ADDI r1, r2, #10` | — operand is literal |
| Register | `ADD r1, r2, r3` | — operand is in register |
| Register-indirect | `LD r1, (r2)` | `r2` |
| Base + displacement | `LD r1, 8(r2)` | `r2 + 8` |
| Base + index | `LD r1, (r2, r3)` | `r2 + r3` |
| Base + index + scale | `LD r1, (r2, r3, 4)` | `r2 + r3*4` |
| PC-relative | `JAL label` | `PC + imm` |
| Pre/post-increment | `LD r1, (r2)+` | `r2`, then `r2 += size` |

Addressing modes matter because:

1. **Code density**: rich modes reduce instruction count — important for I-cache pressure.
2. **Critical path**: wide modes (base+index+scale+displacement) require adders in the AGU (address generation unit) and can limit load latency.
3. **Compiler complexity**: more modes mean more pattern matching in instruction selection.
4. **Decoder cost**: variable-format modes bloat the decoder.

RISC architectures deliberately restrict addressing modes. RISC-V, for example, has only **base+displacement** for loads and stores — no indexing. The compiler must synthesise other modes with an extra add.

### Q4. What is a calling convention? What does it specify?

**Answer:**

A **calling convention** (also called an ABI — Application Binary Interface) is the set of rules that allows functions compiled independently to interoperate. It specifies:

1. **Argument passing**: which registers carry arguments, and when to spill to the stack.
2. **Return values**: where single and multi-word return values are placed.
3. **Caller-saved vs callee-saved registers**: which registers a callee must preserve.
4. **Stack layout**: frame pointer usage, alignment (e.g. 16-byte at entry to every `call` on x86-64 SysV), and red zones.
5. **Variadic handling**: how `printf`-style functions locate their arguments.
6. **Name mangling and exception unwinding** (at the language-ABI level above).

**Example — x86-64 System V:**
- Integer args 1–6: `rdi, rsi, rdx, rcx, r8, r9`
- Float args 1–8: `xmm0–xmm7`
- Return: `rax` (or `rax:rdx` for 128-bit)
- Callee-saved: `rbx, rbp, r12–r15`
- Caller-saved: `rax, rcx, rdx, rsi, rdi, r8–r11`
- Stack 16-byte aligned at every `call`

**Why it matters for an architect:** The ABI shapes register-file pressure. ISAs with more registers (RISC-V's 31 GPRs) reduce spilling; ones with few (x86-32's 8) suffer in leaf functions. The ABI also determines how predictable return addresses are, which affects the efficacy of the **return address stack** in the branch predictor.

---

## Intermediate

### Q5. Compare fixed-length and variable-length instruction encodings.

**Answer:**

| Property | Fixed (e.g. RISC-V 32-bit) | Variable (e.g. x86 1–15 bytes) |
|---|---|---|
| Fetch simplicity | Trivial — next PC = PC+4 | Hard — must parse to find length |
| Decoder width | Parallel decode is easy | Serial length-decode bottleneck |
| Code density | Worse (~30% larger typical) | Better — common ops are short |
| Compressed extensions | Possible (RV32C, Thumb) | Already dense |
| Branch target alignment | Natural (PC always aligned) | Targets can split cache lines |

**The fetch bottleneck:** A wide superscalar must decode N instructions per cycle. With a fixed-length ISA, decoders can run in parallel — each inspects its own 4-byte slot. With variable length, decoder *i* cannot know where its slot begins until decoder *i−1* has finished parsing its own instruction's length. Intel mitigates this with a **pre-decode stage** that marks instruction boundaries before the main decoder, and a **µop cache** that sidesteps decode entirely on hot loops.

**Why x86 survives:** legacy binary compatibility and the fact that modern frontends effectively cache decoded µops, hiding the decode cost on steady-state code.

### Q6. What is the load/store architecture principle and why was it adopted in RISC designs?

**Answer:**

In a **load/store architecture**, memory is accessed *only* via dedicated load and store instructions. All arithmetic and logical operations take register operands only. Contrast with x86, where `ADD [rax+8], rbx` reads memory, adds, and writes memory in one instruction.

**Reasons RISC designs chose this:**

1. **Pipeline regularity**: Every non-memory instruction completes in one cycle at the execute stage. Memory instructions take one cycle at the execute stage for address calculation plus memory-access latency. This uniformity makes forwarding logic trivial and allows a clean 5-stage pipeline (IF, ID, EX, MEM, WB).

2. **Exceptions are precise**: With a combined "read-modify-write" memory operation, a page fault during the write leaves the register in an undefined state — hard to restart. Separating into `LD`, `ADD`, `ST` gives the hardware a natural restart point at every instruction boundary.

3. **Compiler freedom**: The compiler can keep values in registers across multiple uses, rather than having the ISA implicitly re-fetch from memory. Register allocation is the single biggest win of RISC on real workloads.

4. **Simpler decode**: Only two instructions touch memory. The decoder can route them to a dedicated memory pipe and keep the ALU pipe simple.

**Cost:** Code density. An expression like `a[i] += b[j]` becomes four instructions (two loads, add, store) instead of one. This cost was judged acceptable because I-cache and frontend bandwidth grew to match.

### Q7. How do delay slots work and why are they considered a design mistake today?

**Answer:**

A **branch delay slot** is the instruction immediately following a branch, which the hardware *always* executes, regardless of whether the branch is taken. Early MIPS and SPARC exposed 1–2 delay slots as part of the ISA.

**Original rationale:** In a 5-stage pipeline with no branch prediction, a taken branch creates a 1-cycle bubble (the instruction fetched after the branch must be squashed). Making that slot architecturally visible lets the compiler fill it with a useful instruction, eliminating the bubble.

**Example:**

```
    ADD  r1, r2, r3
    BEQ  r4, r5, target
    SUB  r6, r7, r8    ; delay slot — always executes
target:
    LD   r9, 0(r10)
```

The `SUB` runs even if the branch is taken.

**Why it was a mistake:**

1. **Leaks microarchitecture into the ISA**: The delay slot count is tied to a specific pipeline depth. Deeper pipelines need more slots, but the ISA is fixed at one or two.
2. **Hurts code density**: Slots that cannot be filled usefully require a NOP.
3. **Complicates exceptions**: What is the PC to save when an exception occurs on a delay-slot instruction? Every MIPS implementation carries an EPC + "branch delay" bit to disambiguate.
4. **Modern branch prediction makes the bubble disappear**: A correctly predicted branch has no penalty, making the architectural visibility of the slot pure downside.

RISC-V, designed 25 years after MIPS, deliberately **omits delay slots**. Nothing in modern RISC-V or ARM encoding carries this legacy.

---

## Advanced

### Q8. What is the difference between architectural state and microarchitectural state? Give examples of where this distinction matters.

**Answer:**

**Architectural state** is the state defined by the ISA and visible to software. On x86-64 this is: the 16 GPRs, the XMM/YMM/ZMM registers, RIP, RFLAGS, CR0–CR4, MSRs, etc. A context switch preserves and restores architectural state.

**Microarchitectural state** is everything else the implementation carries: pipeline latches, branch predictor tables, reorder buffer entries, store buffers, TLB entries, cache lines, line-fill buffers, prefetcher history, physical register file beyond the architectural rename mapping.

**Where the distinction matters:**

1. **Virtualisation and migration**: A VM snapshot copies architectural state. Microarchitectural state is implicitly discarded — it repopulates as the VM runs. A VM can migrate between physical CPUs of the same ISA precisely because it depends only on architectural state.

2. **Spectre and Meltdown**: These attacks exploit the fact that speculative execution leaves traces in microarchitectural state (cache lines, TLB entries) that architecturally "never happened". Because microarchitectural state is not protected by the ISA's access-control rules, it can leak information across privilege boundaries. The fix requires new architectural guarantees (e.g. Intel's IBRS, ARM's CSDB) that constrain how speculation may affect microarchitectural state.

3. **Precise exceptions**: The hardware must be able to reconstruct the architectural state "as if" the faulting instruction and all subsequent instructions had never executed, while silently discarding microarchitectural state. This is the job of the reorder buffer.

4. **Debug and performance counters**: Debug interfaces expose architectural state. Performance counters expose aggregate microarchitectural state (cache misses, branch mispredicts) without exposing any specific entries.

5. **ISA extensions**: Adding a new architectural register requires an ISA version bump and OS support. Adding a microarchitectural resource (bigger ROB, more TLB entries) is invisible to software.

### Q9. What is a "weak" vs "strong" memory model and how does it interact with ISA design?

**Answer:**

A **memory consistency model** specifies the order in which memory operations from one core become visible to other cores. It is part of the ISA.

**Sequential consistency (SC)** — the strongest, simplest model — requires that all loads and stores appear in some global interleaving consistent with each thread's program order. SC is intuitive for programmers but expensive to implement because store buffers, load-to-store forwarding, and out-of-order execution all threaten it.

**Total Store Order (TSO)** — x86, SPARC — relaxes only one rule: a younger load may bypass an older store to a different address. This matches what a store buffer naturally does. Loads are still ordered, and stores are still ordered among themselves. Most code "just works" without explicit fences.

**Weak models** — ARMv8, POWER, RISC-V — allow nearly arbitrary reordering of independent memory accesses. The programmer (or more typically, the language runtime) must insert explicit fences or use acquire/release loads and stores to enforce ordering.

**Trade-off:**

- Strong models (SC, TSO): simpler programmer model, but the hardware pays for it in store-buffer snooping, ordered commit of load-queue entries, and bus traffic.
- Weak models: faster hardware, simpler out-of-order commit, lower power, but every synchronisation primitive (mutex, atomic, lock-free queue) requires careful fence placement.

**Why this is an ISA decision, not a microarch one:** The memory model is a *promise* to software. If you ship TSO hardware, a future microarchitectural change that exploits load-load reordering for speed would break every existing binary. You can strengthen a weak model in hardware (pay the cost, still correct) but you cannot weaken a strong one without breaking compatibility.

**Modern language-level interaction:** C11/C++11 atomics define a model above the ISA. `memory_order_seq_cst` maps efficiently to x86 (almost-free) but requires expensive double fences on ARM. `memory_order_acquire`/`release` maps efficiently to ARM's `LDAR`/`STLR` but is still free on x86. This asymmetry is why lock-free code performance differs so dramatically between architectures.

### Q10. A new ISA proposal includes 64 architectural general-purpose registers. Discuss the trade-offs.

**Answer:**

Doubling the register count from 32 (RISC-V, AArch64) to 64 affects many parts of the system. The answer reveals whether the candidate thinks system-wide, not just at the decoder.

**Upsides:**

1. **Less spilling**: In hot loops, the compiler can keep more live values in registers. For numerically intense code with many temporaries, this can reduce store/load traffic significantly. Empirically the diminishing returns of extra registers set in around 32 for scalar integer code, but vector and matrix kernels benefit further.

2. **More leaf-function arguments in registers**: Deeply recursive or highly-inlined code passes more data register-to-register.

3. **Better ILP exposure**: More renaming targets means the compiler can break false dependencies without scratch spills.

**Downsides:**

1. **Encoding cost**: A 3-operand instruction with 64 registers needs 3×6 = 18 bits of register fields versus 3×5 = 15. That is 3 extra bits in every compute instruction. If the ISA uses 32-bit fixed-length encoding, opcode/immediate space must shrink. If it uses variable length, code density suffers.

2. **Context switch cost**: Every context switch saves and restores 64 GPRs instead of 32, doubling the memory traffic for this operation. On a heavily oversubscribed system this matters.

3. **Register file area and ports**: The physical register file must still serve the rename mapping, but now the *architectural* register file (for precise state) is bigger. Area scales roughly linearly with register count; wire delay scales worse.

4. **Caller/callee-save trade-off**: Adding registers complicates the ABI. Too many callee-saved registers and every leaf function pays a save/restore cost. Too many caller-saved registers and the callee cannot hold state across calls.

5. **Existing code cannot benefit**: Legacy binaries compiled for 32 registers get nothing. Recompilation is required. If the ISA is extended (rather than replaced), the encoding cost hits new instructions only, but the compiler must then manage a split register space.

**Empirical note:** SPEC CPU studies from the early 2000s showed scalar integer workloads saturate around 32 registers. Floating-point and vector workloads continue to benefit past 32 — which is why ARM SVE and AVX-512 grow the *vector* register count to 32, not the scalar count.

**Verdict:** For a general-purpose ISA in 2026, 32 is the sweet spot for integer registers. Put the silicon into more vector registers, a larger ROB, or bigger caches instead.
