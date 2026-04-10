# Domain-Specific Accelerators — Interview Questions

**Subject:** Computer Architecture
**Topic:** Systolic Arrays, TPUs, NPUs, Dataflow vs Von Neumann
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. Why have domain-specific accelerators proliferated in the last decade?

**Answer:**

Two forces converged around 2012–2015:

**1. General-purpose CPUs stopped getting faster.** Dennard scaling ended around 2005. Single-thread frequency improvements shrank to a few percent per year. Moore's law still delivered more transistors, but the power budget capped how many could be active simultaneously ("dark silicon"). Adding more CPU cores gave only linear speedup and was limited by Amdahl's law on real workloads.

**2. Specific workloads grew explosively.** Deep learning (2012 onward), cryptocurrency mining, real-time video encoding, network packet processing, and genomics created workloads where the vast majority of compute was a small set of operations (matrix multiply, hashes, convolution, regular expression matching) executed trillions of times.

**The economic argument:** A general-purpose CPU spends ~99% of its silicon on frontend, OoO scheduling, caches, and control, with only ~1% doing actual arithmetic. A specialised accelerator spends 70–90% of its silicon on arithmetic and data movement for its target task. The efficiency gain is 10–100× for the right workload — enough to justify building a whole separate chip.

**What accelerators trade for efficiency:**

- **Programmability:** They run a narrow set of operations, not arbitrary code.
- **Memory model:** Often non-cache-coherent or with explicit data movement.
- **Toolchain:** Custom compilers, runtimes, and debuggers required.
- **Upfront engineering cost:** An accelerator design is a multi-year multi-million-dollar project.

The economic crossover happens when the workload's total compute demand over the chip's lifetime justifies the NRE cost of specialisation. Google concluded (correctly) that its neural network inference load crossed that threshold in 2013; the TPU project started the next year.

### Q2. What is dataflow architecture and how does it differ from the von Neumann model?

**Answer:**

**Von Neumann:** Instructions and data share a common memory. Execution proceeds sequentially under program counter control. A single instruction stream directs what happens next. Parallelism is extracted from an otherwise sequential description (by OoO execution, branch prediction, caches).

**Dataflow:** There is no program counter. Each operation executes as soon as its input operands are available. The program is a directed graph: nodes are operations, edges are data dependencies. Execution "flows" through the graph as data arrives.

**Key differences:**

1. **No control flow bottleneck.** Von Neumann is fundamentally sequential; dataflow is naturally parallel. Wherever data is ready, work can happen.

2. **No shared register file.** Data flows from producer to consumer directly over wires or FIFOs. No centralised state.

3. **No cache hierarchy in the classical sense.** On-chip storage is local to each processing element or between dataflow stages.

4. **Determinism.** A pure dataflow program produces the same result regardless of the order its nodes fire, as long as each node's input order is preserved.

**Why dataflow historically failed as a general-purpose approach:** The research dataflow machines of the 1970s–80s (MIT Tagged-Token, Manchester Dataflow) ran into serialisation bottlenecks from dynamic token matching and struggled with irregular control flow. Conventional CPUs outcompeted them on general code.

**Why dataflow is back — for specific domains:** Modern neural network inference and training are almost perfect dataflow workloads. The computation is a static graph of tensor operations with known dependencies. There is no data-dependent control flow at the node level. Deterministic parallel execution is the whole game.

**Examples of modern dataflow-style accelerators:**

- **Google TPU:** A systolic matrix multiply unit with a programmable sequencer that feeds it instructions.
- **Groq LPU:** Explicitly dataflow-compiled — the compiler schedules every cycle at compile time, eliminating the need for dynamic scheduling.
- **Cerebras WSE:** A wafer-scale "chip" with hundreds of thousands of cores arranged in a 2D mesh; workloads are compiled as dataflow graphs mapped spatially onto the fabric.
- **FPGA-based AI accelerators:** The dataflow graph is literally etched into the routing fabric, with no instruction decode at all.

**Interview insight:** The re-emergence of dataflow is not a rejection of von Neumann — it is a recognition that some workloads are sufficiently narrow to justify a specialised execution model, and that narrowness is where the power and area wins come from.

### Q3. What is "arithmetic intensity" and why does it matter for accelerator design?

**Answer:**

**Arithmetic intensity** is the ratio of compute operations to memory accesses in a workload, usually measured in FLOPs per byte.

