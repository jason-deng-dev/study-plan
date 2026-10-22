# Career — resumes, interview territory, algorithms, strategy

> **Canonical plan:** [00-master.md](00-master.md). Sizing, cut order, gates, and operating rules live there.
> **Rule 4 applies here:** the resume describes work that exists. Until every bracket is filled, Appendices A and B are *targets*, not documents.

## Contents

§1 house style and keyword discipline · §2 claim ↔ code audit · §3 Appendix A (quant resume) · §4 Appendix B (AI-infra resume) · §5 Appendix E (interview territory) · §6 Appendix C (algorithms) · §7 Appendix D (career strategy)

---

## 1. House style

Match the Orderbook/Backtester entries: verb-led, mechanism stated, measured number attached. One idea per bullet — the constraint that binds is total page lines, not bullets per project; correctness second (the differentiator — almost no candidate claims fuzzed invariants + deterministic replay); ablation numbers are probe-bait for "how did you measure that?" questions; "loopback" keeps the feed claim honest.

**The inference-engine bullets live in Appendix B below** — that is their single home, so the wording can't drift between a draft block and the shipped page. Appendix A re-spines the same facts for quant roles: Orderbook, Backtester, and the engine framed as low-latency systems evidence. The per-phase resume lines in the master spec are the *raw material* those bullets are cut from.

**Space-constrained fallback:** cut whole lines, never merge two claims into one bullet. Stacking ideas into a line is what makes a project block unreadable, and it saves less space than dropping a weak line does.

**Bracket rule:** any bullet with an unfilled placeholder ships as the fallback — an `[X]` on a resume is worse than a shorter line.

**Keyword discipline — what NOT to add:** industry terms you haven't built (FIX/OUCH parsing, DPDK, RDMA, FPGA) are probe-bait, not signal — every one converts a defensible bullet into a question you can't survive. They live in the design doc's industry-context Q&A and in interview answers attached to your measurements, never on the page.

**Conditional tier — keywords that graduate when the code lands, and only on the project that built them:** *kernel bypass* and *ITCH parsing* are no longer permanent exclusions; the Orderbook ingestion ladder builds both ([00-master.md](00-master.md) §15.2).

- **ITCH:** the feed bullet may say "ITCH-style" today, with "I modelled rather than parsed ITCH, here's the tradeoff" as the answer. Once the real file runs the same harness, it upgrades to "parsed NASDAQ ITCH 5.0 (MoldUDP64 framing, sequence-gap handling) and replayed a real message mix" — on the Orderbook block, with the boundary that it is a sample file, not a production feed handler.
- **Kernel bypass:** stays off the page until a rung ships, then appears **only on the Orderbook block**, in the form the rung supports ("AF_XDP user-space data path, copy mode" is a different and equally honest claim from "zero-copy"). It never appears on the engine block — that repo's ingress is loopback by design.
- **Still excluded regardless of anything:** DPDK and RDMA, unless built. Explaining them well is a design-doc answer, not a page claim.

---

## 2. Claim ↔ code audit

Run against the repos on this machine by grepping the code each claim names. **Shipped** = the code exists and the sentence is defensible as written. **Target** = does not ship until the work lands. **Unverifiable** = no code on this machine to check against.

### Backtester

| Claim | Status | Evidence |
| --- | --- | --- |
| Serial and parallel samplers agree bit-for-bit | **shipped** | `tests/montecarlo_test.cc:385`, `serialAndParallelAgree` |
| Index-partitioned sampling, per-run seeding | **shipped** | `src/montecarlo.cpp:148-155`, `:177` (`std::mt19937 gen(seed + i)`) |
| Trade-level Monte Carlo, Markov regime switching | **shipped** | `include/montecarlo.h:14-31` (`Regime`, `RegimeData`, transition matrix) |
| Sharpe, max drawdown | **shipped** | `src/analytics.cpp:48`, `:137` |
| CAGR | **shipped** | `src/analytics.cpp:335`, `include/analytics.h:134` |
| `lock-free SPSC queues` | **removed** | zero `spsc`/`lock-free`/`ring`/`queue` hits, and no `mutex`/`atomic` anywhere in `include/` or `src/`. Corrected to index partitioning |
| `null ensembles separating signal from luck` | **target** | zero `null`/`permutation` hits |
| `point-in-time datasets → PyTorch → ONNX → walk-forward` | **target** | zero `torch`/`onnx`/`point-in-time`/`walk-forward`/`leakage` hits |
| GPU Monte Carlo (CUDA, T4), CPU↔GPU bit-identical | **target — no fallback** | zero `cuda`/`hip`/`__global__`/`nvcc`/`rocm` hits across `include src tests CMakeLists.txt CMakePresets.json` |
| Arrow/Parquet point-in-time dataset layer | **target** | zero `arrow`/`parquet` hits; step B carries the format decision ([00-master.md](00-master.md) §11) |

### Orderbook

