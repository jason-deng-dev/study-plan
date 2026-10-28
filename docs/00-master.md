# Inference Engine — Master Spec (v4)

*Read → Build → Measure → Explain.*

**What it is.** A low-latency inference serving engine in C++17: streaming clients → admission control → continuous-batching scheduler → a from-scratch transformer decoder → responses, all under measured TTFT and TPOT budgets. Every request passes a sub-microsecond NUMA-aware admission check — rate limits, per-tenant concurrency caps, circuit breaker, restart-safe kill switch — before the scheduler, which admits on a projected KV budget and batches at iteration level. An open-loop load generator (loopback) drives it with a sequenced request stream, so latency is measured under realistic concurrent load, coordinated-omission-correct. The CUDA kernel ladder (GEMM naive → register-blocked, roofline-analyzed) and the decoder's achieved-bandwidth claim share one machine, one harness, one ceiling table. Every optimization claim is backed by `perf stat` and an ablation study.

**v4.** The third project's domain pivots from a pre-trade risk engine to an inference serving gateway. The curriculum does not pivot — Phases 0–5, the never-cut list, the correctness machinery, and the perf playbook survive as a re-skin (§17 has the component mapping). New: Phase 6, the continuous-batching scheduler, previously "know-only, deliberately skipped." The AI spine is no longer an extension — the decoder (step G) and the kernel bench (step A) build inside this repo, and the old two-track ordering amendment is superseded.

**Sequence (v4):** Phase 0 → A (kernels) → G (decoder) → Phases 1–5 (gateway) → Phase 6 (scheduler), with H/B/C/D running on Orderbook and Backtester whenever their own gates are green. The kernels bench and the decoder ship as finished artifacts before the gateway needs them; the decoder's token-for-token gate is the last cheap moment to stop.

**This file is canonical.** It holds the operating rules, the sequence, and the cut order — the things that arbitrate disagreements. Phase sections say what to read and build. Weeks and deadlines live only in §2, never restated in a phase.

**Contents.** §1 operating rules · §2 sequence, sizing, cut order, non-goals · §3 foundations · §4–§10 phases 0–6 · §11 AI spine (steps A, G, H, B, C, D + post-engine tiers) · §12 reading map · §13 explain loop · §14 verification & observation · §15 quant anchors · §16 the final form (claim set, definition of done) · §17 decision record. The `perf stat` playbook is a standalone document with its own numbering, §1–§7: [perf.md](perf.md). Career material — resumes, interview map, algorithms, strategy — is [01-career.md](01-career.md).

---

## 1. Operating rules

1. **Bracket rule** — any `[N]` placeholder must be replaced with a measured value before the claim ships anywhere (resume, interview answer). Unfillable bracket = delete the clause. Extended: **an unbracketed assertion needs code behind it, not just a bracketed number.**
2. **Honesty rule** — every claim needs mechanism, measurement, boundary. Never claim what you can't defend under cross-examination.
3. **Weekly review (10 min) + cold recall (20 min)** — review: planned vs. actual hours per Toggl category. A stalled category is next week's first priority. Categories: Algorithms / Systems / Concurrency / Projects / Anki (≤30% of reading time) / Interview Prep (the 🎯 drill blocks). Then run the week's cold-recall session per §13 — it is the leg of the loop that otherwise quietly disappears.
4. **Resume is output, not input** — the resume appendices describe work that exists; they are targets until brackets are filled (single home: [01-career.md](01-career.md)).
5. **Hardware**: 9800X3D desktop (single NUMA node, Omarchy/Arch), RTX 3060 laptop; cloud for CUDA/NUMA measurement days. **Pace calibration**: ~100–140 dense pages/week with Anki; keep units small and phase-gated.
6. **Session setup**: new chat → attach this file → brief: *"You are my AI-infra / systems prep assistant. This spec is canonical. Operating rules as above. Check plan-vs-actual against Toggl when I report progress."* Nothing else needs attaching — the whole plan is this file.
7. **Document form — every work section opens with a `**Sequence:**` line.** One line, arrows, reading items and build items interleaved in the order they run. Reading and building are never two tracks: a build is numbered where its prerequisites are met, not collected at the end. A section that describes work but cannot state its sequence in one line is not yet specified enough to start. Reference sections (§12's reading map, [perf.md](perf.md)'s playbook, §16–§17) are exempt — they are lookup tables, not work.

---

## 2. Sequence, sizing, cut order, non-goals

**Two spines.**

| Spine | Purpose | Completion rule |
| --- | --- | --- |
| **Inference Engine Spine** | Systems, concurrency, correctness, and measurement through the serving engine — phases 0–6, with the CUDA kernel bench (A) and the decoder (G) built inside the same repo | Must stay green; nothing else activates at the cost of a skipped gate here |
| **Quant Anchors** | Keep Orderbook and Backtester sharp: the deepening queue, the ingestion ladder, plus AI steps H (feature layer), B (point-in-time datasets), C (training literacy), D (evaluation rigor) | Gated on each project's own prerequisites; pauses whenever the engine spine needs the hours |

**Relative size.** These are ratios, not a schedule.

| Phase | Content | Size |
| --- | --- | --- |
| 0 | OSTEP ch. 1–34 sequential w/ carve-outs + CiA full (ch. 1–11, depth-noted) + McKenney + Drepper §1–3 + harness + false-sharing toy + SPSC ring v1/v2 | the dominant block |
| 1 | Admission control core + correctness (fuzz/TSan/differential/replay) | ~1.5 units |
| 2 | Micro-arch: false sharing, NUMA, huge pages, branchless | ~1 unit |
| 3 | Open-loop load generator + pipeline integration + latency methodology | ~1 unit |
| 4 | Zero-alloc + ablation | ~1 unit |
| 5 | Docs + post-project reading queue | ~0.5 unit |
| 6 | Continuous batching + KV-budget admission (+ chunked prefill) — the centerpiece | ~2 units; a concurrent policy system, not a filter |
| A | CUDA + kernels bench (GEMM ladder, roofline, one quantized kernel) | 4–5 weeks |
| G | From-scratch decoder (token-for-token gate) | 1–1.5 weeks |

**On the clock — there isn't one.** Phase 0 is roughly equal to everything after it combined; each build phase is about one unit of work. Track **hours per category** (rule 3), not weeks elapsed. A phase that takes twice as long and lands green is not behind schedule; a phase that lands on time with a skipped gate is. **Shipped > ambitious.**

**Cut order if time runs short:**

1. SIMD batch checks (already a stretch — the design-doc Q&A bullets survive the cut)
2. **Second backend (ONNX Runtime) — cut before any core item**; the decoder backend alone carries the story
3. Half of Phase 2's experiments (keep NUMA + false sharing; branchless and huge pages are the cut candidates)
4. Phase 6 rung 2 (chunked prefill) — the scheduler ships at rung 1 with the ladder documented and rung 2 named as designed-but-unbuilt
5. **Phase 6 rung 2b (prefix caching)** — survives compression a little longer than chunked prefill (cheaper, and the more common production feature); the shared-prefix load-generator flavor is cut with it, since without that workload the rung is unmeasurable
6. Phase 6 preemption rungs (recompute/swap) and rung 3 (graph capture)
7. Load-generator realism (keep one recorded stream + one synthetic flavor)
8. Snapshot-recovery stretch (already demoted)
9. Step G's stretch items (sampling variants, GPU path); then the step-A Triton comparison (A6); then cut A5 (the quantized kernel). The GEMM ladder is never cut — the roofline it produces is what the decoder's bandwidth claim rests on

The decoder is not cuttable: it is the scheduler's backend, and phase 6 has nothing to serve without it.

**Never cut:** Phase 0 reading and the SPSC ring; the correctness gates (fuzz / TSan / differential / replay); **the scheduler correctness gate** (output equality across batch compositions); the decoder's token-for-token gate against llama.cpp; ablation; zero-alloc verification.

**What this project is not:**

- Not a protocol implementation — no OpenAI-compatible API, no HTTP/2 stack, no gRPC, no SSE streaming layer. A simple binary request protocol on loopback carries the signal (arrival rates, sequencing, backpressure) without protocol plumbing.
- Not a product — no auth, no model routing, no multi-tenancy beyond quota accounting, no admin UI. Tenant quotas exist to give admission control something to enforce.
- Not a model-serving framework — one model (GPT-2 124M), one backend by default. ONNX Runtime is an optional second backend and the first thing cut.
- Not an ML project — the model is a fixed, well-understood checkpoint and a deliberate placeholder. The work is the serving and performance layer; say exactly that when probed.
- Not a training system — no fine-tuning, no distributed training, no NCCL. Know the vocabulary, claim the boundary.
- Not kernel bypass **in this repo** — explain *why* it exists (syscall/context-switch overhead, measured in Phase 2) and know the physics cold. The engine's ingress stays a loopback request stream by design; the build lives where a real feed exists, in the Orderbook's ingestion ladder (§15.2). One repo builds it, one repo explains it, no claim in the wrong one.
- **Not distributed at all, deliberately.** One process, one node, a loopback load generator — no replication, no consensus, no multi-node anything. This is a real boundary for AI-platform roles, where the hiring signal is distributed systems. State it as a boundary rather than hiding it. The artifact for that door is chosen and post-engine (§11, Tier A item 1 and Tier B item 6): a hand-written ring all-reduce, its alpha–beta cost model, and the crossover measurement that decides when tensor parallelism pays — a *decision rule*, not a component. Do not add it pre-emptively.
- Not an AVX2 showcase — the design-doc Q&A bullets prove literacy; the build adds little.

**Interface rule.** The two spines meet at exactly two interfaces, both designed in Phase 1 before either consumer exists — the **model backend interface** (decoder today, ONNX Runtime optionally) and the **KV allocator interface** (contiguous today, paged in Phase 6). Those two seams are the only forward-looking design the engine carries. Inline hooks are marked **AI hook**.

**Learning-transfer checkpoint (end of every phase), answered out loud first, then written:** What did I learn? Where did I apply it? What measurement or test confirmed it? What is the boundary or exception? Where else does this concept transfer?

| Phase | Transfer you should be able to explain without notes |
| --- | --- |
| 0 | Memory ordering, SPSC mechanics, context switches, virtual memory, cache basics |
| 1 | Invariant design, fail-safe consistency, fuzzing, TSan, deterministic replay |
| 2 | False sharing, branch prediction, TLB pressure, SMT/NUMA topology, experimental control |
| 3 | Open-loop vs. closed-loop load, backpressure, tail-latency methodology, coordinated omission |
| 4 | Allocation behavior, object lifetime, interposition, ablation discipline |
| 6 | Iteration-level scheduling policy, KV memory management, admission under two SLOs, batch economics |
| Kernels (A) | CUDA execution model, coalescing, tiling, occupancy, roofline, quantization cost |
| Decoder (G) | Attention, KV cache, prefill vs. decode, transformer internals, correctness against a reference |
| Quant anchors (H/B/C/D) | Feature pipelines, leakage control, training literacy, model export, evaluation rigor |
| Python bridge (P) | pybind11 exposure, GIL boundary, pytest-driven verification, vectorized feature reference, notebooks over committed artifacts |

---

## 3. Foundations (what this build is quietly teaching you)

Foundations are a small set of first principles that *generate* answers — not book coverage. This plan builds five, deliberately:

| # | Foundation | Generative for | Where it's built |
| --- | --- | --- | --- |
| 1 | The memory hierarchy (caches, coherence, NUMA, TLB) | False sharing, `alignas`, huge pages, dense-vs-pointer-chasing, PMR pools | Phase 0 (Drepper §1–3) → Phase 2 (Drepper §4–6.4.1 + COD structured read + CSAPP 3.9.3 alignment + OSTEP virt) |
| 2 | The hardware memory model (store buffers, ordering) | Breaker/kill-switch signalling, SPSC correctness, seq_cst cost, every "why is this atomic enough" question | Phase 0 (OSTEP 28 + McKenney → CiA 5 full incl. 5.3 → SPSC ring v1/v2 — **built from zero**) → Phase 1 (5.3 re-read, immediately applied) |
| 3 | What the OS does for you and what it costs (syscalls, context switches, VM) | Kernel-bypass rationale, pinning, zero-alloc hot paths | Phase 0 (OSTEP processes + memory half, ch. 6/28) → Phase 2 (re-skim before experiments) → **Orderbook ingestion ladder** (§15.2), where the cost is removed in rungs and each rung is measured against the last |
| 4 | Transport-layer semantics (reliability, flow control, ordering) | TCP vs. UDP, flow control vs. admission control as backpressure, head-of-line blocking, capacity math | Phase 3 (Top-Down structured read incl. Ch. 3 full, applied to the request path and load generator) |
| 5 | The C++ object model (lifetime, UB, aliasing) | Interposition, placement new, every "is this UB?" code review | Phase 4 (cppreference) + your Effective C++/Modern C++ base |

If a phase feels like "just following steps," return to this table — each foundation is the *why* behind a cluster of build decisions.

---

## 4. Phase 0 — Pre-project reading

**Baseline: you arrive with zero concurrency background and no SPSC queue in your backtester.** Everything is built here from nothing. Front-loaded because atomics/memory-ordering bugs are correctness bugs in a serving gateway — you can't ablate your way out of a data race.

**Sequence:** OSTEP 1–11 → CiA 1–3 ∥ OSTEP 13–23 → 🔨 tools → 🔨 harness → 🔨 false-sharing toy → CiA 4 → OSTEP 28 → CSAPP 3.2–3.7 → 🔨 ring v1 → Preshing → McKenney → CiA 5 → 🔨 ring v2 → CiA 6–11 → OSTEP 32/33 → OSTEP 31 → Stokes → Drepper §1–3 → COD part 1.

**The list below is that sequence, item by item.** Reading items and build items are numbered together, in the order they run; each build appears where its prerequisites are met, not collected at the end.

