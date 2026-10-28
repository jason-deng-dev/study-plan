# AI Performance / Career Research Findings (2026-10-26)

Goal: can Jason's completed profile land AI-performance roles? Where else to aim for maximum
transferability? Sources: live JDs pulled directly + Sept 2026 "Voodoo Doll" AI-infra guide.

## Can the completed profile land AI perf roles?
Yes — top quartile on evidence, but with 3 specific gaps.

### What live JDs require
- **SambaNova — AI Performance Engineer (New Grad), San Jose** — $131k–$197k base. Required:
  B.S. CS/EE · strong Python **or C++** · PyTorch/TF/JAX · vLLM/TensorRT/SGLang · CUDA or Triton.
  Preferred: quantization, memory hierarchy/scheduling, internship, **OSS contributions in ML systems**.
- **Nuance Labs — Model Optimization & Inference (New Grad), Seattle, in-person, visa sponsorship**
  — $200k–$300k base + equity. "You don't need to have shipped at production scale — you need to
  learn fast and go deep." Keywords: KV caching, memory layout, attention kernels, batching,
  serving, vLLM/SGLang, CUDA/Triton. "Measure first, then optimize."
- Other new-grad openings: Cerebras (AI Inference Platform Eng, New Grad), Perplexity
  (AI Inference Engineer, entry), Tesla (Compiler Eng AI Inference, internship), plus a wave of
  campus inference roles in China (Baidu/Tencent/miHoYo — disaggregated serving, KV cache pooling).

### Box-by-box
| Box | You |
|---|---|
| C++ systems depth | ✅✅ strongest asset |
| Measure-first / optimize culture | ✅✅ bracket rule + perf playbook; rare in applicants |
| KV cache / batching / prefill-decode | ✅ Phase 6 maps 1:1 |
| Attention kernels from scratch | ✅ Phase G, bit-for-bit vs llama.cpp |
| CUDA / Triton | ⚠️ only after Phase A; 3060 weak hardware |
| PyTorch | ❌ real gap — decoder is from-scratch C++, no PyTorch line |
| vLLM/SGLang familiarity | ❌ gap — they ask about *their* ecosystem |
| Open-source contributions | ❌ none yet (preferred qual on SambaNova) |
| Degree | ⚠️ Econ + CS/Math minor = "related field"; defensible with work history |

### Cheapest gaps to close (order)
1. **PyTorch reference of decoder** — run GPT-2 in PyTorch, bit-for-bit compare C++ decoder.
   Days, not weeks. Turns gap into a resume line.
2. **One bounded vLLM change** — trace scheduler/KV path, add one metric/fix, PR it. Clears SambaNova's
   OSS pref-qual AND Nuance's "hobby level is fine."
3. **Phase A (GEMM ladder + 1 INT8 kernel)** — already in plan, non-negotiable.
4. **Nsight Systems/Compute** alongside perf counters — the profiling vocabulary in kernel JDs.

## Entry-level vs aggregate demand (2026-10-26 data)
Premise check: "AI demand is much higher than quant dev." **True in aggregate, false at entry level.**

### Aggregate AI demand (high and growing)
- AI/ML tech postings **+88% YoY** (Sept 2026 vs Sept 2025), ~5× the 17% growth for tech overall (Dice).
- Data-center listings **doubled in 2 years**; ~6 per 1,000 US postings (Indeed).
- AI-infra sector: **3,065 open roles** across 21 companies; OpenAI ~700-780 (HireIndex/Tinda).

### Entry-level AI (the opposite picture)
- Only **6.2%** of 8,967 classified AI/ML engineering roles are junior-open (Q2 2026); **11 senior+ per entry-level** (8bitconcepts).
- AI-native cos: **20.8 senior per junior**; junior+intern = **4.6%** of leveled postings (Landed, 2026).
- Oct 2026: only **2.1%** of 1,288 "AI Engineer" postings entry-level; **half of those stating a number ask 5+ yrs** (Apiva).
- US **entry-level postings down ~35% since Jan 2023**; SWE employment ages 22-25 **down ~20% since 2024** (Revelio/Stanford). Senior +29% since autumn, junior flat (SynapseScope).
→ The SambaNova/Nuance/Cerebras new-grad roles above are **exceptions** — startup waves, not a deep junior pipeline.