| Claim | Status | Evidence |
| --- | --- | --- |
| p50 add 142ns, cancel 125ns, 3-level sweep 357ns | **stale — re-measure** | Those numbers exist only in `docs/prototype/benchmark/benchmark_results.md` (`BM_BuyEmptyBook` et al.) — the **pre-rewrite prototype**, measured through the old `Orderbook`/`Trader` API, which is different code from today's book. There is no `docs/benchmark/` at the repo root; the README's results table is empty pending the harness; and `benchmarks/main.cpp` is fully commented out, so `orderbook_bench` builds an empty binary. Do not ship this row. The replacement comes from the `Orderbook_Vector` vs `Orderbook_Pool` replay comparison |
| Pooled intrusive FIFO per level, O(1) best-quote, zero hot-path allocations | **target** | levels are a searchable vector in `orderbook_vector.h`; orders are `std::list<Order>` nodes; zero `pmr`/`intrusive`/pool hits. Ladder dropped (would be a fourth container rewrite — cap at [00-master.md](00-master.md) §15.1) |
| Generational slot map for the order-ID index | **target** | `idMap` is `std::unordered_map<OrderId, std::list<Order>::iterator>` — `orderbook_vector.h:39` |
| PMR-pooled intrusive FIFO per level | **target** | zero `pmr`/`intrusive`/pool hits |
| OFI / microprice / queue-position feature layer | **target** | zero `microprice`/`imbalance`/`queuePosition` hits |
| AF_XDP kernel-bypass ingestion | **target** | zero `xdp`/`af_xdp` hits; spec and rung ladder in [00-master.md](00-master.md) §15.2 |
| Wire-to-wire, hardware-timestamped latency | **target** | no `SO_TIMESTAMPING`/PHC code; NIC capability unverified (`ethtool -T` is step 0) |
| Parsed NASDAQ ITCH 5.0 message mix | **target** | zero `itch`/`moldudp` hits; the feed is modelled, not parsed |
| Linear-vs-binary-vs-branchless search comparison | **implementation shipped, comparison still target** | `include/search_policies.h` ships all three; `orderbook_vector.h` is templated on the policy; `tests/search_policies_test.cc` passes (58 of the 123 in the suite). Only the head-to-head *comparison* is missing, and it is gated on the replay harness, not the implementation |

### Inference engine (v4 pivot)

| Claim | Status | Evidence |
| --- | --- | --- |
| All engine bullets — gateway, scheduler, decoder, kernels | **target** | the repo holds `TODO.md` and `docs/` only — there is no `src/`. The pre-trade risk-engine claims audited here previously are superseded wholesale; the pre-pivot spec is in git history before the v4 commit |

### GoodSoft work bullet 1

**Unverifiable from this machine.** No C++ Monte Carlo source exists under `/home/jason/repos`, and `github.com` is blocked from this session, so the pybind11/FastAPI engine cannot be checked. Two things to reconcile before it ships: the linked `automation-ecosystem` README documents **five** Docker services while the bullet claims **six** (the sixth is presumably this engine — make the README show it, or drop the count), and no C++ service appears in that README at all.

### Consequences

The Orderbook block currently ships one defensible bullet of four, and the GPU bullet ships none. Appendix A is a TARGET document and rule 1 covers `[N]` placeholders — but it does not cover an *unbracketed* assertion. `Cacheline-aligned, NUMA-pinned hot path` has no bracket, no number, and no code, and it reads as shipped. Extend rule 1: **an unbracketed assertion needs code behind it, not just a bracketed number.**

**The `Replay & Validation` bullet is the same failure, and worse, because it is specific.** Appendix A and B both ship *"Implemented ITCH 5.0 ingestion with sequence-gap detection and a Synced/Recovering resync state machine"* — with no hedge, no bracket, and no `itch`/`moldudp` in the repo. The conditional-tier note above says the bullet "may say ITCH-style today"; the bullet does not say that, so the deployed page is ahead of the code on the one claim a quant interviewer probes first. Two ways out, and only two: build it (§15.2 makes it rung-0 work on the existing loopback feed — ~2–4 days, no NIC, no root), or downgrade the wording to "ITCH-style" and keep the tradeoff answer. Leaving it as written is the option that fails in the room.

Separately, the C++ version claim has **inverted**: `Orderbook/CMakeLists.txt:4` now sets `CMAKE_CXX_STANDARD 23`, so the README's `C++23` is correct and the resume's `C++17` is the stale side. Fix the resume. The lesson matches the one this audit exists for: both claims were checked against a snapshot and neither was re-checked. **An audit that is not re-run is an assertion.**

The pipeline lead-in under **Projects** is itself one of these unbracketed assertions: it names four steps and all four are targets. It ships when the first link ships — step H, the Orderbook feature layer — not before.

---

## 3. Appendix A — Resume, quant variant (TARGET — brackets must be filled)

Jason Deng
778-847-3750 | jasondeng.dev@gmail.com | linkedin.com/in/jason-deng-dev | github.com/jason-deng-dev
Canadian citizen | Fluent in English and Mandarin

**Education**
B.Sc. (Honours), Economics, Minor in Computer Science & Mathematics — University of Toronto, June 2025

**Work History**
**GoodSoft Co., Ltd., Osaka, Japan** — Software Engineer (June 2025 – May 2026); Technical Consultant (June 2026 – Present, retained post-handoff)
(github.com/jason-deng-dev/automation-ecosystem)

- Built a multithreaded C++17 Monte Carlo engine (bootstrap resampling, EWMA decay, Sharpe-ratio optimization) deriving content-type allocation weights from 60k+ observed outcomes; shipped to production via pybind11/FastAPI.
- Sole engineer on 6 services in production (AWS Lightsail, Docker, PostgreSQL, per-service DB isolation, GitHub Actions CI/CD): owned schema, deploys, and incident response for a year; hardened the public APIs with Redis rate limiting and a proxy layer.
- Built semaphore-bounded concurrent ingestion of 2,000+ products and a weekly race calendar — DeepL-translated, zero manual steps.

**Projects**

*Three systems, one standard: a matching engine, a statistical simulation engine, and a low-latency serving engine — nanosecond-scale latency measured with controls, determinism verified bit-for-bit.*

**Order Book Matching Engine — C++23** (github.com/jason-deng-dev/Orderbook)

