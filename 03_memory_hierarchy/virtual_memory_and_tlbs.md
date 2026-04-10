# Virtual Memory and TLBs — Interview Questions

**Subject:** Computer Architecture
**Topic:** Paging, Multi-Level Page Tables, TLB Coverage, Huge Pages, ASIDs
**Difficulty tiers:** Fundamentals / Intermediate / Advanced

---

## Fundamentals

### Q1. Why does virtual memory exist?

**Answer:**

Virtual memory is a layer of indirection between the addresses programs use (virtual) and the addresses physical memory uses (physical). It solves several problems simultaneously:

1. **Isolation:** Each process has its own virtual address space. Process A cannot name — let alone touch — process B's memory, because A's virtual addresses translate to different physical addresses than B's.

2. **Relocation:** A program compiled to run at virtual address `0x400000` can run even if another program is already at that physical address. The loader and OS map each process's virtual `0x400000` to any free physical page.

3. **Memory abstraction beyond physical:** A process can use more address space than the machine has physical memory (on-disk paging covers the gap; this matters less today when 256 GB RAM is common and swap is mostly vestigial, but the abstraction remains).

4. **Protection:** Page-table entries carry permission bits (R, W, X, U/S). An attempt to write a read-only page, execute a non-executable page, or access a kernel page from user mode raises a fault.

5. **Sharing:** Multiple processes can map the same physical page into their virtual address spaces — the basis for shared libraries, copy-on-write fork, and shared memory IPC.

6. **Lazy allocation:** The OS can defer physical allocation until the page is first touched. malloc can hand out gigabytes that only materialise as you use them.

The cost is address translation on every memory access, which is where the TLB comes in.

### Q2. Describe a multi-level page table and why it is used instead of a single-level table.

**Answer:**

A single-level page table would be an array indexed by virtual page number, with one entry per VPN. For a 48-bit virtual address space and 4 KB pages, there are 2³⁶ pages — 64 Gi entries. At 8 bytes per entry, this is **512 GB per process** just for the page table. Impossible.

**Solution: multi-level (hierarchical) page tables.** The virtual address is split into fields that index successive levels of the table:

**x86-64 with 4 KB pages:** 48-bit virtual address (canonical form), 4-level page table.

```
[unused 16 | PML4 9 | PDPT 9 | PD 9 | PT 9 | offset 12]
```

- Top-level table (PML4): 512 entries, pointed to by the CR3 register.
- Each PML4 entry points to a 512-entry PDPT.
- Each PDPT entry points to a 512-entry PD.
- Each PD entry points to a 512-entry PT.
- Each PT entry points to a 4 KB physical page.

Five-level paging (introduced in Ice Lake) adds a PML5 level for 57-bit virtual addresses.

**Memory savings:** Only the top-level table must exist in full (4 KB). Sub-tables are allocated on demand. A process using 1 MB of address space needs maybe 4 KB (PML4) + 4 KB (PDPT) + 4 KB (PD) + 4 KB (PT) = 16 KB of page-table storage, not 512 GB.

**Translation cost:** A page walk traverses all levels. At worst, 4 memory accesses per translation on x86-64. This is why the TLB (which caches completed translations) is essential — without it, every load and store would trigger 4 extra loads, crippling performance.

### Q3. What is a TLB and why is it fully associative (or nearly so)?

**Answer:**

A **Translation Lookaside Buffer (TLB)** is a small cache of recent virtual-to-physical address translations. On each memory access, the CPU looks up the virtual page number in the TLB. On a hit, the physical page number is returned in a single cycle. On a miss, a page walk is triggered (by the hardware or a software handler).

**Why TLBs are typically fully associative:**

1. **Small size** (64 to a few hundred entries). Full associativity is cheap at this scale.

