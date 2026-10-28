if AI has to write suggestions store in ai suggestions

# Plan repo cheat-sheet (don't re-scan 00-master.md)

## Canonical files
- `docs/00-master.md` — THE spec (Inference Engine v4). Arbitrates all disagreements.
- `docs/resume.md` — resume status.
- `docs/perf.md` — perf stat playbook (own numbering §1–§7).
- `history.md` — what's actually been done, by date.

## The project: Inference Engine (C++17)
Low-latency inference serving gateway, one repo / one build / one CI:
`gateway/` (admission control) · `scheduler/` (continuous batching) · `decoder/` (from-scratch GPT-2) · `kernels/` (GEMM ladder). Pivoted v4 from a pre-trade risk engine (component mapping in §17). Model is a deliberate placeholder — work is the serving/perf layer.

## Sequence
Phase 0 → A (CUDA kernels) → G (decoder) → Phases 1–5 (gateway) → Phase 6 (scheduler).
- P0: reading (OSTEP/CiA/Drepper/McKenney/CSAPP 3.2–3.7) + 3 builds: measurement harness, false-sharing toy, SPSC ring v1(seq_cst)→v2(relaxed).
- P1: admission core (token bucket, caps, breaker, lock-free ledger) + fuzz/TSan/differential/replay gate.
- P2: micro-arch (false sharing, SMT/NUMA, huge pages, branchless, mem-hierarchy sweep).
- P3: open-loop loadgen + TCP path + coordinated-omission-correct TTFT/TPOT + overload run.
- P4: zero-alloc proof (LD_PRELOAD interpose) + ablation table.
- P5: README + design doc + proof pack.
- P6: continuous batching + KV-budget admission + chunked prefill + prefix caching (centerpiece).
- A: GPU Monte Carlo + GEMM ladder + roofline + 1 INT8/fused kernel. G: GPT-2 decoder, token-for-token vs llama.cpp.

## Two quant anchors (pause whenever engine needs the hours)
Orderbook + Backtester: AF_XDP ingestion ladder, microstructure features (step H), leakage-free datasets (B), training literacy (C), eval rigor (D). Step B → Parquet/Arrow not CSV.

## Python bridge (P) — all three repos
pybind11 wrapper of each C++ core + pytest under `py/tests/` + pandas/NumPy reference for H's features + Jupyter in `notebooks/`. Two halves: anchor (Orderbook/Backtester, gates on replay) and engine (after G + Phase 4). ~1 wk.

## Operating rules (from the spec)
1. Bracket rule: no `[N]` claim ships without a measured number; unfillable → delete clause.
2. Honesty rule: mechanism + measurement + boundary for every claim.
3. Weekly review + cold recall; track hours/category, not weeks. Shipped > ambitious.
4. Resume is output, not input.
5. HW: 9800X3D desktop (1 NUMA), RTX 3060 laptop, cloud for CUDA/NUMA days.

## Already done (per history.md)
Full-stack (JS/React/Node/DB, 46d) · LearnCPP c1.5–c25 · Tour of C++ targeted chs · Effective C++ · Effective Modern C++ · CMake · OSTEP thru 34, CiA thru 5, McKenney, CSAPP 3.2–3.5. → Phase 0: reading spine mostly done; harness + SPSC ring (incl. false-sharing delta) built — ring lives in Orderbook (`docs/spsc.md`, `docs/notes/spsc_fifo_notes.md`). Remaining: CSAPP 3.6–3.7 + CiA/OSTEP tail.