- **Low-Latency Core:** Price-time priority matching engine in C++23 with self-trade prevention and partial-fill semantics; reverse-ordered contiguous price levels, best price at the back so book updates never shift the container, with the search policy as a compile-time template parameter.
- **Replay & Validation:** Implemented ITCH 5.0 ingestion with sequence-gap detection and a Synced/Recovering resync state machine, and a post-match apply path (execute/cancel/replace against resting orders) decoupled from the order-entry matching path.
- **Data Structure Optimization:** Replaced the std::map prototype with a reverse-ordered level vector (**[X]×** throughput on the [N]M-message replay) and the order index with a generational slot map (cancel p50: **[A→B]**ns); fills bit-identical across rewrites. Pooled slab with intrusive FIFO links removes the per-order heap node.
- **Hot-Path Optimization:** Zero hot-path allocation via a pre-committed slab arena and a hand-rolled free-list, verified by interposing malloc/free and operator new/delete rather than by inspection; tail latency attributed with per-op LLC and dTLB miss counts. Benchmarked 3 search strategies on the same replay, located the depth **[D]** crossover where linear search wins on hardware prefetching, confirmed against `perf stat` branch-miss and IPC counters.
- **Correctness:** Replays one capture through every backend and asserts identical book-state hashes plus continuous invariants (uncrossed book, per-level volume conservation, order-index consistency), catching **[C]** divergences the unit suite missed.
- **ML Feature Integration:** Zero-alloc microstructure feature layer (order-flow imbalance, microprice, queue-position estimate) in the hot path at **[X]** msgs/sec, feeding the backtester's signal inputs. *— gated on step H; do not ship until it exists.*
- **Kernel-Bypass Ingestion:** Replaced the socket feed with an AF_XDP user-space data path at rung [R]: kernel-stack cycles per message [A]→[B] (perf-verified), syscalls per message [N]→0, wire-to-book p99.9 [C]ns against NIC hardware timestamps; fills bit-identical to the socket path on the same recorded stream. *— rungs 0–1 need no special hardware; rungs 2–3 pause first.*

**Backtesting + Monte Carlo Engine — C++17** (github.com/jason-deng-dev/Backtester)

- **Orthogonal Architecture:** Strategy logic factored into Signal, Sizing, and Risk axes composed at runtime; zero-allocation CSV ingest via `std::from_chars`; FIFO lot accounting with per-position analytics (Sharpe, max drawdown, CAGR).
- **Execution Realism:** Depth-aware fills that consume visible book depth with slippage as a first-class output, plus TWAP child-order slicing — the simulator prices its own execution instead of assuming a fill.
- **Streaming Ingest:** SPSC ring as bar producer with the strategy loop as consumer, replacing the blocking full-file load; the Monte Carlo sampler stays index-partitioned parallel-for rather than queued, because a queue buys nothing where every path costs the same and writes one slot.
- **Statistical Rigor & Regime Awareness:** Trade-level Monte Carlo with percentile distributions over EV, max drawdown, and terminal equity; two-state regime classification with a fitted transition matrix for non-stationary dynamics; null-ensemble baselines separating signal from luck.
- **Parallel & GPU Execution:** Index-partitioned writes with per-run seeding across **[C]** cores — serial and parallel samplers agree bit-for-bit under test, **[S]×** speedup; path generation ported to CUDA (cloud T4) at **[N]×** the 16-thread CPU path with seeded replay bit-identical across CPU and GPU.
- **ML Data Pipeline:** Leakage-free point-in-time datasets stored as memory-mapped Parquet, handed to PyTorch zero-copy through the Arrow C Data Interface → ONNX export → walk-forward evaluation with null-ensemble validation.

**Low-Latency Inference Engine — C++17** (repo) — *framed for this page as systems evidence; the serving-layer bullets live in Appendix B*

- Sub-microsecond O(1) admission control on the request path (token-bucket rate limits, per-tenant concurrency caps, circuit breaker, restart-safe kill switch): [A]× under a mutex + unordered_map baseline measured on the same harness.
- Cacheline-aligned, NUMA-pinned hot path: zero hot-path allocations under malloc interposition, cross-node atomics [D]× a local atomic.
- Ablation on that layout: false-sharing elimination cut p99 [B]×, branchless checks cut branch misses [C]%.
- Verification: [10M] fuzzed request sequences differential-tested against a reference implementation with zero divergences; TSan-clean lock-free SPSC ledger with decisions bit-identical across runs and thread counts.
- Driven end-to-end from an open-loop load generator over a loopback TCP request stream; per-stage latency measured coordinated-omission-correct.

**Editing rules:** cut adjectives, tool-drops, redundant tails — never numbers, never "bit-identical", never method names. Cut order under pressure: engine bullet 5 (load generator) → engine bullet 3 (ablation) → backtester parallel bullet. The Orderbook ingestion bullet is the one line that is *first* to cut on page pressure and *last* to cut on content value: it is the strongest single line on this page if a rung ships, and it is nothing if none does. One page, always.

**Bracket fallbacks (an unfilled `[X]` ships as the shorter line):** engine bullet 1 drops the `[A]× … same harness` parenthetical, keeping "sub-microsecond O(1) admission control". Engine bullet 2 drops the cross-node clause if `[D]` is unfilled, keeping the layout and interposition clauses. Engine bullet 3 drops the two ablation factors and the line goes with them. Engine bullet 4 drops the fuzz/differential clause if `[10M]` is unfilled, keeping the TSan/bit-identical clauses. Engine bullet 5 drops entirely until the load generator exists. Orderbook bullet 3 drops the `[125 → N]ns` parenthetical. Orderbook bullet 4 drops entirely if `[X]` is unfilled. Backtester bullet 3 (GPU) does not ship at all until step A exists in the repo: `grep -rniE "cuda|hip/|__global__|nvcc|rocm"` over `include src tests CMakeLists.txt` returns zero hits, so there is no fallback for that bullet, only deletion. Backtester bullet 4 drops entirely if `[S]×` and `[C]` are unfilled.