### Quant dev entry-level (deep and junior-hungry)
- **335-336 open Quant Developer roles** across 54 hedge funds/prop firms/market makers (Oct 2026), median disclosed comp **$200k** (kadoa).
- **926** quant dev postings US (LinkedIn); **1,463** live quant vacancies at 21 firms (KnowQuant, Sept 2026) — NY 391, London 262, Chicago 173.
- **C++ quant dev surging** — Citadel/Jane Street/Jump ramping; six-figure **entry-level** base (cppforquants, 2025).

### Conclusion
AI demand that's growing is **senior** demand. You'd compete for a handful of new-grad slots in AI-perf.
Quant dev = most meritocratic **and** highest-ROI entry: deep junior pipeline, six-figure entry pay, and a
documented path up into AI-perf (NVIDIA/G-Research) once senior enough to be in the pool those roles draw from.

### Risks
- New-grad AI-perf openings are few and mostly US-based; **quant-dev has far more junior volume** —
  dual-track hedge is correct (AI perf = ceiling, quant-dev = probability).
- Canadian openings are thin except Cohere (Toronto board was JS-rendered — unverified).
- Skill mix shifts fast (Sep 2026: domestic accelerators, RL infra, cache pooling hot). Durable
  systems skills survive; tool names don't.

## Other roles for maximum transferability
Transferable asset = **measured performance engineering on latency-critical systems** (profiling,
memory hierarchy, concurrency, determinism, GPU) — not "quant" or "HPC" specifically. Crossover is
two-way and documented: NVIDIA hires "AI and Quant DevTech Engineer"; G-Research (quant fund) hires
"NLP Performance Engineer" (LLM inference).

| Role | Feasible | → AI perf | Why | Targets |
|---|---|---|---|---|
| 1. Quant dev / trading systems | 9/10 | 8/10 | orderbook is entry ticket; prop funds now run LLM teams | prop shops, funds, market-data vendors |
| 2. HPC | 7/10 | 9/10 | CUDA/parallelism/profiling = daily job; purest AI pipeline | HPC teams, NVIDIA, research labs |
| 3. Database / storage engine | 8/10 | 7/10 | memory mgmt, concurrency → KV cache, continuous batching, tiered storage | ClickHouse, TiDB, Databricks, Redis |
| 4. Network/kernel (DPDK/AF_XDP/eBPF) | 6/10 | 8/10 niche | orderbook ingestion already is this; AI networking (NCCL/RDMA) most acute shortage | infra startups, cloud, NVIDIA |
| 5. Compilers | 4/10 | 9/10 | ML compilers hot but ~no junior openings — enter via HPC/quant | — |
| 6. Game engine / graphics | 7/10 | 6/10 | C++/SIMD/GPU overlap, weak narrative to AI | — |

### Strategy
1. Quant dev = primary entry — **most junior openings (335+ at 54 firms) + six-figure comp; AI exit
   documented.** Confirmed by 2026-10-26 data: AI demand is senior-biased (only 2-6% junior-open),
   quant entry-level is deep.
2. HPC = purest AI pipeline if wanting AI *now*.
3. Database engine = best hedge if quant/HPC miss.
4. Inference Engine repo = glue that makes any track read AI-adjacent.
5. One resume per track, same evidence, different framing: quant = msg/s + ns; HPC = CUDA + parallelism;
   DB = memory management + concurrency. Bracket-rule numbers serve all three.

### Avoid
Generic SWE and applied ML engineering — waste the C++/systems edge, transfer poorly to perf track.