$$\text{Arithmetic Intensity} = \frac{\text{number of arithmetic operations}}{\text{number of bytes transferred to/from memory}}$$

**Why it matters:**

A processor has two independent bottlenecks:

- **Peak compute:** operations per second (determined by ALU count × frequency).
- **Peak memory bandwidth:** bytes per second (determined by memory interface speed and width).

A workload can only be limited by one at a time. If compute is the bottleneck, adding more ALUs helps. If bandwidth is the bottleneck, adding more ALUs does nothing — the new ALUs starve. The crossover point is:

$$\text{Balance Point} = \frac{\text{Peak Compute}}{\text{Peak Bandwidth}}$$

If a workload's arithmetic intensity is above this, it is compute-bound; below, memory-bound.

**Typical values:**

| Workload | Arithmetic Intensity |
|---|---|
| Vector add (a[i] = b[i] + c[i]) | 1 FLOP / 12 bytes ≈ 0.08 |
| Dot product | 2 FLOPs / 8 bytes = 0.25 |
| Matrix-vector product (N×N · N) | ~2 FLOPs / byte |
| Matrix-matrix multiply (N×N · N×N) | ~N/8 FLOPs / byte (grows with N) |
| Convolution (large kernel, dense) | 10–100 FLOPs / byte |

**What accelerator designers do about it:**

1. **Raise effective intensity via on-chip data reuse.** A systolic array reuses each loaded element across many PEs, multiplying the effective ops per memory access by the array dimension.

2. **Include large on-chip scratchpad memory.** TPUs have MBs of on-chip SRAM for activations and weights to avoid repeated DRAM fetches.

3. **Use high-bandwidth memory (HBM).** Stack DRAM directly on the accelerator package to get 10× the bandwidth of commodity DDR.

4. **Data compression.** Weights in neural networks are often compressible (pruned, quantised). Compressing them for memory transfer and decompressing on-chip reduces bandwidth requirements.

**Example calculation:** An A100 GPU has ~19.5 TFLOPs (FP32) and 1.5 TB/s HBM2e bandwidth. Balance point = 13 FLOPs per byte. A workload below that is memory-bound; above it, compute-bound. Matrix multiply at moderate size is above; vector add is far below.

**Interview insight:** The roofline model graphs this: y-axis is throughput, x-axis is arithmetic intensity. A workload's achievable throughput is $\min(\text{peak compute}, \text{bandwidth} \times \text{intensity})$. Tuning a kernel often means raising arithmetic intensity (via blocking, tiling, reuse) until the workload moves above the balance point.

---

## Intermediate

### Q4. What is a Tensor Processing Unit, architecturally?

**Answer:**

A TPU (Google) is a domain-specific accelerator for neural network inference (TPU v1) and training (TPU v2+). The architectural ideas are:

**Core: A systolic matrix multiply unit.** TPU v1 had a 256×256 8-bit systolic array — 65,536 multiply-accumulate units arranged in a 2D grid. Every cycle, this array can do 65k MACs. At ~700 MHz, that is ~92 trillion ops/sec (8-bit) — at launch, orders of magnitude more than contemporary CPUs or GPUs for inference.

**Large on-chip unified buffer (UB).** 24 MB of SRAM holding activations between layers. Network activations stream from UB → matrix unit → accumulators → activation function → back to UB. Rarely touches off-chip DRAM during a forward pass.

**Weight FIFO.** Weights are streamed from off-chip memory into the matrix unit as the computation proceeds. A given weight matrix feeds one layer.

**Accumulators.** 4 MiB of accumulator storage catches matrix multiply results before applying nonlinearities.

**Activation unit.** Implements the typical activation functions (ReLU, sigmoid, tanh) in hardware.

**No cache hierarchy, no branch predictor, no OoO.** The TPU has none of the traditional CPU complexity. Execution is a stream of simple high-level instructions (`matmul`, `conv`, `activate`, `pool`) queued by the host CPU.

**Instruction set.** ~12 high-level instructions in TPU v1. The interface is not "C code" but "neural-network compiled to TPU instructions" via TensorFlow or XLA.

**Data types.** TPU v1 used 8-bit integer inference to maximise throughput per bit of memory bandwidth. TPU v2+ support bfloat16 for training (smaller than FP32 but enough dynamic range to train effectively).

**What the TPU sacrifices:**