---

## 4. Appendix B — Resume, AI-infra variant (same facts, re-spined)

Jason Deng
778-847-3750 | jasondeng.dev@gmail.com | linkedin.com/in/jason-deng-dev | github.com/jason-deng-dev
Canadian citizen

C++ systems engineer — low-latency inference serving, GPU computing, and performance engineering.
Skills: C++17/20 · CUDA · Python/PyTorch · ONNX Runtime · Linux (perf, NUMA, GDB, TSan) · TCP/UDP · CMake

**Education** — identical to Appendix A.

**Work History**

**GoodSoft Co., Ltd., Osaka, Japan** — Software Engineer (June 2025 – May 2026); Technical Consultant (June 2026 – Present, retained post-handoff)
(github.com/jason-deng-dev/automation-ecosystem)

- Built a multithreaded C++ Monte Carlo engine (bootstrap resampling, EWMA, Sharpe optimization) deriving allocation weights from 60k+ observations; shipped to production via pybind11/FastAPI.
- Sole engineer on a 6-service production platform (AWS, Docker, PostgreSQL, CI/CD): automated content and catalog operations end-to-end — eliminated routine manual ops and gave non-technical operators self-serve control without SSH; hardened public APIs with Redis rate limiting and a proxy layer.
- Built and operated a Claude-API content pipeline in production — prompt strategy and content weights tuned from historical performance data across 4 content types.

**Projects** (reordered: the engine leads — it is the flagship)

**Low-Latency Inference Engine — C++17/CUDA** (github.com/jason-deng-dev/inference-engine)

- End-to-end low-latency serving stack built as one system: from-scratch transformer decoder, continuous-batching scheduler, admission-control gateway, and a measured CUDA kernel study — single CMake build, every claim regenerable from `bench/`.
- Decoder: from-scratch GPT-2-class transformer (multi-head attention, KV cache behind a swappable allocator, greedy decode); output verified token-for-token against llama.cpp on fixed prompts; [X] tok/s single-stream, [N] ms TTFT at 2k context; decode at [B]% of peak DRAM bandwidth, roofline-analyzed against the kernel ladder's measured ceiling.
- Scheduler: iteration-level continuous batching with KV-budget admission (reject-at-capacity); outputs token-identical to the single-stream reference across batch compositions and thread counts; [N]× throughput at [B]× TPOT, explained by achieved decode bandwidth; TTFT/TPOT under concurrent load, coordinated-omission-correct.
- Gateway: sub-microsecond O(1) admission control (token-bucket rate limits, per-tenant concurrency caps, circuit breaker, restart-safe kill switch) over SPSC dispatch; cacheline-aligned, NUMA-pinned, zero hot-path allocations (LD_PRELOAD-verified); [A]× over a mutex baseline; perf c2c-verified false-sharing elimination cut p99 [B]×.
- Kernels: GEMM ladder naive→coalesced→shared-memory-tiled→register-blocked at [A]% of peak FP32 FLOPs and [B]% of peak DRAM bandwidth; occupancy sweep; one INT8 kernel with measured throughput gain and accuracy cost.
- Correctness: [10M]+ fuzzed request sequences differential-tested against a naive reference scheduler with zero divergences; deterministic replay bit-identical across thread counts (FP non-associativity analyzed, not assumed); TSan-clean.

**Backtesting + Monte Carlo Engine — C++17** (github.com/jason-deng-dev/Backtester)

- **Orthogonal Architecture:** Signal / Sizing / Risk axes composed at runtime; zero-alloc CSV ingest via `std::from_chars`; FIFO lot accounting with per-position statistics (Sharpe, max drawdown, CAGR, annualized volatility).
- **Execution Realism:** Depth-aware fills consuming visible book depth, slippage as a first-class output, TWAP child-order slicing — execution is priced by the simulator rather than assumed.
- **Streaming Ingest:** SPSC ring as bar producer, strategy loop as consumer, replacing the blocking full-file load; the Monte Carlo sampler is index-partitioned parallel-for instead, since a queue has no job where per-path cost is uniform and each path writes exactly one slot.
- **Statistical Rigor & Regime Awareness:** Trade-level Monte Carlo — percentile distributions over EV, max drawdown and terminal equity; two-state regime classification with a fitted transition matrix (`CalmToCalm`/`CalmToVol`/`VolToCalm`/`VolToVol`); null-ensemble baselines separating signal from luck.
- **Parallel & GPU Execution:** Index-partitioned writes with per-run seeding across **[C]** cores — serial and parallel samplers agree bit-for-bit under test, **[S]×** speedup; CUDA path generation (cloud T4) at **[N]×** CPU, Nsight-profiled, seeded replay bit-identical across CPU/GPU.
- **ML Data Pipeline:** Leakage-free point-in-time datasets as memory-mapped Parquet → PyTorch zero-copy via the Arrow C Data Interface → ONNX export → walk-forward evaluation with null-ensemble validation.

**Order Book Matching Engine — C++23** (github.com/jason-deng-dev/Orderbook)

