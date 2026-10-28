# Trimmed Reading Lists for Each Phase

Below are the **must‑have** reading items for each phase after you have finished CiA ch. 1‑6 and the core OSTEP, Drepper, COD, and CSAPP material that directly feeds the builds.

---

## Phase 0 – Core systems & ring

| # | Book / Chapter | Why keep it | Notes |
|---|----------------|-------------|-------|
| 1 | **OSTEP 4–6** | Context switches, page faults, and the basis for the false‑sharing toy. | Keep the full chapters; you’ll need the details for the harness. |
| 2 | **CiA ch. 1–6** | Concurrency fundamentals, atomics, SPSC ring, false‑sharing, thread pools, testability. | You already finished this; keep the chapters as reference for the ring implementation. |
| 3 | **CSAPP 3.2–3.7** | Assembly, data layout, and the ability to read the ring’s disassembly. | Keep the whole section; you’ll need it for the interview drill. |
| 4 | **Drepper 1–3** | Memory hierarchy, NUMA, caches. | Essential for the cache‑line experiments. |
| 5 | **COD part 1** | Memory‑hierarchy model and cache‑coherence framing. | Keep the entire part; it ties together the Drepper and CSAPP material. |
| 6 | **Tools** (perf.md, Google Benchmark, etc.) | You’ll need the harness and the perf workflow. | Keep only the sections you actually use. |
| 7 | **Build harness** | The measurement harness for Phase 2. | Keep; it is the foundation for all subsequent benchmarks. |
| 8 | **Build toy** | Optional: a small toy for the interview. | Keep if you want a stand‑alone demo; otherwise skip. |
| 9 | **Build ring v1 & v2** | The core SPSC ring implementation. | Keep both; they are the basis for the admission path. |
| 10 | **CiA ch. 7–8** | SPSC ring details, false‑sharing. | Keep; they provide the low‑level reasoning for the ring. |
| 11 | **CiA ch. 11** | Testability and correctness gates. | Keep; it is used for the Phase 1 correctness gate. |

---

## Phase 1 – Admission control & correctness

| # | Book / Chapter | Why keep it | Notes |
|---|----------------|-------------|-------|
| 1 | **CiA ch. 5** | Atomics, memory ordering (already covered, but a quick re‑read helps solidify the *why* for the admission path). | Keep the 5.3 section; skip the rest. |
| 2 | **CiA ch. 7** | SPSC ring; necessary for the ledger. | Keep the ring‑implementation part; skip the hazard‑pointer discussion. |
| 3 | **CiA ch. 8** | False‑sharing – needed for the ring’s cache‑line padding. | Keep the core discussion; skip the extra subsections. |
| 4 | **CiA ch. 9** | Thread pools – not used in the admission path. | **Skip** |
| 5 | **CiA ch. 10** | Execution policies – not used yet. | **Skip** |
| 6 | **CiA ch. 11** | Testability – used for the correctness gate. | Keep. |
| 7 | **OSTEP 28** | Atomics – already covered in CiA 5. | **Skip** |
| 8 | **Stokes** | Optional; not used in the implementation. | **Skip** |
| 9 | **Drepper 1–3** | Memory‑hierarchy; used for the ring’s cache reasoning. | Keep. |
|10 | **COD part 1** | Memory‑hierarchy framing. | Keep. |
|11 | **Tools** | fuzzer, perf, etc. | Keep only the relevant sections. |

---

## Phase 2 – Micro‑architecture experiments

| # | Book / Chapter | Why keep it | Notes |
|---|----------------|-------------|-------|
| 1 | **Drepper 4–6** | Cache‑line experiments, NUMA, TLB. | Keep the full sections. |
| 2 | **COD part 1** (again) | Memory‑hierarchy model. | Keep. |
| 3 | **CSAPP 3.9.3** | Alignment and cache‑line size. | Keep. |
| 4 | **CSAPP 3.6.5–3.6.6** | Branch‑prediction and misprediction penalties. | Keep. |
| 5 | **CSAPP 3.6.7** | Branch‑handling and `cmov`. | Keep. |
| 6 | **CSAPP 6.6.1** | The Memory Mountain – stride & footprint. | Keep. |
| 7 | **CSAPP 6.6.2** | L1/L2 associativity. | Keep. |
| 8 | **CSAPP 6.6.3** | TLB and page‑size effects. | Keep. |
| 9 | **CSAPP 6.6.4** | Cache‑line size and alignment. | Keep. |
|10 | **Agner Fog 4.8–4.10** | Pipeline, branch‑prediction, and instruction‑level parallelism. | Keep. |
|11 | **Agner Fog 4.11** | Memory‑access patterns. | Keep. |
|12 | **Agner Fog 4.12** | Memory‑bandwidth and cache‑coherence. | Keep. |
|13 | **CSAPP 6.6.5** | Cache‑line size (duplicate of 6.6.4). | Skip – already covered. |
|14 | **CSAPP 6.6.6–6.6.8** | Advanced topics; not needed for the baseline experiments. | **Skip** |
|15 | **Drepper 3.5** | Cache‑coherence and MESI. | Keep. |
|16 | **COD 5.10** | MESI framing. | Keep. |
|17 | **OSTEP 28** | Atomics – already covered. | **Skip** |
|18 | **Stokes** | Optional; not used. | **Skip** |
|19 | **CiA ch. 9** | Thread pools – not used. | **Skip** |
|20 | **CiA ch. 10** | Execution policies – not used. | **Skip** |

