# Computer Architecture — Interview Preparation

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Subject: Computer Architecture](https://img.shields.io/badge/Subject-Computer%20Architecture-blue)](https://en.wikipedia.org/wiki/Computer_architecture)

## Overview

This repository provides comprehensive interview preparation material for computer architecture roles. The content targets engineers interviewing for CPU, GPU, and accelerator architecture positions, as well as performance-engineering and low-level software roles where microarchitectural understanding is required.

The material progresses from instruction set architecture and single-issue pipelines through modern out-of-order execution, memory hierarchy, and data-parallel accelerators. Each section contains explanatory material and worked problems reflecting the depth expected in senior-level interviews.

## Table of Contents

- [01 ISA and Pipelines](#01-isa-and-pipelines)
- [02 Out-of-Order Execution](#02-out-of-order-execution)
- [03 Memory Hierarchy](#03-memory-hierarchy)
- [04 Parallelism and Accelerators](#04-parallelism-and-accelerators)
- [05 Quizzes](#05-quizzes)
- [How to Use](#how-to-use)
- [Related Repositories](#related-repositories)
- [Contributing](#contributing)

### 01 ISA and Pipelines

Instruction set architecture fundamentals and in-order pipelined execution.

- `isa_design_principles.md` — RISC vs CISC, orthogonality, addressing modes, calling conventions
- `classic_pipeline_and_hazards.md` — Five-stage MIPS pipeline, structural/data/control hazards, forwarding, stalls
- `branch_prediction.md` — Static and dynamic predictors, BTB, RAS, tournament and TAGE predictors
- `worked_problems/` — Problems on pipeline CPI, forwarding logic, and predictor accuracy

### 02 Out-of-Order Execution

Dynamic instruction scheduling, renaming, and speculation.

- `tomasulo_and_renaming.md` — Register renaming, reservation stations, ROB, WAW/WAR elimination
- `speculation_and_recovery.md` — Speculative execution, precise exceptions, checkpointing, squash
- `wide_issue_and_scheduling.md` — Multi-issue, wakeup-select loops, load/store queues, memory disambiguation
- `worked_problems/` — Problems on ROB sizing, issue-width scaling, and Tomasulo cycle-by-cycle traces

### 03 Memory Hierarchy

Caches, virtual memory, and coherence.

- `cache_organisation.md` — Direct-mapped, set-associative, replacement policies, write policies, inclusion
- `virtual_memory_and_tlbs.md` — Paging, multi-level page tables, TLB coverage, huge pages, ASIDs
- `coherence_and_consistency.md` — MESI/MOESI, snoop vs directory, TSO vs weak memory models, fences
- `worked_problems/` — Problems on miss-rate decomposition, TLB reach, and coherence traffic analysis

### 04 Parallelism and Accelerators

Multicore, vectorisation, and domain-specific acceleration.

- `smt_and_multicore.md` — Simultaneous multithreading, cache partitioning, NUMA, scaling bottlenecks
- `vector_and_simd.md` — SSE/AVX/SVE, predication, gather/scatter, vector length agnostic code
- `domain_specific_accelerators.md` — Systolic arrays, TPUs, NPU tiles, dataflow vs von Neumann

### 05 Quizzes

Self-assessment quizzes covering each major topic area.

- `quiz_isa_and_pipelines.md` — Pipeline hazards, forwarding, branch prediction
- `quiz_ooo_execution.md` — Renaming, ROB, speculation
- `quiz_memory_hierarchy.md` — Caches, TLBs, coherence, consistency

## How to Use

This repository is structured as a progressive course in modern computer architecture:

1. **Start with ISA and pipelines**: Build intuition for in-order execution before engaging with the subtleties of dynamic scheduling.

2. **Study out-of-order execution**: This is the core of every modern high-performance CPU. Be able to draw a Tomasulo pipeline diagram on a whiteboard.

3. **Memory hierarchy is half of performance**: Interviews will probe cache math, TLB coverage, and coherence invariants. Practice the worked problems until the arithmetic is fluent.

4. **Parallelism and accelerators**: Understand how classical CPU techniques generalise (and break down) in multicore and GPU contexts.

5. **Quiz yourself**: Use the quizzes to identify weak areas and return to the relevant section.

## Related Repositories

- **[Interview_SoC_Architecture](https://github.com/BrendanJamesLynskey/Interview_SoC_Architecture)** — System-level integration, interconnects, and peripheral design
- **[Interview_RISC_V](https://github.com/BrendanJamesLynskey/Interview_RISC_V)** — RISC-V ISA specifics and privileged architecture
- **[Interview_CUDA](https://github.com/BrendanJamesLynskey/Interview_CUDA)** — GPU architecture and CUDA programming
- **[Interview_AI_Accelerator_Architecture](https://github.com/BrendanJamesLynskey/Interview_AI_Accelerator_Architecture)** — ML accelerator design principles

## Contributing

Contributions are welcome. Please ensure:

1. Content is technically accurate and clearly explained
2. Worked problems include complete derivations and final answers
3. Diagrams and numerical examples are reproducible
4. Quiz questions reflect realistic interview scenarios

For significant additions, please open an issue first to discuss scope and approach.

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