- **Low-Latency Core:** Price-time priority matching engine in C++23; reverse-ordered contiguous price levels, best price at the back, search policy as a compile-time template parameter; self-trade prevention and partial-fill semantics.
- **Replay & Validation:** ITCH 5.0 ingestion with sequence-gap detection and a Synced/Recovering resync state machine; post-match apply path (execute/cancel/replace against resting orders) decoupled from the order-entry matching path.
- **Data Structure Optimization:** Replaced std::map with a reverse-ordered level vector (**[X]×** throughput) and the order index with a generational slot map (cancel p50: **[A→B]**ns); fills bit-identical across rewrites; pooled slab with intrusive FIFO links removes the per-order heap node.
- **Hot-Path Optimization:** Zero hot-path allocation via a pre-committed slab arena and hand-rolled free-list, verified by interposing malloc/free and operator new/delete; tail attributed with per-op LLC and dTLB miss counts. Benchmarked linear vs binary vs branchless search, located the depth **[D]** crossover where linear wins on hardware prefetching, confirmed via `perf stat` branch-miss and IPC counters.
- **Correctness:** Replays one capture through every backend, asserting identical book-state hashes plus continuous invariants (uncrossed book, per-level volume conservation, order-index consistency) — caught **[C]** divergences the unit suite missed.
- **ML Feature Integration:** Zero-alloc microstructure feature layer (order-flow imbalance, microprice, queue position) at **[X]** msgs/sec, feeding the backtester's signal inputs. *— gated on step H.*
- **Kernel-Bypass Ingestion:** AF_XDP user-space ingestion at rung [R]: kernel-stack cycles per message [A]→[B], syscalls per message [N]→0, wire-to-book p99.9 [C]ns against NIC hardware timestamps; fills bit-identical to the socket path on the same recorded stream; real ITCH 5.0 message mix replayed through the same harness. *— rungs 0–1 need no special hardware.*

**Editing rules:** the skills strip is the FIRST cut under space pressure (bullets carry the keywords). Never submit as "ML engineer" — boundary discipline applies to applications too. Engine-block cut order: correctness line → kernels line → gateway ablation clause. The decoder and scheduler bullets are the block's reason to exist; they go last.

**Engine bracket fallbacks:** every engine bullet is a target until the code exists. An engine bullet with an unfilled bracket ships as its shorter form or not at all — and the block's lead-in line ships only when the first two subsystems (kernels and decoder) are measurable.

**Orderbook ingestion bullet:** ships at the rung that was actually measured, and **names the rung** — "AF_XDP" with no mode is the claim an interviewer dismantles in one question ("copy or zero-copy?"). If no rung ships, the existing loopback wording stays and the bullet does not exist. Same for the ITCH clause: it appears only once a real file has run the same harness.

**Post-engine tier — the claims this block gains if the stretch tier ships, in the order they become claimable.** Each is a *target* until its measurement exists, and each is capped at one line: **ring all-reduce + the TP crossover** ("implemented ring all-reduce in C++ and measured the interconnect crossover where tensor parallelism stops paying — [X] µs latency term, [Y] GB/s bandwidth term, 1GbE → NVLink"); **architecture breadth** ("GPT-2-class and Llama-class decoders — RoPE, RMSNorm, SwiGLU, grouped-query attention — both behind a token-for-token gate"); **quantization** ("INT8 decoder path at [X]% throughput for [Y] perplexity delta, token-agreement [Z]% against the FP32 gate"); **speculative decoding** ("[X]× at batch 1, decaying to [~1]× at batch [N] — the technique and continuous batching pull the same knob in opposite directions"); **TP on the decoder** ("[N]-way tensor parallelism on a rented node, token-identical across rank counts"). Nothing here displaces the engine block; on page pressure these are the first lines cut.

**Underused asset: the GoodSoft bullets are framed as automation work, and for AI-infra and platform roles they should be framed as distributed-systems operations work** — six services, the interfaces and failure modes between them, rate limiting and a proxy layer as defence in depth, and incident ownership as the retained consultant. It is the only entry on the page that proves you have operated something in production and been accountable for it when it broke, which is the thing most junior candidates cannot claim. It currently reads as a side note. One rewriting pass at application time: same facts, distributed-systems vocabulary, no invented specifics.

**Artifact placement.** The kernels bench and the decoder are subsystems of the **Inference Engine** project block, not separate blocks and not fold-into-Backtester clauses. The Backtester keeps its own block (GPU Monte Carlo + ML pipeline); the kernels ladder's numbers, the decoder's tok/s and TTFT, and the scheduler's batching tables are all engine bullets.

- **Skills strip:** "Nsight" and a kernel-optimization phrase may be added once A ships, and "KV cache / attention" once G ships. Not before. The strip stays the first cut under pressure.
- **Bracket discipline** applies to every engine bullet: the ladder's bandwidth and FLOPs percentages, the quantized kernel's accuracy delta, the decoder's tok/s and TTFT, the scheduler's throughput multiple, and every gateway ablation factor all ship as fallbacks until measured. A blocked module with no measured number is worse than omitting the bullet, because "roofline-analyzed" or "continuous batching" with nothing behind it is the easiest claim in this project to cross-examine.

---

## 5. Appendix E — Quant dev interview territory (the full map)

**When to open this:** interview season, and whenever a phase's 🎯 drill block references a territory. Each per-phase drill block drills the subset relevant to that phase; this is the master checklist.

Weighting: ~70% coding/systems, ~20% design + math, ~10% behavioral/domain. Gaps marked **[ADD]** are cheap post-ship/interview-season items.

### A. C++ language depth (~25–30%)