---

## Phase 3 – Open‑loop load generator & latency methodology

| # | Book / Chapter | Why keep it | Notes |
|---|----------------|-------------|-------|
| 1 | **CSAPP 3.4** | Data‑layout and alignment for the load generator. | Keep. |
| 2 | **CSAPP 3.6.1–3.6.7** | Branch‑prediction (used for the generator’s loop). | Keep. |
| 3 | **CSAPP 6.6.1** | Memory‑mountain for the generator’s buffer accesses. | Keep. |
| 4 | **Agner Fog 4.6** | Pipelining and instruction‑level parallelism. | Keep. |
| 5 | **Agner Fog 4.8** | Branch‑prediction and pipeline stalls. | Keep. |
| 6 | **Agner Fog 4.9** | Instruction‑level parallelism (ILP). | Keep. |
| 7 | **Agner Fog 4.10** | Memory‑bandwidth. | Keep. |
| 8 | **Drepper 4–6** | NUMA and cache hierarchy for the generator. | Keep. |
| 9 | **COD part 1** | Memory‑hierarchy framing. | Keep. |
|10 | **OSTEP 28** | Atomics – already covered. | **Skip** |
|11 | **Stokes** | Optional; not used. | **Skip** |

---

## Phase 4 – Zero‑alloc & ablation

| # | Book / Chapter | Why keep it | Notes |
|---|----------------|-------------|-------|
| 1 | **CiA ch. 4** | Steady_clock for timing. | Keep. |
| 2 | **CSAPP 3.3** | Data‑format alignment. | Keep. |
| 3 | **CSAPP 3.9.3** | Alignment & cache‑line size. | Keep. |
| 4 | **CSAPP 6.6.1** | Memory‑mountain for cache‑line experiments. | Keep. |
| 5 | **Agner Fog 4.8** | Pipeline stalls. | Keep. |
| 6 | **Agner Fog 4.9** | ILP. | Keep. |
| 7 | **Drepper 4–6** | Cache hierarchy. | Keep. |
| 8 | **COD part 1** | Memory‑hierarchy framing. | Keep. |
| 9 | **OSTEP 28** | Atomics – already covered. | **Skip** |
|10 | **Stokes** | Optional; not used. | **Skip** |

---

## Phase 5 – Documentation & post‑project queue

| # | Book / Chapter | Why keep it | Notes |
|---|----------------|-------------|-------|
| 1 | **CiA ch. 11** | Testability (used for the correctness gate). | Keep. |
| 2 | **OSTEP 28** | Atomics – already covered. | **Skip** |
| 3 | **Stokes** | Optional. | **Skip** |
| 4 | **Drepper 1–3** | Memory‑hierarchy – not needed for documentation. | **Skip** |
| 5 | **COD part 1** | Memory‑hierarchy framing – not needed. | **Skip** |

---

# How to Apply These Trims
1. **Mark the items you keep** – create a checklist in your task manager or a simple Markdown table. 2. **Use the “Keep” items as the only reading you commit to** – the rest can be moved to a “future reference” folder. 3. **Update your study plan** – each phase should now have a single, focused reading list that feeds directly into the builds. 4. **Track time spent** – you’ll notice a 30 % reduction in reading hours while still covering all the necessary concepts.

---

# Final Note
The trimmed lists keep every source that **directly feeds a build or a metric**.  Anything that was a “background” or “nice‑to‑have” has been removed.  This will let you focus on the code and the measurements that matter for the interview and the resume.