- **General programmability.** You cannot compile arbitrary C++ to a TPU.
- **Fine-grained control flow.** TPUs execute straight-line tensor ops without data-dependent branching.
- **Low-latency single-request response.** TPU v1 was optimised for throughput, not latency; later versions improve here.

**What it gains:**

- **30–80× performance per watt** compared to contemporary CPUs and GPUs on inference workloads at launch.
- **Predictable latency.** The whole execution is a planned pipeline; no cache misses, no branch mispredicts, no GC pauses.
- **Scale.** A single TPU pod (v4) combines 4096 chips into one coherent ML supercomputer with a custom optical interconnect.

**Interview insight:** The TPU embodies the principle "know your workload, build exactly what it needs". It is not "better than a GPU"; it is *different*, optimised for a much narrower envelope, and dominates within that envelope.

### Q5. How does a Neural Processing Unit (NPU) in a mobile SoC differ from a TPU?

**Answer:**

Both target neural network inference, but at very different power and cost points.

**NPU (mobile SoC NPU — Apple Neural Engine, Qualcomm Hexagon NPU, Samsung NPU, etc.):**
- Power budget: ~1 W.
- Peak compute: 10–30 TOPS (int8) in 2024 designs.
- Memory: Small on-chip buffer (a few MB) + shares DRAM with the rest of the SoC.
- Integrated into the same package as the CPU/GPU/ISP.
- Optimised for **inference of small-to-medium models** (image recognition, speech, generative models on device).

**TPU (datacentre):**
- Power budget: ~200 W per chip.
- Peak compute: ~275 TFLOPs (bfloat16) on TPU v4.
- Memory: Tens of GB of HBM per chip.
- Standalone or in multi-chip pods.
- Optimised for **training and batch inference of huge models**.

**Architectural differences driven by the power budget:**

1. **Matrix unit size.** Mobile NPU matrix units are much smaller than TPU's — maybe 8×8 or 16×16 MACs per cycle, not 256×256. Smaller arrays have better power characteristics for intermittent inference workloads.

2. **Data types.** Mobile NPUs lean hard on int8 or int4 quantisation; training quality is not a concern since they only do inference. TPUs need at least bfloat16 for training stability.

3. **Integration with other SoC blocks.** Mobile NPUs sit alongside the CPU, GPU, ISP, and DSP on the same die, sharing memory fabrics and power planes. They must cooperate with these for pipelined workloads (camera → ISP → NPU → display).

4. **Workload mix.** Mobile NPUs often accelerate many small models running briefly (face unlock, voice wake, photo processing, image enhancement) — they must boot fast, release quickly, and support fast context switching. TPUs run a single large model for hours.

5. **Memory hierarchy.** NPUs must deal with the shared DRAM being contended by CPU and GPU. TPUs own their HBM entirely.

6. **Software stack.** Mobile NPUs are driven by frameworks targeting developer portability (Core ML, TensorFlow Lite, NNAPI). TPUs have custom compilers (XLA) deeply tied to the training framework.

**Modern trend:** NPUs are becoming a standard SoC block alongside CPU and GPU. The 2024 Microsoft "Copilot+ PC" spec mandates 40+ TOPS NPU specifically so laptops can run local LLM inference. The NPU is now a first-class citizen of consumer hardware, not an exotic accessory.

### Q6. What is the difference between "scratchpad" and "cache" memory, and why do accelerators prefer scratchpad?

**Answer:**

- **Cache:** Hardware-managed. The CPU accesses memory normally; the cache quietly holds a subset of recent lines and evicts old ones per a replacement policy. The programmer does not control which data lives in cache.

- **Scratchpad:** Software-managed. The accelerator has on-chip SRAM with its own address space. The programmer (or compiler) explicitly moves data to and from scratchpad with DMA or explicit load/store instructions.

**Why accelerators prefer scratchpad:**

1. **Predictable latency and throughput.** Scratchpad access is always one cycle (or whatever the SRAM provides). No cache misses, no tag comparisons, no coherence delays. Performance is exactly what you analyse on paper.

2. **No cache metadata.** No tags, no state bits, no replacement logic — just raw SRAM. Typically 30–50% more effective capacity per area than an equivalent cache.

3. **No coherence hardware.** An accelerator with scratchpad only needs its own view of memory. No snoop logic, no MESI, no cross-core coordination. Massive simplification.

4. **Workload-specific reuse patterns.** The compiler knows the dataflow graph of the neural network layer being computed. It can compute the optimal data movement schedule and orchestrate scratchpad residency exactly — something a general-purpose cache replacement policy cannot match for this specific workload.