- **Types & objects:** trivial/POD/standard-layout, object layout & padding, alignment/`alignof`/`alignas`, sizeof quirks
- **Functions:** overload resolution, ADL, `noexcept`, `constexpr`/`consteval`, trailing return types
- **Classes:** special members (rule of 0/3/5), virtual dispatch (mechanism + cost), inheritance access, virtual inheritance, CRTP, friendship
- **Templates:** deduction, deduction guides, partial specialization/ordering, concepts & requires, fold expressions, variadics, `if constexpr`, instantiation/ODR, `extern template`, tag dispatch, template bloat
- **Value semantics:** rvalue/forwarding references & reference collapsing, `std::move` vs `std::forward`, when moves don't happen, guaranteed copy elision, copy-and-swap
- **Smart pointers & ownership:** `unique_ptr` (deleters, arrays), `shared_ptr` (control block, aliasing ctor, `make_shared` tradeoffs), `weak_ptr` (cycles, caches), `enable_shared_from_this`
- **Type erasure — the probe:** three mechanisms (virtuals, `std::function`, hand-rolled vtable) + performance comparison
- **UB minefield:** strict aliasing & common violations, signed overflow, OOB, use-after-free/double-free, uninitialized reads, **iterator invalidation per container**, dangling temporaries, evaluation order, **data races = UB**, static-init-order fiasco
- **Exceptions:** mechanism & when it costs, exception guarantees, `noexcept` as an optimization lever, `terminate` paths
- **Std internals:** vector growth/invalidation, SSO, deque chunking, `unordered_map` (buckets, load factor, iteration guarantees), `std::function` internals, `std::variant` + `visit`
- **Allocators:** allocator traits, `std::pmr` (monotonic buffer, pools), stateful allocators
- **C++20/23:** concepts, ranges, coroutines (what/why/when-not), modules, spaceship, `jthread`, latch/barrier/semaphore, atomic `wait`/`notify`, `stop_token`
- **Compile & link:** translation units, ODR, linkage, static vs. shared libs (PIC, PLT/GOT), symbol visibility, **interposition (`LD_PRELOAD` — Phase 4)**, ABI (mangling, layout, pImpl rationale)
- **Tooling:** ASan/TSan/UBSan/LSan (what each catches), gdb (breakpoints, watchpoints, core dumps, attach), `perf`, clang-tidy vocabulary

*Coverage: 4 C++ books + Phase 4. **[ADD]:** deliberate UB whiteboard reps; "explain vtable/`std::function`/SSO" drills.*

### B. Concurrency & memory model (~20%)

- Thread lifecycle, `jthread`, arg passing, `hardware_concurrency`, oversubscription
- Mutex taxonomy (timed/recursive/shared), guards, `std::lock`, `call_once`, upgrade locking
- **Deadlock:** four conditions, prevention (ordering, hierarchical mutex, try-lock), livelock/starvation
- **Condition variables:** predicate waits, spurious/lost wakeups, `notify_one` vs `notify_all`, bounded buffer, covering condition
- Futures/promises/`packaged_task`/`shared_future`, `async` launch policies, cross-thread exceptions
- **Atomics:** all specializations, `is_lock_free`, CAS loops, ABA, the **six orderings with x86 mappings**, happens-before / synchronizes-with / release sequences, fences, atomic wait/notify
- **Memory model:** data race = UB, SC-DRF, compiler vs. hardware reordering, store buffers, TSO vs. ARM weak model
- **Lock-free:** SPSC/MPMC rings (their artifact), reclamation (hazard pointers, epochs, RCU concept), seq_cst-prototyping discipline, busy-wait tradeoffs, backoff
- **Sync cost model:** uncontended/contended mutex (futex path), atomic RMW costs, cache-line ping-pong, convoying, priority inversion, spin-then-block, two-phase locks
- Thread pools & work stealing; why refuse `std::execution`

*Coverage: CiA full + OSTEP 28 + the ring. Strongest axis — resume-probed twice; expect the deepest drilling here.*

### C. OS / systems (~15%)

- Processes/threads, creation, states, context-switch mechanics & cost, user vs. kernel mode, **syscall path & cost**
- **Virtual memory:** VA→PA walk, multi-level page tables (x86-64), **TLB (reach, associativity, huge-page interplay)**, page faults (minor/major), `mmap` family (anon/file, `mprotect`, `madvise`, `mlock`), `brk`, CoW, swap policies, thrashing, overcommit, OOM killer, fragmentation, **NUMA + first-touch + interleave**
- Scheduling: MLFQ, **CFS (vruntime, min_granularity)**, EEVDF, real-time classes, affinity (`taskset`, `cpu_set`), `isolcpus`
- IPC & signals: pipes, unix sockets, shared memory, signal delivery/masking, async-signal-safety, eventfd/timerfd/signalfd
- **IO:** blocking/non-blocking, `select`/`poll`/`epoll` (LT/ET), reactor, `io_uring` (SQ/CQ), `SO_BUSY_POLL`, zero-copy (`sendfile`, `splice`)
- Allocation: malloc internals (bins/tcache), mmap threshold, fragmentation, pools/arenas/slabs, jemalloc/tcmalloc
- Observability: `strace`/`ltrace`, `/proc` (maps, smaps, status), `perf stat/record/top`, `vmstat`, `numastat`
- Kernel-bypass: **AF_XDP is built** on the Orderbook ([00-master.md](00-master.md) §15.2) — UMEM, the four rings, copy vs. zero-copy modes, generic vs. native XDP, `SO_BUSY_POLL`, hardware timestamps. DPDK and ef_vi stay know-why-not-how

*Coverage: OSTEP spine. Second-strongest axis.*

### D. Architecture / micro-architecture (~10%)

- Pipeline: stages, hazards, forwarding, **branch prediction & misprediction cost**, speculation, OoO (ROB, reservation stations), SMT (shared resources)
- Memory hierarchy latency table; cache organization (sets/ways); **MESI/MOESI** & false-sharing mechanism; hardware memory ordering
- ISA-level atomics (`lock xadd`, `cmpxchg`, LL/SC), fences (`mfence`/`lfence`/`sfence`)
- SIMD: SSE/AVX2/AVX-512, intrinsics, auto-vectorization, gather/scatter, alignment, when SIMD loses
- Prefetching (hardware stride/pattern; software `prefetch`; when harmful)
- **NUMA quantified:** local vs. remote atomics (~5×), the cloud measurement
- Perf analysis: counters (IPC, branch-misses, cache-misses, dTLB-misses), top-down vocabulary, `rdtsc` vs `clock_gettime`
- FP: IEEE-754, rounding modes, denormal cliffs, FMA contraction, fast-math dangers, cross-vendor determinism (a **step A** claim, and only the CPU↔GPU half is yours to own; HIP/ROCm is ruled out, so do not carry a cross-*vendor* claim you are not building)