1. **OSTEP 1–2** — read fully
2. **OSTEP 3** (dialogue) — optional (test the device; keep if it refreshes you)
3. **OSTEP 4–6** — read fully · **ch. 6 is the anchor**: LDE (context switches, traps) is the mechanism behind every data race
4. **OSTEP 7** — summary level (~15 min: intro + summary)
5. **OSTEP 8** — partial (intro + summary + §8.1; skip proofs/tuning)
6. **OSTEP 9–10** — summary level (~15 min each)
7. **OSTEP 11** — skip (summary dialogue)
8. **CiA ch. 1–3** — read fully, **∥ parallel with OSTEP 13–23** (orthogonal content, deliberate change of pace): what concurrency is, launching threads, mutexes/races/deadlock — the vocabulary, the first tools, and the "spotting race conditions in interfaces" eye the correctness gate needs
9. **OSTEP 13–15** — read fully
10. **OSTEP 16** — narrative pace (no cards, no re-reads; exists to explain why paging won)
11. **OSTEP 17–21** — read fully · ch. 21's page-fault mechanics are the whole zero-alloc story
12. **OSTEP 22–23** — skim
13. **Tools** (~2 hr): work through [perf.md](perf.md) (Sections 1–3 first, then the learning layers), `man perf-stat`, and the Google Benchmark README. **This item sits here, not at the end of the list** — it is the last prerequisite artifacts 1 and 2 are waiting on, and nothing below it that builds can run without it.
14. 🔨 **Build — artifact 1: the measurement harness.** *What:* a `perf stat` + Google Benchmark wrapper producing machine-readable output with reproducible, seeded workloads. *Why:* every claim for the next five phases depends on this being trustworthy; build it before you need it. *Where:* in the repo that owns the workload being measured. The first instance is the **Orderbook replay harness** named in §11 and gated in §15.1 — the Orderbook is the only artifact in the portfolio that already has a real workload. The engine's `bench/` instance arrives at Phase 2 and shares the *contract*, not the code: the [perf.md](perf.md) §3 fourteen-field experiment log, the [perf.md](perf.md) §5 event sets (one row per experiment), and raw output plus environment metadata archived under `bench/results/` on every run. *First workload:* a **synthetic seeded message stream** — deterministic input is [perf.md](perf.md) §3 step 3's own requirement, and it keeps the harness independent of the ITCH parse, which §15.2 gates *on this harness being stable*. The ITCH stream becomes the second workload at §15.2 rung 0, not the first. *Done when:* an experiment is defined by name, runs A/B/A with pinning and warmup separated from the measured interval, and writes `bench/results/<experiment>.md` with the [perf.md](perf.md) §3 log above the raw counter output. *Reach for:* [perf.md](perf.md) §3's ten-step workflow. **Not §14**, which holds no method section; earlier revisions of this file pointed there and the reference was wrong. *Learn:* measurement is infrastructure; an untrustworthy harness makes every later number worthless.
15. 🔨 **Build — artifact 2: the false-sharing toy** — your first interview prop. *What:* two threads updating separate variables that happen to share a cache line, measured before and after separation. *Standalone on purpose:* it carries no other work, so placement is the only variable that moves — do not fold it into the ring or the ledger, where a real handoff confounds the delta. *Reach for:* the [perf.md](perf.md) §5 false-sharing event set, `perf c2c` where available, and the controlled padding ablation as the actual evidence when generic counters stay ambiguous. *Done when:* the ✅ question below is answerable in two sentences, and the raw output is archived with the prediction written above it (§13.4). *Learn:* coherence traffic is invisible to the counters you'd expect to show it — the ablation, not the counter, carries the claim.
16. **CiA ch. 4** — full read; CVs/futures at comprehension pace; **§4.3 with full attention** (`steady_clock` vs `system_clock` — correctness for latency measurement and the rate limiter)
17. **OSTEP 28** — read properly, NON-NEGOTIABLE (test-and-set, CAS, LL/SC, fetch-and-add, spinlock costs, futex/two-phase locks — the hardware foundation under `std::atomic`, the ring, and the deep interview chains; CiA has no equivalent). **Read COD 2.11 *Parallelism and Instructions: Synchronization* alongside this item** (~10 pp): the ISA-level view of atomic exchange and LL/SC — the same subject, and synchronization rather than ISA mechanics, so it does not belong in the skip list.
18. **CSAPP 3.2–3.7** (~85 pp; v4.4 slice, widened v4.6): program encodings and AT&T formatting (3.2), data formats and the b/w/l/q size suffixes (3.3), operand specifiers (3.4), arithmetic — `leaq`, `imul`, `idiv` (3.5), condition codes, conditional moves and loops (3.6.1–3.6.7), procedures/stack frames/ABI (3.7). **Read contiguously from 3.2** — this is a skill chapter, and a skill does not survive being read at 60% (v4.6): 3.2 and 3.3 are the conventions everything after them assumes, and 3.5 and 3.6.7 are what you actually meet in real disassembly. This is the plan's only source that teaches reading a disassembly, and it is what makes the ring's verify question — *why x86 makes acquire/release nearly free but `seq_cst` needs a full fence, from your ring's disassembly* — answerable rather than quotable.
19. 🔨 **Build — artifact 3a: SPSC ring v1, all-`seq_cst`.** *What:* a bounded, pre-allocated single-producer/single-consumer queue. *Why:* it's the transport under the ledger, and where memory ordering stops being theory. *Reach for:* write it **by faith in `seq_cst`** on purpose — obviously correct, and it is CiA 7.3's own prototype-then-relax guideline, so writing it before CiA ch. 5 costs nothing pedagogically and buys an artifact early. TSan clean. *Learn:* the shape of the queue first, so item 23 has something concrete to relax. **Note what you dodged:** a bounded pre-allocated ring has no node reclamation, so hazard pointers and ABA (CiA 7.2) don't apply — know *why*, because you'll be asked.
20. **Preshing's two memory-ordering posts** — optional on-ramp (~1 hr): *The Synchronizes-With Relation*, *Acquire and Release Fences*
21. **McKenney, *Memory Barriers* §1–4** — read (the store-buffer model underneath Ch. 5)
22. **CiA ch. 5 — FULL read incl. 5.3** (synchronizes-with, happens-before, orderings, release sequences, fences). Budget 2–3× normal time for 5.3 — the densest material in the book and the one real wall; re-reading at Phase 1 still wins
23. 🔨 **Build — artifact 3b: SPSC ring v2.** Relax to acquire/release **one change at a time**, re-running TSan and the tests after each, with a written justification per relaxation in the commit message. *Done when:* v2 is TSan-clean, every memory order carries a written justification, and the disassembly question is answerable from your own build. That justification document is interview gold. *Learn:* which orderings are load-bearing and which are superstition. Its first real consumer is the Orderbook's parser → book-builder pipeline, the workload §15.2 rung 0 puts a real ITCH payload behind; the reasoning transfers to the gateway's dispatch path, not the Backtester (see the Unlocked note below).
24. **CiA ch. 6** — full read: design *reasoning* with attention (§6.1 guidelines); lock-based implementations are reference material — comprehend, don't type
25. **CiA ch. 7** — full read: **7.1 + 7.3 properly** (definitions; guidelines: seq_cst prototyping, reclamation, ABA, busy-wait); **7.2 at literacy depth** (hazard pointers, reference counting — know them, you'll cite why your bounded ring needs neither)
26. **CiA ch. 8** — full read: **8.2 read twice** (cache ping-pong, false sharing, NUMA distance, oversubscription — the C++-side view of the Phase 2 experiments; first pass now, second when Phase 2 starts); 8.3–8.4 normal; **8.5 at comprehension depth** (know exactly what parallel `for_each` does — your refusal answer)
27. **CiA ch. 9** — full read: thread pools; weight the *concepts* (queue contention, work stealing) over the implementations — you'll contrast pools with your SPSC model in interviews
28. **CiA ch. 10** — full read (short): `std::execution` policies — know precisely what you're refusing and why
29. **CiA ch. 11** — full read: designing for testability — read right before the correctness gate (Phase 1's last item)
30. **OSTEP 32 & 33** — read fully
31. **OSTEP 31 — skim.** Semaphores: producer/consumer, readers/writers concepts only. C++20 has `std::counting_semaphore`, but the POSIX API still appears in legacy OS-level IPC (shared memory)
32. **Stokes, *Inside the Machine*** — optional but recommended (~3–4 evenings; before Drepper): concept chapters fully, skim the Pentium 4/PowerPC case studies, never quote its (2006) implementation details
33. **Drepper §1–3** — read fully (~60 pp; Foundation #1 first half). Disposition of the rest so nothing is silently dropped: **§4–6 read in Phase 2**; **§7 *Memory Performance Tools* superseded by [perf.md](perf.md)** — skim only if it names a tool you don't recognize; **§8 *Future Technologies* skipped** (2007 predictions; measure your own machine instead)
34. **COD structured read, part 1 — alongside Drepper §1–3.** Read each COD section *after* the Drepper section it backs (second sources reinforce a model; they don't build it):

    | Drepper (item 33) | COD part 1 | Connection |
    | --- | --- | --- |
    | §1 Introduction — CPU/memory performance gap, commodity x86/x86-64 scope, and its standing caveat that vendor documentation is optimistic and measurement is the arbiter | — no counterpart — | Framing. Sets the model; the caveat is why every claim in this plan is measured rather than cited |
    | §2 Commodity Hardware Today — RAM types, DRAM access details | 5.2 Memory Technologies (skim) | SRAM/DRAM characteristics at textbook altitude; Drepper is deeper, COD gives the vocabulary |
    | §3 CPU Caches — structure, associativity, measurement; **§3.5 cache coherency + false sharing** | 5.1 Introduction, 5.3 The Basics of Caches, 5.4 Measuring and Improving Cache Performance, **5.10 Cache Coherence** | §3.5 ↔ 5.10 is the direct pair: MESI is the protocol behind Drepper's false-sharing measurement, and Phase 2's experiments measure it |
    | — no counterpart — | 1.6 Performance, 1.7 The Power Wall, 1.8 The Sea Change | Read these *first*, before the paired sections; framing only (CPU-time math, Amdahl, why multicore exists) |

    Numbering below is the **6th edition**, verified against its table of contents. Ch. 2, 3, 5, and 6 match the 5th edition; **only ch. 4 shifts** — the 6th inserts §4.5 *A Multicycle Implementation*, so pipelining starts at 4.6 and every later ch. 4 number is the 5th-edition number + 1. In the 4th edition, cache coherence is 5.8 — match by *title* when in doubt. Ch. 1 detail: **read 1.2** — the seven great ideas (common case fast, performance via parallelism/pipelining/prediction, hierarchy of memories) are the generative principles §3's foundations table restates, and they are the framing that survives into interviews; **skim 1.10**; skip 1.3–1.5, 1.9, 1.11–1.12.
    Ch. 5 detail — **Read** 5.1, 5.3, 5.4, 5.10 (5.3 + 5.4 are the mechanics behind Phase 2's experiments: placement, write policy, the 3Cs, replacement, multilevel caches; 5.10 is MESI). **Skim** 5.2. **Optional:** 5.8 *A Common Framework for Memory Hierarchies* — 4 pp unifying cache and VM under the same four questions. **Skip** 5.5, 5.6, 5.7 (VM — OSTEP owns it), 5.9, 5.11–5.12.

Scheduled elsewhere, not dropped: COD 2.12 in Phase 4 — **as a skim; CSAPP ch. 7 owns that ground from v4.4**; 4.6–4.11 in Phase 2; 3.5 and the GPU chapters (6.6, 6.7, App. C) in step A; the **matrix-multiply thread** (2.20 → 3.8 → 4.13 → 5.14 → 6.13) at steps A3–A4; 6.9–6.10 at Tier A item 1. Full skip-map in §12. **CSAPP likewise (added v4.4, targeted slice only):** 2.1.3 at §15.2 rung 0; 3.4/3.6.1–3.6.6/3.7 here at item 18 and 3.9.3/3.6.6 in Phase 2; 7.5–7.6 + 7.9–7.13 and 9.8.4–9.9 in Phase 4; 9.9 again at step G rung 0. Its ch. 4–6 and 10–12 are out of scope by design, and every lab is skipped.

**Recorded for completeness — do not schedule:** OSTEP 24–25 (dialogues) · 26–27 and 29–30 (covered by CiA in C++ form; **ch. 28 is read properly at item 17** — the earlier "26–30" range contradicted that item, corrected v4.6) · 34 (dialogue) · persistence 36–51 (optional 2-hr skim of 36/39/42; rest cut) · security 52–57 (skipped entirely).

**Anki discipline:** card what you'll use within weeks, not everything.

**Three gates, not a second track.** Three things sit deliberately outside the sequence above, each because something else must be stable first. The **ITCH parse** is §15.2 rung 0, gated on artifact 1 — it follows this phase rather than running inside it. The **engine's `bench/` instance** of the harness arrives at Phase 2, when the engine finally has a workload of its own. The ring is **carried into Phase 1** as the ledger's transport, which is what makes item 23's justifications load-bearing rather than academic.

**What artifact 1 gates downstream** — the reason it is built first and in the Orderbook: §15.1's slot map (bit-identical fills on the 1M-message replay), §11's step H (microstructure features), and §15.2's whole ingestion ladder. Building it is Phase 0 work; nothing downstream of it is.

**Transferable benchmark hygiene** (banked from the OSTEP ch. 19/14 homeworks — applies to every number you produce):

- **Observable side effects:** the compiler deletes loops whose results nobody reads — print/accumulate results, or use `DoNotOptimize` (ch. 19 hw Q5).
- **`steady_clock`, never `system_clock`;** warmup iterations separated from measured ones; enough repetitions that timer precision (ch. 19 hw Q1) can't distort the mean.
- **First-touch:** freshly allocated pages cost demand-zeroing on first access (ch. 19 hw Q7) — pre-touch buffers before timing, or your first iteration lies. (Ch. 21's measurement homework makes this visible; the sweep it describes becomes a Phase 2 artifact.)
- **Pin before measuring** (`sched_setaffinity` — ch. 19 hw Q6): unpinned threads bounce cores and inherit foreign TLB/cache state.

**🔓 Unlocked — the ring's reasoning transfers to the gateway's dispatch path, not the Backtester.** The obvious move is to port the ring into the backtester as the Monte Carlo job/result transport. Don't: that sampler is embarrassingly parallel with uniform per-path cost and exactly one output slot per path, so it is already correctly parallelized by index partition; a queue there adds handoff latency and a contended cacheline for no throughput. What transfers is the *justification discipline* — every memory order you argue for here governs the gateway's request → accounting ledger, a genuine single-producer/single-consumer path. If you want the ring in the backtester, §15.1 routes it to streaming ingest, where a producer/consumer pair actually exists.

**✅ Verify:** For the harness — can you name the primary metric, the single changed variable, and the archived provenance for one experiment without opening the file? Can you explain why x86 makes acquire/release nearly free but `seq_cst` needs a full fence — from *your own ring's* disassembly? Can you explain the toy's cache-miss spike mechanism in two sentences? Is your ring v2 TSan-clean with a written justification for every memory order?

---

## 5. Phase 1 — Admission-control core + correctness

**Sequence:** re-read CiA 5.3 + 4.3 → read the policy half (Nygard's stability patterns, rate-limiting algorithms, Little's Law) → build the admission core, tests first → read the libFuzzer and custom-mutator docs → build the breaker, ledger, and replay → read CiA 11 → run the correctness gate.

**Shape:** build the decision path, then attack it. Everything is judged by the correctness gate at the end, not by whether it runs.

**📖 Read:** **Re-read CiA 5.3** (the heart of Ch. 5 lands a second time with the breaker and ledger on your desk — re-reading beats first-reading for retention). Plus re-skim **CiA 4.3** as you write the rate limiter (`steady_clock`, not `system_clock`).

**📖 Read (domain reading, not systems reading):** admission control as a discipline. The systems half is covered; the *policy* half is not covered anywhere in the plan, and it is what makes each check a defended decision instead of a guess. In order: (1) **Nygard's circuit-breaker / stability patterns** (*Release It!* chapter, or Fowler's article as the short version) — breaker, bulkheads, load shedding: the vocabulary the gateway's control set is built from. (2) **Rate-limiting algorithms** — token bucket vs. leaky bucket vs. sliding window vs. GCRA; say what each bounds and what it lets through. (3) **Little's Law** and the queueing relationship between concurrency, arrival rate, and latency — the arithmetic behind every admission decision, and the vocabulary for phase 6's batching curves. *(Optional, only if the control list still feels guessed: skim one production gateway's documented policy — nginx `limit_req`/`limit_conn` or Envoy's circuit breakers and load shedding. One hour, no more; boundary reference, not a reading assignment.)*

**Why:** every design question in this phase has a policy answer a serving interviewer already knows. Why reject rather than queue, why concurrency caps and not just rate limits, what a breaker protects that a kill switch does not, what happens to in-flight requests when the breaker trips. **Boundary to state in the design doc:** a technical implementation of those patterns at loopback scale, not a production gateway.

**🔨 Build: the admission core.**

- **What:** the per-tenant state an admission decision reads, and the O(1) checks that turn it into accept/reject — rate limits (token bucket), concurrency caps, quota/notional-style accounting, request-size limits, duplicate-request detection.
- **Why:** this code runs on every request; its data layout sets the latency floor before a single micro-optimization.
- **Reach for:** cache-dense state — one tenant's decision inputs in one cache line (Drepper §3.3.2 finally pays off); allocation-free check paths; a token bucket for rate limiting; the branch-elimination habit tested properly in Phase 2.
- **Learn:** "O(1)" is meaningless until you say *where the data lives*; a latency floor is a layout decision, not a codegen decision.
- **Decide and document the rejection policy:** reject-at-capacity vs. queue-and-delay. Pick reject (fail-fast), state what it costs the refused client, note where queueing would be right instead. Same question phase 6 asks about the KV budget, one level down.
- *Tests first, implementation second.*

**Adversarial lens** (banked from the OSTEP ch. 8 homework audit): any allocation mechanism whose penalty rules trigger on naive behavior gets gamed by whoever understands the rules — the MLFQ gaming attack (fake I/O to avoid demotion) is canonical. Apply it: when you build the token bucket, ask *can a tenant pattern its requests to stay under the rate limit while maximizing consumed capacity?* (The MLFQ designers' answer was accounting-based: demote on allotment regardless of behavior.) Write the question and your answer into the design doc — it is also the fairness question phase 6 inherits.

**🔓 Unlocked — Orderbook (after the Phase 1 gate):** the dense, index-addressed state you just built is the pattern behind a generational slot map — the rewrite queued for the orderbook's order-ID index. You'll understand it well enough to replace your own cancel-by-ID container and justify it from cache behavior rather than fashion. (The generational counter is the same stale-access idea as the slot map's ABA defense — CiA 7.3 connects.) Queued in §15.1.

**📖 Read:** libFuzzer getting-started doc (~30 min), **plus the custom-mutator documentation (~30 min) — not optional here.** Your input is a structured request sequence, not a byte blob. A mutation fuzzer over raw bytes spends nearly all its cycles exploring the deserializer and generating sequences a real client could never emit, so the 10M-sequence number measures the parser rather than the admission logic. Two fixes, use both: a **custom mutator** emitting structurally valid requests and sequence-level edits (cancel a live request mid-generation, complete a request, duplicate an ID, reuse a tenant, arrive exactly at a quota boundary), and **differential testing** against a deliberately naive reference implementation — same input, both engines, assert identical accept/reject decisions. Differential testing is the strongest correctness claim in this project because it needs no one to enumerate the invariants: divergence *is* the bug report.

**🔨 Build: circuit breaker + ledger + replay.**

- **What:** a circuit breaker that stops admitting when the system is unhealthy and a restart-safe kill switch that refuses everything on command; an accounting ledger fed by completions (tokens served, bytes, per-tenant usage, in-flight counts); a replay log of the accept/reject stream.
- **Why:** the breaker is the engine's own overload protection; the ledger is the state every check reads; replay is the only way to prove the system is *deterministic* rather than merely *passing*.
- **Reach for:** a lock-free breaker signal readable on the hot path without a syscall or a lock — Phase 0's memory-ordering work, cashed in; your SPSC ring as the transport between acceptor and serving threads; `steady_clock` for anything time-based.
- **Learn:** how to design an invariant that survives concurrency instead of a lock that hides the question; determinism is a design constraint you build in, not a test you run at the end.
- **Decide and document the consistency model:** who owns the ledger, what the acceptor thread reads, what happens if it changes between check and admit. Pick fail-safe (reject) and defend it in code.
- **Before designing the replay format, read one writeup on deterministic record-replay / deterministic simulation** — Wilson's *Testing Distributed Systems w/ Deterministic Simulation* (talk) or Antithesis's deterministic-simulation posts. What you are buying is the taxonomy of nondeterminism sources (thread interleaving, time, addresses, FP) — which ones your replay must pin, and which you are quietly assuming away. Skipping it is how a "bit-identical" claim turns out to hold only on one machine.
- **Breaker state must survive a restart.** A trip flag held only in process memory means a crash, config reload, or supervisor restart silently re-admits traffic — a real incident class. Decide where the flag is persisted, what must be durable *before* the next request can be accepted, and how a restarting process learns it is still tripped. Cheap now, awkward to retrofit, and a good answer to "what happens if your process dies?"
- **Limits change while the engine runs — the config-publication problem.** New quotas arrive from an operator or config service while requests are in flight. Publish them to the hot path with no lock and no torn read. The mechanism has a name and a literature — read the seqlock and RCU chapters of McKenney's *Is Parallel Programming Hard, And, If So, What Can You Do About It?* (free, targeted) before choosing; then one design-doc paragraph on which mechanism you picked and why a plain `double` store would be wrong.
- **AI hook:** version every decision input and configuration in the replay format now, so a model version can be pinned without redesigning replay — and define the **model backend interface** here (a request in, tokens out), even though the decoder it will front does not exist yet. It is the seam phase 6 and step G both build against.

**📖 Read + ✅ Correctness gate (phase is not done until all pass):** Skim **CiA Ch. 11** first (~40 min — designing for testability, stress-testing techniques); then:

- Unit tests at boundaries: exactly at limit, one over, negative, zero, max int.
- **Ledger edge cases, as unit tests and as fuzz invariants:** a duplicate completion for an already-recorded request; a completion arriving out of order; a completion for an unknown or cancelled request; in-flight count going negative (the bug that means the cap leaked); a cancel-acknowledgement arriving after the request already completed; and the reconciliation question underneath all of them — what the ledger does when the client's view and its own view disagree. These are the probes an interviewer uses to find out whether the ledger was designed or merely written.
- libFuzzer: structured request sequences via the custom mutator (not raw bytes); invariants — in-flight ≤ cap after any accept, rate ≤ burst, breaker rejects everything post-trip, every admitted request is accounted exactly once. Run [10M] sequences.
- **Differential gate:** the same generated sequences through the reference implementation built alongside this gate and through the real engine; decisions must match exactly. Any divergence is a find, not a nuisance — log it either way.
- **Reference implementation — build it once, here.** A deliberately straightforward version of the same admission path: a mutex around shared state, `std::unordered_map` for the ledger, heap allocation per decision, early-exit branches, no cache work. Around a day. Not throwaway: it is the differential-testing oracle above, row 0 of the Phase 4 ablation table, and the honest answer to "why would I want yours instead of the obvious implementation." Building it in Phase 4 instead means writing it twice.
- TSan clean under concurrent load.
- Known-answer accounting tests (hand-computed usage totals over a fixed sequence).

**Artifact:** `gateway/` library + `gateway/test/` + fuzz harness.
**Claims this phase should earn:** *"Fuzzed admission path with [10M] random request sequences; zero invariant violations across rate-limit, concurrency-cap, breaker, and accounting guarantees; bit-identical decision replay across runs and thread counts."*

**🎯 Interview drills (Phase 1 ships → you answer these):**

- 5-level chain: why no mutex → contended mutex → futex → syscall cost → would you spin? why not on one core → how does the OS know you're descheduled?
- acquire/release vs seq_cst — what x86 actually emits (store buffers; McKenney).
- Walk through your SPSC ring; justify EVERY memory order. Why no hazard pointers/ABA (bounded pre-allocated ring)?
- Branchless: when is it WORSE? (your forced always-taken-branch benchmark story.)
- Why reject instead of queue? What does Little's Law say about the alternative?
- What does a circuit breaker protect that a rate limit does not?
- Thread pools: what they are, why serving frameworks use them, how your SPSC-spin model differs — the contrast is the answer.

---

## 6. Phase 2 — Micro-architecture

**Sequence:** read Drepper §4–6.4.1 + the OSTEP re-skim + the COD set → build the four cache and topology experiments → read Agner Fog + Lemire + CiA 8.2 + CSAPP 3.9.3/3.6.6/6.6.1 + the Mytkowicz paper → build branchless, the memory-hierarchy sweep, huge pages, and the dense-vs-hash writeup → verify.

**Shape:** controlled experiments on a machine you understand. Every claim here is a before/after with a prediction made *before* the run.

**📖 Read:** Three sources, two foundations (#1 and #3, §3):

- **Drepper §4–6.4.1** — virtual memory from the hardware view, NUMA systems (§5), and the programmer-facing sections (§6.4.1 false sharing, §6.5). Completes the Drepper backbone started in Phase 0.
- **OSTEP re-skim (30 min + Anki cards):** re-skim the address-translation/paging/TLB and `malloc`-internals chapters so the OS-altitude view sits fresh alongside Drepper's hardware-altitude view. Nothing new; any shaky card → re-read that section. (The syscall/context-switch cost model from ch. 6 — already in your deck — is the boundary condition for every "is this optimization worth it" judgment here.)

Read COD as the structured second source (§12 has the full skip list), sequenced *after* the corresponding Drepper sections:

- Drepper §1–3 → **COD 5.1, 5.3, 5.4, 5.10** — done in Phase 0 item 34; re-skim your highlights only.
- Drepper §4 *Virtual Memory* (4.3 TLB reach, huge pages) → **COD 5.7 (skim — OSTEP already built this; the value is the hardware view) + 5.8 *A Common Framework for Memory Hierarchies*** (the four questions applied to both cache and VM).
- Drepper §5 *NUMA Support* → **COD 6.5 *Multicore and Other Shared Memory Multiprocessors***, and **6.4 *Hardware Multithreading*** — read 6.4 *before* making pinning decisions, since physical-vs-logical core placement depends on it. (6.4 has no Drepper counterpart; it pairs with the SMT experiment.)
- Drepper §6 *What Programmers Can Do* → read as an index of levers, not a build guide. Its multi-threaded material revisits false sharing — **the primary treatment is §3.5, already read in Phase 0**; if the §6 subsections are renumbered in your copy, follow §3.5.
- Agner Fog → **COD 4.6 *An Overview of Pipelining* through 4.11 *Parallelism via Instructions***, with **4.8 *Data Hazards: Forwarding versus Stalling*** and **4.9 *Control Hazards*** the two that explain what a misprediction actually costs (a flushed pipeline is the mechanism behind every branch-miss number you will quote). 6th-edition numbering — the 5th called these 4.7 and 4.8.
- Phase 4 preview → **COD 2.12 *Translating and Starting a Program*** — **demoted to a skim (v4.4).** **CSAPP 7.5–7.6 + 7.9–7.13** is the real treatment of the link/load model behind symbol interposition, and it is read in Phase 4 where the interposition build actually is; there is no reason to spend the Phase 2 slot on the shorter version.

**Hardware reality (9800X3D, Omarchy/Arch):** 8 Zen 5 cores / 16 threads, **single CCD → one NUMA node**, SMT 2, 96MB L3. Consequences: (a) no cross-node memory exists at home — the real cross-node number comes from one cloud measurement day; (b) SMT topology and the huge L3 make your machine a *better* instrument for most of this phase than a server. Everything assumes native Linux (`perf`, `numactl`, `libnuma`, `madvise` all first-class on Arch).

**🔨 Build: the cache and topology experiments.** Four questions, four before/afters in the Phase 0 harness:

- **Does isolating hot shared state pay?** Apply cacheline separation to the state two threads actually touch — the ledger, the kill signal, the rate limiter. *Reach for:* `alignas` on the shared lines, `perf c2c` when generic cache counters stay ambiguous. *Learn:* false sharing is coherence traffic, not a cache miss — and why the ablation, not the counter, is the evidence. (The Phase 0 toy, now applied to real code.)
- **🔓 Unlocked — Backtester:** the same fix applies to the Monte Carlo queue head/tail counters — small work once you've measured the effect here, and it extends the false-sharing story across two projects.
- **What does SMT placement actually cost?** Pin hot threads to sibling threads vs. distinct physical cores and measure. *Learn:* why co-locating hot threads on siblings is a production mistake, and why your 96MB L3 makes it *less* punishing than a server. This is the at-home replacement for NUMA, and the COD 6.4 material applied.
- **Where do the cache-level crossovers sit on *your* die?** Dense-vs-hash and cache-pressure experiments have different thresholds on a 96MB-L3 consumer part than on a Xeon. Note provenance honestly in the writeup — "measured on a 96MB-L3 consumer die" is an interesting result, not a caveat. (Reasoning exercise, not a benchmark — see the dense-vs-hash item below.)
- **What is the real cross-node penalty?** Develop and unit-test the NUMA code path at home; it will correctly discover one node and pin everything there. Then rent a 2-socket bare-metal instance for one afternoon and measure same-node vs. cross-node atomics with the *same harness*. *Learn:* the ~5× remote-atomic cost your hardware physically cannot demonstrate. Annotate provenance ("developed on 9800X3D; cross-node numbers measured on 2-socket bare metal"). This number goes on your resume.
  - Rent the machine twice if the budget allows — a cold harness on a rented box is the classic way to come home with nothing. Validate the harness against a trivial workload first, then measure.
  - (Free, for code-path validation only: boot with one NUMA node faked into two to exercise per-node allocation and pinning. Emulated latency deltas are structural, not physical — never let those numbers into a claim.)

**📖 Read:** Agner Fog's microarchitecture manual — the branch-prediction chapter **plus the out-of-order execution and scheduler chapters** (added v4.6: COD 4.11 supplies the model, and these are the instance for it; the branch chapter alone leaves you able to name a misprediction penalty without being able to say what a reorder buffer does with the instructions behind it. Match chapters by title against the manual's own numbering) — plus two Daniel Lemire posts on branchless programming (~2 hours). Plus **CiA 8.2** (data contention/cache ping-pong, false sharing, NUMA distance, oversubscription) — the C++-code-side companion to Drepper's hardware view; read it back-to-back with the experiments. Re-read [perf.md](perf.md) §5–6 before running them. Also `man numa`, `numactl`. **Plus CSAPP 3.9.3 *Data Alignment*** (~3 pp, added v4.4) read with the false-sharing experiment — the padding and offset rule the toy and the ledger line both rest on — **and 3.6.6 *Implementing Conditional Branches with Conditional Moves*** read with the branchless build: `cmov` is the compiler-side form of the same decision, which is also the reason to check whether the compiler already made it before hand-writing it. **Plus CSAPP 6.6.1 *The Memory Mountain*** (~4 pp, added v4.5) read with the memory-hierarchy sweep — it supplies the two-axis (footprint × stride) framing the sweep borrows. *(Optional, third tier: **CSAPP 5.7.2–5.7.3 + 5.11.2 + 5.12**, ~30 pp — functional-unit latency/throughput, the abstract dataflow model, misprediction penalties, and *Understanding Memory Performance* (5.12, added v4.5): load/store ports, store-to-load forwarding, memory aliasing. It is the framing for "why do multiple accumulators beat one accumulator," and 5.12 is the vocabulary that makes the ablation's IPC column explainable rather than merely recorded. Where Agner Fog is deeper and correct for your specific core, use him.)*

**Plus one paper, and it is the most important read in this phase for the ablation's credibility: Mytkowicz, Diwan, Hauswirth & Sweeney, *Producing Wrong Data Without Doing Anything Obviously Wrong* (ASPLOS 2009, ~20 min).** Innocuous setup choices — memory layout, environment size, alignment, the length of a path string, link order — can shift measured performance by 5–10% in either direction, in otherwise methodologically correct experiments. That is larger than most of the effects this phase is trying to measure. Read it before you trust any single-digit before/after, then apply it: the ablation compares *separately built binaries*, exactly the situation the paper describes. The playbook's A/B/A interleaving and "don't claim 2% inside 10% noise" rule are the operational defence; this paper is why the rule exists, and citing it when you defend the method is a strong answer to "how do you know that delta is real?"

**🔨 Build: branchless, the memory-hierarchy sweep, and huge pages.**

- **Branchless decision path.** *What:* the combined admission check rewritten as straight-line arithmetic instead of a chain of early-exit branches. *Reach for:* prior prediction of `branches` and `branch-misses` before you run it — the experiment most likely to disappoint if you skip the prediction. *Learn:* when removing branches *loses* — build one benchmark with a forced, perfectly-predictable branch where predication is slower, and write the paragraph explaining why. That paragraph is the interview answer.
- **The memory-hierarchy sweep.** *What:* sweep the working set from L1-resident up past RAM into swap, recording per-access time (or achieved bandwidth) against footprint. *Why:* the resulting curve — L1 → L2 → L3 → RAM → the swap cliff — is the single best figure this project produces, and it's the same experiment as the TLB sweep with a longer x-axis. *Reach for:* `vmstat 1` in a second terminal for the swap regime (`si`/`so`/`free`/`swpd`); the OSTEP ch. 21 `mem` homework is this experiment already written — `mem.c` is in your `ostep-homework` checkout and prints bandwidth per loop, so the harness is mostly "sweep the size, collect, plot." *Learn:* the hierarchy as *your machine's* numbers rather than a textbook table — and that loop 0 pays first-touch faults while loops 1+ are warm, until past RAM, where loop 1 re-faults everything loop 0 evicted. **Predict both shapes before running** (§13).
  - **Add the stride axis (v4.5).** As written the sweep is one-dimensional — footprint only. Run it a second time at *fixed* footprint with a swept stride: the valleys at power-of-2 strides **are** L1/L2 associativity conflicts, which is how you measure your die's associativity instead of looking it up. Cost is one extra loop around the `mem.c`-derived harness; the payoff is that the dense-vs-hash item below stops being pure reasoning and gets a measured half. *Reach for:* CSAPP 6.6.1 *The Memory Mountain* (~4 pp) for the two-axis framing; Drepper §3.3 already supplies the mechanism (critical stride), so this is an experiment, not a reading.
  - **Boundary, honestly:** a beyond-RAM run thrashes the NVMe for minutes and can trip the OOM killer (`overcommit_memory=0` means the allocation *succeeds* and the process dies later). Run it from a TTY, capped; or measure the curve to RAM and label the swap regime *"read from OSTEP ch. 21, not measured."* Either is defensible. An unlabelled one is not.
- **Huge pages and TLB characterization.** *What:* apply huge pages to the hot data once the sweep has told you where TLB pressure actually bites. *Reach for:* `madvise`, `smaps` (`AnonHugePages`) for verification, `dTLB-load-misses` for the mechanism, the three THP modes as your controlled variable. *Learn:* why a page walk is mostly misses — the upper levels are tiny and shared across every translation so they stay cached; leaf entries are scattered across the whole address space so they don't. Huge pages attack both halves: fewer levels to walk, and each leaf entry covering more of the footprint. The elbows in the sweep curve are *your* machine's L1/L2 dTLB reach, not trivia.
  - **Pre-register the likely null:** if the hot working set already lives inside 96MB of L3, huge pages may buy nothing at the ledger. A measured null with a mechanism ("the footprint never leaves L3, so TLB pressure was never the bottleneck") is a *better* result than an invented win. Record it either way.
- **Dense-vs-hash:** half-day reasoning exercise (footprint vs. pointer chasing, cache-level crossover) written into the design doc. Not a benchmark.

**✅ Verify:** Can you explain why an always-taken branch beats predication? Why huge pages can hurt small/sparse working sets? Where the swap cliff sits on your curve — and why loop 1 costs more than loop 0 past it? Why remote atomics cost ~5× local — and why your 9800X3D can't demonstrate it (single CCD) but a 2-socket machine can?

**Skip:** prefetching (noise), AVX2 (stretch only), per-node breaker replication (understand it, cite it, don't build it).

**Artifact:** bench report — topology deltas, false-sharing before/after, branch-miss delta, memory-hierarchy sweep (L1 → swap cliff) + its stride × footprint surface + TLB curve, cross-node penalty (cloud-measured, provenance annotated), all perf-verified, all with predictions logged.
**Claims this phase should earn:** *"NUMA-aware admission path: [X]× p99 degradation on cross-node atomics (measured on 2-socket bare metal); cacheline-aligned ledger (false-sharing p99 improved [Y]×); branchless checks ([Z]% fewer branch misses, perf-verified, with documented case where branchless loses)."*

**🎯 Interview drills (Phase 2 ships → you answer these):**

- **Super-linear speedup**: N CPUs exceed N× when a working set fits the aggregate cache but not one core's — warm beats cold; migration re-colds. (Banked from OSTEP ch. 10 hw Q7 — concept only.) Same physics as your pinning experiment.
- **Guarantee math**: starving job needs ≥ f of CPU with quantum q → boost/rebalance interval ≤ q/f (10ms, 5% → every 200ms). One-liner, not a derivation. (OSTEP ch. 8 hw Q5.)
- Linux scheduling vocabulary: CFS = vruntime, slice = sched_latency/n with min_granularity floor, nice→weights, sleepers reset to tree-min on wake; CFS was default until Linux 6.6 → now EEVDF. (Lottery/stride = theory behind fair share; MLFQ = the other paradigm.)
- Kernel bypass: why does it exist? (your ch. 6 syscall-cost knowledge — the explanation is graded here, and built on real hardware in §15.2.) Huge pages: when do they HURT? Why your 9800X3D can't show remote-atomic cost but a 2-socket machine can.
- Event loop vs spinning receiver; what io_uring solved (post-dates OSTEP ch. 33).
- MESI at the protocol level (false-sharing mechanism; remote atomic ~5× local). Dense array vs hash: footprint vs pointer chasing, cache-level crossover. SIMD: gather/scatter, AVX2 vs AVX-512 downclocking, lane semantics.

---

## 7. Phase 3 — Load generator + integration

**Sequence:** read the Top-Down structured pass → build the open-loop load generator → read the socket API, Beej, and the wrk2 material → build the request path, staleness detection, and the full pipeline → read Tene and the HdrHistogram docs → build the latency methodology and run the overload measurement.

**Shape:** stop measuring a component and start measuring a *system*. Everything before this was a microbenchmark; from here the numbers include a network hop, a real arrival process, and concurrency.

**📖 Read:** **Top-Down structured pass** (§12): read 1.3–1.5 first — 1.4's delay/loss/throughput math is the back-of-envelope material interviews probe — then 2.1 (transport services frame) and 2.7 (socket concepts), then **2.2.2 *Non-Persistent and Persistent Connections* + 2.2.6 *HTTP/2*** — that is where the book's head-of-line-blocking discussion actually lives, and it is the argument your design doc has to make — then **Ch. 3 in full**: TCP vs. UDP, reliability, flow control, congestion control. Pay special attention to **3.4 *Principles of Reliable Data Transfer* — sequence numbers, ACKs, timeouts (3.4.1–3.4.2 are the stop-and-wait derivation; if time compresses, the load-bearing parts are 3.4.3 Go-Back-N and 3.4.4 Selective Repeat)**: your request stream's sequencing and staleness design reimplements 3.4's ideas at application layer, and you should say exactly that in the design doc. **3.7 (TCP congestion control: 3.7.1 classic, 3.7.2 ECN, 3.7.3 fairness)** and **3.5.5 (flow control)** matter more here than they would in a market-data framing: this is the first project where the transport pushes back on you.

**🔨 Build: the load generator — an open-loop request source.**

- **What:** a generator that replays recorded request streams (prompt lengths and generation lengths drawn from a real distribution) at a **configurable offered arrival rate**, independent of how fast the server responds.
- **Why:** closed-loop generators — send a request, wait for the response, send the next — measure a system that is never actually loaded: when the server slows down, a closed-loop client politely slows down with it, and the latency number is a lie. Open-loop arrival is the only way TTFT and TPOT mean anything under load, and it is the prerequisite for coordinated-omission correction in the latency-methodology build below.
- **Reach for:** a target rate as the single controlled variable; Poisson or bursty arrivals as a second flavor; a recorded trace as the realistic flavor; request IDs and application-level sequence numbers from the start.
- **Learn:** the load model is part of the measurement instrument, and an instrument you did not calibrate produces numbers you cannot defend.

**📖 Read:** socket API specifics — `man 7 tcp`, `man 7 ip`, **Beej's Guide to Network Programming (targeted: stream sockets + non-blocking + `poll`/`epoll`)**, and Top-Down's socket-programming section (~1.5 hours, reference-style, look up as you code). Plus **one hour on load-testing practice** — how wrk2 differs from wrk and why (open-loop + coordinated-omission correction is the whole difference; Tene below is the theory, this is the tool that implements it).

**🔨 Build: the request path, staleness detection, then the full pipeline.**

- **What:** (a) a loopback **TCP** request stream carrying a simple sequenced binary protocol, with the gateway accepting, admitting or refusing, and responding; (b) staleness detection on the input side — a generator that silently stops is more dangerous than one that is obviously broken, so the gateway needs a heartbeat/timeout path to notice, plus the sequence-edge cases: **32-bit wraparound**, where `next != prev + 1` is wrong at the boundary and the correct test is a distance computation on the unsigned difference (**RFC 1982**, *Serial Number Arithmetic* — ~4 pp, added v4.5: the canonical treatment of this exact test, and neither COD nor CSAPP covers it), and **stale-without-gap**, where the sequence advances perfectly but the workload is old; (c) the whole chain wired end to end — generator → gateway admission → **decoder behind the model-backend interface** (built in step G; a deterministic stub is acceptable here only if G has not landed) → responses → accounting ledger.
- **Why:** (a) is the transport decision made explicitly — TCP, because serving traffic wants backpressure and reliability, and because **head-of-line blocking is now a real design constraint rather than a market-data footnote**: one slow response on a shared connection delays the ones behind it, which is the entire argument for HTTP/2 multiplexing. State in the design doc why a real gateway would use HTTP/TLS and what the simple binary protocol deliberately omits. (b) is the failure mode that actually bites: a stalled client or a wedged generator looks like a healthy system with no traffic. (c) is what turns separate artifacts into one system whose latency you can defend.
- **Reach for:** monotonic request IDs as the application-level truth about what arrived; non-blocking accept with `epoll` on the server side; in-process wiring for everything except the loopback hop — that's the point, not a shortcut.
- **Learn:** application-level reliability is a reimplementation of transport-layer ideas with different tradeoffs; integration latency is not the sum of component latencies; a transport's flow control and your admission policy are two different backpressure mechanisms that must agree.
- **AI hook:** keep a clean, zero-alloc boundary where request features can later feed a model-version-pinned backend without coupling feature code to the serving loop (step H).

**📖 Read:** Gil Tene's *"How NOT to Measure Latency"* (talk or writeup, ~1 hour) on coordinated omission + HDR histograms, then the **HdrHistogram library's documenting pages** (recording, rescaling, merging, and what "p99.9" means when you merge two histograms — ~30 min). Tene gives you the theory; the library docs give you the mechanics that decide whether your percentiles are actually right.

**🔨 Build: latency methodology.**

- **What:** a per-stage percentile breakdown across the whole pipeline — accept → admit → enqueue → decode/prefill → first token → completion — under live load, reported as **TTFT and TPOT separately**.
- **Why:** this is where the project stops being a codebase and becomes a measured system. "TTFT p99 is [N]ms and TPOT p99 is [M]ms while serving [R] req/s with a [B]-deep resident batch on the same pinned node" is the sentence the whole phase exists to produce.
- **Reach for:** HDR-style histograms, warmup separated from the measured interval, coordinated-omission correction (the Tene read above is the prerequisite, not the background reading), and `perf stat -p` for aggregate CPU-state evidence alongside the histograms — never instead of them. Record the offered rate with every percentile; a latency number without its load level is not a number.
- **Learn:** why TTFT and TPOT are different measurements answering different questions, and why one "latency" number hides the tradeoff that phase 6 is entirely about.

- **Overload run — the failure-mode measurement.** *What:* one deliberately oversubscribed run, with the generator at roughly 5–10× the rate the receiver can drain, instrumented with the same histograms. *Why:* steady-state latency is the number you want; overload behaviour is the number that proves you understand your own system. Which stage saturates first, whether the transport's flow control or your shedding policy absorbs the excess, what p99.9 and the maximum do, whether the accept queue grows without bound, whether the breaker trips and at what threshold, and whether the kill switch still responds under pressure — a serving interviewer will ask at least one of these, and "I never ran it overloaded" is a bad answer. *Reach for:* the offered rate as the only controlled variable; listen-backlog and connection counters to tell transport drops from application refusals; your own admission counters as the leading indicator. *Learn:* a system's failure mode is part of its specification — a gateway that refuses fast is a different product from one that queues until it dies, and the design doc should say which one this is.

**✅ Verify:** Kill the generator mid-run — does the gateway detect staleness and stop waiting? Can you explain why TCP head-of-line blocking is a real constraint for multiplexed request streams, and what HTTP/2 does about it? Why does the receiver spin instead of using `select()`/`poll()`, and what problem did `io_uring` (post-dating OSTEP ch. 33) solve? Can you state your admission policy's behaviour at 2× capacity from the overload run's data?

**Artifact:** `loadgen/` + integrated pipeline + latency report (percentile tables committed as JSON under `bench/results/`; the plots rendered from them in `notebooks/` — step P).
**Claims this phase should earn:** *"Drove the engine end-to-end from an open-loop load generator over a loopback TCP request stream: per-stage TTFT/TPOT pipeline measured under concurrent load with coordinated-omission-corrected HDR histograms; documented overload policy and shedding behaviour at [X]× offered capacity."*

**🎯 Interview drills (Phase 3 ships → you answer these):**

- Open-loop vs. closed-loop load generation: what does a closed-loop generator hide, and why does that flatter your p99?
- Why TCP for request/response and UDP for market data (the reverse of the archived risk-engine design): what does each transport give the application that the other cannot?
- Head-of-line blocking: why it is fatal for multiplexed request streams, and what HTTP/2 multiplexing actually fixes (Top-Down §2.2.6).
- Capacity math: nodal delay components; propagation ≈ 5µs/km of fiber → the entire colocation conversation, and why loopback measurements have no propagation term.
- Flow control vs. admission control: two backpressure mechanisms, and what happens when they disagree.
- Socket tuning vocabulary: `SO_RCVBUF`, listen backlog, `epoll` edge vs. level trigger, `TCP_NODELAY` and why Nagle interacts with small responses.

---

## 8. Phase 4 — Zero-alloc verification + ablation

**Sequence:** read cppreference's object-model pages + CSAPP 7.1–7.13 (contiguous, v4.6) → build the zero-allocation proof → build the ablation table.

**Shape:** one proof, one table. Prove the hot path never allocates, then account for every optimization you've made.

**📖 Read:** cppreference on `operator new`/`operator delete`, **plus the object-model pages: object lifetime, strict aliasing / type punning, and placement new** (~1 hour). Foundation #5 made concrete: interposition hooks the allocation functions, and any future pool/arena work (including the PMR deepening on your Orderbook) leans on lifetime and aliasing rules. Plus **CSAPP 7.1–7.13** (~42 pp; v4.4 slice, widened to contiguous in v4.6): compiler drivers and static linking (7.1–7.2), object files and relocatable object files (7.3–7.4 — the ELF section layout that 7.5's symbol table lives inside), symbols and symbol tables, symbol resolution, **relocation (7.7** — the mechanism 7.12's PIC is built on, previously held as a drop-through and now simply read), executable object files and loading (7.8–7.9), dynamic linking with shared libraries, PIC, and **7.13 *Library Interpositioning*** in all three flavors — compile-time, link-time, run-time. This is the canonical treatment of exactly the mechanism this phase builds: the `LD_PRELOAD` library is 7.13.3 and the `--wrap` build is 7.13.2 under different names, so read it before writing either. **COD 2.12 is demoted to a skim** as the second source. The v4.4 drop-through of 7.7–7.8 is retired (v4.6): 7.8 is a prerequisite for 7.9, not a bonus, and 7.7 is what makes PIC more than a flag. That's all this phase needs — the concepts are already built.

**🔓 Unlocked — Orderbook:** this material is exactly what a pooled, intrusive per-level FIFO needs — nodes placed into a pre-committed arena rather than allocated, with lifetime and aliasing rules respected. Two artifacts from it: (1) re-run the 1M-message benchmark and cite the p99 tightening — the zero-allocation story then spans two projects and reads as an engineering principle; (2) **the determinism check**: fills must be bit-identical before/after the rewrite on the same golden replay — same fill sequence, same partial-fill behavior. That turns the rewrite into a correctness story, not just a perf one. Queued in §15.1. **Prerequisite read before that rewrite (added v4.4): CSAPP 9.9 *Dynamic Memory Allocation*** (~26 pp) — implicit and explicit free lists, placement and splitting, coalescing with boundary tags, segregated free lists. That is the design space your hand-rolled free-list reclaim is one point in, and it is the teaching version of what glibc's bins and tcache implement. Read it *before* the `MallocInternals` wiki, not instead of it: the wiki is glibc's instance of this design space and it postdates the book by a decade, so the order is design space first, then the instance.

**🔨 Build:**

- **Zero-allocation proof.**
  - *What:* every allocation in the process routed through your own accounting, with the benchmark failing if a hot path allocates after warmup.
  - *Why:* an allocation on the hot path is an unbounded latency event wearing a disguise — a lock, a page fault, a syscall, a cache eviction, all at once, at a moment you don't choose.
  - *Reach for:* global operator interposition rather than a custom allocator — interposition catches the hidden allocations inside third-party code, which is exactly where they hide.
  - *Learn:* "we don't allocate" is a claim requiring an instrument, and the interesting failures are in code you didn't write.
  - **Boundary of the instrument: interposing `operator new`/`delete` does not see raw `malloc`/`free`.** Any C library, and Google Benchmark itself, allocates through C functions that never touch the C++ operators — so a hidden allocation on the measured path would pass the gate while the gate reports clean. Close it rather than document it: interpose `malloc`/`free`/`realloc` as well, in the same preloaded library as the operators, and run the check as an `LD_PRELOAD` build so dynamically-linked callers are covered too (a `--wrap` at link time only catches calls resolved inside your own binary). Cross-check at least once with a heap profiler on the full run. The claim you are buying is "the process did not allocate on the hot path," and it is only as strong as the widest hook you installed.
  - **What the proof does not cover:** it shows nothing *new* was allocated — not what happened to the pages you already hold. Pair it with `page-faults` and RSS under live load, and answer this for the design doc: after warmup, why doesn't RSS shrink when the process frees memory? (Pages go back to the allocator and become page cache; the OS reclaims them only under pressure.) Twenty minutes of `vmstat 1` alongside the running engine answers it on your own machine. That's the honest boundary on the zero-alloc claim — *no allocation* is not the same as *no page traffic*.
- **AI hook:** keep the benchmark harness able to swap the synthetic workload for the real decoder behind the model-backend interface, and to record batch size as a first-class dimension. Do not build the backend here — the interface is defined in phase 1 and the decoder lands in step G.

- **Ablation study — the centerpiece.** One table, every "what did X buy you?" answered:

| Configuration | p50 | p99 | p99.9 | IPC | Branch-miss | dTLB-miss |
| --- | --- | --- | --- | --- | --- | --- |
| Reference (straightforward implementation) |  |  |  |  |  |  |
| Baseline (this architecture, optimizations removed) |  |  |  |  |  |  |
| + cacheline isolation |  |  |  |  |  |  |
| + branchless |  |  |  |  |  |  |
| + huge pages |  |  |  |  |  |  |
| + NUMA pinning |  |  |  |  |  |  |

Ablation discipline: one fixed workload and one focused `perf stat` event set across all rows; A/B/A or interleaved repetitions; every raw output and environment log under `bench/results/` per [perf.md](perf.md) §7. The table and its plots are rendered from that committed JSON in a Jupyter notebook (step P) — the notebook is a view, never the source of the number. IPC is explanatory, not the primary claim. Rows that measure as null stay in the table — a null with a mechanism is a finding, and removing it is how a table becomes marketing.

**Two baselines, and the difference matters.** The **reference** row is the implementation a competent engineer writes in a week: a mutex around shared state, `std::unordered_map` for the ledger, heap allocation per decision, a chain of early-exit branches. It is not a strawman — it is what the code looks like before you know what you know. The **baseline** row is *this* architecture with the optimizations removed one at a time, which isolates each optimization's contribution. The ablation table answers "what did each optimization buy"; the reference row answers the question a reviewer actually asks, which is "why would I want yours instead of the obvious thing." Report the final-vs-reference delta as the headline sentence and keep the ladder underneath it as the evidence. The reference implementation is built **once**, in Phase 1, and serves two purposes: the differential-testing oracle for the fuzzer and row 0 of this table.

**Never cut:** correctness (Phase 1), ablation, zero-alloc.

**Artifact:** `bench/` with ablation table + histograms.
**Claims this phase should earn:** *"Verified zero hot-path allocations via global new/delete interposition; ablation study quantifying each micro-arch optimization's p99.9 contribution; measured on the integrated serving pipeline under open-loop load."*

---

## 9. Phase 5 — Docs + post-project reading queue

**Sequence:** calibrate the genre against two public design docs → write the README → write the design doc and its Industry-context appendix → write the vocabulary layer → queue the post-project reading.

**📖 Read: calibrate the genre first.** Read one or two respected public design documents before writing yours — a RocksDB or LLVM design doc for the systems genre, and one inference engine's architecture documentation (vLLM's design docs, TensorRT-LLM's architecture pages, or llama.cpp's internals notes) so you know how this specific system is described by the people who build them. Two hours. The failure mode for a first design doc is writing a README with more words: the genre answers *why this design and what it cost*, never *what the code does*.

**🔨 Build: README and design doc.** README (every claim linked to `perf stat` output, Google Benchmark JSON, or a `bench/` table); design doc (3–4 pages, four subsystems):

- **The two latency budgets — the denominator for every TTFT/TPOT claim.** A latency claim without the budget it lives inside is a benchmark, not an argument. State what TTFT and TPOT each cost at this scale and why (admission, queueing, prefill compute, decode bandwidth), state the SLO you set for each under a stated offered rate, and state what would have to be true for your numbers to matter. This section turns headline numbers into design decisions, and it is the first thing a sharp interviewer asks for. **Make it operational:** an error budget per SLO, what burns it, and the snapshot-published counters that report it — published from a seqlock-protected snapshot rather than read from the hot path, because a sub-microsecond path cannot be scraped. Mechanism in §14; the build is post-engine Tier A item 3.
- **The overload policy.** From the Phase 3 overload run: which stage saturates first, whether the transport's flow control or your shedding policy absorbs the excess, what the breaker does and at what threshold, and what happens to in-flight requests when it trips.
- **The admission policy.** Reject-at-capacity vs. queue-and-delay, why fail-fast, and the Little's Law arithmetic behind the choice. One paragraph, defensible on a whiteboard.
- **Final-versus-reference delta.** The headline comparison against the straightforward implementation (mutex, `std::unordered_map`, allocating), with the Phase 4 optimization ladder underneath as the evidence.
- **The scheduler section — stub it here, fill it in phase 6** (phase 5 runs first in the v4 sequence; leave the heading and the questions in place so phase 6 appends rather than improvises): the rung ladder and its measured deltas, the KV-budget arithmetic, the admission projection, the chunked-prefill decision, scheduler overhead per admission, and the **FP non-associativity / output-equality** writeup — why batched kernels can differ in the last bits, what your gate therefore proves, and where the guarantee stops.
- **The decoder correctness story.** Single-stream token-for-token against the reference first, concurrency second; why the KV allocator sits behind an interface and what that buys.
- **The kernel ladder.** Rung-by-rung numbers against the roofline, the occupancy sweep, and the quantized kernel's measured throughput gain and accuracy cost.
- **The observation-cost section** (from §14): what it costs to watch a sub-microsecond path — the measured uprobe perturbation, why tracing was replaced by an in-band seqlock-published histogram, and which observations were kept in eBPF because they cannot perturb the path. This answers "how would you debug this in production?" with a measurement instead of a tool name.

The README, design doc, raw benchmark outputs, fuzz/TSan evidence, replay hashes, and batching tables together form the **core proof pack** (career timeline: [01-career.md](01-career.md)).

**Design doc appendix — "Industry context" Q&A** (the home for keywords that must NOT go on the resume, each answered from your own measured system):

- **vLLM / PagedAttention, Orca, TensorRT-LLM, SGLang**: what each contributes (paging, iteration-level scheduling, kernel fusion and graph capture, prefix caching), and how your scheduler's choices map onto theirs and where they differ — answered from your rung ladder, not from the papers.
- **Continuous vs. static batching**: the head-of-line argument, from your own rung 0 vs. rung 1 numbers.
- **Quantization (INT8/FP8/GPTQ/AWQ)**: what each buys and costs; your INT8 kernel's measured accuracy delta is the anchor, and the precision ladder (COD 3.5) is the mechanism.
- **KV cache management**: paging, eviction, prefix caching, and the memory math that caps batch size — from your own budget arithmetic.
- **Speculative decoding**: know the vocabulary and the tradeoff (draft model cost vs. tokens accepted); don't claim a build.
- **Distributed inference (tensor/pipeline parallelism, NCCL)**: boundary answer — single-node, deliberately. **This script is owed at the Stage-1 gate, not here.** Applications start at Phase 1 ([01-career.md](01-career.md)) and Phase 5 lands months behind that, so write it when applications start. The argument: the artifact for this door is a *decision rule, not a component* — a ring all-reduce costs `alpha + beta*n` (one latency term, one bandwidth term), so there is a message size below which tensor parallelism loses and above which it pays; TP needs two all-reduces per layer, so the latency term is multiplied by the layer count before it is weighed against the compute it saves; the crossover is a per-interconnect measurement (1GbE, 2.5GbE, loopback, PCIe, NVLink), not a rule of thumb. Then the boundary: the project is single-node by design, multi-GPU hardware is rented rather than owned, and the extension is priced and sequenced as Tier A item 1 / Tier B item 6 rather than omitted. Close on the frame — *"I priced tensor parallelism instead of demoing it, and on my interconnects the crossover says don't"* — because a decision rule survives cross-examination and a two-GPU demo of a model that fits on one GPU does not. **NCCL is never claimed as a build**; on the rented node it is the reference your hand-rolled ring is validated against.
- **Serving stacks you did not build (HTTP/2, gRPC, Triton, Ray Serve)**: one paragraph each on what problem they solve, framed against your simple binary protocol.
- **Kernel bypass / DPDK / RDMA**: the mechanism is your Phase 2 measurement (syscall + copy overhead). Frame it with the scope stated: this repo's ingress is loopback *by design* — the bypass build lives in the Orderbook (§15.2). So: "I know the measured cost these eliminate, I built the AF_XDP version where it applies, and here is why it does not apply to a load generator." **DPDK and RDMA stay explain-only** — know the mechanisms (poll-mode drivers; kernel-bypass verbs, one-sided operations, why RDMA needs a lossless fabric) and say plainly that you did not build them.

**Vocabulary layer — the standard AI-infra probes this project does not build.** One paragraph each plus a boundary line, capped the way the entry above is. Sources in §12. The premise: *"I know what it solves, and here is why this project does not have it"* beats silence — and both beat a claim you cannot defend.

- **Prefix caching**: KV reuse across requests that share a prefix. *Built*, as phase 6 rung 2b — answered from your own hit rate and TTFT delta, and from the ownership change it forced (exclusive blocks → reference-counted, immutable while shared).
- **Training-side distributed — FSDP/ZeRO, pipeline parallelism, mixed precision, checkpointing and fault tolerance**: what each shards or bounds, and why the failure modes differ from inference (optimizer state, activation memory, straggler tolerance). Boundary: single-node inference; distributed *training* shares vocabulary and little else.
- **Tensor cores and architecture-specific paths — CUTLASS, Hopper TMA and `wgmma`**: what a tensor core does differently from a CUDA core (a small matrix multiply per instruction, with a precision contract attached), and what TMA changes about data movement. Not purely boundary — it is the mechanism behind one of the causes A5's cuBLAS gap attribution names.
- **Multi-LoRA serving — S-LoRA, Punica**: many adapters over one base model, batched; why an adapter is a small rank decomposition and how that interacts with the batch dimension. Boundary: one model, one backend, by design.
- **Structured / constrained decoding — `xgrammar`, `outlines`**: masking logits so output conforms to a grammar or schema; why it is a per-step mask rather than a post-hoc parse. Boundary: not built — this project samples greedily.
- **KV cache offloading — FlexGen and vLLM's offloading**: moving KV to host memory or disk under a bandwidth budget, and why the transfer cost rather than the capacity is the binding constraint. Boundary: device-memory-only.
- **GPU sharing and multi-tenancy — MIG, MPS, time-slicing**: the three ways one device gets split, and what each isolates (memory, fault domain, QoS) versus what it does not. Boundary: one process, one device.
- **Evaluation harnesses — `lm-evaluation-harness`**: what a benchmark suite measures, and why a token-for-token equivalence gate is a different instrument answering a different question. Boundary: correctness is equivalence to a reference, not a score.

**📖 Post-project queue — only after shipping, sized for interview prep. Nothing on it gets read during the build; it exists so you don't feel the itch to "finish the list" instead of shipping.**

- **DDIA Ch. 1–2, 5–6 skim** (consistency models, replication) — for distributed-flavored system design rounds.
- **Alex Xu Vol. 1 as Q&A flashcards** (caching, rate limiting, load balancing, and the "design a rate limiter" chapter especially). **Vol. 2: skip** (it's for "design Twitter" rounds you won't face).
- **OSTEP persistence skim (ch. 36, 39, 42 only — ~2 hours)** — I/O-device model, file API, journaling vocabulary. The rest of the persistence half is cut, not deferred.

---

## 10. Phase 6 — The scheduler (continuous batching)

**Sequence:** read Orca + vLLM/PagedAttention + Pope et al. + a working scheduler's source + CSAPP 9.9 → build rung 0, the naive reference scheduler → build rung 1, continuous batching with KV-budget admission → read Sarathi-Serve → build rung 2, chunked prefill → read the vLLM prefix-caching docs + RadixAttention → build rung 2b, prefix caching → rung 3 is stretch.

**Shape:** the first concurrent *policy* system in the project. Everything before it was a filter or a pipeline with a fixed path; here the path itself is the thing being designed — which requests run together, which wait, which are refused, and what that costs the ones already running.

**Gate:** phase 1 green (admission path, ledger, correctness machinery exist), step A shipped (roofline and bandwidth reasoning), and step G's decoder passes its token-for-token gate single-stream. The scheduler is a concurrency layer over a correct engine — never a way to make an incorrect engine look busy.

**Why it is the centerpiece:** continuous batching separates a serving *engine* from a serving *wrapper*. It is also the only part of this project whose design is policy rather than mechanism, which makes it the most interviewable thing built here — and the easiest to get subtly wrong.

**Decisions taken at adoption, so phase 6 does not silently choose them:**

- **Ladder, not monolith.** Rung 0: the naive reference scheduler (one request at a time, no batching) — the differential oracle. Rung 1: iteration-level continuous batching with KV-budget admission. Rung 2: chunked prefill. Each rung is measured before the next is written; a rung that buys nothing is a finding worth keeping. Same idiom as the GEMM ladder.
- **Chunked prefill: yes, as rung 2.** It is the modern design (vLLM V1, TensorRT-LLM, SGLang) and the only way the TTFT/TPOT tradeoff is real rather than nominal — without it a long prompt stalls the whole batch. It is also the strongest "why is this hard" answer the project produces.
- **KV allocation goes behind an interface from day one**, even while the implementation is contiguous. Paging can then replace the allocator in phase 6 without touching attention.
- **The differential invariant is output equality**, not merely "zero divergences": batching is a scheduling change, not a numerical one, so token outputs must be identical across batch compositions and across thread counts.
- **Admission projects the worst case** — prompt length + `max_new_tokens` — and rejects at capacity rather than preempting. Preemption (recompute or swap) is a stretch rung. Reject-at-capacity is the direct analog of the risk engine's fail-safe reject, and defensible in one sentence.
- **Scheduler overhead per admission decision is measured and reported**, because "admission control adds [N]ns to a decision that costs [M]µs" is the sentence that makes the gateway a systems project rather than a wrapper.

### Rung 0 — 📖 Read, then 🔨 the naive reference scheduler

**📖 Read:** **Orca** (Yu et al., OSDI 2022) for iteration-level scheduling, then **vLLM / PagedAttention** (Kwon et al., SOSP 2023) for KV paging and the memory-budget mechanism. Both are short. Read for the *decision* each paper makes and what it trades away, not for implementation detail. Skim **Pope et al. (2022)** for the batch-economics and KV-memory arithmetic you will reproduce in your own sweep. Then **read one working scheduler's source** the way you read llama.cpp for the decoder: vLLM's scheduler and block manager — Python, read for the policy, not the craft. It is a reference implementation of exactly the design you are about to build, and the papers leave out the parts that decide your data structures (how the running/waiting/swapped sets are represented, when preemption fires, how block tables are allocated and freed). **Plus CSAPP 9.9 again, here (added v4.4):** the KV block allocator behind PagedAttention's block table is an allocator problem in its purest form — fixed-size blocks, allocate and free out of order, fragmentation as the enemy — so 9.9 supplies the vocabulary for the interface phase 1 stubs and this rung fills in. Read it as the design space, and vLLM's block manager as the instance.

**🔨 Build:** the deliberately naive reference: one request at a time, no batching, FCFS, no admission policy beyond "queue everything." Around a day.

**Why first:** it is the differential oracle, exactly as the naive admission path was in phase 1 — and row 0 of the batching table, the honest answer to "what did batching actually buy." Build it here or write it twice.

**Learn:** what a scheduler *is* — an ordering policy over work that arrives asynchronously — before it is an optimization.

### Rung 1 — 🔨 Continuous batching + KV-budget admission (the ship line)

**What:** iteration-level scheduling. Each decode step iterates over the resident batch; new requests join mid-generation at the next iteration boundary; finished requests drain and free their KV. Admission is a projection: a request enters only if `prompt_len + max_new_tokens` fits the KV budget (worst case, reject-at-capacity). No preemption at this rung.

**Reach for:** the KV allocator behind the interface chosen in the decoder phase — contiguous at this rung, replaceable without touching attention; iteration-boundary joining so no request waits for the whole batch to finish; an O(1) admission test against a running KV total; the same SPSC dispatch and admission-control patterns the gateway already uses.

**Why it pays:** the batch is the unit of throughput, and decode is bandwidth-bound — every additional resident sequence amortizes the same weight read. That is the whole economic argument, and the benchmarks below are where it becomes your number instead of a paper's.

**🔓 Unlocked — decoder:** this is where KV *memory management* becomes a design problem rather than an allocation. The decoder's contiguous cache is the honest single-stream choice; paging exists because of this phase, and the interface you built in step G is what lets you say so with evidence.

**Decisions to document (one paragraph each):**
- Why worst-case projection and reject-at-capacity, versus admit-on-prompt and preempt-on-growth. Name what the alternative would cost (recompute or swap) and why it is a stretch rung rather than the default.
- Where the KV budget number comes from: total device/CPU memory, weights, activations, and the arithmetic from Pope et al. reproduced on your own machine.
- Fairness: what prevents one long-lived tenant from consuming the budget. State the policy and its failure mode. The standard machinery has a name — weighted fair queueing and deficit round robin — and you already own the primary source: **Top-Down §4.2.4 *Where Does Queuing Occur?* and §4.2.5 *Packet Scheduling*** cover FIFO, priority queuing, round robin, and WFQ in ~6 pp, which is the treatment this decision needs (the router's scheduler and yours solve the same problem). Read them before inventing a policy — "how does your scheduler stay fair across tenants" is a standard serving-interview question, and "the caps handle it" is a weaker answer than a named policy with a stated starvation boundary.

### Rung 2 — 🔨 Chunked prefill

**📖 Read:** **Sarathi-Serve** (Agrawal et al., OSDI 2024) — the origin of chunked prefill and the source of its actual mechanism: why splitting prefill into fixed-size chunks and piggybacking them on decode steps bounds the stall, and what it costs. Read it *before* building rung 2; the paper's throughput-latency argument is the thing you will reproduce on your own machine, and defending without it means defending "vLLM does it."

**What:** split a long prompt into chunks so prefill work interleaves with decode steps instead of stalling the batch. TTFT for the newcomer trades against TPOT for the residents *by policy*, not by accident.

**Why rung 2 and not rung 1:** it only becomes meaningful once rung 1's numbers exist to compare against. Its value is the delta it produces in the rung-1 curves — without those, it is unmeasurable and therefore unclaimable.

**Learn:** prefill is compute-bound and decode is bandwidth-bound, and co-scheduling them is a resource-partitioning problem — the same reasoning as mixing two workloads on one core, which is Phase 0 material applied to a new resource.

**Cut rule:** cut item 4 in §2. If time compresses, the scheduler ships at rung 1, the ladder is documented, and rung 2 is named as designed-but-unbuilt. Never claim it unbuilt.

### Rung 2b — 🔨 Prefix caching (KV reuse across requests)

**Letter, not number, on purpose:** this is an orthogonal axis to the prefill/decode ladder — it reuses work across requests rather than changing how one request's work is scheduled — so it slots in beside rungs 1 and 2 rather than after them.

**📖 Read:** vLLM's prefix-caching and block-hashing documentation, then **SGLang's RadixAttention paper** (Zheng et al., 2023) for the data structure. Read the paper for the *choice*: a radix tree over token sequences rather than a flat map keyed on complete prefixes, and what that buys when prompts share partial prefixes instead of exact ones. Half a day.

**What:** retain the KV blocks of prefixes more than one request has computed, and let a later request that shares a prefix begin decoding from those blocks instead of re-running prefill over them. Today the prefill for a shared system prompt is repeated once per request; this converts that compute into a lookup.

**Why it earns a rung:** it is the standard production feature — every serious serving stack has it — and it is the first point where the KV allocator stops being an allocator and becomes a *cache*, which means ownership stops being exclusive. That is the real change. Rung 1 gives each sequence its own blocks and frees them on completion; rung 2b lets two sequences reference one block, so a block is freed only when its last reference goes.

**Reach for:** block-level hashing — hash a full block's token IDs chained with its parent's hash, so a block's identity encodes its entire prefix; a reference count per block; and the immutability rule that makes sharing safe: a shared block is never written, only the final partial block of a sequence is mutable, so appending allocates a fresh block rather than mutating a shared one. Get that rule wrong and the failure is silent cross-request corruption.

**Measure:** hit rate on a workload with genuinely shared prefixes — which means the load generator needs a shared-prefix flavor, because on distinct prompts the rung's delta is zero and *that* is the finding; TTFT delta on hits against the same workload with caching off; the memory cost of retained blocks and the eviction policy that bounds it (LRU over refcount-zero blocks).

**Learn:** caching as a serving mechanism rather than a memoization trick; why prompt structure — shared system prompts, multi-turn history — decides whether a cache is worth its memory; that introducing a cache changes the ownership model of the thing it caches.

**Cut rule:** cut after chunked prefill and before the preemption rungs. Cheaper than chunked prefill (no new theory beyond hashing and reference counting) and the more common production feature, but the TTFT/TPOT tradeoff is the scheduler's core story and this is an optimization layered on top of it. Never claim it unbuilt.

### Rung 3 (stretch) — 🔨 Decode graph capture

**📖 Read:** the CUDA Graph sections of the CUDA C Programming Guide, plus the graph-capture notes in vLLM's or TensorRT-LLM's design docs. Half a day.

**What:** capture one decode iteration into a CUDA graph and replay it, instead of launching its kernels one at a time. **Why here and not in step A:** a Monte Carlo path generator is one wide kernel whose launch overhead is invisible against its runtime, while a decode iteration is dozens of small kernels whose launch overhead is a visible fraction of the iteration — the same technique is a null result in one place and a real one in the other, and knowing which is which is the lesson. The scheduling-shaped part is what makes it phase-6 material: batch composition changes every iteration, so a graph is captured **per batch-size bucket** and short batches are padded up to their bucket. The padding is real work you are choosing to do, and its cost is measured separately from what capture saves.

**Why a stretch rather than a ship line:** it optimizes on top of rung 1's numbers, it needs a GPU decode path the decoder deliberately does not have in v1, and it is only honest once **launch overhead is visible in your own Nsight Systems trace**. Profile first — if the trace shows launch overhead is noise on this workload, that is the finding and the rung stays unbuilt.

**Learn:** launch overhead as a first-class cost; shape specialization vs. generality; why every serious serving stack trades flexibility for this.

**Cut rule:** cut before rung 2. Never claim it unbuilt.

### ✅ Correctness gate (phase is not done until all pass)

- **Output equality across batch compositions.** The same seeded request stream, run at batch size 1 and at capacity, must produce token-for-token identical outputs per request. Batching is a scheduling change, not a numerical one — if compositing changes a token, the bug is in your attention masking or position handling. Report it the way phase 1 reports replay: bit-identical, across runs and across thread counts.
- **Differential vs. rung 0.** Fuzzed request sequences (varying prompt lengths, generation lengths, arrival patterns) through both schedulers; per-request outputs must match exactly. Divergence is a find, not a nuisance.
- **Admission invariants under fuzz:** resident KV never exceeds budget; every admitted request eventually completes or is explicitly refused; refused requests are refused once, with a reason; no request is admitted twice.
- **Concurrency edge cases, as fuzz invariants:** a request cancelled by the client mid-generation frees its KV exactly once; a client that disconnects without reading does not block drain; a batch where every request finishes on the same iteration; a single request longer than the budget (must refuse, not hang); arrival exactly at the budget boundary.
- **Prefix-cache invariants (gated on rung 2b):** a block is freed exactly once, and only at refcount zero; no write ever lands in a shared block; and two requests sharing a prefix produce token-identical output with caching on and off, over the same seeded stream. The last one carries the weight — it is the output-equality gate re-run against the cache, and it separates a working cache from one that quietly serves another request's KV.
- **TSan clean** under concurrent arrivals and completions.
- **Explain-loop gate (shipping requirement):** cold, spoken, no notes — Orca's iteration-level scheduling and vLLM's paging, why each beats the naive design, and how your design differs and why. This is the feedback substitute the plan cannot otherwise provide.

### Benchmarks — the phase's actual output

Every number is a table in the README's claims pack, generated by `bench/`, not transcribed by hand.

- **Throughput vs. per-token latency, by batch size.** The headline curve. Present it with the mechanism, not next to it: achieved DRAM bandwidth during decode against the roofline ceiling measured in step A. The curve's shape is the argument — it flattens exactly where bandwidth saturates.
- **TTFT and TPOT under concurrent load**, coordinated-omission-correct, on the phase-3 harness, at stated offered rates. Two SLOs, because admission trades one against the other.
- **Rung deltas.** Rung 0 vs. rung 1 vs. rung 2, same workload: what batching bought, what chunked prefill bought. A rung that buys nothing is a finding worth keeping and stating.
- **Scheduler overhead per admission decision** — nanoseconds against a per-token cost measured in milliseconds.
- **Per-iteration launch cost (if rung 3 runs)** — from Nsight Systems, before and after graph capture, with the batch-bucketing padding cost as its own row. A graph result reported without the padding cost is half a result.
- **Overload behavior** on the phase-3 overload harness: what the shedding policy does when the offered rate exceeds capacity, whether TTFT degrades gracefully or collapses, and where the breaker trips.

**Writeups (both required):**
- **FP non-associativity** — why batched kernels can produce different bits than single-stream (reduction order, kernel selection), what your output-equality gate therefore proves, and where the guarantee stops. COD 3.5 plus the ONNX Runtime execution-provider determinism docs.
- **The KV budget arithmetic** — your machine's numbers, your model, your context length.

**Artifact:** `scheduler/` + `scheduler/test/` + batching tables in `bench/`.
**Claims this phase should earn:** *"Iteration-level continuous-batching scheduler with KV-budget admission over a from-scratch C++17 decoder: [N]× throughput at [B]× TPOT vs. single-stream, explained by [P]% of peak DRAM bandwidth during decode; outputs token-identical to the single-stream reference across batch compositions and thread counts; TTFT/TPOT under concurrent load, coordinated-omission-correct."*

**🎯 Interview drills (Phase 6 ships → you answer these):**

- Walk through one scheduler iteration: who joins, who decodes, who drains, who is refused, and in what order you decide.
- Why is decode bandwidth-bound and prefill compute-bound, and what does that mean for co-scheduling them? (Answer from your own achieved-bandwidth figure.)
- What does the KV cache cost at 2k context on this machine, and how does that number set the batch ceiling? (Arithmetic, on a whiteboard.)
- Why reject at capacity instead of preempting? What would preemption cost, and what does vLLM do?
- How do you know batching did not change your outputs? (Output-equality gate, differential oracle.)
- What is your scheduler's overhead per admission decision, and why is that number the one that matters?
- What happens at 2× offered capacity? Which policy sheds, and what does the tail do?
- When does CUDA graph capture pay and when is it a null result? (Answer from your own trace: many small kernels vs. one wide kernel, and what the batch-size bucketing costs.)
- How do you observe a system whose hot path you are not allowed to perturb? (§14 is the answer's evidence.)

**Next:** run the transfer checkpoint out loud, then a short docs pass to append this phase's sections (rung ladder, KV-budget arithmetic, FP writeup, batching tables) to the proof pack started in §9. Phase 5 runs before phase 6 in the v4 sequence, so the design doc is written expecting this append — leave the scheduler section stubbed, not improvised.

---

## 11. AI spine (dependency-gated)

**Sequence (engine spine, per §2):** phase 0 → A (kernels bench) → G (decoder) → phases 1–5 (gateway) → phase 6 (scheduler). **Sequence (quant anchors, each on its own gate):** Orderbook replay harness stable → H (microstructure features) → B (point-in-time datasets) → C (one model, one export, then stop) → D (evaluation rigor). Step **P (Python bridge)** has two halves with separate gates: the anchor half (Orderbook/Backtester wrappers, the pandas reference for H) lands once each anchor repo's replay gate is green; the engine half (engine wrapper, notebooks over the Phase 3–4 artifacts) lands after G and Phase 4. P pauses like the anchors when the spine needs the hours. **Sequence (post-engine, gated on phase 6 green):** Tier A items 1 → 2 → 3, then Tier B items 4 → 5 → 6, the marginal study last.

The ordered per-step detail — concepts, resources, drills — is the **Learning curriculum** subsection at the end of this section. The tables in between are reference (gate, skill, effort, resume shape), not a second ordering.

Two parts. Steps **H, B, C, D** run as work on the *existing* Orderbook and Backtester. Steps **A** (CUDA + kernels bench) and **G** (the decoder) build *inside* the inference engine repo, where they are the scheduler's prerequisites rather than a side track. Resume framing stays the serving-and-performance layer, not ML research.

**Activation rule:** AI modules activate only when their prerequisite artifact exists **and the current gate is green**, and they pause rather than cannibalize the engine. Exception: A and G are on the engine spine's own critical path, gated only on their own prerequisites. H, B, C, D pause first when hours compress.

**Hooks already built into the phases:**

| Artifact | Hook preserved for the ML curriculum |
| --- | --- |
| Deterministic replay (Phase 1) | Versioned decision inputs/configuration → model-version pinning |
| Model-backend interface (Phase 1) | Decoder today, ONNX Runtime optionally — the seam step G fills |
| KV allocator interface (step G) | Contiguous today, paged in phase 6 without touching attention |
| Latency/ablation harness (Phases 3–4) | The batch-size dimension → the phase-6 sweep |
| Orderbook replay harness | Clean zero-alloc feature boundary → step H's microstructure features |
| Backtester Monte Carlo | Seeded deterministic partitioning → future CPU/GPU replay |
| Backtester simulation data | Point-in-time boundaries → future leakage-free datasets |
| Step P pybind11 wrappers | The differential idiom gains a second driver: the same replay/bit-identical gates invoked from pytest; notebooks render committed bench JSON |

**The pipeline** — each step feeds the next. H→B→C→D is ~4–5 weeks of small items on the two anchor projects; A is 4–5 weeks; G is ~1–1.5 weeks; D can run anytime after C; P is ~1 week split across its two halves. H is the microstructure-features row in §15.1. **E and F are retired as separate steps** — E's learning (per-execution-provider determinism, model versioning) survives as a phase-6 writeup topic and its build becomes the optional ONNX second backend; F's batch sweep is a phase-6 benchmark.

| Step | Project | Gate | Build | Skill | Effort | Resume line shape |
| --- | --- | --- | --- | --- | --- | --- |
| H | Orderbook | After the Orderbook has a stable replay harness | Zero-alloc feature layer: OFI, microprice, queue-position estimate | Feature engineering at latency | 1–2 wks | "Zero-alloc microstructure feature layer at X msgs/sec" |
| B | Backtester | After H produces usable features and the Backtester replay is stable | Point-in-time dataset generation from sims with strict leakage discipline (labels from future data = banned), **stored as Parquet and memory-mapped through Arrow** (format note below) | Data engineering for ML; columnar layout as an API | 1–1.5 wks | "Leakage-free point-in-time dataset layer over memory-mapped Parquet, feeding C++ backtests and PyTorch training from the same buffers" |
| C | Backtester | After B produces a validated dataset | Tiny model (logistic/GBM, PyTorch/Python) trained on B's data, exported to ONNX | Training-loop literacy | 2–3 days | "Trained/evaluated baseline models, ONNX export" |
| E | ~~Risk engine~~ | **Retired** — absorbed into the engine | Learning survives as a phase-6 writeup (per-provider determinism); build becomes the optional ONNX second backend (first thing cut) | — | — | — |
| F | ~~Risk engine~~ | **Retired** — absorbed into the engine | The batch-size sweep is now a phase-6 benchmark, measured as throughput vs. TTFT/TPOT | — | — | — |
| A | Backtester (GPU Monte Carlo) + engine repo (kernels module) | Kernels bench: engine path, after Phase 0. GPU Monte Carlo: after the CPU Monte Carlo has a deterministic seeded baseline | CUDA Monte Carlo (path generation on GPU, seeded reproducibility CPU↔GPU) **plus the kernels bench**: GEMM ladder naive→coalesced→tiled→register-blocked, occupancy sweep, roofline figure, one INT8 or fused kernel. The decoder's achieved-bandwidth claim is measured against this ladder's ceiling, same machine, same harness | CUDA — the screening currency; kernel optimization is the accelerator-company probe | 4–5 wks | "GPU Monte Carlo (CUDA), bit-identical replay across CPU/GPU; GEMM ladder at [A]% of peak DRAM bandwidth and [B]% of peak FP32 FLOPs, roofline-analyzed with Nsight Compute" |
| G | engine repo (decoder module) | After A (roofline and bandwidth reasoning). The old E/F gate is retired | From-scratch C++17 GPT-2-class decoder: weight dump, naive forward pass, KV cache behind an allocator interface, greedy decode, single-stream benchmarks; correctness by token-for-token match against llama.cpp. Becomes the engine's model backend (phase 6 batches it) | Transformer internals on real hardware — converts know-only LM vocabulary into a measured artifact | 1–1.5 wks | "From-scratch GPT-2-class decoder in C++17 (KV cache, greedy decode): [X] tok/s single-stream, [N] ms TTFT at 2k context; verified token-for-token against llama.cpp" |
| D | Backtester | Anytime after C; can run in parallel with A | Model evaluation harness: walk-forward validation + null ensembles applied to model signals vs. baseline | ML evaluation rigor | 2–3 days | "Walk-forward eval with null ensembles separating model edge from luck" |
| P | All three repos (anchor half: Orderbook + Backtester; engine half: engine repo) | Anchor half: each anchor repo's replay gate green. Engine half: after G (the decoder is what the wrapper exposes) and Phase 4 (the artifacts the notebooks render) | **The Python bridge.** Per-repo pybind11 wrapper exposing the C++ core as an importable package (extends the GoodSoft Monte Carlo pattern); a pytest harness under `py/tests/` driving the same replay/bit-identical/differential gates from Python; a pandas/NumPy implementation of step H's features (OFI, microprice, queue position) over the replayed stream, diffed row-for-row against the C++ zero-alloc layer; Jupyter notebooks in `notebooks/` rendering the committed bench JSON (A4 roofline, Phase 4 ablation, Phase 3/6 TTFT/TPOT scaling) | The C++↔data boundary — the rare combination, not a second language | ~1 wk (split) | "Exposed the C++ engine and matching core as a pybind11 package; Python research layer (pandas/NumPy/Jupyter) drives replay, differential feature checks, and benchmark analysis; pytest harness wraps the C++ core" |

**Step B's format decision — Parquet + Arrow, not CSV.** The argument is not "faster ingest" — your `from_chars` CSV path is already fast — it is that the format changes what the pipeline *can* do: random access to history by timestamp is exactly what a point-in-time join needs, so leakage discipline becomes a property of the layout instead of a habit; the schema is explicit, which kills a class of silent column-order bugs; dictionary-encoded symbol columns are the string→integer transform a feature pipeline otherwise hand-rolls; and the **Arrow C Data Interface** hands the same buffers to PyTorch with no serialization. That last one is the AI-infra claim — the zero-copy *handoff*, not the ingest. Keep the CSV path and benchmark both: two ingest paths make the before/after free. Boundary: zero-copy stops at any transform (non-native layout, string handling, a rolling window crossing a row-group boundary) — state where it stops rather than claiming it never does.

**Cost triage when the engine needs the hours:**

- **H — keep.** The only *built* microstructure artifact in the portfolio, 1–2 weeks; no amount of serving work teaches OFI and queue position. The engine cannibalizes its hot-path technique but not its domain content.
- **B — keep, scoped.** Bounded to the dataset layer (point-in-time boundaries, embargo, Parquet, the zero-copy handoff). If it grows a feature store, a scheduler, or a catalog, it has left the plan.
- **D — keep.** Two to three days of statistics (purged CV, embargo, deflated Sharpe) that nothing else in this curriculum teaches.
- **P — keep, scoped.** The boundary is the rare combination the listings name (C++-first plus the data stack). Capped at wrappers + pytest + notebooks + the one pandas reference; the moment it grows a second implementation of any engine subsystem, it has left the plan.
- **C — literacy only, permanently.** One model, one training loop, one export, then stop. A half-built training pipeline reads as dabbling, and the decoder is a far stronger ML artifact.
- **The pause rule is the time answer.** These are anchors: they pause when the engine needs the hours, by design. Cutting them to "free up time" trades a cheap, differentiated, non-substitutable item for hours the engine's own gates will absorb anyway.

### Post-engine stretch tier — the AI-infra gap closers

Gated on the engine having shipped (phase 6 green): none of it competes with the spine, and all of it answers the two standard objections — *"the model is small"* and *"there is no distributed or production story."* Tier A is ~2 weeks total and is the part worth committing to; Tier B is pick-by-remaining-hours.

**Tier A — cheap, high density.**

1. **Ring all-reduce + the alpha–beta crossover (~4–6 days).** Implement ring all-reduce over TCP — the general N-rank form, run as N=2 across the desktop and laptop and N=4 as local processes — then measure latency and bandwidth against message size and fit the latency-plus-bandwidth cost model. Plot the crossover: the message size above which tensor parallelism pays, per interconnect (1GbE, 2.5GbE, loopback, and with the rented node, PCIe and NVLink). **The artifact is the measurement that decides when TP is worth it**, and it needs no special hardware. Drills: why ring and not tree; name the two terms of the cost model and which dominates at which size; why TP needs two all-reduces per layer. **Read COD 6.9 *Introduction to Multiprocessor Network Topologies* (+ 6.10 cluster networking) as a skim first** (~30 min): hop count, bisection bandwidth, and why topology sets the per-interconnect constants the crossover is measured against — otherwise the plan prices the alpha–beta model with no source for the topology term. **NCCL enters as the reference, not the build:** on the Tier B node, run NCCL's all-reduce at the same message sizes and rank counts on the same hardware and diff its curve against your hand-rolled fit. The deliverable is the comparison — where your implementation tracks NCCL and where it falls away, and which cost-model term explains the divergence. Same differential idiom as phase 6's rung-0 oracle: the keyword lands as a *validated-against reference* with a measurement behind it, while staying off the build-claim list. **Sources:** Thakur, Rabenseifner & Gropp, *Optimization of Collective Communication Operations in MPICH* (2005) — the alpha–beta model and its lineage; Gibiansky, *Bringing HPC Techniques to Deep Learning* (Baidu, 2017) — the ring algorithm and the ring-not-tree argument.
2. **Decoder architecture breadth — RoPE, RMSNorm, SwiGLU, GQA (~3–5 days).** Extend the decoder past GPT-2 to a Llama-shaped config, behind the *same* token-for-token gate against a reference running the same tiny checkpoint. **GQA is why this earns the days**: grouped-query attention is the KV-cache reduction technique every serving stack uses, and owning it converts the KV-budget arithmetic from a GPT-2 curiosity into the standard story (KV per token drops by the query-to-KV-head ratio). Closes "GPT-2 is small" with real technical content at a fraction of the cost of serving an actual 7B — and needs no GPU, because correctness runs on a tiny config. **Source:** Ainslie et al., *GQA* (2023).
3. **Snapshot-published metrics + the SLO write-up (~3–4 days).** A sub-microsecond path cannot be scraped, so counters are published from a seqlock-protected snapshot the hot path writes and the exporter reads — the same mechanism phase 1 already assigns the McKenney reading for, now with a second consumer. Then the policy half: TTFT and TPOT SLOs at a stated offered rate, an error budget each, and what burns them. **No dashboards, no operator, no OpenTelemetry**; the claim is the publication mechanism and the policy, not the tooling.

**Tier B — the big items, in order.**

4. **Decoder quantization, end to end (1–2 weeks).** A5 measures one quantized kernel; this measures what quantization does to *the serving path* — throughput gain against accuracy cost (perplexity plus token-agreement rate versus the FP32 gate), and where the error lands. Gated on A5, cheaper than it looks because the correctness harness already exists. **The deliverable is a perplexity-versus-bits curve, not a single point:** the same small real checkpoint (the Tier A item 2 tiny Llama checkpoint, so the gate and the weights are already in place) quantized at FP32 → FP16 → INT8 weight-only, one row per precision — perplexity delta, size, and decode throughput on one machine. That curve is the difference between "I wrote an INT8 GEMM" and "I can quantize a model and tell you what it costs," which is the exact artifact the GGUF/GPTQ/AWQ community reports. **Boundary:** weight-only, block-level, one checkpoint family — no format building, no GPTQ/AWQ implementation; those are Tier 3 vocabulary. **Sources:** Frantar et al., *GPTQ* (2022); Lin et al., *AWQ* (2023) — the two methods, and why activation-aware scaling exists as a correction to weight-only error; Dettmers, *The Big LLM Squeezing Guide* (2024) — the conceptual overview of why low-bit works (GPTQ/AWQ/k-quants are its named instances), read once before building the curve so the tradeoff surface has a frame.
5. **Speculative decoding (~1 week).** Draft (GPT-2) plus target, in your own decoder. The finding is the interesting half: it is a batch-1 latency technique, so its gain collapses exactly as continuous batching's throughput rises — two techniques pulling the same knob in opposite directions, measured on one system. **Source:** Leviathan, Kalman & Matias (2023) — states the batch-1 framing, so the finding is predictable from theory rather than a surprise from the data.
6. **Tensor parallelism on your own decoder, one rented node (2–3 weeks).** Shard attention heads and MLP columns, place the all-reduces, shard the KV cache, re-run the token-for-token gate across rank counts. Highest-ceiling claim available without a cluster, tractable because item 1 built the collective's moving parts by hand first. **Report scaling efficiency explicitly:** throughput at 1, 2, and 4 ranks against the linear ideal, per batch size, with the shortfall attributed to a named term — all-reduce latency against per-layer compute, KV-shard imbalance, or the absence of communication/compute overlap — never as a bare percentage. **State the model-size boundary in the same breath:** GPT-2 124M fits on one GPU with room to spare, so the honest result at low rank counts is that TP *loses*, and the artifact's value is the measured crossover. If the number is meant to come out positive, it needs Tier A item 2's Llama-shaped config ahead of it. **Source:** Shoeybi et al., *Megatron-LM* (2019) — column- and row-parallel sharding, and the origin of "two all-reduces per layer."

**Marginal — only if everything above ships:** a disaggregated prefill/decode study (3–5 days) — model the KV-transfer cost, validate the model against your measured TTFT/TPOT, report the crossover. Honest because it is a model validated against a real system rather than a deployed one.

**Skipped, with reasons (so they do not creep back).** *Kubernetes* — a Dockerfile and one manifest are two hours and earn no claim; cluster operations is a different job. *Serving a real 7B* — adds no code the decoder does not already have; item 2 closes the same vocabulary gap with engineering. *Multi-node GPU across your own two machines* — impossible rather than hard: ROCm and CUDA GPUs cannot join one collective, and consumer Ethernet adds ~4–6 ms of pure latency per token across 64 collectives — which is why item 1's crossover measurement exists. *Grafana/OpenTelemetry/operators* — tooling, not findings.

**Hardware (cloud route):** CUDA runs only on NVIDIA — the RX 9070 XT can't, the laptop's RTX 3060 can, but the chosen path is cloud: Colab's free tier (T4) or ~$1/hr instances cover the entire CUDA workload, kernels bench and decoder included — same rent-the-hardware pattern as the Phase 2 NUMA cloud day. Use real `nvcc` + Nsight in the cloud, never a translation layer (ZLUDA / ROCm-on-consumer): the profiling toolchain is part of what you're learning, and the resume currency is *CUDA* specifically.

**Retired in v4 — the optional API-vs-ONNX backend comparison.** Its gate (old step E) no longer exists; its content is superseded by the phase-6 writeup on per-provider determinism. If it ever returns, it returns as a fresh decision with its own gate, not as drift back into this plan. **The one form it was offered in (not adopted):** a TensorRT crossover study — engine build, FP16/INT8 calibration, batch-size sweep against the CPU path to find where a GPU actually wins. A real deployment skill with a real lesson (a sub-microsecond decision runs on CPU, so moving it to a GPU *costs* latency at batch 1, and the win only appears as an intersection on the curve). Unadopted because it re-introduces the second backend that cut-order item 2 removes *first*, and because the phase-6 sweep plus the A4 roofline already answer the "CPU vs. GPU at batch=1" drill analytically. Re-open only post-shipping, with the crossover framing — never as a speedup claim, which at batch 1 would be false.

**Two rules (non-negotiable):**

1. **Models are plumbing, not alpha.** Every model is tiny and honest; all claims are about the pipeline and its measurement. Scripted probe answer: "my work is the serving and performance layer; the model is a placeholder — that's a deliberate boundary, not a gap."
2. **Bracket rule applies.** No AI bullet ships without its measurement; the ONNX stub reaches the resume only when replay-pinned determinism is demonstrated.

**Deliberately skipped (know the vocabulary, can't claim the build):** distributed training / NCCL, RAG/vector DBs, MLOps-at-scale. (NCCL is run as Tier A item 1's reference implementation and never as a build — the collective you claim is the hand-rolled ring.) KV caches left this list when step G was specified; continuous batching left it in v4. **Resurrected by the AI spine:** COD 6.6 + Appendix C and 6.7 (slotted into step A), COD 3.5/3.6–3.7 upgraded to real reads (thrive layer), DDIA ch. 10–11 figures skim. Still skipped: OSTEP persistence half, Top-Down ch. 7–8. (CiA ch. 9–10 are **read** at Phase 0 items 27–28 — the thread-pool contrast and the `std::execution` refusal answer. This line listed them as skipped until v4.6 caught the contradiction with those items.)

**End-state narrative:** data → features → training → export → serving → measurement — one engine carrying kernels, decoder, gateway, and scheduler, with the Orderbook and Backtester as the quant anchors. Small bonus: note "HIP-portable kernel design" in the CUDA writeup (knowing where the portability boundary sits is itself a signal).

### Learning curriculum (concepts → resources → drills)

Keyed to the pipeline steps so each concept is applied within days of learning it. Your systems background is a shortcut throughout: GPU memory hierarchy, tail latency, and throughput-vs-latency are the same physics as Drepper/COD/Gil Tene, re-skinned — that's the interview edge.

**Step H — feature layer (Orderbook).** *Concepts:* order-flow imbalance, microprice, queue-position estimation; why these predict short-horizon prices (adverse selection, informed flow). *Resources:* Cont, Kukanov & Stoikov, *The Price Impact of Order Book Events* (2014 — the OFI original); Harris, *Trading and Exchanges*, queue/limit-order chapters (skim). *Drill:* "why does order imbalance predict short-horizon moves?"

**Step B — dataset pipeline (Backtester).** *Concepts:* point-in-time correctness; label leakage / lookahead bias (the #1 probed topic); time-series splits — never shuffle; purged CV + embargo periods; class imbalance; rolling z-scores. *Resources:* López de Prado, *Advances in Financial Machine Learning*, ch. 3–7 only (the reference); scikit-learn docs for mechanics. *Drills:* "why does random k-fold fail on time series?" "where does leakage hide in feature engineering?"

**Step C — training literacy (Backtester, Python).** *Concepts:* the floor — linear/logistic regression, gradient boosting (LightGBM/XGBoost), loss functions, overfitting vs. regularization, bias-variance; PyTorch floor — tensors, autograd, one training loop (learn to *talk to* ML engineers, not become one); ONNX as exchange format. *Resources:* PyTorch *60-Minute Blitz* (an evening); LightGBM/XGBoost docs; `onnx`/`onnxruntime` docs. **Skip ML textbooks** — a time sink for plumbing-level literacy. *Drills:* "why GBM over a neural net here?" "what's actually inside an ONNX file?"

**Steps E+F — retired, absorbed into the engine.** *What survives as material:* **numeric determinism** — FP non-associativity, why CPU vs. GPU inference can differ bitwise, and the **ONNX Runtime determinism and execution-provider documentation** (determinism is not a global switch but a per-provider property — kernel selection, reduction order, and provider fallbacks all vary; the reading is "which knob pins which provider, and where the guarantee stops"). It is the source for phase 6's required writeup. *Also surviving:* ONNX Runtime C++ API docs (only if the second backend ships), TensorRT docs skim (the quantization reading behind A5). *Drills survive verbatim:* "CPU vs. GPU inference at batch=1 — walk the tradeoff" (answered from your own phase-6 sweep), "what is quantization and what does it cost?", "why can inference be non-deterministic?"

**Step A — CUDA (Backtester + the engine's kernels module, the big one).** Five sub-steps, each read unlocking the build under it. A5 comes last because quantization is the only part that needs the precision ladder. The kernels bench lives in the **engine repo**: a reviewer opening the Backtester should find a simulation engine, and a reviewer opening the engine should find measured kernel tables next to the decoder whose bandwidth claim they explain. The GPU Monte Carlo stays in the Backtester, where its CPU counterpart and replay harness already are.

- **A1. 📖 *GPU architecture.* COD 6.6 (GPU intro) + Appendix C skim, then 6.7 (DSAs/TPU) skim, then 6.12 *Benchmarking the Google TPUv3 Supercomputer and an NVIDIA Volta GPU Cluster* as a skim** — SIMT, warps, thread blocks, and the GPU memory hierarchy (registers → shared → L2 → HBM) at textbook altitude before touching code; also the accelerator-landscape vocabulary (TPU as archetypal DSA) for AI-chip interviews, and 6.12 puts measured numbers on that landscape rather than leaving it as taxonomy. 6.12 is also the primary source for the Tier 3 "tensor cores and architecture-specific paths" boundary answer. Drepper §1–3 and COD ch. 5, re-skinned. 🔨 **Build:** NVIDIA DLI free intro course (few evenings, hands-on). ✅ **Verify:** why is a GPU bandwidth-rich but latency-poor?
- **A2. 📖 *Programming model and coalescing.* CUDA C Programming Guide** as desk reference — kernel launches, grid/block/thread geometry, `__global__`, memory **coalescing** — plus *streams and async*: `cudaMemcpyAsync`, overlapping H2D/compute/D2H, directly analogous to your SPSC pipelining. 🔨 **Build:** *GPU Monte Carlo in the Backtester.* Path generation on GPU, coalesced, seeded CPU↔GPU bit-identical replay, profiled with Nsight Systems on the cloud T4. Do this first among the CUDA work: it is the "make CUDA run at all" step and reuses your shipped CPU Monte Carlo, so the correctness story is a replay comparison rather than a new invariant. **AI hook preserved:** per-path seeding and the deterministic partition scheme carry over unchanged; the CPU↔GPU equivalence test is the same harness you use for thread-count invariance. ✅ **Verify:** same path hashes, same terminal equities, across the device boundary.
- **A3. 📖 *Tiling and shared memory.* Kirk & Hwu, *Programming Massively Parallel Processors* (4th ed., ch. 1–4)** — thread/block geometry, coalescing, shared memory, tiling, bank conflicts. **Plus COD's matrix-multiply thread, read here as one block (~2 h): 2.20 *Going Faster: Matrix Multiply in C* → 3.8 *Subword Parallelism* → 4.13 *Instruction-Level Parallelism* → 5.14 *Cache Blocking* → 6.13 *Multiple Processors*.** It is the CPU-side ancestor of this ladder — 5.14 is the tiling rung, 4.13 the register-blocking and unrolling rung, 6.13 the parallel rung — and it builds the arithmetic-intensity intuition the roofline formalizes at A4. Skip 1.10 (Python); it carries nothing the thread does not. 🔨 **Build:** *the GEMM ladder, part 1.* Naive → coalesced → shared-memory tiled. Measure each rung as you add it; the ladder table is the artifact, and a rung that buys nothing is a finding worth keeping. 🎯 **Drill:** "walk me through coalescing."
- **A4. 📖 *Occupancy and roofline.* Kirk & Hwu ch. 5–6** — occupancy, performance limits, the matrix-multiplication case study — plus the **Nsight Compute** tutorials and the roofline model in its primary source: **Williams, Waterman & Patterson, *Roofline* (2009)**, short and citable, the origin of the arithmetic-intensity framing you will be asked to reproduce on a whiteboard. 🔨 **Build:** *the GEMM ladder, part 2.* Register-blocked kernel, occupancy sweep, and the **roofline figure** — achieved FLOPs and achieved DRAM bandwidth against the machine's ceilings. Document where the ladder stops scaling and what the limiter is. Profiling is part of the build: every rung's number gets Nsight Compute output linked. ✅ **Verify:** roofline a kernel out loud; explain how occupancy interacts with shared memory; say which rung hit which ceiling and why.
- **A5. 📖 *Precision and quantization.* COD 3.5 (IEEE-754) as a real read** — you cannot answer "what does quantization cost" without knowing what a mantissa is — **plus TensorRT's quantization docs upgraded from skim to read, and one INT8 GEMM reference.** 🔨 **Build:** *one fused or quantized kernel.* INT8 dequant-GEMM or fused attention — pick one, not both. Measure throughput gain and accuracy cost against the FP32 baseline; write both numbers down. **cuBLAS as the upper reference:** same shape, same dtype, same GPU, reported as a percentage of cuBLAS rather than a standalone number. The percentage is not the artifact — the attribution is. Name which of the things cuBLAS does that you do not explains the gap (tile shape tuned per architecture, split-K for small matrices, `cp.async`/TMA pipelining to hide global latency, double buffering, tensor cores or their absence, launch-configuration and occupancy tuning), and say which of them the A4 roofline already predicted. Expect to lose: a first-attempt kernel that beats cuBLAS is measuring the wrong thing. **If the branch is fused attention rather than INT8:** Dao et al., *FlashAttention* (2022) is the reference, and its IO-awareness argument is a second worked example of A4's roofline reasoning. 🎯 **Drill:** "what is quantization and what does it cost?", answered from your own accuracy delta.
- **A6. 📖 *Triton, as a comparison.* The Triton documentation and its GEMM tutorial** — a language, not a textbook, so the tutorial is the entire reading. ~Half a day. 🔨 **Build:** the same GEMM as A3/A4, written in Triton, measured against both your CUDA kernel and cuBLAS on the same shape and GPU. A three-way table. **Why it earns a slot:** Triton appears in a large share of AI-infra postings, and the finding is the abstraction's price and payoff — where Triton matches hand-written CUDA, where it beats it (autotuning a search space you did not sweep by hand), where it does not reach (no explicit control of pipeline depth or shared-memory layout, exactly what the ladder taught you to control). Stated as a comparison, it strengthens the CUDA claim. **Cut rule:** cut before A5.

**Not here: CUDA Graphs.** The Monte Carlo path generator is one wide kernel, so launch overhead is invisible against its runtime and graph capture is a null result at A2. The technique is real one workload away — the decoder's decode iteration — and it is built as phase 6's stretch rung. The general rule: a technique is not a plan item until the workload makes it pay, and knowing which workload that is *is* the learning.

*RISC-V note:* if the accelerator-company path (Tenstorrent/Openchip-style) firms up, skim COD's RISC-V "Real Stuff" sections — "RISC-V vs. x86" is a known probe.

**Step G — decoder (transformer internals, the engine's backend, ~1–1.5 weeks).** *Gate:* after A (roofline and bandwidth reasoning make decode legible). This step converts know-only LM vocabulary into a measured artifact **and** becomes the model backend the scheduler batches — but the decoder is built and verified single-stream first, and the cut list below keeps it from becoming a serving engine itself.

*Cut list, fixed in advance:* no tokenizer beyond a naive BPE over a dumped vocab; no HTTP server, no OpenAI-compatible API, no transport of any kind (the gateway owns ingress); **no batching inside the decoder** — it runs a batch handed to it by phase 6, it does not schedule one; no quantization of the decoder; no GPU path in v1; nothing larger than GPT-2 124M.

*Interface obligation:* the KV cache is built behind an allocator interface even while the implementation is contiguous, and the decoder implements the model-backend interface defined in phase 1. Both seams exist so phase 6 can add paging and scheduling without rewriting attention.

*Concepts:* scaled dot-product attention, multi-head attention, causal masking, layernorm, the residual stream, prefill vs. decode, the KV cache and what it costs in memory, why decode is bandwidth-bound and prefill is compute-bound, sampling (greedy, temperature, top-k), TTFT vs. inter-token latency.

*Resources:* Jay Alammar, *The Illustrated Transformer* (~1 hr — attention mechanics without the math overhead) → Karpathy, *Let's build GPT: from scratch, in code, spelled out* (~2 hrs — mechanism only, the Python is incidental) → llama.cpp's architecture docs and a read of its KV-cache and sampling source as a **C++ reference implementation** — reading a working C++ implementation of the thing you are about to write is the highest-leverage hour in this step. **Add Karpathy's `llm.c`** as the second reference: GPT-2 in C/CUDA with no dependencies, the closest existing artifact to what this step builds, and what your minimal forward pass gets diffed against. llama.cpp is the reference for KV-cache and sampling craft; `llm.c` is the reference for the forward pass itself → *Attention Is All You Need*, skim for vocabulary only. Bandwidth reasoning comes from A4, the precision ladder from A5; nothing new needed for either.

*Build, in this order:* (1) a one-file Python weight-dump script — GPT-2 124M weights and vocab to flat binaries; Python stops there. (2) Naive decoder: embeddings → single-head attention → MLP → layernorm → logits → greedy decode, CPU, fp32. (3) **Correctness gate before any optimization:** greedy output must match the reference token-for-token on a fixed prompt for the first N tokens. **The reference must run the same weights and the same tokenizer as your decoder** — GPT-2 124M weights dumped from HF compared against a llama.cpp GGUF of a different model family is not a gate, it is a guaranteed mismatch. Pick the pair deliberately (HF `transformers` running the same checkpoint is the natural reference; a llama.cpp GGUF of the *same* GPT-2 model works too) and write down which one, so the claim "verified token-for-token against llama.cpp" is true as stated rather than approximately true. This is your bit-identical-replay habit applied to a new domain, and it is the single most credible thing about the artifact — most from-scratch decoders in portfolios have no correctness claim at all. (4) KV cache, re-run the correctness gate, then re-benchmark. (5) Multi-head, then sampling if cheap. (6) Measure: tok/s single-stream; TTFT at 512/1024/2048 context; achieved DRAM bandwidth during decode against the A4 roofline number; KV-cache footprint vs. context length. (7) *Stretch only, after everything above ships:* a static batch-size sweep for curiosity (tok/s rises, per-token latency rises, and the bandwidth ceiling is the explanation) — the real sweep is phase 6's — then a GPU path. Everything in (6) is single-stream and stays that way; concurrency belongs to the scheduler, and keeping the decoder's numbers clean of it is what makes the token-for-token gate meaningful.

*Drills:* "walk me through attention and the KV cache"; "why is decode bandwidth-bound and prefill compute-bound?" (answer from your own achieved-bandwidth figure); "what does the KV cache cost at 2k context and why does that cap batch size?"; "how did you verify your implementation?" — token-for-token against the reference.

*Architecture breadth (post-engine, Tier A item 2):* the decoder ships GPT-2-shaped; extending it to a Llama-shaped config is what turns the KV-cache arithmetic into the standard story.

**Step D — evaluation rigor (Backtester).** *Concepts:* walk-forward (known); purged/embargoed CV applied; **multiple testing and the deflated Sharpe ratio** (your null ensembles were always a homemade version of this); information coefficient. *Resources:* Bailey & López de Prado, *The Deflated Sharpe Ratio* (short paper, very readable); ALFM ch. 11–12 skim. *Drills:* "how many backtests did you run before finding this?" "why does Sharpe overstate edge after selection?"

**Step P — the Python bridge (all three repos).** *Concepts:* pybind11 — the GIL and `py::gil_scoped_release`, ownership and lifetime across the boundary (`return_value_policy`, owning vs. non-owning references), what a wrapper hides (exceptions crossing, `py::error_already_set`); pandas/NumPy — vectorized feature construction, groupby/rolling over replayed streams, where vectorization stops (per-row control flow → C++); pytest — fixtures and parametrize, driving a C++ core as a black box; Jupyter — a notebook as a *view* over committed artifacts, never the source of a number. *Resources:* pybind11 docs (the tutorial is the whole read — the GoodSoft Monte Carlo wrapper is the house reference); the **NumPy Quickstart** (numpy.org, ~1 hr — indexing, broadcasting, ufuncs, vectorized-vs-loop); *10 Minutes to pandas*; pytest docs (fixtures, parametrize); the **Jupyter docs' "Jupyter Notebook — the classical environment" page + one hour of driving your own bench JSON in a cell** (it's a tool, not a subject — the notebook in part 3 *is* the practice). **Skip:** the rest of the Python ecosystem — web, frameworks, heavy scripting. The boundary is the point, not fluency. *Build, in this order:* (1) a `py/` directory per repo — the pybind11 wrapper of the C++ core: engine (loadgen + decoder + gateway), Orderbook (matching core + replay), Backtester (sampler + Monte Carlo, extending the GoodSoft pattern). (2) the pytest harness under `py/tests/` — the same replay/bit-identical/differential gates invoked from Python; a bug that reproduces only through the wrapper is a boundary finding, and every gate gains a second entry point for free. (3) `notebooks/` at each repo root — the A4 roofline figure, the Phase 4 ablation table, and the Phase 3/6 TTFT/TPOT scaling plots, rendered from the committed bench JSON. (4) the pandas/NumPy implementation of step H's features over the replayed stream, diffed row-for-row against the C++ zero-alloc layer — the same differential idiom as every other gate, and the pandas side doubles as the C++ layer's reference implementation. *Placement convention:* `py/` (wrapper + tests) and `notebooks/` at the repo root in every repo — intentional integration, not bolt-ons. *Drills:* "where does the GIL bite in an exposed C++ engine, and how do you keep the hot path fast?" "why a pybind11 boundary instead of JSON/RPC?" "how do you keep a Python-driven benchmark honest (no accidental closed-loop or sleeps)?" "what does the pandas reference buy that the C++ tests do not?"

**Cross-cutting "thrive" layer:** FP precision ladder (FP32/TF32/FP16/INT8 — what quantization buys and costs; **COD 3.5 is now a real read**; 3.6–3.7 SIMD likewise — the CPU-side ancestor of GPU vector lanes); GPU-vs-CPU physics (bandwidth vs. latency, why batch amortizes); tail latency under serving load (Gil Tene — done); batch vs. stream vocabulary (DDIA ch. 10–11 figures-only skim, ~1 hour, ten Anki cards — step B/H are batch and streaming respectively); **thread pools, one paragraph** — what they are, why serving frameworks use them, how your SPSC-spinning model differs (the contrast is the interview answer); the know-only list (NCCL/all-reduce vocabulary, distributed training concepts, MLOps). Every one is answered with either a measurement from your build or the boundary script — that pairing is the signal.

**Sequencing note:** the engine path is **Phase 0 → A → G → Phases 1–5 → Phase 6**, so the kernels ladder and the decoder exist before the gateway and scheduler need them. The anchor path is H→B→C→D on Orderbook and Backtester, landing whenever their own gates are green. CUDA's curriculum is self-contained and the heaviest — if motivation dips, run DLI + Kirk & Hwu *in parallel* as the "second book"; cloud GPUs are on-demand either way.

**🎯 AI-extension drills** (answer from your own measurements; run them through §13 like any other drill — cold, spoken, probed, colored, logged):

- *Serving (phases 3 + 6):* CPU vs GPU inference at batch=1 — walk the tradeoff (your own sweep histograms); what is quantization and what does it cost; why can inference be non-deterministic (FP non-associativity); transformer vocabulary: attention, KV cache, prefill/decode (built in G), continuous batching (**built in phase 6** — answer from your own rung ladder, not from the vLLM paper).
- *CUDA (A):* walk through coalescing; warp divergence; occupancy vs shared memory; roofline a kernel; the GEMM ladder and where it stops scaling; what quantization costs (your measured accuracy delta); why a GPU is bandwidth-rich but latency-poor; your CPU↔GPU seeded-replay determinism story.
- *Decoder (G):* walk me through attention and the KV cache; why is decode bandwidth-bound and prefill compute-bound; what does the KV cache cost at 2k context and what caps batch size; how did you verify the implementation.
- *Data/eval (B/D):* where does leakage hide in feature engineering; why never shuffle time series; purged CV + embargo; "how many backtests did you run before finding this" (deflated Sharpe).
- *Boundary script (all):* "My work is the serving and performance layer; the model is a placeholder — that's a deliberate boundary, not a gap."

---

## 12. Reading map (your list → this project)

Every book on your list gets a verdict and a landing spot. Nothing is "read because it's on the list."

| Your reading list | Verdict | Where it lands |
| --- | --- | --- |
| Concurrency in Action | **Read fully (Ch. 1–11)** — slotted into the Phase 0 execution order with per-chapter depth notes there (Ch. 5 with 2–3× time budget; 7.1/7.3 + 8.2 full attention; Ch. 6/9–11 comprehension depth). Phase 1 keeps a targeted 5.3 re-read | Phases 0–1 |
| OSTEP (Three Easy Pieces) | **Spine of Phase 0: ch. 1–34 in order with carve-outs** — exact read/skip map in Phase 0 item 1 (~70% read; concurrency half mostly covered by CiA except ch. 28/33). Persistence (36–51) cut to a 2-hr skim of 36/39/42; security (52–57) skipped | Phase 0 |
| Computer Networking: Top-Down (C1–5) | **Structured read (~50% of Ch. 1–6).** Read fully: 1.3–1.5 (1.4 delay/loss/throughput is the highest-value section in the book — the interview 'capacity math'), 2.1 (esp. 2.1.3–2.1.4, transport services), **2.2.2 + 2.2.6 (persistent connections, HTTP/2 — the HOL-blocking source)**, 2.7, **Ch. 3 in full** (3.4.3–3.4.4 carry the most weight; 3.5.5 flow control and 3.7 congestion control are design inputs now, not background — phase 3 uses TCP), 4.1 + **4.2.4–4.2.5 (queuing, packet scheduling — the WFQ source phase 6 needs)** + 4.3.1–4.3.2, 6.4.1–6.4.2 + 6.7. Skim: 1.1–1.2 (**read the 1.2.2 sentence on fiber propagation speed — it is the 5 µs/km constant the capacity-math drill uses**), 2.2 (rest), 2.4, 4.2 (rest), 4.3.3, 5.2, 5.4, 5.6, 6.1–6.3 (6.2.3 CRC is the mechanism behind "checksum weakness"), **6.6 (data-center architectures — the only DC-networking source on the list, and AI-infra probes it)**. Skip: 1.6–1.7, 2.3/2.5/2.6, 4.3.4/4.4/4.5, 5.3/5.5/5.7, 6.5, Ch. 7–8 (optional: 8.6 TLS "big picture" is 2 pp and grounds the HTTP/TLS paragraph the phase-3 design doc owes). Socket mechanics: Beej's + `man 7 tcp` at phase 3 | Phase 3, just-in-time |
| Computer Organization and Design | **Structured read with a skip list (~55–60%) — structured second source, sized by skip-ratio, not by rank.** Read: **1.2** (seven great ideas — the generative principles), **1.6 + 1.7 + 1.8**; 2.4, 2.8, 2.12 (link/load — **skim only from v4.4; CSAPP ch. 7 supersedes it in Phase 4**), 2.14 (aliasing/UB intuition); 3.5 *Floating Point* (a real read — quantization cost needs the mantissa), 3.6–3.7 skim; **4.1** (skim — the chapter's frame, v4.6), **4.3–4.4** (skim **4.3** for the datapath elements; **4.4 is a declared drop-through, not a skip** — 4.6 pipelines the single-cycle datapath that 4.4 builds, so if 4.6's IF/ID/EX/MEM/WB stages land cold, read 4.4's ~12 pp before continuing; v4.6), **4.6–4.11** (pipelining, 4.8 data hazards, 4.9 control hazards, ILP); **5.1, 5.3, 5.4, 5.10** + 5.7–5.8 skim (VM — OSTEP owns it; 5.8 is the unifying framework); **6.1–6.5** (incl. 6.4 *Hardware Multithreading* — feeds Phase 2 pinning); **6.6 + Appendix C + 6.7 skim (step A1's GPU material)**. Skip: ISA mechanics (2.2–2.10, 2.13, 2.15–2.19 — but **read 2.11, synchronization**), arithmetic circuits (3.1–3.4), logic design (4.2, 4.14) and the multicycle implementation (4.5 — a stepping stone pipelining supersedes), ECC/RAID/cache controllers (5.5, 5.9, 5.11–5.12), warehouse-scale (6.8, 6.11 — but **skim 6.12**, the TPUv3/Volta cluster benchmark, at step A1; **6.13 is the parallel rung of the matrix-multiply thread and is read at step A4, not skipped** — the earlier list double-listed it, corrected v4.6), Appendices A–B, D–E (A.3–A.4 dropped from v4.4 — CSAPP 7.7–7.8 is the drop-through depth for Phase 4's link/load). **Numbering is the 6th edition; ch. 2/3/5/6 match the 5th, ch. 4 is +1 (the 6th adds §4.5 *A Multicycle Implementation*), and the 4th's cache coherence is 5.8 — match by title; per-slot itemization in Phase 0 item 34 and Phase 2's first read.** **Why COD gets the larger second-source budget:** its unread chapters (ISA mechanics, arithmetic circuits, logic design and datapath, ECC/RAID) are cleanly skippable, which is what lets a ~370-page structured read carry the architecture ground; CSAPP's corresponding chapters are dense end to end, so a CSAPP-primary spine costs more pages for less architecture | Phase 0 (Ch. 1 + 5.1–5.4/5.10) → Phase 2 (4.6–4.11 with Agner Fog; 6.4 before pinning; 6.5) → Phase 4 (2.12 as a skim) → step A1 (3.5 re-read, 6.6/6.7/6.12/App. C; 2.20–6.13 matmul thread at A3–A4) |
| Computer Systems: A Programmer's Perspective (3rd ed.) | **Targeted supplement (~175 pp) — not a replacement for COD, and not a full read.** The two books overlap by about a third (caches, VM, floating-point representation); where they diverge, COD holds the architectural *model* — pipelining and hazards through out-of-order execution, ISA-level synchronization, the multicore-to-DSA arc — with Agner Fog, Drepper and PMPP supplying the instance depth, and CSAPP holds the *mechanisms the builds touch directly*, which no other source in this plan covers: **how to read a disassembly**, **linking and interpositioning**, the **allocator design space**, and **byte order**. Read: **2.1.2–2.1.3** data sizes and byte ordering (§15.2 rung 0's big-endian ITCH/MoldUDP64 decode); **3.2–3.7** program encodings, AT&T formatting, data formats and size suffixes, operand specifiers, arithmetic (`leaq`/`imul`/`idiv`), condition codes, conditional moves, loops, procedures/stack frames/ABI (~85 pp, with the SPSC ring — the only source that makes Phase 0's disassembly gate answerable, since COD's ISA chapters are skipped; read **contiguously from 3.2**, v4.6 — a skill chapter does not survive being read at 60%, and 3.2/3.3 are the conventions 3.4 onwards assume); **3.9.3 Data Alignment** (~3 pp, with the false-sharing work); **6.6.1 *The Memory Mountain*** (~4 pp, added v4.5 — read with Phase 2's memory-hierarchy sweep for the footprint × stride framing; Drepper §3.3 owns the mechanism, so this is a label on an experiment, not a chapter to study); **7.1–7.13** object files, ELF layout, symbols and resolution, relocation, loading, dynamic linking, PIC, and **7.13 library interpositioning** in all three flavors (~42 pp, contiguous from v4.6 — 7.3–7.4 are the ELF sections 7.5 indexes, 7.7 is what PIC is built on, 7.8 is a prerequisite for 7.9; the v4.4 drop-through of 7.7–7.8 is retired — the canonical treatment of Phase 4's mechanism, which demotes COD 2.12 to a skim); **9.8.4 + 9.9** `mmap` and dynamic memory allocation (~32 pp — implicit/explicit/segregated free lists and boundary tags: the design space glibc's bins and tcache implement, owed to the Orderbook PMR rewrite and reusable at step G's KV block allocator). Optional, third tier: **8.5** signals, only if the kill switch delivers by signal; **3.11** FP code (xmm/ymm, so FP-loop disassembly reads); **5.7.2–5.7.3 + 5.11.2 + 5.12** functional-unit latency/throughput, the abstract dataflow model, misprediction penalties, and memory performance (~30 pp; 5.12 added v4.5 — store-to-load forwarding and memory aliasing, the explanation behind the ablation's IPC column. Framing only; Agner Fog is deeper and correct for your core); **12.7.1–12.7.2** thread safety and reentrancy (~8 pp, added v4.5 — only if the kill switch delivers by signal, read alongside 8.5: async-signal-safety is the constraint, and the four thread-unsafety classes are vocabulary CiA does not supply); **3.10.2** gdb (~2 pp, added v4.6 — the plan has no other debugging-tool source, and the ring's first crash wants one); **3.9.1** structures (~4 pp, with 3.9.3 — member offsets and the layout the alignment rule is applied *to*); **9.11** common memory-related bugs (~5 pp, with 9.9 — 9.11.9 *referencing data in free heap blocks* and 9.11.10 *memory leaks* are exactly the failure modes the arena rewrite invites); **10.5** the Rio package (~6 pp, with Phase 3's request path — short counts on `read`/`write` are the bug the buffered wrappers exist to prevent, and Beej's passing mention is the plan's only other treatment). Skip: **ch. 4** (Y86-64/HCL/pipelining — a simulator build for a toy ISA, and COD 4.6–4.11 owns hazards), **ch. 5** otherwise (Agner Fog), **ch. 6** except 6.6.1 (COD ch. 5 + Drepper), **10–12** (Beej's + Top-Down + CiA + OSTEP own that ground), **1, 3.1, 3.6.8, 3.8, 3.9.1–3.9.2, 3.10, 9.1–9.7, 9.10, App. A** (2.2–2.3 optional for the wraparound arithmetic RFC 1982 governs; 2.4 is the FP drop-through behind COD 3.5), and **every lab** — cachelab and malloclab rehearse ground Phase 2's experiments and the Orderbook rewrite already cover, with no artifact attached | Phase 0 (3.2–3.7 with the ring) → Phase 2 (3.9.3, 3.6.6, 6.6.1) → Phase 4 (7.1–7.13, 9.9) → step G rung 0 (9.9, KV block allocator) → §15.2 rung 0 (2.1.2–2.1.3) |
| DDIA | **Not a build-time read.** Skim Ch. 1–2, 5–6 pre-interview (consistency models, replication) | Post-project queue |
| System Design (Alex Xu Vol 1/2) | **Not a build-time read.** Vol. 1 as Q&A flashcards pre-interview; **Vol. 2 skip** | Post-project queue |

**Foundation supplements added by this plan (not on your original list):** McKenney's *Memory Barriers: a Hardware View for Software Hackers* (Phase 0), Preshing's memory-ordering posts (Phase 0, optional), Stokes' *Inside the Machine* (Phase 0, optional intuition primer), Sutton, *The Bitter Lesson* (Phase 0, ~15 min — scaling beats hand-built cleverness; the frame for every "why bother optimizing this" decision in the plan), Beej's Guide to Network Programming (Phase 3), a two-half full read of Drepper (Phases 0 + 2) that **supersedes COD's memory-hierarchy chapters (ch. 5) but not COD as a whole** — Drepper has no pipelining, no IEEE-754, no GPU, and no ISA-level synchronization, all of which stay COD's — and a **targeted CSAPP slice (v4.4–v4.6, ~175 pp)** covering the ground neither source has: x86-64 assembly/ABI and disassembly reading, linking and interpositioning, and the allocator design space.

**Added by v4 (load-bearing for decisions the plan makes you defend, ~2–3 weeks total):** Orca (iteration-level scheduling), vLLM/PagedAttention (KV paging) **plus vLLM's scheduler and block-manager source** (read the way step G reads llama.cpp), Pope et al. 2022 (batch economics and KV arithmetic), **Sarathi-Serve, OSDI 2024** (chunked prefill — read before building rung 2), one circuit-breaker/stability-patterns chapter, one rate-limiting-algorithms reference, Little's Law, **McKenney's *Is Parallel Programming Hard* — seqlock and RCU chapters** (phase 1's config publication), **HdrHistogram library docs** (phase 3's percentile mechanics), **one deterministic-replay / deterministic-simulation writeup** (Wilson's talk or Antithesis's posts), **one short fair-queueing treatment** (WFQ / deficit round robin — phase 6's tenant-fairness policy; **primary source is now Top-Down §4.2.5**, so this is the optional deeper treatment, not the only one), and the ONNX Runtime determinism docs as the source for phase 6's FP writeup.

**Added by v4.1 (all just-in-time, all free, each unlocking a specific build):** the kernel's `Documentation/networking/af_xdp.rst`, the `xdp-project/xdp-tutorial` AF_XDP lessons, and the `libbpf`/`libxdp` ring API (§15.2); the NASDAQ **ITCH 5.0 specification** including its MoldUDP64 framing section, plus the public sample data file; the **Arrow C++ / Parquet** docs and the **Arrow C Data Interface** spec (step B); and bpftrace/bcc one-liners plus the BPF verifier and map-type material (§14). Sanitizer documentation is tooling, not reading — it lives with the CI matrix in §14.

**Added by v4.2 (the Orderbook pooled-intrusive refactor's rationale — ~4h total, and four of the five are already *owed*: [01-career.md](01-career.md)'s interview map names malloc internals and `std::pmr` on the must-answer list with no source anywhere in this map):** the glibc **`MallocInternals`** wiki (bins, tcache, arenas — without it the "why not just use malloc" answer is undefendable, because glibc's per-thread tcache makes the common-case allocation genuinely cheap); cppreference on **`std::pmr::memory_resource`**, `monotonic_buffer_resource`, `unsynchronized_pool_resource` and `polymorphic_allocator` (read the `deallocate` semantics for monotonic specifically — it is a no-op, which makes it structurally wrong for any container that churns); the **Boost.Intrusive** docs (intrusive vs non-intrusive, hooks, `constant_time_size` — the canonical treatment of the one-container-at-a-time constraint and the slot-reuse aliasing hazard); **Berger, Zorn & McKinley, *Reconsidering Custom Memory Allocation* (OOPSLA 2002)** (the boundary condition — region/arena allocators win for specific lifetime patterns, custom *general-purpose* allocators lose to modern mallocs); and **Chandler Carruth, *Efficiency with Algorithms, Performance with Data Structures* (CppCon 2014)** (locality over big-O, measured). Two re-read targets change emphasis rather than content: **Drepper §3.3 and §6**, and **Agner Fog's data-structures and memory-access chapters** — the plan currently assigns only his branch-prediction section, the wrong chapter for this question. Full rationale, measurement plan, and six cold-recall questions: `Orderbook/docs/perf_refactor_rationale.md`.

**Added by v4.3 — the AI-infra completeness pass.** An audit against what AI-infra roles actually probe. Governing rule: **a reading that produces no claim is a boundary answer, not signal** — each item is tagged for which it is. Three tiers.

**Tier 1 — sources for builds already planned (~2–3 evenings).** The post-engine tier had drills and builds with no primary sources behind them, which produces "I built it and can't defend why."

- **Thakur, Rabenseifner & Gropp, *Optimization of Collective Communication Operations in MPICH* (2005)** — the alpha–beta cost model behind Tier A item 1. Without it the fitted terms have no lineage.
- **Gibiansky, *Bringing HPC Techniques to Deep Learning* (Baidu, 2017)** — ring all-reduce, and the ring-not-tree argument.
- **Shoeybi et al., *Megatron-LM* (2019)** — tensor parallelism: column- and row-parallel sharding, and the origin of "two all-reduces per layer."
- **Frantar et al., *GPTQ* (2022)** and **Lin et al., *AWQ* (2023)** — post-training quantization, for Tier B item 4. Both named as interview topics with no source behind them.
- **Leviathan, Kalman & Matias, *Fast Inference from Transformers via Speculative Decoding* (2023)** — Tier B item 5. States the batch-1 framing that makes the item's finding predictable.
- **Dao et al., *FlashAttention* (2022)** — required if A5 takes the fused-attention branch, and the cleanest second example of the IO-awareness reasoning A4 teaches.
- **Ainslie et al., *GQA* (2023)** — Tier A item 2's grouped-query attention, and the query-to-KV-head ratio.
- **Karpathy, `llm.c` (source read)** — GPT-2 in C/CUDA, no dependencies: the closest existing artifact to step G's build. llama.cpp stays the KV-cache and sampling reference; llm.c is what the forward pass gets diffed against.

**Tier 2 — two builds this pass adds (both chained off interfaces the plan already specifies).**

- **Prefix caching** — phase 6 rung 2b. Chains off step G's KV allocator interface; no new seam. Reading: vLLM's prefix-caching / block-hashing docs and the SGLang RadixAttention paper (Zheng et al., 2023) — the latter for *why a radix tree rather than a hash map*.
- **Triton GEMM** — step A6. The same GEMM as the CUDA kernel, diffed against both your CUDA and cuBLAS. Reading: the Triton docs and its GEMM tutorial — a language, not a textbook.

**Tier 3 — vocabulary with sources (~1 week of evenings, boundary answers only, no build claims).** Every item lands in the industry-context Q&A (§9) with a boundary line. These are standard probes, and "never heard of it" is a worse answer than "I know what it solves, here is why this project does not have it."

- **Training-side distributed:** FSDP/ZeRO (Rajbhandari et al., 2019; PyTorch FSDP docs), pipeline parallelism (Huang et al., *GPipe*, 2019), mixed precision (Micikevicius et al., 2017), checkpointing and fault tolerance (Megatron/DeepSpeed docs). One paragraph each; the boundary is the project's own non-goal.
- **Tensor cores and architecture-specific paths:** CUTLASS docs, and the Hopper TMA + `wgmma` sections of the CUDA C Programming Guide. Also sharpens A5's cuBLAS gap attribution.
- **Quantization formats and post-training methods:** the GGUF spec (quantization block layout, k-quants), Frantar et al., *GPTQ* (2022), and Lin et al., *AWQ* (2023) — already cited at Tier B item 4, promoted here so the *format* vocabulary (weight-only vs. activation, block size, mixed precision, size-vs-perplexity tradeoff surface) has a boundary answer, not just the methods. Boundary: the plan builds weight-only INT8 at block level on one checkpoint (Tier B item 4's curve); it does not build a format or a quantizer tool. "I know what each format solves and where my INT8 point sits on that tradeoff surface" is the complete answer.
- **Multi-LoRA serving:** S-LoRA (Sheng et al., 2023) and Punica (Chen et al., 2023). Boundary: one model by design.
- **Structured / constrained decoding:** Willard & Louf, *Efficient Guided Generation* (2023), plus the `xgrammar` or `outlines` docs. A real production topic with no analogue here — state it as a gap.
- **KV cache offloading:** FlexGen (Sheng et al., 2023) and vLLM's offloading docs. Boundary: device-memory-only.
- **GPU sharing and multi-tenancy:** MIG and MPS documentation, time-slicing. Boundary: one process, one device.
- **Evaluation harnesses:** `lm-evaluation-harness`. Boundary: a token-for-token correctness gate, not a benchmark suite.

**Not added, and the reasons stand:** Kubernetes / Ray / OpenTelemetry / operators (tooling, not findings); multi-model serving (conflicts with the non-goal); distillation, pruning, mixture-of-experts (frontier-*model* vocabulary, not infrastructure work); serving a real 7B (the architecture-breadth argument closes the same gap with engineering). Training-infrastructure as a *built* door remains out of scope — Tier 3 is the boundary answer, not a plan branch.

**Added by v4.4 — the CSAPP supplement (~125 pp, at existing phase slots; no new phase).** CSAPP was evaluated as a *replacement* for COD and rejected. The overlap is roughly a third (caches, VM, float representation), and the non-overlapping halves point in opposite directions: CSAPP's back half duplicates sources this plan already has and prefers (networking → Beej's + Top-Down; concurrency → CiA + OSTEP; memory hierarchy → Drepper + COD ch. 5; optimization → Agner Fog), and it stops short of the material step A1, Phase 2's branch work, A5's quantization reasoning, and COD 2.11 rest on: CSAPP's ch. 4 carries the classic in-order pipeline hazards but stops before superscalar, out-of-order execution, and speculation — which is what COD 4.11 adds, and what reasoning about a modern core's branch-miss cost actually requires. It has no GPU or accelerator chapters, and no ISA-level synchronization (no LL/SC to describe, since x86 lacks it). Its float chapter (2.4) is a peer of COD 3.5, not a replacement for it. What CSAPP owns outright, with no other source in the plan, is four things. **How to read a disassembly** (ch. 3): Phase 0's ring gate asks for the acquire/release vs. `seq_cst` explanation *from your own ring's disassembly*, and the plan skips COD's ISA chapters — this is the only reading that makes that answer reachable. **Linking, loading, and the three interpositioning flavors** (ch. 7, whose 7.13 is the canonical treatment of Phase 4's mechanism): it replaces an existing assignment rather than adding one, demoting COD 2.12 to a skim. **The allocator design space** (9.9: implicit and explicit free lists, boundary tags, coalescing, segregated free lists) — the teaching version of the structures glibc's bins and tcache implement, owed to the Orderbook pooled-intrusive rewrite and reusable at step G's KV block allocator, where the same problem reappears with fixed-size blocks. **Byte ordering** (2.1.3), for the big-endian ITCH/MoldUDP64 decode §15.2 rung 0 needs. Two marginal additions are optional and named as such (8.5 signals; 5.7.2–5.7.3 + 5.11.2 processor model, where Agner Fog is the deeper source). The labs are skipped deliberately: cachelab and malloclab rehearse ground Phase 2's experiments and the Orderbook rewrite already cover, and produce no artifact this portfolio can cite. Net effect on hours: approximately neutral — the slice is small, and it is paid for by the COD 2.12 demotion and by CSAPP's back half being explicitly out of scope.

**Added by v4.5 — a citation gap and one experiment axis (~8 pp of new reading, near-neutral).** Two findings from re-auditing COD against CSAPP for the quant-dev and AI-infra targets. Neither changes the v4.4 verdict; one of them corrects how v4.4 justified it.

- **(1) Phase 3's sequence-wraparound test had no source.** The staleness design names the exact hazard — `next != prev + 1` is wrong at the boundary, and the correct test is a distance computation on the unsigned difference — and cited nothing. That test is **RFC 1982, *Serial Number Arithmetic*** (~4 pp). Neither textbook covers it, so the plan was asking for a decision it gave no mechanism for; the citation is now attached where the decision is made. Optional depth behind it, if COD 2.4 reads thin on modular arithmetic and conversion: **CSAPP 2.2–2.3** (~25 pp), contiguous with the 2.1.3 byte-ordering read.
- **(2) Phase 2's memory-hierarchy sweep was one-dimensional.** It swept footprint but not stride, which leaves L1/L2 associativity conflicts unmeasured — and those conflict valleys are how you *measure* a die's associativity rather than look it up. The sweep gains a second pass at fixed footprint with a swept stride (one extra loop around the existing `mem.c` harness), with **CSAPP 6.6.1 *The Memory Mountain*** (~4 pp) supplying the framing label. Drepper §3.3 already carries the mechanism (critical stride), so this is an experiment, not a chapter. Side effect: the dense-vs-hash item, previously reasoning-only, gains a measured half.
- **Two optional third-tier CSAPP sections join the existing optional line** (neither scheduled): **5.12 *Understanding Memory Performance*** (~10 pp) — store-to-load forwarding and memory aliasing, the vocabulary that makes the ablation's IPC column explainable, which the plan otherwise records without explaining; and **12.7.1–12.7.2** thread safety and reentrancy (~8 pp) — conditional on the kill switch delivering by signal, where async-signal-safety is the real constraint and the four thread-unsafety classes are vocabulary CiA does not supply.

**Correction carried in the same pass, no page cost:** §12's COD and CSAPP rows and §17's v4.4 rationale called COD the plan's "primary source, not backup" and claimed it holds "the material the AI-infra half is built on." Both overstate it, and the first contradicts Phase 0 item 34's own rule that second sources reinforce a model rather than build it. The plan's primaries are Drepper, OSTEP, CiA, Agner Fog and PMPP (§3). COD holds the architectural *model*; CSAPP holds the *mechanisms the builds touch directly*; COD's larger second-source budget is page-economics — its skippable chapters make it cheaper per unique page — not precedence.

**Added by v4.6 — the continuity audit (+~45 pp CSAPP, ~+2 pp COD, three contradictions fixed).** Every skip list on this page was re-tested against a question the earlier passes did not ask: *is this skip safe because another source owns the ground, or is it a hole in a chapter being read as a skill?* The distinction matters because a list selected on "produces a claim" is the wrong instrument for a chapter whose output is an *ability*. Findings:

- **CSAPP ch. 3 was being read at 60% and it is a skill chapter.** 3.2 *Program Encodings* (AT&T formatting, how the `.s` is produced) and 3.3 *Data Formats* (the b/w/l/q suffixes) are the conventions everything from 3.4 onward assumes; 3.5 *Arithmetic and Logical Operations* is `leaq`, `imul`, `sar`, `idiv` — what real disassembly is actually made of; 3.6.7 *Loops* is the control structure surrounding every conditional move 3.6.6 teaches. The slice becomes **3.2–3.7 contiguous**, ~55 → ~85 pp.
- **CSAPP ch. 7 had three prerequisites sitting on the skip list.** 7.3–7.4 (object files, ELF section layout) are what 7.5's symbol table lives *inside*; 7.8 is a prerequisite for 7.9, not a bonus; 7.7 (relocation) is what makes 7.12's PIC more than a compiler flag. The v4.4 drop-through of 7.7–7.8 is retired and the read becomes **7.1–7.13 contiguous**, ~30 → ~42 pp.
- **CSAPP 2.1.2 *Data Sizes*** joins 2.1.3 at §15.2 rung 0 — a byte-order example is unreadable without knowing a `long` is 8 bytes and a `short` is 2 on LP64.
- **COD ch. 4 had the same shape.** 4.6 pipelines the single-cycle datapath 4.4 builds, so **4.4 becomes a declared drop-through** (read its ~12 pp if the IF/ID/EX/MEM/WB stages land cold), 4.3 is skimmed for the datapath elements, and 4.1 is skimmed for the chapter's frame. 4.2 (logic design) and 4.5 (multicycle) stay safe skips — you are not building gates, and multicycle is a stepping stone pipelining supersedes.
- **Agner Fog had one chapter assigned where the argument needs three.** COD 4.11 supplies the out-of-order *model*; the branch-prediction chapter alone leaves you naming a misprediction penalty with no reorder buffer behind it. His out-of-order execution and scheduler chapters join it.
- **Three internal contradictions, all caught by the same pass.** OSTEP's "do not schedule 26–30" contradicted Phase 0 item 17's non-negotiable ch. 28; §11's "still skipped" list contradicted Phase 0 items 27–28, which read CiA ch. 9–10; and the COD row skipped 6.13 while the matrix-multiply thread read it at step A4. Corrected in place.
- **One tool gap named rather than fixed:** the plan has no gdb source anywhere, despite a C++ project with a hand-rolled lock-free ring. CSAPP 3.10.2 (~2 pp) joins the optional tier as the pointer; the real source is the manual, reached when the first crash happens.

**Net:** CSAPP ~130 → ~175 pp — still ~16% of the book, still a targeted slice. COD gains ~2 pp of skims. The hours are not new work: every addition is a prerequisite the plan was already implicitly assuming.

**Nothing else joins the list.** If a new reading appears that is not on this page, it is drift, not curriculum.

---

## 13. The explain loop

The other three legs of the loop have instructions. This is the leg that collapses into "yeah, I get it" — so it gets a mechanism.

**Design principles:** **Retrieval beats review** (producing an answer from nothing builds memory; re-reading builds familiarity, which feels identical and isn't). **Speech beats writing** (written answers let you hedge, edit, and look things up mid-sentence; interviews are spoken). **Spacing beats volume** (one question re-asked at 1 / 2 / 4 weeks beats the same question answered once, thoroughly). **Probing beats self-grading** (you can't grade your own gaps; the follow-up question is the instrument). **Cheap beats thorough** (20 min/week survives a busy month; a 3-hour monthly review does not).

### 1. Weekly cold recall (~20 min)

Run in a **fresh chat** with this file attached. Prompt:

> Cold recall. Follow §13 of the master spec. Draw 3 questions: 2 from the current phase, 1 from an earlier phase chosen by the re-ask schedule in my log. Ask one at a time, wait for my spoken answer, then probe once with the "why not X / what's the boundary / what breaks it" question. Then color it and tell me exactly what I missed.

**Question sources, in priority order:** (1) the phase's 🎯 drill block; (2) the phase's row in the transfer table (§2); (3) any claim you've shipped — provenance, mechanism, boundary; (4) every Amber and Red in your own log (always eligible).

**The run:**
1. Answer **out loud, no notes, ~2–3 min each.** If you can't start within 15 seconds, that's a Red — don't grind.
2. Take **one probe** per answer. The answer you can't give is the real gap.
3. **Color it:** **Green** — clean, with both mechanism and boundary. **Amber** — right shape, missing mechanism or boundary. **Red** — couldn't produce it.
4. **Log it** in `drills/recall-log.md`: date, phase, question, color, and the exact thing you missed.
5. **Promote:** Amber/Red → an Anki card (front = the question; back = *your own* answer, transcribed and corrected — your gaps build the deck, not a premade one). Green → paste the answer into `drills/answers/<topic>.md`; you'll reuse it verbatim in an interview.

**Re-ask schedule (a Leitner box, no app needed):** Red → next week. Amber → 2 weeks. Green → 4 weeks. Green twice → 8 weeks, then retire. The color column in the log *is* the scheduler.

### 2. Monthly recorded walkthrough (~30 min)

Once a month, record **5 minutes, no notes, no edits**: explain the current artifact to an imaginary interviewer — what it does, why this design over the obvious alternative, what you measured, where it breaks. One take. No re-recording. Watch it back once, and timestamp the moment you started hedging or hand-waving. That timestamp is next month's syllabus. Keep every recording: the month-1 → month-5 delta is progress evidence, and the last one per artifact is a real interview prop.

### 3. End-of-phase transfer checkpoint

The five prompts in §2 — **answer them out loud first, then write.** Writing first lets prose hide exactly the gaps that speech exposes.

### 4. The prediction ledger (attached to every measurement)

Before each experiment, write down: the predicted direction of the primary metric, and the predicted direction of the mechanism counter. Run it. Record hit or miss. A mental model is only real if it predicts. This is the cheapest calibration instrument in the plan, and it turns `perf stat` from a reporting tool into a testing tool ([perf.md](perf.md) §4 already asks for this in Layer 2 — the ledger makes it a habit). Lives at the top of each `bench/results/<experiment>.md`, above the raw output.

**What you have by interview season:** `drills/recall-log.md` (dated weak-spot list sorted by color — drill the Reds) · `drills/answers/` (your own answers, already in interview register, already correct) · `drills/recordings/` (the delta, plus one clean walkthrough per artifact) · Anki deck (the atomic facts your answers depend on, generated by your own gaps).

**Budget:** ~20 min/week + ~30 min/month ≈ **1.5 hr/month.** If it costs more, it's being done wrong.

---


## 14. Verification & observation (sanitizers, CI, and the cost of watching)

The plan declares its correctness gates never-cut in several places; this section says how they run. A gate that only exists on the machine where it was written is a claim about a moment, not about the code. And the second, sharper point: **every tool that watches a running system costs something**, and on a sub-microsecond path that cost is a design constraint.

### 14.1 The sanitizer matrix (all three repos)

**Sequence (per repo):** add a CMake preset per sanitizer mode → write the boundary tests and the fuzz corpus → wire the CI matrix over `{asan+ubsan, tsan}` plus the fuzz job → commit the corpus and every crash as a regression case → link the run from the README.

Same matrix for Orderbook, Backtester, and the inference engine. It is cheap, it is the highest signal-per-hour item available outside the engine spine, and none of the three repos has any CI today.

| Tool | Catches | Blind to | Cost |
| --- | --- | --- | --- |
| **ASan** (+LSan) | heap/stack/global buffer overflow, use-after-free, double-free, invalid free, leaks; use-after-return with `ASAN_OPTIONS=detect_stack_use_after_return=1` | uninitialized reads, data races | ~2× slowdown, large shadow memory |
| **UBSan** | signed overflow, bad shifts, misaligned access, invalid casts, VLA misuse, null deref | anything merely *wrong*, not undefined | ~free (a few %) |
| **TSan** | data races, lock-order inversion, use of destroyed synchronization | races the run never schedules; races in hand-rolled synchronization it cannot recognize | 5–15× slowdown, heavy memory |
| **MSan** | uninitialized-memory reads | **everything your dependencies do** | ~3× |
| **libFuzzer** | whatever the sanitizer underneath can see, over generated inputs | inputs your mutator cannot produce | the sanitizer's cost, times the corpus |

**Rules that make the matrix real:**

1. **UBSan must abort in CI.** By default it warns and continues, which means a green build that printed an undefined-behaviour line nobody read. `-fno-sanitize-recover=all` or it does not count.
2. **ASan and TSan are separate builds, never combined** — mutually exclusive, and TSan's memory overhead means you run it on the concurrency-focused targets, not the whole world.
3. **MSan only if you build the world.** It requires *every* dependency — including libc++ — instrumented; against a stock libstdc++ it reports false positives inside library code. Practical rule: ASan+UBSan everywhere, TSan on concurrent targets, MSan in a purpose-built container or not at all. Write the limitation down; knowing why MSan lies is worth more than a green badge you cannot trust.
4. **Never benchmark under a sanitizer.** Different allocator, timing, memory layout. ASan numbers are not latency numbers.
5. **TSan and the SPSC ring.** Hand-rolled lock-free code may be flagged or invisible to TSan depending on how the synchronization is expressed; annotations (`ANNOTATE_HAPPENS_BEFORE`/`_ACQUIRE`) exist for exactly this. If annotation is not worth it, say so explicitly in the design doc and lean on the other two legs — deterministic replay and the fuzz machinery — rather than quietly accepting a TSan exclusion.

**Fuzzing under the sanitizer is the point.** The corpus already exists in spirit ([10M]-sequence admission fuzz, the request-sequence fuzzer, and — with §15.2 — a frame deserializer that eats untrusted bytes). Wire it as: short fuzz run on every PR, long run nightly, corpus committed, every crash kept as a regression case. That converts "fuzzed with zero divergences" from a past event into a property that holds on every commit.

**Wiring (per repo, ~a day each):** a GitHub Actions matrix over `{asan+ubsan, tsan}`, a third job for the fuzz corpus, `sccache`/`ccache` plus a CMake preset per sanitizer mode (the repos already have `CMakePresets.json` — add a preset rather than inventing a second build system), and a README line linking the run. No Bazel, no remote execution, no build-system migration: the signal is that the matrix exists and is enforced.

**Claim this section earns (proof-pack sentence):** *"CI matrix green across ASan/UBSan and TSan on every push; the [N]-sequence fuzz corpus runs under ASan nightly with zero findings since [date]."*

### 14.2 The cost of watching (eBPF, and where it is legitimate)

**The problem:** you cannot observe a sub-microsecond hot path without changing it. Instrumentation has a price — a trap, a probe, a map update, a cache line touched — and on a 200ns decision the price is measured in multiples of the thing being measured. The question is never "how do I trace this?" It is **"how much does this observation cost, and where is observation still legitimate?"**

**What eBPF is genuinely for here** — not the hot path, but the things your own code cannot see about itself: **kernel involvement, proven absent** (syscall counts, `sched:sched_switch` off-CPU time, CPU migrations on a pinned thread, page faults — how you *prove* the zero-alloc and zero-syscall claims, and it corroborates the §15.2 rung deltas from the outside); **lock contention and futex behaviour** on the reference implementation (the mutex-based oracle) — the "what the obvious implementation actually costs" number, observed from outside the process; and **the observation-cost measurement itself**.

**The experiment: measure the observer.** Design it like any other ablation:

1. Pick a function on the hot path with a known cost (`perf stat` cycles/op over [N] iterations, pinned, no probes).
2. Attach a bpftrace uprobe on entry, then entry+exit; re-run the *same* harness.
3. Report the delta in cycles/op and in p99 — the perturbation, in nanoseconds, on a path you already know the length of.
4. State the conclusion in one sentence: at this cost, in-band tracing of this function is or is not viable, and what the production alternative is.

Expect "not viable" on a sub-microsecond path, and expect that to be the *interesting* result rather than a failure. **The alternative is in-band:** an append-only event log (the pattern the Orderbook already uses) plus a histogram published to a reader through a seqlock — the exact mechanism Phase 1 already assigns you the McKenney reading for. One continuous idea: the same seqlock that publishes config to a hot path can publish a latency histogram out of one, and the cost is a store plus a fence. The same snapshot then feeds the metrics exporter — the hot path writes counters, the exporter reads a consistent snapshot, and no scraper ever touches the decision path.

**Reads (all free):** bpftrace one-liners and the `bcc` tool list first (an evening, and enough for everything above); then the BPF verifier constraints (bounded loops, 512-byte stack, no arbitrary pointers) and per-CPU vs. hash maps, which explain why the cheap probes are cheap; `ringbuf` vs. perf buffer if you go beyond counters. Half a day total, just-in-time.

**Claim this section earns:** *"Measured uprobe overhead at [N]ns on a [M]ns decision path ([P]% perturbation); replaced in-band tracing with a [K]ns seqlock-published histogram and reserved eBPF for syscall, scheduler, and off-CPU observation, where it cannot perturb the path."*

**Drills:** why can't you just add a timer around the hot path? What does a uprobe cost, and what does that cost *mean* on a 200ns path? What can eBPF see that your process cannot see about itself? Why a per-CPU map instead of a shared one? What does the verifier refuse to let you do, and why does that make the program safe to run in the kernel at all?

**Boundary:** observability literacy plus one honest measurement, not a production telemetry stack. No OpenTelemetry, no metrics pipeline, no dashboards. Say so if probed; the finding is the cost, not the infrastructure.

**Sequencing:** 14.1 is the "in a gap" work — no hardware needed, unblocks nothing else, fills a half-week between engine phases, and should be done **before** the Phase 1 fuzz gate is claimed as shipped. 14.2 is engine-side, runs after the gateway exists (Phase 4's zero-alloc verification is the natural pairing — same subject, two observation methods), and its measurement needs a pinned, quiet machine and the playbook's A/B/A discipline to be believable.

---

## 15. Quant anchors

Anchors are the Orderbook and Backtester workstreams. They pause whenever the engine spine needs the hours — the pause is built into the design, not a failure of it. Resume discipline: each deepening adds at most one bullet with a before/after measured number.

### 15.1 Deepening queue

**Sequence:** after Phase 1 → Orderbook generational slot map. After Phase 4 → Backtester streaming ingest, then the Orderbook PMR intrusive list. Once the Orderbook replay harness is stable → the ingestion ladder, starting at rung 0 with the ITCH parse. After Phase 3 if green → Orderbook microstructure features (this is step H). After shipping → Backtester TWAP slicing. The SPSC port into the Backtester sampler is not on this list — it is cancelled, and the paragraph below says why.

**Re-gated; the SPSC port is cancelled, not deferred.** The generational slot map still runs as Phase 1's first deepening: it retires a claim currently **on the resume and not in the code**, so its priority is credibility, not marginal signal. The SPSC port into the Backtester sampler is cancelled outright — a queue has no job there. The Monte Carlo workload is embarrassingly parallel with uniform per-path cost and exactly one output slot per path, so an index-partitioned parallel-for is both correct and faster than a queue: no handoff latency, no contended head/tail cacheline, no load imbalance for a queue to smooth. Building one to justify a bullet would have produced a design an interviewer dismantles in a single question ("why not a static partition?"). The bullet was corrected instead. The PMR intrusive list stays gated on Phase 4, because it needs the object-lifetime and interposition material taught there; until it lands, the PMR clause stays off the page — **it is currently on the page anyway, in the Orderbook block, and that is a live contradiction** (see the claim audit in [01-career.md](01-career.md)). General rule: **a bullet whose claim depends on unshipped work does not ship until that work lands.**

| When | Project | Addition | Effort | Why |
| --- | --- | --- | --- | --- |
| Gate: Orderbook replay harness stable and its correctness gate green. **First item (rung 0, needs no NIC): real ITCH 5.0 + MoldUDP64 parse on the existing feed — the resume already claims it.** *(Audit update v4.7: the ITCH 5.0 parse over an mmap'ed NASDAQ sample file and the map → vector → slab ladder have since shipped and are measured — 38.15M msg/s, 26.2 ns, in `docs/optimization_log.md`; only the MoldUDP64 framing half remains unshipped. The rung-0 item is therefore down to the framing/sequence-gap layer, not the parser.)* Then the ladder proper | Orderbook | **Network-ingestion ladder** — AF_XDP rungs 0–3, NIC hardware timestamps, a real ITCH message mix. **Full spec: §15.2** | rung-sized: ~2–4 days (rung-0 ITCH parse) or ~2–3 days (rung 1) to ~4 weeks (all rungs) | The only place in the portfolio where the kernel network path is the measured bottleneck. Cashes in the Phase 2 syscall/context-switch physics; makes the feed handler part of the artifact instead of a fixture. Rungs 0–1 + timestamps need no special hardware |
| After Phase 4 | Backtester | Streaming ingest: the SPSC ring as bar producer with the strategy loop as consumer (`include/datafeed.h` currently exposes only a blocking full-file `load`) | ~3 days | Puts SPSC where a real producer/consumer exists. Retrofit it here and you also get to answer "why is a queue right here and wrong in the sampler?" — a better answer than the queue itself |
| After Phase 1 | Orderbook | Generational slot map for the order-ID index (dense arena, IDs = (index, generation)); **fills must be bit-identical vs. the old index on the 1M-message replay** | ~2–3 days | Dense-vs-hash + cache reasoning from Phases 0–1 turned into a measured rewrite. *(Audit update v4.7: the map→vector→slab ladder has shipped and is measured — the 3.9×-class numbers are real, not targets. This slot map remains unshipped and is the next container rewrite.)* |
| After Phase 4 | Orderbook | Per-level FIFO as PMR-pooled **intrusive list** — placement-new'd slots carved from a pre-committed arena owned by a `std::pmr` resource, with a hand-rolled free-list reclaiming slots. **Not** a `monotonic_buffer_resource`: its `deallocate()` is a no-op, so it cannot serve a FIFO whose orders churn all day; same bit-identical replay check; re-benchmark | ~3–5 days | Zero-alloc hot path on two projects; reuses Foundation #5. The resume claims this clause as done today while the re-gating note says it stays off the page until this lands — resolve in favour of the page |
| After shipping | Backtester | TWAP child-order slicing + depth-aware fills (fill at best bid/ask, consume visible depth, slippage as first-class output) | ~2–3 weeks | The biggest conceptual upgrade available — makes the backtester honest about execution; keep it to TWAP + depth, don't balloon into VWAP/POV |
| After Phase 3 if green; otherwise after shipping | Orderbook | Microstructure features: order-flow imbalance, microprice, queue-position estimate — feedable as backtester signal inputs; this is AI step H | ~1–2 weeks | Closes the loop: orderbook → features → backtester; first AI module |

**Cap the internal-container rewrites at two or three total across all projects** — the signal saturates fast, and a fourth reads as compulsive rather than principled. The cap counts *container rewrites* (the ladder, the slot map, the PMR list); **infrastructure work does not count against it**, because the ingestion ladder changes what the system *is* rather than how its containers are spelled. The anchor set is five items: the ingestion ladder (new surface), H (new domain), B (new dataset layer), D (new statistics), and the two container rewrites already queued. Cap the third kind, not the others.

**Recorded, not adopted:** a TensorRT crossover study — a batch-size sweep locating where a GPU beats the CPU path for a sub-microsecond decision. Real skill, real finding, but it belongs to a *different* project than the two this queue covers, and it re-opens the cut-first second backend (§11).

### 15.2 Orderbook: network ingestion ladder (AF_XDP)

**Sequence:** rung 0 (the loopback feed, plus the real ITCH 5.0 + MoldUDP64 parse, plus NIC wire timestamps) → rung 1 (`SO_BUSY_POLL` + `PACKET_MMAP`/tpacket) → rung 2 (AF_XDP copy mode) → rung 3 (AF_XDP zero-copy). Each rung is measured against the last before the next is written, on the same recorded stream. Rung 3 is the first cut; wire timestamps, the fills-bit-identical gate, and gap/loss testing are never cut.

**Shape:** the Orderbook's matching core is measured; its *ingestion path* is not. Everything arrives through a synthetic loopback feed, and the claim on the page says so. This replaces the synthetic path with a real one, in rungs, each measured against the last — so the feed handler becomes part of the artifact rather than a test fixture, and the latency number gains a wire end.

**Why this is the Orderbook's next problem and not a new project:** a matching engine's latency budget is dominated by what happens *before* the book sees the message. The kernel network path is interrupt → NAPI poll → `skb` allocation → protocol processing → socket queue → wakeup → `recvmsg` → copy. That is two copies and a scheduler round trip per packet, all of it measurable, all of it removable. Phase 0's syscall cost model and Phase 2's kernel-bypass drill explain *why* it exists; this is where the explanation gets a number attached.

**Placement and gate:** a Quant Anchor. Gate: the Orderbook's replay harness is stable and its correctness gate is green (bit-identical fills on the 1M-message replay). Like every anchor it **pauses** whenever the engine spine needs the hours — and of all the anchors, this is the one sized to need that rule.

| Rung | Path | What it isolates | Hardware needed | Effort |
| --- | --- | --- | --- | --- |
| **0** | the existing UDP-multicast loopback feed — **plus the real ITCH 5.0 + MoldUDP64 parse, done here, on this path** | baseline: everything the kernel does per message; the payload stops being synthetic | none | done + ~2–4 days |
| **1** | `SO_BUSY_POLL` (`SO_PREFER_BUSY_POLL`, `net.core.busy_poll`) + `PACKET_MMAP`/tpacket ring | the interrupt/wakeup cost alone, and one of the two copies | none — works on any NIC, including a VM | ~2–3 days |
| **2** | AF_XDP **copy mode** (`XDP_REDIRECT` into a UMEM socket, generic or native XDP) | the socket layer and `skb` gone; DMA→kernel→UMEM copy remains | any NIC with XDP support; generic/SKB mode works everywhere | ~1 week |
| **3** | AF_XDP **zero-copy** (`XDP_ZEROCOPY` bind) | the last copy — NIC DMAs straight into your frames | a driver that implements it, plus a packet source | ~1–2 weeks |

**Rung 0 reading (added v4.4):** the NASDAQ **ITCH 5.0 specification** with its MoldUDP64 framing section (already in this map, v4.1) plus **CSAPP 2.1.2–2.1.3, *Data Sizes* + *Addressing and Byte Ordering*** (~10 pp; 2.1.2 added v4.6 — a byte-order example is unreadable without knowing that a `long` is 8 bytes and a `short` is 2 on LP64). ITCH is a big-endian wire format, and this is the one place in the plan where the host's byte order and the wire's disagree — read the spec without that background and a field decode ends up silently byte-swapped, which is a bug the replay's fill comparison will surface as a mystery rather than as a diagnosis.

Build it in rungs, exactly like the GEMM ladder and the scheduler. Each rung is measured before the next is written, and **a rung that buys nothing is a finding worth keeping** — the rung-2 or rung-3 number is often "less than expected, here is why," and that answer is worth more in an interview than a stack of wins.

**Rung 1 is the highest value per hour in this workstream.** `SO_BUSY_POLL` converts the wakeup path into a spin and tcpdump-visible behaviour does not change at all; the tpacket ring removes one copy with no new dependency. If the ladder stops after rung 1, the workstream has still produced a real before/after.

**Hardware reality — three separate constraints, each one can stop you:**

1. **XDP mode.** Native XDP needs a driver that implements it. Consumer onboard NICs are the usual blocker — Intel `igc` (i225/i226, common on X670 boards) does native XDP but **not** zero-copy; Realtek typically does neither. Check with `xdp-tools`' `xdp-features` and `ethtool -i`, and treat a failed `XDP_ZEROCOPY` bind as information rather than an error to work around. Cheap enablers: a used Intel X520/X710 (~$50–80, `ixgbe`/`i40e`, solid XDP + zero-copy), or one cloud bare-metal afternoon.
2. **The packet source.** The practical blocker nobody mentions: AF_XDP needs packets arriving on a real NIC, and a loopback feed is not one. Options, best first: (a) **two ports and a cable** — transmit the recorded feed from port A, receive on port B with AF_XDP (`tcpreplay` or your own sender); (b) **a second machine** on a direct link; (c) **`pktgen`** (in-kernel, `/proc/net/pktgen`) for throughput work where realistic content does not matter; (d) a QEMU VM with `virtio-net`, which supports XDP and, on recent kernels, AF_XDP zero-copy. **WSL2's `hv_netvsc` supports neither native XDP nor zero-copy**, so the Windows-side dev loop can build and unit-test the ring plumbing but cannot produce rung-2/3 numbers. Develop anywhere; measure on bare metal.
3. **Root.** AF_XDP, XDP program loading, and `ethtool -T` all need privileges. Note in the design doc that the data path is unprivileged *once bound* — "the loader needs root, the hot path does not" is itself a good boundary answer.

**Recorded-feed rule:** reuse the same recorded stream as the workload across every rung. Same messages, same order, same seeds — otherwise the rungs are not comparable and the ladder is decoration. **With the rung-0 ITCH parse landed, that stream is real traffic rather than a synthetic mix**, which is what makes every rung delta defensible: message *mix* is what makes a book's latency distribution bimodal (the modify/cancel fast path vs. the price-level walk), and a synthetic mix hides exactly that.

**The architectural claim: UMEM is your PMR pool.** AF_XDP hands you a **UMEM**: one pre-committed, contiguous region you split into fixed-size frames and register with the socket. The kernel and (in zero-copy mode) the NIC allocate nothing — they hand frames back and forth through four rings: **FILL** (you give empty frames to the kernel), **RX** (the kernel gives filled frames back), **TX / COMP** (the transmit side and its completion ring; build them only if you transmit). Every ring is a single-producer/single-consumer index pair with explicit barriers — the same structure as the Phase 0 SPSC ring, with one difference that is the whole lesson: **the producer is hardware**, so the C++ memory model no longer covers the relationship and `libbpf`'s `xsk_ring_prod`/`xsk_ring_cons` accessors exist precisely because of that. An order is deserialized *in place* inside the frame, and the book takes an index into the arena. That is the claim worth making: **the same arena allocator serves the order book and the NIC.** It is the single most cohesive sentence across the two C++ projects, and it only becomes true if you resist the urge to copy frames into a "real" buffer on arrival. Resist it. If a rung requires a copy to work, write down why — that is a finding about the driver, not a licence to abandon the design. Cache-line discipline carries over unchanged: ring index pairs get their own lines, and `perf c2c` now shows contention between a DMA engine and a core rather than between two cores.

**Wire timestamps (do this first, at rung 0).** Without a hardware timestamp, every latency number you own is userspace-to-userspace, and the rung-3 story is unfalsifiable. `ethtool -T <iface>` tells you what the NIC can timestamp; `SOF_TIMESTAMPING_RX_HARDWARE` is what you want, and if the NIC exposes a PHC (`/dev/ptp*`) you can discipline it with `phc2sys`/`ptp4l` for a genuinely trustworthy clock. Use `SO_TIMESTAMPING` on the socket, or the AF_XDP equivalent via `xsk_umem__get_data` + the RX ring's timestamp field where the driver provides it. Record, per message: wire-arrival (hardware), userspace-visible, book-updated. The **wire-to-book** number is the one that goes on the page; the other two are the mechanism. If the NIC cannot hardware-timestamp, say so and label the claim userspace-to-userspace — an honest boundary beats an unlabelled number.

**Real message mix: a real ITCH file — and it is not ladder work.** The resume already claims this outright ("Implemented ITCH 5.0 ingestion with sequence-gap detection and a Synced/Recovering resync state machine"), and the audit says zero `itch`/`moldudp` hits — so this is a live contradiction to resolve, not an optional upgrade. **It is also independent of the rungs**: the ladder changes the *transport*, the ITCH parse changes the *payload*. Ship it **at rung 0, on the existing loopback UDP feed** — no NIC, no root, no XDP — and every later rung inherits a real message mix for free. **Having parsed the real thing removes the question entirely**, at 2–4 days against ~4 weeks for the ladder, and it pays immediately rather than at rung 1. NASDAQ publishes the **ITCH 5.0 specification** and public **sample data** files — real symbol universe, real message mix, real order sizes. Transport framing is part of the work: **MoldUDP64** for the multicast feed (sequence numbers, heartbeat, gap detection) or the TCP variant. **MoldUDP64 is the half the resume claim actually rests on** — "sequence-gap detection and a Synced/Recovering resync state machine" is protocol-layer behaviour, and ITCH alone does not supply it; both halves are needed for the sentence to be true as written. Gap handling is what makes this more than a parser — it is the same staleness/sequencing story Phase 3 keeps for the engine, in its native habitat. Then the same replay harness runs a real message mix, and the latency distribution stops being synthetic: message *mix* is what makes a book's latency distribution bimodal (the modify/cancel fast path vs. the price-level walk), and a synthetic mix hides exactly that. **Scope fence, or it balloons:** decode only the message types the apply path consumes (add, execute, cancel, delete, replace, plus whatever else `orderbook_vector.h`'s post-match apply already handles) — not the full spec's twenty-odd types; sample file only; no book rewrite, same replay harness with a different input source. **Boundary:** parsing a sample file is not a production feed handler. No arbitration, no redundancy, no A/B feed comparison. Say that; it is true and it is fine.

**✅ Correctness gate (the workstream is not done until these pass):**

- **Fills bit-identical across rungs.** The same recorded stream through rung 0, rung 1, and the AF_XDP path must produce bit-identical fills and an identical event log. This is the Orderbook's existing replay claim extended to the transport — the transport is not allowed to change the book's behaviour, only its latency.
- **Gap and loss behaviour, as a test not a hope.** Drop packets deliberately (netem, or a feed that skips sequence numbers): the book must detect it, and the policy (halt vs. continue-stale) must be the documented one. A feed handler that silently accepts a gap is the classic production incident.
- **Backpressure:** FILL ring starvation and RX ring overflow are both real and both have defined behaviour. Assert it; do not discover it under load.
- **TSan clean** with the socket thread and the book thread running; **ASan/UBSan clean** per §14, with the fuzz corpus extended to malformed frames (truncated, oversized, wrong message type) — a network path is the one place untrusted bytes enter this portfolio, and the fuzzer should be pointed at the deserializer with real hostility.

**Benchmarks — the workstream's actual output.** Every row is a table in the Orderbook README, generated by the harness, raw `perf` output committed under `bench/results/`.

| Measurement | Why it is the one that matters |
| --- | --- |
| **Cycles in the kernel network stack, per message, rung 0 → 3** | `perf stat -e cycles` without `:u` — the mechanism number, isolating exactly what the rung removed |
| **Syscalls per message** (expect 1–2 → 0) | `strace -c` or `perf trace`; the bluntest and most convincing single line |
| **Wire-to-book latency: p50 / p99 / p99.9 / p99.99**, hardware-timestamped | the headline, and the reason the timestamp work comes first |
| **CPU cost per message at sustained rate** (10k/s and at the NIC's practical ceiling) | at low rates latency dominates; at high rates the CPU saving is the story. Measure both or the finding is half-told |
| **FILL-ring starvation and RX overflow counts under load** | proves the ring sizing is a decision, not a default |

**Boundary to state in the writeup:** rung 3 on a 1–10G link in a home lab is not a colocated exchange feed. The finding is the *shape* of the removal (what the kernel path cost, what removing it bought), with the machine annotated — the same provenance discipline as the cross-node NUMA number.

**Claim this workstream should earn:** *"Replaced a loopback feed with an AF_XDP user-space data path over a real NIC: cycles in the network stack per message [A] → [B] (perf-verified), syscalls per message [N] → 0, wire-to-book p99.9 [C]ns measured against NIC hardware timestamps; fills bit-identical to the socket path on the same recorded stream."* — with the rung stated, and the ITCH parse as its own bullet once it runs the same harness.

**Cut order (within this workstream):** (1) rung 3 (zero-copy) — needs a specific driver and a packet source; (2) rung 2 (AF_XDP copy mode) — falls back to rung 1's numbers without it; (3) **never cut:** wire timestamps (without them nothing here is claimable), the fills-bit-identical gate, and gap/loss testing. **The ITCH parse is not on this list** — it is rung-0 work that runs on the existing loopback feed, it is cheaper than any rung, and the resume already depends on it. Rungs 0–1 plus timestamps need no special hardware and no privileges beyond `perf`. If the whole workstream stalls, that much still ships.

**📖 Readings (all free, all just-in-time):**

| Resource | Covers |
| --- | --- |
| Kernel `Documentation/networking/af_xdp.rst` | authoritative ring semantics, UMEM, copy vs. zero-copy modes. Read *before* any blog |
| `xdp-project/xdp-tutorial` (the AF_XDP lessons) | the working skeleton; fastest path to a bound socket |
| `libbpf`/`libxdp` docs + `xsk_ring_*` accessors | the ring API and why the barriers are where they are |
| `man 7 packet`, `SO_BUSY_POLL` kernel docs | rung 1 — the cheapest rung needs the least reading |
| `ethtool -T`/`-i` output, `xdp-features` | capability discovery as a habit |
| NASDAQ **ITCH 5.0 spec** (+ MoldUDP64 section) and the public sample file | the real message mix |
| Top-Down ch. 3 | the transport framing you are bypassing; re-read §3.7 |

**🎯 Drills (this workstream ships → you answer these, cold):** Walk the kernel receive path and say what each stage costs — then which stages AF_XDP removes and which it does not. · What is the difference between generic/SKB mode, native mode, and zero-copy — and what does each cost you? · Why does the FILL ring exist at all? What happens when it runs dry, and how do you size it? · Why do the ring accessors use explicit barriers instead of `std::atomic`? What breaks if you use the C++ memory model here? · Where do your frames live, who allocates them, and what does that have to do with your order book's allocator? · How do you know a packet was lost, and what does your book do about it? · What is your wire-to-book number, and how was the wire timestamp obtained? · Why is `SO_BUSY_POLL` a different tradeoff from AF_XDP — what does each buy and what does each cost in CPU?

**War-story prompts (write these down as they happen):** **the ring that appeared to work and dropped frames intermittently** (usually a barrier, a ring index not being committed, or a descriptor returned to the wrong ring — and it will present as a book bug first), and **the number that got worse at rung 2 before it got better** (busy-polling a core against SMT siblings, or a NIC on the wrong NUMA node — Phase 2's topology lesson arriving as a production bug). Keep both. They are the interview answers this workstream exists to produce.

---

## 16. The final form (target document — nothing here is built)

Every other section says what to *build* and in what order. This one says what the project *claims* when finished, which artifact produces each claim, and what the claim costs to earn. It adds no commitments of its own: every row is derived from a phase section, §11, or §14, and cites it. If a row here is not in the plan, the row is wrong and the plan wins.

**The system, finished.** One C++17 repository, one CMake build, one CI, one benchmark directory. A loopback request stream enters through a sub-microsecond admission path — token-bucket rate limits, per-tenant concurrency caps, a restart-safe circuit breaker, a lock-free accounting ledger — crosses an SPSC dispatch queue into an iteration-level continuous-batching scheduler that admits on a projected KV budget, and is served by a from-scratch transformer decoder whose attention runs over a KV cache behind a swappable allocator interface. The decoder's achieved memory bandwidth is quoted against a ceiling measured by a GEMM ladder in the same repo, on the same machine, under the same harness. Every number in the README regenerates from `bench/`; every optimization claim has an ablation or a profiler counter under it; every correctness claim has a differential oracle or a replay hash behind it. The model is a deliberate placeholder — the work is the serving and performance layer, and the docs, the resume, and the interview script all say so in the same words.

**Claim set.** Tier: **C** = core (phases 0–6, steps A and G — the spine) · **P** = post-engine (stretch tiers) · **N** = never claimed (§16.3).

*Kernel and GPU*

| Claim | Producing artifact | Measurement | Tier | Source |
| --- | --- | --- | --- | --- |
| GEMM ladder: naive → coalesced → shared-memory tiled → register-blocked | `src/kernels/`, tables in `bench/` | Achieved FP32 FLOPs and achieved DRAM bandwidth against the machine's ceilings; occupancy sweep; the roofline figure; every rung's Nsight Compute output linked | C | §11 A3–A4 |
| Where the ladder stops scaling, and which ceiling stops it | Same | Named limiter per rung; a rung that buys nothing reported as a finding | C | §11 A4 |
| One quantized or fused kernel (INT8 dequant-GEMM **or** fused attention, not both) | A5 | Throughput gain and accuracy cost against the FP32 baseline; **percentage of cuBLAS** with the gap attributed to a named cause | C | §11 A5 |
| The same GEMM in Triton, diffed against hand-written CUDA and cuBLAS | A6 | Three-way table on one shape and one GPU: where the abstraction matches, where autotuning a search space beats a hand sweep, where explicit pipeline depth and shared-memory control win | C (cut early) | §11 A6 |
| GPU Monte Carlo with seeded replay bit-identical across the device boundary | Backtester | Same path hashes and same terminal equities on CPU and GPU | C | §11 A2 |

*Decoder*

| Claim | Producing artifact | Measurement | Tier | Source |
| --- | --- | --- | --- | --- |
| From-scratch GPT-2-class transformer in C++17: multi-head attention, KV cache behind an allocator interface, greedy decode | `src/decoder/` | Correctness gate first: output token-for-token identical to a reference running **the same weights and the same tokenizer** on a fixed prompt | C | §11 step G |
| Single-stream performance | Same | tok/s; TTFT at 512 / 1024 / 2048 context; achieved DRAM bandwidth during decode as a percentage of the ladder's measured ceiling; KV footprint against context length | C | §11 step G |
| Architecture breadth: a Llama-shaped config — RoPE, RMSNorm, SwiGLU, grouped-query attention — behind the *same* token-for-token gate | Decoder, extended | GQA's measured KV reduction by the query-to-KV-head ratio; correctness against a reference on a tiny checkpoint | P | §11 Tier A item 2 |

*Scheduler*

| Claim | Producing artifact | Measurement | Tier | Source |
| --- | --- | --- | --- | --- |
| Iteration-level continuous batching with KV-budget admission (worst-case projection, reject-at-capacity, no preemption at rung 1) | `src/scheduler/` | Throughput vs. per-token latency by batch size, presented **with** achieved decode bandwidth against the roofline — the curve flattens where bandwidth saturates | C | §10 rung 1 |
| Rung ladder: naive → continuous batching → chunked prefill → prefix caching | Same, plus the rung-0 reference scheduler | Rung deltas on one workload; a rung that buys nothing stated as such | C | §10 |
| Prefix caching: KV reuse across requests sharing a prefix | Scheduler rung 2b, over the decoder's KV allocator interface | Hit rate and TTFT delta against a shared-prefix workload with caching on and off; memory cost of retained blocks and the eviction policy that bounds it; token-identical output with caching on and off | C (cut early) | §10 rung 2b |
| Batching is a scheduling change, not a numerical one | Output-equality gate | Token-for-token identical per request across batch compositions **and** thread counts; fuzzed streams differential-tested against rung 0 with zero divergences | C | §10 |
| Scheduler overhead per admission decision | Same | Nanoseconds, quoted against a per-token cost measured in milliseconds | C | §10 |
| Two SLOs, two budgets | Phase-3 harness | TTFT and TPOT under concurrent load at stated offered rates, coordinated-omission-correct; an error budget per SLO and what burns it | C | §9 |
| Fairness policy with a stated starvation boundary | Scheduler | A named policy (WFQ / deficit round robin family), not "the caps handle it" | C | §10 rung 1 |
| Decode graph capture per batch-size bucket (stretch) | Scheduler rung 3 | Launch cost before/after from Nsight Systems, **with the bucketing padding cost as its own row**; unbuilt if the trace shows launch overhead is noise | C (cut early) | §10 rung 3 |

*Gateway, correctness, and measurement*

| Claim | Producing artifact | Measurement | Tier | Source |
| --- | --- | --- | --- | --- |
| Sub-microsecond O(1) admission control over cache-dense per-tenant state | `src/gateway/` | p99 decision latency, and the multiple over the deliberately naive reference — mutex, `std::unordered_map`, heap allocation per decision | C | §5, §8 |
| Layout and micro-arch ablation | Phases 2 and 4 | Cross-node atomics vs. local (2-socket bare metal); false-sharing elimination's p99 contribution; branch-miss reduction with the documented case where branchless *loses*; each factor's p99.9 contribution on the integrated pipeline | C | §6, §8 |
| Zero hot-path allocations, verified not asserted | Phase 4 | Interposition of `malloc`/`free`/`realloc` **and** `operator new`/`delete` in one `LD_PRELOAD` library, cross-checked once with a heap profiler | C | §8 |
| Correctness: fuzz, differential, replay | Phase 1 (+ 3) harness | Structured sequences via a custom mutator (not raw bytes) through both the reference and the real path; decisions must match exactly; replay bit-identical across runs and thread counts | C | §5 |
| Open-loop, coordinated-omission-correct latency | `bench/` load generator | Per-stage TTFT/TPOT under concurrent load; overload behaviour at a stated multiple of capacity, with the shedding policy and breaker threshold named | C | §7 |
| A system whose hot path cannot be scraped, still observable | Phase 1 seqlock + snapshot publisher | Measured uprobe perturbation on the path; counters published from a seqlock-protected snapshot; what was kept in eBPF because it cannot perturb | P | §14 |
| Ring all-reduce in C++ with an alpha–beta cost model and a measured crossover | Standalone, plus the decoder | Latency and bandwidth terms fitted against message size; the crossover per interconnect (1GbE, 2.5GbE, loopback, PCIe, NVLink); **NCCL run as the reference at the same sizes and ranks** and diffed against the hand-rolled fit | P | §11 Tier A item 1 |
| Tensor parallelism on the decoder across a rented node | Decoder, sharded | Scaling efficiency at 1, 2, and 4 ranks against the linear ideal per batch size, shortfall attributed to a named term; token-identical across rank counts. **At GPT-2 124M the honest result is that TP loses** — the artifact is the measured crossover | P | §11 Tier B item 6 |
| Quantization end to end | Decoder, INT8 path | Throughput gain against accuracy cost: perplexity delta plus token-agreement rate versus the FP32 gate | P | §11 Tier B item 4 |
| Speculative decoding | Decoder, draft + target | Speedup at batch 1 decaying toward 1× as batch grows — the technique and continuous batching pulling one knob in opposite directions, measured on one system | P | §11 Tier B item 5 |

### 16.3 What this project never claims

Each is a boundary with a prepared answer, not a gap to hide. §9 holds the scripts.

| Not claimed | The answer |
| --- | --- |
| Distributed training; NCCL as a *build* | Out of scope by design. NCCL appears only as the reference implementation the hand-rolled ring is validated against (§11). |
| Multi-GPU across the two home machines | Impossible, not deferred: ROCm and CUDA cannot join one collective, and consumer Ethernet costs roughly 4–6 ms of pure latency per token across 64 collectives (§17). |
| Kubernetes, cluster operations, Grafana/OpenTelemetry/operators | Tooling, not findings. Two hours of manifests earn no claim (§11). |
| Serving a real 7B model | Deliberate. Architecture breadth (RoPE/GQA) closes the vocabulary gap with engineering instead of a download (§11). |
| Kernel bypass **in this repo** | Structural: the ingress is a load generator, so bypassing the kernel removes cost from the wrong side of the measurement. The build lives in the Orderbook (§15.2). |
| A second backend (ONNX Runtime, TensorRT) | First item in the cut order (§2). |
| Production deployment, on-call, SLO ownership at scale | A different claim, made on the GoodSoft work-history block ([01-career.md](01-career.md)). |
| Frontier-scale models, MLOps at scale, RAG | Vocabulary only. |

**The model is a placeholder.** Every document says it in the same words: *my work is the serving and performance layer; the model is a deliberate boundary, not a gap.*

### 16.4 Why the resume projects thin, and why that is not fixable by editing prose

Three rules compress this file into one page: **the bracket rule** (an unfilled `[N]` ships as the shorter line, or the clause is deleted — every measurement in §16.2 is therefore a deletion until it exists); **resume is output, not input** (an unbracketed assertion needs code behind it); **the cut order** (under page pressure the ablation clause goes, then the kernels line, then the correctness line; the page keeps the decoder and the scheduler last, because they are the block's reason to exist).

So the page's strength is bounded by **what has shipped**, and the plan's own sequencing decides what that is first: the kernels bench and the decoder are finished artifacts before the gateway needs them, and the Stage-1 application gate is Phase 1 ([01-career.md](01-career.md)). Two consequences:

- **The Stage-1 page cannot contain the scheduler, the kernels ladder, or the distributed work** unless the sequence changes. It carries the admission core, the correctness machinery, and whatever else has landed.
- **Editing the block cannot substitute for shipping.** A stronger-sounding bullet with no measurement behind it is the exact failure the claim-versus-code audit exists to catch, and it fails on the first probing question — a worse outcome than a thin page.

**The honest ceiling.** Three things in §16.2 are genuinely rare and worth protecting through every cut: the correctness machinery (a differential oracle plus output-equality under batching; most portfolios have no correctness claim at all), the kernel ladder tied to the decoder's bandwidth number on one machine under one harness, and the scheduler's token-identical invariant. Model scale, distributed systems, and production operations are *not* closed by this project, and no amount of planning closes them.

### 16.5 Definition of done

The project is finished when every item below is true, each verified by running something rather than by inspection.

**Gates (never cut):**
- The decoder's output matches the reference token-for-token on a fixed prompt, with the reference named and the weights and tokenizer shared.
- The scheduler's outputs are token-identical across batch compositions and thread counts.
- Fuzzed request streams produce zero divergences against the naive reference implementations — the phase-1 admission oracle and the phase-6 rung-0 scheduler.
- Replay is bit-identical across runs and thread counts, with FP non-associativity analyzed rather than assumed, and the boundary of that guarantee written down.
- The zero-allocation claim survives `LD_PRELOAD` interposition of both the C and the C++ allocation paths.
- TSan is clean under concurrent arrivals and completions.

**Artifacts that must exist:**
- `bench/` regenerates every number in the README; no number is transcribed by hand.
- A roofline figure with the decoder's achieved bandwidth plotted against the measured ceiling.
- A batching table with rung deltas, and an ablation table with each factor's p99.9 contribution.
- A design doc carrying the two latency budgets, the overload policy, the admission policy, and the final-versus-reference delta.
- An overload run at a stated multiple of capacity, with the shedding policy and breaker threshold named.

**Explain-loop gates (shipping requirements, not suggestions):**
- Orca's iteration-level scheduling and vLLM's paging, cold and spoken, reconciled against your own scheduler's choices.
- Why decode is bandwidth-bound and prefill is compute-bound, answered from your own achieved-bandwidth figure.
- The KV-budget arithmetic on a whiteboard, on your own machine, at your own context length.
- The distributed boundary script — the alpha–beta terms, the two all-reduces per layer, the per-interconnect crossover.

**Cut, not failed.** Items the cut order removes — chunked prefill, graph capture, preemption rungs, the second backend, load-generator realism — are named as designed-but-unbuilt. Never claimed. A rung that bought nothing is a finding worth keeping and stating.

---

## 17. Decision record (why the project exists as it does)

**Status: adopted.** Supersedes the pre-pivot risk-engine framing and the "execution order amendment (2026-09-28)," which proposed running H→B→C→D→A→G ahead of the risk engine's Phase 2–4 grind. That amendment was never adopted and v4 retires it: with the pivot, the kernels bench (A) and the decoder (G) are built *inside* the engine repo, so there is no separate track to run ahead of. Its one surviving insight is kept: the kernels bench and the decoder ship as finished artifacts before the gateway needs them.

**The decision.** The third project's **domain** pivots from a pre-trade risk engine to a **low-latency inference serving gateway** in C++17. The **curriculum does not pivot** — Phases 0–5, the never-cut list, the honesty and bracket rules, the perf playbook, and the deepening queue survive. This is a re-skin of the skeleton, not a new plan.

| Consideration | Verdict |
| --- | --- |
| Quant-dev track already anchored by Orderbook + Backtester; a third finance project adds no new signal there | Pivot |
| AI-infra screeners parse "pre-trade risk / SEC 15c3-5" as fintech C++, regardless of systems quality | Pivot |
| The skeleton is domain-agnostic: ~80% of the spec maps component-for-component | Pivot is cheap |
| AI spine becomes *more* coherent: decoder = backend, batch sweep = core benchmark, continuous batching = centerpiece instead of know-only vocabulary | Pivot strengthens the spine |
| What is lost: the 15c3-5 compliance narrative and tight Orderbook coupling | Accepted; Orderbook replay survives as one workload flavor |
| What is lost: UDP gap/snapshot recovery is a smaller learning item than it protects | Accepted; recovery drops to stretch status |
| What is lost: the quant resume drops from three projects to two | Accepted and stated plainly: Orderbook + Backtester carry that page, and the gateway still reads as low-latency systems evidence on it |

**Framing rule:** the lead project must be domain-legible to both audiences. Low-latency systems discipline reads the same in any domain; financial regulation does not.

**Component mapping:**

| Risk engine component | Gateway equivalent |
| --- | --- |
| UDP multicast feed simulator | Open-loop load generator: recorded request streams replayed at configurable arrival rates |
| SPSC order-entry queue | SPSC request dispatch queues |
| Position/notional/price-band checks | Admission control: token-bucket rate limits, per-tenant concurrency caps — O(1), sub-microsecond, ablated vs. a mutex baseline |
| Kill switch (restart-safe) | Circuit breaker + load shedding (restart-safe) |
| Lock-free position ledger | Lock-free request/accounting ledger, restart-safe metrics |
| 5µs tick-to-trade budget | **Two** budgets, not one: TTFT and TPOT, each with a stated SLO under a stated offered rate. The admission decision is the knob between them — admitting a request grows the batch, raising TPOT for every resident request and setting TTFT for the newcomer |
| Fuzz / differential / TSan / replay gates | Identical — the correctness machinery is domain-blind |
| Phase 4 zero-alloc ablation | Identical, including `LD_PRELOAD` verification |

**What changed, phase by phase:**

| Phase | Change |
| --- | --- |
| 0 | CSAPP 3.2–3.7 attached to the SPSC ring (v4.4, widened to contiguous in v4.6) — program encodings and AT&T formatting, data formats, operand syntax, arithmetic, condition codes, loops, procedures/ABI: the plan's only disassembly-reading source, and what the ring's verify question needs. The SPSC capstone's framing note points at the dispatch queue it becomes |
| 1 | Risk checks → admission control; ledger becomes the accounting ledger; domain reading (15c3-5, RTS 6) → serving-side policy reading. Fuzz/TSan/differential/replay gate kept verbatim |
| 2 | CSAPP 3.9.3 added with the false-sharing work and 3.6.6 with the branchless work (v4.4); 6.6.1 added as the framing for a new stride axis on the memory-hierarchy sweep, and 5.12 to the optional tier with the ablation's IPC column (v4.5); Agner Fog widened from the branch-prediction chapter alone to include out-of-order execution and the scheduler, and COD 4.4 declared a drop-through to match 4.6 (v4.6); otherwise none — the experiments are on the admission path, which has the same shape |
| 3 | Multicast feed handler → open-loop load generator + TCP request stream. Sequencing keeps a staleness-detection story; **snapshot recovery drops to stretch**. Overload run gains the shedding policy. The wraparound half of that staleness story now cites RFC 1982 (v4.5) — the test it names had no source |
| 4 | CSAPP ch. 7 supersedes COD 2.12 for the interposition material, and CSAPP 9.9 is added as the prerequisite for the Orderbook pooled-intrusive rewrite (v4.4). Read contiguously 7.1–7.13 from v4.6 — the prerequisites 7.5 and 7.9 assume were previously on the skip list |
| 5 | README/design/proof pack reframed for serving; industry-context Q&A becomes serving context; boundary script updated |
| 6 | **New** — the centerpiece |

**What survives unchanged:** Phase 0 (OSTEP/CiA/Drepper/McKenney spine, harness, false-sharing toy, SPSC ring v1/v2, plus CSAPP ch. 3 in the ring's slot from v4.4); [perf.md](perf.md) (claim discipline transfers verbatim); §13 (cold recall, prediction ledger — *more* important now); operating rules 1–6; the never-cut list with the scheduler correctness gate added; the deepening queue (cache/NUMA work on Orderbook and Backtester is domain-independent); AI steps B, C, D; step A (content untouched, now built inside the engine repo — roofline and bandwidth reasoning become the decoder's and scheduler's vocabulary).

**Curriculum edit — CSAPP supplement (v4.4).** The open question was whether the quant-dev + AI-infra target favors Bryant over Patterson, i.e. whether CSAPP replaces COD on the reading list. **Verdict: supplement, not replacement.** The two overlap by roughly a third, and the divergence is not one of rank but of role: COD holds the architectural *model* (pipelining and hazards through out-of-order execution, ISA-level synchronization, the multicore-to-DSA arc, network topology), with Agner Fog, Drepper and PMPP supplying the x86-specific instance depth, while CSAPP holds the *mechanisms the builds touch directly*, which no other source in the plan covers (reading disassembly, linking and interpositioning, the allocator design space, byte order). **Neither is the plan's primary text.** That role belongs to Drepper, OSTEP, CiA, Agner Fog and PMPP, per §3's foundations table; COD and CSAPP are both second sources, sized differently — COD larger because its skippable chapters make it cheaper per unique page, not because it ranks higher. A ~125-page slice is added at existing phase slots with no new phase, drawn from ch. 2.1.3, 3.4–3.7, 3.9.3, 7.5–7.13, and 9.8.4–9.9. **Hour cost is approximately neutral**, because the slice is small and it is paid for twice: COD 2.12 demotes to a skim, and CSAPP's back half (ch. 5–6, 10–12) is explicitly out of scope as duplicating Agner Fog, Drepper, COD ch. 5, Beej's, Top-Down, CiA, and OSTEP. The labs are skipped on the plan's own rule — a reading that produces no claim is a boundary answer rather than signal — since cachelab and malloclab rehearse ground Phase 2's experiments and the Orderbook rewrite already cover and produce no artifact this portfolio can cite. Itemization and full skip-map in §12.

**Curriculum edit (v4.5).** A re-audit of the COD-vs-CSAPP question against the two targets, prompted by the observation that §12 called COD a "primary source" while §3's foundations table and Phase 0 item 34 both treat it as a second source. **The verdict does not change** — COD keeps the larger budget, CSAPP keeps its targeted slice — but two things follow. **First, the rationale is corrected.** COD is not the plan's primary text, and the v4.4 claim that it "holds the material the AI-infra half is built on" is not supported by the plan's own itemization, where the AI-infra sources are PMPP, the Williams roofline paper, the serving papers, and the builds themselves. The accurate division: COD holds the architectural *model* (pipelining and hazards through out-of-order execution, ISA-level synchronization, the multicore-to-DSA arc); CSAPP holds the *mechanisms the builds touch directly* (disassembly reading, linking and interpositioning, the allocator design space, byte order); the plan's primaries are Drepper, OSTEP, CiA, Agner Fog and PMPP. COD's larger second-source budget is page-economics — its unread ISA, arithmetic, logic-design and ECC chapters are cleanly skippable, which is what lets a structured read of ~370 pages carry the architecture ground. **Second, two readings are added:** **RFC 1982** for Phase 3's sequence-wraparound test, which was a named decision with no source anywhere in the plan, and **CSAPP 6.6.1** as the framing for a stride axis on Phase 2's memory-hierarchy sweep, previously footprint-only and therefore blind to the associativity conflicts it was supposed to characterize. Optional third tier, unscheduled: **CSAPP 5.12** read with the ablation's IPC column, and **12.7.1–12.7.2** read with 8.5 if the kill switch delivers by signal. Itemization in §12.

**Curriculum edit (v4.7) — quantization depth, and one audit correction.** Two changes, one strategic and one factual. **Strategic:** the target-role analysis ("people extracting the most out of models" — the llama.cpp/vLLM/GGUF community) showed the plan's quantization coverage was light relative to that community's centre of gravity. Tier B item 4's deliverable is upgraded from a single INT8 point to a **perplexity-versus-bits curve** (FP32 → FP16 → INT8 weight-only on the same small real checkpoint) — the exact artifact that community reports, at ~1–2 weeks with no new gate. GGUF/GPTQ/AWQ join Tier 3 as *format* vocabulary with a boundary answer (the plan builds weight-only INT8 at block level; it does not build a format or a quantizer tool). No new phase, no new hardware. **Factual:** the §15.1 audit text was stale in two places — the ITCH 5.0 parse over an mmap'ed NASDAQ sample file and the map → vector → slab ladder have since shipped and are measured in `Orderbook/docs/optimization_log.md` (38.15M msg/s, 26.2 ns slab-linear). Both passages are corrected in place; the resume claims they support are now code-true. The MoldUDP64 framing half (sequence-gap detection, resync state machine) remains unshipped and stays off the page until it lands.

**Curriculum edit (v4.6) — the continuity audit.** Motivated by a question the earlier passes never asked of the skip lists: whether a skip is safe because another source owns the ground, or is a hole in a chapter being read as a *skill*. A list selected on "produces a claim" is the wrong instrument for a chapter whose output is an ability, and three of the plan's skips failed that test. **CSAPP ch. 3** becomes a contiguous 3.2–3.7 read (~55 → ~85 pp): 3.2 and 3.3 are the encoding conventions 3.4 assumes, 3.5 is the arithmetic that real disassembly is made of, and 3.6.7 is the loop structure surrounding the conditional moves 3.6.6 teaches — the chapter's stated job is teaching you to read a disassembly, and a skill does not survive being read at 60%. **CSAPP ch. 7** becomes 7.1–7.13 contiguous (~30 → ~42 pp) and the v4.4 drop-through of 7.7–7.8 is retired: 7.3–7.4 are the ELF layout 7.5 indexes, 7.8 is a prerequisite for 7.9, and 7.7 is what makes PIC a mechanism rather than a flag. **CSAPP 2.1.2** joins 2.1.3 at §15.2 rung 0. On the COD side, **4.4 becomes a declared drop-through** (4.6 pipelines the datapath 4.4 builds), with 4.3 skimmed for the datapath elements and 4.1 for the chapter's frame; 4.2 and 4.5 remain safe skips. **Agner Fog's out-of-order execution and scheduler chapters** join the branch-prediction chapter — COD 4.11 supplies the model, and the branch chapter alone leaves a named penalty with no reorder buffer behind it. The same pass caught **three internal contradictions** (OSTEP's "do not schedule 26–30" against item 17's non-negotiable ch. 28; §11's "still skipped" list against items 27–28, which read CiA ch. 9–10; and the COD row skipping 6.13 while the matrix-multiply thread read it at step A4) and **one tool gap** — no gdb source anywhere in the plan — named rather than fixed. Net: CSAPP ~130 → ~175 pp, still ~16% of the book; no new hours, since every addition is a prerequisite the plan was already assuming. Itemization in §12.

**AI spine edits:** the order amendment is superseded; steps E and F are absorbed (E's *build* is cuttable — cut item 2 — and E's *learning* survives as a phase-6 writeup; F's sweep becomes a phase-6 benchmark); step G becomes the gateway's in-process backend behind a pluggable interface, with "no continuous batching" removed from G's cut list (batching lives in the gateway; the decoder stays single-stream-correct first, the gateway adds concurrency); continuous batching is **built**, in phase 6; new reads added (§12).

**Resume edits:** Appendix A (quant) deletes the risk-engine bullets — the pre-pivot spec is preserved in git history as a future cycle; the quant page = Orderbook + Backtester, plus the gateway only if framed as low-latency systems evidence. Appendix B (AI-infra) makes the inference engine the lead project block — gateway, scheduler, decoder, and kernels under one heading, all bullets bracketed until measured. Boundary script updated: "my work is the serving and performance layer; the model is a placeholder — a deliberate boundary, not a gap." Full text in [01-career.md](01-career.md).

**Risk register:**

| Risk | Mitigation |
| --- | --- |
| Scheduler is harder than the risk checks it replaces | Orca/vLLM as reference designs; rung-0 naive reference for differential testing; phase 6 gated on phase 1 being green; chunked prefill is rung 2, not rung 1 |
| No external feedback on new material | §10's explain-loop gate is a shipping gate, not a suggestion |
| Timeline slips while a hiring window opens | Kernels bench and decoder both ship before the gateway does — two finished artifacts stand alone. Never present planned work as built (bracket rule) |
| Scope creep back toward a product | The non-goals list in §2 is load-bearing: no OpenAI-compatible API, no multi-model serving, no HTTP/2 stack, no auth |

**Repo mechanics.** Nothing is built yet, so the pivot is cheap.

1. **Rename** `pre-trade-risk-engine` → `inference-engine`; GitHub keeps redirects. The rename is a separate step from the doc adoption — it changes the working directory and the remote.
2. **Layout** — one repo, one CMake build, one CI, because the decoder's bandwidth claim needs the kernels ladder's measured ceiling on the same machine and the same harness, and because one README can present the claims as one system:

```
inference-engine/
├── README.md            ← the proof pack: architecture diagram + claims table,
│                          every number linked to the bench that produced it
├── src/
│   ├── gateway/         ← admission control, SPSC dispatch, breaker, ledger
│   ├── scheduler/       ← continuous batching, KV-budget admission  (phase 6)
│   ├── decoder/         ← transformer decoder, KV cache behind an allocator interface
│   └── kernels/         ← GEMM ladder, roofline figures, the quantized kernel
├── bench/               ← all benchmark harnesses + measured tables
├── docs/                ← this plan (rename to design docs as they ship)
└── tests/               ← fuzz, differential, replay gates
```

3. Phase docs 11, 13, and 15 were edited per the pivot (now §5, §7, §9); phase 6 (§10) exists before phase 1 starts, so the scheduler's hooks (KV allocator interface, backend interface) are in the phase-1 design.
4. Commit history carries the arcs: kernels ladder rung-by-rung, then the decoder with its correctness gate, then gateway, then scheduler. Nothing lands before the thing it depends on.

**Kernel bypass, scoped.** The pivot parked kernel bypass as explain-only, and the non-goals list says the same. **That decision is scoped to this repo**, and the scope is explicit because two other documents were written on the assumption that the keyword is permanently unclaimable. **This repo** keeps a loopback request stream — the reasoning is structural, not budgetary: a load generator is a *client simulator*, so bypassing the kernel for it removes cost from the wrong side of the measurement. **The Orderbook** gains a real ingestion ladder (§15.2) — AF_XDP in rungs, NIC hardware timestamps, a real ITCH message mix. A market-data feed is the one place in this portfolio where the kernel path is the measured bottleneck, which is exactly the condition the original non-goal named as its own exception. **Resume consequence:** "kernel bypass" moves from the never-add list to a conditional keyword — defensible on the Orderbook block once a rung ships, permanently off the engine block. Also in this cycle: the sanitizer CI matrix (§14.1, repo-agnostic, covers all three repos), the eBPF observation-cost measurement (§14.2, engine-side, gated on the gateway existing), and one correction to the AI spine — step B's dataset layer is stored as memory-mapped Parquet rather than CSV (§11). The cut order does not change: these are anchor workstreams and rungs, and they pause before any engine item does.

**The distributed door, priced.** The "not distributed, deliberately" non-goal now names its own extension concretely: a hand-written ring all-reduce, its alpha–beta cost model, and the crossover measurement that decides when tensor parallelism pays — the cheapest distributed artifact available on hardware Jason actually owns, because the desk and the laptop on one LAN are enough to produce the failing half of the comparison. Tensor parallelism on the decoder is the same door's expensive half, gated on a rented multi-GPU node. Both live in §11's post-engine tier, both are gated on phase 6 shipping, and multi-node GPU across the two home machines is recorded as *impossible* rather than deferred (ROCm and CUDA cannot join one collective; consumer Ethernet costs ~4–6 ms of latency per token at 64 collectives).