2. **High cost of conflict misses.** A TLB miss costs 4+ memory accesses (or the walker's cache hit rate), much more than an L1 cache miss. Even a handful of conflict misses per million instructions would show up as a measurable slowdown.

3. **Non-uniform access patterns.** TLB entries are not accessed in a strided pattern; they are accessed in whatever order the program's working-set pages appear. A set-associative TLB with any reasonable index function can suffer conflict misses on unlucky page layouts.

**Modern TLB structures:** A two-level hierarchy is typical:

- **L1 DTLB:** ~64 entries, fully associative, 1-cycle lookup.
- **L1 ITLB:** ~64 entries for instruction translations.
- **L2 TLB:** ~1500–2000 entries, often set-associative with high associativity (8–16 way) because the entry count makes full associativity costly.

Modern L2 TLBs are also split for different page sizes (4 KB, 2 MB, 1 GB).

### Q4. What are huge pages and why do they matter?

**Answer:**

A **huge page** is a larger-than-default page, typically 2 MB or 1 GB on x86-64. The x86 ISA has supported huge pages since the Pentium Pro; ARM supports 2 MB, 1 GB, and larger via block translations.

**Why they matter:**

1. **TLB reach multiplies.** A 64-entry TLB with 4 KB pages covers 256 KB of virtual memory. With 2 MB pages, it covers 128 MB — 512× more. For workloads that touch working sets larger than a few hundred KB, this is the difference between TLB-hit-dominated execution and TLB-miss-dominated execution.

2. **Shorter page walks.** A 2 MB page terminates the walk one level earlier; a 1 GB page terminates two levels earlier. Each level saved is one memory access saved on the (already expensive) miss.

3. **Reduced page-table storage.** Fewer leaf entries mean less metadata.

**Costs and complications:**

1. **Fragmentation.** A 2 MB page must be backed by a contiguous, aligned 2 MB physical region. After heavy allocation/deallocation cycles, the OS may not be able to find one, forcing huge-page requests to fall back to 4 KB pages.

2. **Waste.** If an application only touches a few KB of a 2 MB page, most of the page is wasted. Huge pages are inappropriate for sparse access patterns.

3. **Protection granularity.** Permission bits apply to the whole huge page. You cannot mark a single 4 KB chunk read-only within a 2 MB huge page.

4. **Page-level swapping** is impractical at 2 MB granularity, so huge pages are typically pinned.

**Transparent Huge Pages (Linux THP):** The kernel automatically promotes 4 KB allocations to 2 MB when the process's access pattern justifies it and contiguous physical memory is available. Saves programmer effort but can cause latency spikes when defragmentation runs.

**When to use huge pages:**
- Large in-memory databases (Redis, Postgres shared buffers)
- JIT heaps (JVM, V8)
- HPC workloads with large matrices
- Machine learning inference with weight matrices

---

## Intermediate

### Q5. What is an ASID (Address Space Identifier) and what problem does it solve?

**Answer:**

An **ASID** (also called PCID on x86) is a tag attached to each TLB entry identifying which address space it belongs to. It lets the TLB hold entries from multiple processes simultaneously, so a context switch between processes does not require flushing the TLB.

**Without ASIDs:** On a context switch from process A to process B, all A's translations must be invalidated — otherwise B would inherit A's mappings. The simplest way is to flush the entire TLB. This is expensive: the next few hundred memory accesses from B incur TLB misses, each costing a page walk.

**With ASIDs:** The CPU loads process B's ASID register. Subsequent TLB lookups require both the VPN *and* the ASID to match. Entries for process A remain in the TLB but are invisible to B. If process A is resumed soon after, its TLB entries are still warm.

**ASID field size:** On x86 PCID, 12 bits → 4096 ASIDs. On ARM, 8 or 16 bits depending on the implementation. The OS must recycle ASIDs when they run out, which does require a TLB flush but only once per wrap-around.

**Kernel mappings:** A conventional optimisation is to mark kernel page entries "global" so they are shared across all ASIDs — no need to flush them on context switch and no need to duplicate them per-ASID. Meltdown mitigations (KPTI) broke this by forcing kernel pages to be unmapped from user ASIDs and only mapped in a separate kernel ASID, adding TLB flushes at user↔kernel transitions.

**Performance impact of ASIDs:** On kernel-heavy workloads (database with many syscalls), ASIDs can reclaim 5–15% of runtime. Before PCID, Linux used full TLB flushes and paid this cost.

### Q6. What happens on a TLB miss?

**Answer:**

A TLB miss triggers a **page walk** — the sequence of memory accesses that traverse the page tables to find the translation.

**Hardware-walked TLBs (x86, modern ARM):** A dedicated state machine in the MMU walks the page table. For x86-64 with 4 KB pages:

1. Read CR3 → physical address of PML4.
2. Use VA bits [47:39] to index PML4; read the entry.
3. Use VA bits [38:30] to index the PDPT pointed to; read the entry.
4. Use VA bits [29:21] to index the PD pointed to; read the entry.
5. Use VA bits [20:12] to index the PT pointed to; read the entry. This entry has the physical frame number.
6. Combine with VA bits [11:0] → physical address.
7. Insert the translation into the TLB.

**Four memory accesses** (one per level) on a cold walk. The walker issues these through the regular cache hierarchy, so many page-walk accesses hit in L1/L2.

**Software-walked TLBs (MIPS, SPARC):** A TLB miss raises a fault; an OS handler walks the page table in software and writes the translation into the TLB with a privileged instruction. More flexible (any page-table format works) but 10–100× slower per miss.

**Page Walk Cache (PWC):** Modern x86 cores cache intermediate page-walk results (PML4, PDPT, PD entries) in a separate structure. On a TLB miss that happens "near" a previous one in the same region, the walker can skip levels. Reduces the average walk cost to 1–2 memory accesses.

**Exception path:** If the final page-table entry is marked not-present, the hardware raises a **page fault** exception. The OS handler either allocates the page (lazy allocation), reads it from swap, handles a copy-on-write clone, or kills the process.

**Cost model:** A TLB miss that hits L1 page-walk cache ≈ 10 cycles. A TLB miss that goes all the way to DRAM for every level ≈ 4 × ~300 cycles = 1200 cycles. In the worst case, walks themselves can miss in the data caches, multiplying the cost — this is why sparse random-access workloads are hellish.

### Q7. What is a TLB shootdown and why is it expensive?

**Answer:**

A **TLB shootdown** is the process of invalidating a page-table entry across all CPUs that might have cached it. It happens when the OS modifies a page table in a way that changes or removes a mapping — unmapping, permission change, swap-out, etc.

**Why it is multi-CPU:** On a multiprocessor system, every CPU has its own TLB. A mapping cached on CPU 0 is invisible to CPU 1 and vice versa. If CPU 0 changes a page table entry, CPU 1 might still be using the old translation in its local TLB — a correctness violation.

**The shootdown sequence:**

1. CPU 0 acquires the relevant lock (or uses RCU-style updates).
2. CPU 0 modifies the page table entry.
3. CPU 0 sends an **Inter-Processor Interrupt (IPI)** to every other CPU that might have the entry.
4. Each receiving CPU issues a TLB invalidation for the affected VA (or flushes its whole TLB in the bulk case).
5. Each CPU sends an acknowledgement back.
6. CPU 0 waits for all acknowledgements before proceeding.

**Why it is expensive:**

- **IPI latency:** Microseconds per IPI, not nanoseconds.
- **Synchronisation:** CPU 0 blocks until *every* peer acknowledges.
- **Target-CPU disturbance:** The receiving CPUs take an interrupt, flushing their pipelines.
- **Follow-on TLB misses:** After invalidation, the affected pages must be re-walked on next use.

A single shootdown can cost 10,000+ cycles across all affected CPUs. On a large system (128+ cores) touching a shared mapping, shootdowns can dominate runtime.

**Modern mitigations:**

1. **Batch invalidations:** The OS collects several invalidations and issues them in one IPI.
2. **Range invalidations:** On ARMv8.4+, a single instruction invalidates a whole range.
3. **Broadcast TLB invalidate:** Some ISAs have hardware-broadcast invalidation (Intel's INVLPGB introduced with Zen 4). This removes the IPI round-trip.
4. **Restricting mutations:** Linux's RCU-protected page tables avoid shootdowns for many common operations by arranging updates such that stale entries are benign.

**Interview insight:** The existence of shootdowns is why some performance-critical workloads deliberately use **mmap with huge pages or pre-touching** to minimise page-table churn — they pay the setup cost once to avoid thousands of shootdowns at runtime.

---

## Advanced

### Q8. You observe that L2 TLB miss rate is 3% on a memory-bound workload, with average page walk latency of 30 cycles. The workload has CPI 1.5 and executes 0.4 memory accesses per instruction. How much of CPI is page walk overhead? What would 2 MB pages change?

**Answer:**

**Page walk overhead calculation:**

Memory accesses per instruction: 0.4
Fraction that miss L2 TLB: 0.03
Walks per instruction: 0.4 × 0.03 = 0.012
Cycles per walk: 30

Cycles per instruction from walks: 0.012 × 30 = **0.36 CPI**

So 0.36 of the total 1.5 CPI (**24%**) is page-walk overhead. Not a minority contribution — it is the second or third largest slice after instruction execution and cache misses.

**Effect of 2 MB pages:**

The TLB reach increases by 512× (2 MB / 4 KB). A working set that overflowed a 1024-entry L2 TLB at 4 KB pages (covering 4 MB) now fits comfortably in 32 entries at 2 MB pages (covering 64 MB).

Assume the miss rate drops from 3% to 0.1% (realistic for a working set that was marginally too large). Walks per instruction: 0.4 × 0.001 = 0.0004. Cycles per walk: 20 (one level shorter). Overhead: 0.0004 × 20 = **0.008 CPI**.

Net saving: 0.36 − 0.008 ≈ **0.35 CPI**, or about **23%** of original CPI. The new CPI would be ~1.15 — a 23% throughput improvement.

**The interview takeaway:** huge pages are not an esoteric tuning option for a few workloads. On realistic memory-bound code, they routinely deliver 10–25% performance improvements. This is why mature JIT runtimes and database engines go to significant lengths to obtain huge-page backing.

**Caveat:** The example numbers assume perfect prediction of working-set benefit. In practice, huge pages can hurt on workloads with sparse access (only a few KB of each 2 MB page touched) because the prefetcher and cache now see noise from the unused parts of the page. Profiling before deploying huge pages is essential.

### Q9. Describe how nested (two-level) paging works for virtualisation and its performance cost.

**Answer:**

In a virtualised system, the guest OS has its own page tables that map *guest virtual addresses (GVA)* to *guest physical addresses (GPA)*. But GPAs are not real physical addresses — they are a guest abstraction. The hypervisor must further map GPAs to real *host physical addresses (HPA)*.

**Without hardware support:** The hypervisor maintains "shadow page tables" that directly map GVA → HPA, updated whenever the guest modifies its own page tables. This is correct but expensive: every guest page-table write must trap to the hypervisor for shadow-table maintenance. Context switches are also painful.

**Nested paging (x86 EPT, ARM Stage-2, RISC-V H-extension):** The hardware directly supports two levels of translation:

1. **Stage 1 / Guest tables:** GVA → GPA, walked using the guest's CR3.
2. **Stage 2 / Nested tables:** GPA → HPA, walked using a hypervisor-controlled register (EPTP on x86).

**The walker's work:** A single memory access from guest code triggers a walk of the guest page tables, but each guest page-table access is itself a GPA that must be translated through the nested tables. For a 4-level guest walk on x86-64:

- Guest L4 access → needs GPA→HPA (4 more accesses)
- Guest L3 access → needs GPA→HPA (4 more accesses)
- Guest L2 access → needs GPA→HPA (4 more accesses)
- Guest L1 access → needs GPA→HPA (4 more accesses)
- Final data access → GPA→HPA (4 more accesses)

That is up to **25 memory accesses for a single cold nested walk** (5 steps × 5 accesses per step including the final data). In practice, L1/L2 caches catch most of them, but a true cold walk is ruinous.

**Mitigations:**

1. **Nested TLB (combined TLB):** A TLB that caches GVA → HPA directly, bypassing the nested walk on a hit. Needed to make nested paging viable. Filled by the walker on cold misses.

2. **Large pages at both levels:** If both the guest and host use 2 MB pages, the walks are shorter at both stages. Linux KVM hosts are typically configured this way for production VMs.

3. **Nested page-walk caches:** Cache intermediate results of the nested walk itself, further reducing cold cost.

**Real performance impact:** On memory-intensive VM workloads, nested paging typically costs 5–15% runtime compared to bare metal. Most of this comes from cold nested walks and TLB pressure (the VM's working set now competes against other VMs on the same host).

**Why this matters in 2026:** Cloud VMs dominate server deployments. Understanding why VM workloads have higher memory-access overhead — and how huge pages fix it — is essential for anyone sizing cloud workloads or designing hypervisors.

### Q10. Explain Meltdown from the virtual memory perspective. What microarchitectural fact did it exploit, and what did KPTI sacrifice?

**Answer:**

**The background:** Before Meltdown, operating systems (Linux, Windows, macOS) mapped the kernel's address space into every user process's page table, but with the "User/Supervisor" bit set to Supervisor (kernel-only). The idea was that a user-mode instruction touching a kernel page would take a page fault — the permission check would block it. Having kernel pages always-mapped avoided TLB flushes on every syscall.

**The microarchitectural flaw:** On affected Intel CPUs, the permission check was enforced at **retire**, not at **execute**. Speculatively, a load from a kernel address in user mode could read the data, and dependent operations could use that data, before the permission fault was actually raised. The fault was deferred until the load reached the head of the ROB — but by then, side effects had already propagated to microarchitectural state (cache lines, TLB entries).

**The attack (simplified):**

```
mov  rax, [kernel_addr]    ; speculative: loads kernel byte into rax
and  rax, 0xff              ; isolate
shl  rax, 12                ; scale by 4 KB
mov  rbx, [probe_array + rax] ; touches one of 256 possible cache lines
```

After the fault eventually fires and the pipeline squashes, the attacker's cache state still reflects which of the 256 probe lines was touched. Timing each probe line reveals the kernel byte. Repeat to read arbitrary kernel memory.

**Why the virtual-memory design made this possible:** The kernel pages were present in the user process's page tables. The speculative load therefore had a valid translation; only the permission check objected. Without the mapping, there would be no TLB entry, no valid physical address, and speculation would be unable to read kernel data.

**KPTI (Kernel Page Table Isolation):** The fix is to maintain **two separate page tables per process**:
- A **user page table** containing only user pages plus a tiny stub of kernel code needed for syscall entry/return.
- A **kernel page table** containing the full user pages plus the full kernel.

On every user↔kernel transition, CR3 is reloaded, switching between the two tables.

**Costs of KPTI:**

1. **TLB flushes on every syscall and interrupt** (partially mitigated by PCID/ASID if available). Without PCID, KPTI was catastrophic — 20–30% slowdown on syscall-heavy workloads. With PCID (all recent x86), ~5%.

2. **Extra CR3 writes on every privilege transition.** Non-trivial latency.

3. **Duplicated page-table storage.** Each process now has two tables.

4. **Complexity in syscall entry/exit code.** Must carefully manipulate CR3 during the transition without dropping privilege checks.

**What it bought:** Meltdown is completely blocked — the kernel pages simply aren't mapped in the user page table, so there is nothing for speculation to read.

**Lasting architectural lesson:** The security of a microarchitecture depends not only on its committed state but also on what it *allows to happen speculatively*. Before Meltdown, nobody thought the user/supervisor bit needed to be checked before speculative execution; after Meltdown, every major CPU vendor redesigned speculation to enforce permission checks at execute time. Apple's M-series, ARM Cortex-A76 and later, and AMD Zen (from the start) check at execute and are therefore immune without KPTI. Intel's post-Coffee Lake hardware (Ice Lake forward) includes an architectural fix as well.