5. **Energy efficiency.** Without tag comparison, set selection, and coherence snooping, scratchpad access burns less energy per byte than cache access.

**Why CPUs prefer cache:**

1. **Unknown access patterns.** General-purpose code walks data in ways the compiler cannot predict. A hardware cache adapts automatically; a scratchpad would require the programmer to orchestrate every access.

2. **Transparent programming model.** Cache looks like memory; scratchpad requires explicit data movement. Forcing every programmer to manage local memory manually would be an ergonomic disaster (and is why GPU programming is harder than CPU programming).

3. **Backward compatibility.** Existing binaries just work on a larger cache; they do not automatically benefit from a larger scratchpad.

**Middle ground — configurable / partitioned designs:**

- Some designs let the programmer configure how much of the on-chip SRAM is cache vs scratchpad. NVIDIA GPUs (since Fermi) allow each kernel to select the split.
- Intel's Knights Landing (Xeon Phi) had MCDRAM that could be configured as cache, scratchpad, or hybrid.

**Interview insight:** The cache-vs-scratchpad choice is fundamentally a bet on whether the workload is predictable. Deep learning is extraordinarily predictable (the graph is known ahead of time) — that is why every serious ML accelerator uses scratchpad. General-purpose computing is not predictable — that is why CPUs use cache. This single technical choice shapes an enormous amount of the downstream design.

---

## Advanced

### Q7. Describe how a programmable dataflow architecture (e.g. Cerebras WSE, Groq LPU) differs from a TPU, and what the trade-offs are.

**Answer:**

A **TPU** is a programmable sequencer driving a matrix unit. Instructions are issued at runtime; a scheduler queues tensor operations. The execution is logically in-order at the operation level, with the matrix unit fully utilised during each op.

**Fully dataflow architectures** like Cerebras WSE and Groq LPU take a more radical position: the **entire program is compiled to a physical mapping** of operations to processing elements and a cycle-level schedule of data movement. There is no runtime scheduler. Each element executes a fixed op at fixed cycles, and data flows between elements on a predetermined path.

**Cerebras WSE (Wafer-Scale Engine):**
- A full silicon wafer (not cut into chips) containing ~850,000 AI cores in a 2D mesh.
- Each core has a small SRAM and can execute simple tensor operations.
- A neural network layer is mapped spatially across thousands of cores. Activations and weights flow through the mesh in planned movements.
- No DRAM during compute; everything is on-wafer SRAM (~40 GB across the whole device).
- Benefit: enormous bandwidth — the wafer-internal fabric is vastly faster than any DRAM.
- Trade-off: compilation is extreme — a model must be mapped to the physical fabric, and memory/compute boundaries must align with the mesh topology.

**Groq LPU:**
- A deterministic dataflow accelerator optimised for inference latency.
- Compiler schedules every instruction at compile time, including network routing between units.
- No caches, no dynamic scheduling, no runtime variance — execution time is exactly predictable.
- Benefit: lowest latency per token of any major LLM inference accelerator (as of 2024).
- Trade-off: requires substantial compiler effort for every new model. Not suitable for general tensor programs.

**How these differ from a TPU:**

| Property | TPU | Cerebras/Groq |
|---|---|---|
| Runtime scheduling | Yes (op-level) | No (all at compile time) |
| Compilation complexity | Moderate | Extreme |
| Latency predictability | Good | Deterministic to the cycle |
| Workload flexibility | Any tensor graph | Must fit the fabric well |
| DRAM dependency | HBM | Mostly/entirely on-chip |

**Interview insight:** The trade-off is flexibility vs performance. TPU gives you good performance on anything XLA can compile. Cerebras and Groq give you exceptional performance on workloads their compilers can lay out well, and nothing on the rest. The industry is working out which niches justify the extreme specialisation.

### Q8. What is "chiplet-level" specialisation and why is it a natural evolution from monolithic accelerators?

**Answer:**

Historically, accelerator SoCs combined different functions (compute, memory, IO, control) on a single monolithic die. Chiplet-based designs split them into separate dies connected in the same package.

**Why this is a natural evolution for accelerators:**

1. **Process node asymmetry.** Compute benefits from the leading-edge node (3 nm, 2 nm). IO interfaces, analogue memory controllers, and SerDes PHYs gain nothing from process shrinks — they are constrained by wire parasitics and noise, not transistor count. Building them on cheaper 7 nm or 5 nm chiplets saves cost without losing performance.