*Coverage: Drepper + COD targeted + the CSAPP slice (ch. 3 asm/ABI, ch. 7 linking + interpositioning, 9.9 allocators) + Phase 2.*

### E. Networking (~10%)

- Stack vocabulary: L2/L3/L4 roles (frames/MAC, IP/routing, TCP/UDP)
- **TCP:** segments, handshake/teardown states, seq/ack, retransmission (RTT/RTO), flow control (rwnd), congestion control (slow start, AIMD, CUBIC/BBR), **head-of-line blocking**, Nagle + delayed-ACK interaction, keepalive, SACK/timestamps
- **UDP:** header, checksum weakness, MTU/fragmentation (why avoid); multicast (class D, IGMP, joins) — the engine does not build this, but the **Orderbook does**: a loopback multicast feed carried to a real NIC in measured rungs, ending in an AF_XDP user-space path
- **The transport split, both directions:** TCP for the engine's request/response path — flow control as backpressure, head-of-line blocking as a design constraint to reason about — and UDP market data / TCP order entry as quant vocabulary you can defend without having built it. Expect the question phrased either way
- API depth: non-blocking + `EAGAIN`, partial send/recv, `MSG_*` flags, latency-relevant `SO_*` (RCVBUF/SNDBUF/REUSEADDR/REUSEPORT/BUSY_POLL/TCP_NODELAY), `shutdown` vs `close`, `SO_ERROR`
- Kernel path: interrupt → NAPI → softirq → socket buffer → wakeup; **what AF_XDP removes and what it does not** — measured, per-stage, on the Orderbook; `io_uring` for networking
- Protocols: binary framing (length-prefix vs delimiters) as built; **ITCH 5.0 + MoldUDP64 is a build claim** once a real sample file runs the Orderbook's harness — sequence numbers, heartbeats, and gap fill/resend are the same vocabulary FIX uses, so one implementation covers both answers. FIX/OUCH/PITCH/SBE stay *read-only vocabulary*
- Measurement: RTT anatomy, bandwidth-delay product, bufferbloat, **propagation ≈ 5µs/km**, serialization delay

*Coverage: Top-Down structured + Phase 3 + Beej's.*

### F. Algorithms / LeetCode (~10–15%, screen gate)

- Full LC taxonomy: arrays/two-pointer/sliding window; hashing; lists; **monotonic stack**; trees/tries; heaps; intervals; greedy; binary search (incl. on-answer); graphs (BFS/DFS/topo/Dijkstra/union-find); DP (linear/2D/knapsack/interval); backtracking; bit manipulation; math (gcd, binary exponentiation, matrices)
- Bar: medium fluent, hard reachable, **contest rating ~1900**; bug-free-first-submit discipline

*Coverage: NeetCode 150 + weekly contests (ongoing). See §6.*

### G. System design — low-level flavored (~5–10%)

- Repertoire (the serving items are the built ones): inference gateway, request scheduler with admission, load shedder, rate limiter (token bucket vs. sliding window), sequencer/logger (seq writes, batching, MPSC), thread pool, memory pool/arena, LRU cache, heartbeat/failover, config distribution; *quant vocabulary:* market data handler, order gateway/OMS, matching engine, risk gateway, pub/sub, time sync (PTP)
- Scoring axes: correctness under concurrency, latency-budget breakdown, failure handling, observability, capacity math
- Generic SD (AI-infra roles): Xu Vol. 1 flashcard territory

*Coverage: the projects ARE the designs; **[ADD]** whiteboard reps of the repertoire list.*

**[ADD] is a scheduled block, not an annotation.** Every [ADD] marker in this appendix is work with no slot on the calendar, which is the same as work that will not happen. Give them one: whiteboard reps of the repertoire list, the two section-A [ADD] items, and the UB reps — run as a fixed weekly slot from the moment Stage-1 applications start. Design-round reps also need a partner or a mock service; the explain-loop's recorded walkthroughs cover the solo half, and nothing in the plan currently covers the live half.

### H. Math / probability (~5–10% QD)

- EV, conditional probability, Bayes, combinatorics, expected-value games; distributions, CLT, CIs (owned via Monte Carlo); Fermi estimation; fast mental math; Markov chains (their domain); basic linear algebra
- *(QT shots only: full puzzle gauntlet)*

### I. Behavioral (~5–10% — the "pod trust" filter)

- Six story bank (conflict, failure, ambiguity, deadline, deep-debug, disagreement) from real projects; why-trading/firm narrative coherence; ownership signals (sole engineer, retained consultant)
- **The career-change narrative is its own story, and the most likely one to be probed.** Economics honours, a Japan-based automation company, content-pipeline work — the bridge to trading has to be a story you can tell in sixty seconds without hedging. The version that works is the one the other artifacts already support: you built the tooling because you wanted an answer to a trading question, the Monte Carlo engine shipped for a real business decision, and you kept descending the stack until the interesting problems were systems problems. Rehearse it out loud like any other drill. The failure mode is discovering it for the first time in the room.

### J. Market microstructure domain (~5%)

