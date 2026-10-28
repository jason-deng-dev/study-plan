# Current Project Status
Jason Deng
778-847-3750  |  jasondeng.dev@gmail.com  |  linkedin.com/in/jason-deng-dev  | github.com/jason-deng-dev
Canadian citizen  |  Fluent in English dont and Mandarin
Education
B.Sc. (Honours), Economics, Minor in Computer Science & Mathematics                                   University of Toronto, June 2025

Work History 
GoodSoft Co., Ltd., Osaka, Japan   (https://github.com/jason-deng-dev/automation-ecosystem)
Software Engineer (June 2025 – May 2026)                                 Technical Consultant (June 2026 – Present, retained post-handoff)                                                                                                                     
Built multithreaded C++17 Monte Carlo engine for dynamic allocation weights; shipped to production via pybind11 and FastAPI.
Sole engineer on 6 automated services in production (AWS Lightsail, Docker, PostgreSQL, per-service DB isolation, GitHub Actions CI/CD): owned schema, deploys, and incidents; hardened the public APIs with Redis rate limiting and a proxy layer.
Built semaphore-bounded concurrent weekly ingestion of 2,000+ products, zero manual steps.
Projects
Low-Latency Order Matching Engine & Market Data Replay- C++23(https://github.com/jason-deng-dev/Orderbook)
Core: Price-time priority with self-trade prevention and partial-fill semantics. Compile-time search policies (linear, binary, branchless) with O(1) best-quote access and zero hot-path allocations.
Optimizations: Migrated price levels from std::map → contiguous reverse vector (+8.9%) → slab-allocated intrusive nodes (+42.4%), achieving 38.15M msg/s throughput and 26.2ns mean latency.
Ingestion: Engineered a zero-copy ITCH 5.0 feed handler via mmap (165M msg/s). Decoupled ingestion/matching via custom lock-free, cache-line-padded SPSC FIFO, achieving 61.7M msg/s.
Validation: Drove empirical, iterative performance tuning via CPU-pinned Google Benchmark and perf hardware counters.

Backtesting + Monte Carlo Engine  - C++17 (https://github.com/jason-deng-dev/Backtester)
Backtester: Factored strategy logic into runtime-composed Signal, Sizing, and Risk axes. Implemented zero-allocation CSV ingest (std::from_chars). FIFO lot accounting with per-position analytics (Sharpe, max drawdown, CAGR).
Monte Carlo: Validated edge via trade-level Monte Carlo (CIs on EV, max drawdown, terminal equity) and Markov regime-switching models for non-stationary market dynamics.

# To dos
Add to Backtesting + Monte Carlo Engine
- Parallelization: Parallelized Monte Carlo path generation via threading (achieving [10]x speedup on 9800x3d). Ported to CUDA (RTX 3060 Mobile), achieving a further [18]× speedup with strict bit-for-bit determinism under seeded execution.

Inference Engine — C++17/CUDA  (github.com/jason-deng-dev/inference-engine)
Kernels:  Optimized GEMM kernels (naive → coalesced → tiled → register-blocked), achieving [X]% of theoretical FP32 peak and [Y]% of DRAM bandwidth, validated via roofline analysis. Implemented INT8 quantization reaching [Z]% of cuBLAS performance.
Decoder: Built a from-scratch GPT-2-class transformer decoder (multi-head attention, paged KV cache, greedy decode); achieved bit-for-bit parity with reference implementations at [X] tok/s and [Y] ms Time-To-First-Token (TTFT).
Scheduler: Implemented iteration-level continuous batching with dynamic KV-budget admission, yielding [X]× throughput at [Y]× lower Time-Per-Output-Token (TPOT) compared to static batching baselines.
Gateway: Engineered a sub-microsecond request admission gateway using lock-free SPSC dispatch, achieving [X]× throughput over a mutex baseline with zero hot-path allocations (verified via LD_PRELOAD).
Distributed: Implemented distributed tensor parallelism via a custom C++ ring all-reduce, profiling latency/bandwidth tradeoffs across interconnects (1GbE vs. NVLink) and achieving [X]% of linear scaling at [N]-way TP, validated against NCCL.
Correctness: Guaranteed deterministic execution via [10M]+ fuzzed request sequences, achieving zero divergences against a naive reference and maintaining bit-identical replay across varying thread counts (TSan-clean).