2. **Yield.** Accelerators are large — a datacentre GPU can be 800+ mm². At that size, defect rates are brutal. Splitting into smaller chiplets lets the vendor throw away only the bad pieces.

3. **Reticle limit escape.** Lithography reticles max out at around 800 mm². Beyond that, you cannot fabricate a single die. Chiplets let designs scale past this with direct die-to-die bonding.

4. **Mix-and-match product stacks.** One compute chiplet + different IO chiplets = different products. AMD's EPYC uses this heavily — the same compute die powers many SKUs by varying the IO and memory-controller chiplet configuration.

5. **Rapid updates.** You can refresh the compute chiplet (new process node) without redesigning the memory and IO dies. Halves the time-to-market for product generations.

**Examples:**

- **AMD MI300:** Multiple compute chiplets + IO chiplet + HBM3 stacks, all in one package. The compute dies can be GPU or CPU chiplets, composable into different SKUs (MI300X, MI300A).

- **NVIDIA Grace-Hopper:** A CPU chiplet (Grace) and GPU chiplet (Hopper) bonded via NVLink-C2C in a single package. Coherent memory fabric between them.

- **Intel Ponte Vecchio:** 47 chiplets across multiple process nodes in one package — the most aggressive chiplet design shipped.

- **Apple M1 Ultra:** Two M1 Max dies bonded with UltraFusion silicon interposer. Programs see it as a single chip.

**Interconnect standards:**
- **UCIe (Universal Chiplet Interconnect Express)** is the industry standard for cross-vendor chiplets.
- **BoW (Bunch of Wires)** is a simpler alternative.
- **Infinity Fabric, NVLink, CXL** are vendor-specific at present but evolving toward interoperability.

**Trade-offs:**
- **Die-to-die energy.** Off-die signalling is still 3–10× more energy per bit than on-die. Applications that need chiplet-crossing bandwidth pay a measurable cost.
- **Packaging complexity.** Silicon interposers, TSVs, and hybrid bonding are expensive and have their own failure modes.
- **Thermal design.** Heat dissipation is harder with multi-die packages — hot spots in one chiplet may affect its neighbours.

**Looking forward:** By the late 2020s, nearly every leading-edge accelerator and high-end CPU will use chiplets. The interconnect and packaging technology is becoming at least as important a research area as the compute architectures themselves.

### Q9. What is "compute in memory" (CIM) and why is it considered an answer to the memory wall?

**Answer:**

**Compute in memory** (also called **processing in memory, PIM**) moves simple arithmetic operations into or adjacent to the DRAM die, avoiding the round-trip between main memory and the CPU/accelerator for operations that have low arithmetic intensity.

**Why it is interesting:**

The energy cost of moving a byte from DRAM to the CPU is roughly **50–100× greater** than the energy of performing a single 64-bit add on that byte once it arrives. For workloads with low arithmetic intensity (vector reductions, sparse matrix-vector, database scans), the total energy is dominated by data movement, not compute. Moving the compute to the data is the only way to break this.

**Varieties of CIM:**

**1. Near-data processing (near-DRAM):** Small processors placed on the DRAM package, between the DRAM banks and the memory interface. They can perform arithmetic on data read from DRAM before it leaves the package. Examples: Samsung HBM-PIM (2021), SK Hynix AiM.

**2. In-bank processing:** Smaller ALUs placed inside the DRAM die itself, near the sense amplifiers. Operations like "sum all elements in a row" happen without the data ever reaching the memory bus. Research prototypes; some commercial offerings.

**3. Analog in-memory compute:** The DRAM array (or a specialised SRAM/ReRAM array) is used as an analogue compute element. Activations are presented as voltages on wordlines; weights are encoded in the resistance of memory cells; currents on bitlines naturally sum to implement matrix-vector multiplication. Extremely efficient for the narrow class of "dense matrix-vector multiply with small weights" that dominates neural network inference. Examples: Mythic, IBM's analogue AI research.

**4. Bit-serial compute in SRAM:** A variation where the SRAM itself becomes a bit-serial ALU array. Flexible but slower per-op than dedicated ALUs. Academic-leaning.

**Why the memory wall justifies this:** The classic "fetch a 64-byte line from DRAM, do a few ops, write it back" pattern wastes most of the energy on the fetch. Moving compute to memory means the ~100 pJ per byte of DRAM access is no longer multiplied by the op count — you save the round trip for any op done in memory.