- Order types (limit/market/stop/IOC/FOK/peg), price-time priority, book mechanics, spread, adverse selection basics; latency economics (tick-to-trade, jitter, colocation); exchange anatomy (gateways, feeds, matching); light regulatory vocabulary (Reg NMS, market-maker obligations)
- **Serving control set (replaces the pre-trade control set, which is now vocabulary-only):** token-bucket vs. leaky-bucket vs. sliding-window vs. GCRA rate limiting; per-tenant concurrency caps vs. rate caps (what each bounds); reject-at-capacity vs. queue-and-delay; circuit-breaker states and thresholds; load shedding; retry storms and why jittered backoff exists; fail-safe vs. fail-open. Answered from your own admission path and the Phase 3 overload run. The pre-trade set (price bands, self-trade prevention, SEC 15c3-5 / RTS 6) stays available as a boundary answer attached to Orderbook, not as a build claim.
- The industry-context appendix covers the vocabulary layer

### K. Resume cross-examination (runs under everything)

Every number: provenance, why-not-X alternatives, what-broke stories. Served by README ↔ perf output linkage, the ablation table, and the design doc.

---

## 6. Appendix C — Algorithms track (parallel to the build; never competitive with it)

**Diagnosis (first contest: Q1+Q2 after NC150):** the gap is unlabeled retrieval, not volume. NC150 teaches patterns WITH labels; contests test deconstruction of unseen problems.

- **Daily (30–45 min):** 1–2 NEW untagged problems (shuffled pool). Generate candidate approaches out loud before coding. Prioritize mediums + the medium-hard boundary.
- **Weekly:** every contest (virtual if timing is bad) + same-day post-mortem: re-solve Q3(+Q4) WITHOUT editorial → read editorial → Anki the pattern as a decision rule ("constraint X → think Y").
- **Spaced re-solves:** 1-3-7 spacing on FAILED contest problems, not everything solved.
- **Metric:** contest rating, not problems done. Target ~1900 (≈ consistent 3-solves). Trajectory: 2–4 months of weekly contests.
- **Section audit rule:** does this teach a pattern I'll recognize in an unseen problem? (Example audit — Math & Geometry: do Pow(x,n), Spiral Matrix, Rotate Image, Happy Number, Set Matrix Zeroes; sprint the trivial five; skip GCD-of-Strings, Multiply Strings, Detect Squares.)
- **Budget:** ≤45–60 min/day + contest weekend. The day this eats spec hours, the allocation has gone wrong.

---

## 7. Appendix D — Career strategy (re-read quarterly)

**Role gates:**

- **Quant dev (PRIMARY):** interview = C++/systems depth + LeetCode + projects. This spec targets it directly.
- **QT (cheap parallel shots):** pedigree-blind at Jane-Style firms; gate = probability/mental-math/estimation gauntlet (trainable, not resume-driven). Systems knowledge ≈ zero advantage in the room; take shots when invited.
- **QR (2–3 year path):** tier-1 entry near-zero without research evidence; path = internal mobility from a dev seat or funded master's later.
- **AI infra (co-primary in v4):** the inference engine is now the flagship, so this track no longer waits on an extension — mid-tier (inference startups, accelerator companies, quant AI platform teams) is realistic at junior level once the engine's kernels and decoder are measurable, with the scheduler strengthening the application. Frontier-lab infra = later-via-mobility only.
- **Both tracks ride the same build:** the engine is systems evidence for quant rounds and serving evidence for AI-infra rounds; the two resume variants re-spine the same facts. Neither track is waiting on the other.

**The moat:** execution moat, not knowledge moat — the compound (production C++ + measured projects + CUDA artifact + correctness story) is rare because most people attrit at every step. It decays if stalled and converts to a career moat only after the first job. Defense = velocity. AI-resistance: the durable walls are physical grounding, verification judgment, and accountability — knowledge recall is a snapshot moat with an expiration date you don't control. Practice wielding AI inside builds with the measurement discipline applied to its output.

**Contacts / funnel (referral-driven market):** Duncan Bees — Openchip (Barcelona, RISC-V accelerators): informational conversation NOW (team structure, junior screening, remote/relocation). Same profile fits Tenstorrent (Toronto HQ) and North American RISC-V/analogs. Keep contacts warm early.

**Documents:** two one-page resume variants (Appendix A/B); never hybrid. Resume = output of shipped work.

**Sequence (there is no deadline — but there is a hiring calendar, and the plan should not pretend otherwise):**

**Stage 1 — apply at the Phase 1 gate, not at the full proof pack.** The plan as written starts applications only after the complete core proof pack exists, which by the sizing table is 9–12 months out (Phase 0's reading alone is roughly 2,200–2,400 dense pages at the stated 100–140 pages/week, before any build). That is too long to go without interview reps, and interview loops are the instrument that tells you which gaps are real. The Stage-1 proof pack is: the Orderbook and Backtester as shipped (with the credibility patches in the deepening queue landed), plus the admission core (Phase 1), the decoder (G) if it has landed, the correctness gate, and the ablation from Phases 1–2. That is a defensible portfolio. Write the Stage-1 README and design doc at that point, start applying to both tracks, and let the loop tell you what to fix. The gateway and scheduler then ship *while* you are interviewing, as upgrades, rather than as prerequisites.

**Stage 2 — full proof pack, remaining AI modules, AI variant live.** Unchanged from the original sequence, except that it now runs concurrently with an active job search rather than gating it.

**Review trigger (so a slipping plan has a defined failure state):** if the Phase 0–1 reading is not complete within about six months of start, stop treating the curriculum as a prerequisite and switch to applying with the current portfolio plus whatever has shipped, continuing the spine part-time. A plan with no trigger can silently consume a year.

Algorithms parallel throughout. If the core slips, the extension spine pauses; applications do not wait for every AI module.