**Challenges:**

1. **Programming model.** CPUs assume memory is passive. Exposing compute-in-memory to software requires new ISA extensions or driver abstractions.

2. **Flexibility.** In-memory ALUs are necessarily simple. Complex operations must happen somewhere else, and the data movement cost returns.

3. **Coherence with the CPU cache.** If CIM modifies data, the CPU cache must know about it. This requires either cache flushing or new coherence mechanisms.

4. **Manufacturing.** DRAM and logic use different processes. Putting logic in DRAM either compromises DRAM density or requires 3D stacking of logic dies on DRAM.

**Current status (2024–2026):** CIM is moving from research to early commercialisation. Samsung HBM-PIM is shipping in limited quantities; Mythic is shipping analog CIM for edge inference. The first mainstream uses are for LLM inference, where the weight-dominated memory bandwidth argument is strongest. Expect this to become a major part of accelerator architecture over the second half of the 2020s.

### Q10. How would you reason about whether to build an accelerator for a new workload?

**Answer:**

The question comes down to **expected lifetime compute savings vs NRE cost**. A principled analysis has several axes:

**1. Workload characterisation:**
- How much total compute does this workload consume? (Hours of GPU time per day? Trillions of ops per request? Millions of requests?)
- What fraction of that compute is a small set of operations? (Matrix multiply? FFT? Hash? Regex?)
- Is the workload stable or rapidly evolving? (Evolving workloads outrun specialised hardware.)

**2. Energy and cost baseline:**
- What is the current cost (CapEx + OpEx) of running the workload on general-purpose hardware?
- What fraction of that cost is compute vs memory vs networking?
- How much could a specialised accelerator realistically save? (10× is typical; 100× rare; 2× is usually not worth the bother.)

**3. NRE estimate:**
- A custom accelerator ASIC costs $10M–$100M+ in engineering, mask sets, and verification.
- Time to market: 2–4 years from architecture to shipping silicon.
- Software stack: often 3–5× the hardware engineering cost.
- Field deployment: integration, validation, customer support.

**4. Alternative acceleration paths:**
- **GPU** — most flexible, immediate availability, strong toolchain. The right answer for most new workloads until they reach huge scale.
- **FPGA** — moderate specialisation, faster iteration than ASIC, worse energy efficiency than ASIC. Good for workloads with moderate specialisation benefit or that are still evolving.
- **Instruction extension** on existing CPU/GPU — smallest step, fastest path. Good if the workload is "almost" something the general-purpose unit can do.

**5. Risk factors:**
- **Workload drift.** Will the algorithm the accelerator is built for still be the best approach in 3 years? Deep learning architectures have changed dramatically over the last decade.
- **Software portability.** A specialised accelerator creates vendor lock-in. This may be acceptable internally (Google's TPU) but a hard sell for external customers.
- **Scaling.** Does the accelerator scale to your expected demand? If it saturates at 10x the current workload and you expect 100x growth, it may not be worth it.

**6. Success criteria:**
- **Perf/watt improvement ≥ 10×** over the best general-purpose alternative.
- **Projected cost savings ≥ 3× NRE** over the chip's lifetime.
- **Workload stability ≥ 3 years** — long enough that the accelerator stays relevant.

**Canonical case studies:**

- **Google TPU:** Justified because neural network inference was (a) a huge fraction of Google's total compute, (b) stable in its core operations (matrix multiply), and (c) a 30–80× efficiency win over CPUs/GPUs at the time.

- **Bitcoin ASICs:** Justified by the extreme efficiency demand. SHA-256 is narrow enough and stable enough that ASICs can reach 100,000× efficiency over CPUs. General-purpose hardware lost the niche entirely.

- **Video codec accelerators in every phone SoC:** Justified because video playback is universal and the workload is narrow and stable. Every mobile chip ships with H.264/H.265/AV1 decoders; nobody uses the CPU for it.

- **GPU for general compute (CUDA):** A case where an accelerator built for one workload (graphics) turned out to also serve another (HPC, ML) well enough to take over. Lesson: sometimes the right accelerator was already in the building, and the opportunity was in software abstraction, not new silicon.

**Interview signal:** A strong candidate reasons about workload characteristics, alternatives, and the long tail of costs (software, deployment, support) — not just "build an ASIC because it will be faster".
