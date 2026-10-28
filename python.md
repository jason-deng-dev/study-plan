# Python for a C++ Quant/AI-Performance Profile

## The thesis

Python is a **screening filter, not a differentiator** for quant dev / AI
performance roles. Listings at the target firms (HRT, Optiver, Jane Street,
Two Sigma, AI-infra teams) require Python for data handling; without it the
C++ work never gets read. The moat stays in C++/CUDA/systems depth — Python
makes that depth legible to recruiters.

The **combination** is what's rare: most C++ devs can't touch the data stack,
most Python quant devs can't write a lock-free engine. The goal is to own the
C++ ↔ data boundary, not to become a Python developer.

## What has signal value (the core)

1. **NumPy + pandas** — the only data skills truly needed. Ingesting feeds
   (CSV/Parquet), basic analytics, vectorized feature construction.
2. **pybind11** — the bridge. Expose the C++ engines as importable Python
   packages; this is already in use (GoodSoft Monte Carlo engine) and should
   become a portfolio centerpiece pattern.
3. **PyTorch at the export boundary only** — train/export to ONNX, load in
   the C++ inference engine. Directly strengthens the inference-engine
   project.
4. **pytest + Jupyter** — for the validation stories already in the profile
   (roofline analysis, bit-for-bit parity, benchmarking).

## What to skip (low ROI for this target)

- Deep Python ecosystem: zipline/backtrader frameworks, ray.tune, boto3
  orchestration, Django/web, heavy scripting. That's a Python-developer
  curriculum, not a quant/AI-perf one.
- Competing in "data scientist" / "ML engineer" roles — those are the
  commoditized pool; entering there is competing at a disadvantage. Python
  doesn't open new role *types*, it removes a checkbox on the same roles.

## Integration points with existing projects

- **Orderbook** — replay/analysis tooling in Python: feed the ITCH 5.0
  corpus, plot latency histograms, drive benchmark comparisons from Jupyter.
- **Backtester** — pandas for data prep and result analysis; pybind11
  wrapper so strategies can be prototyped in Python against the C++ core.
- **Inference engine** — PyTorch → ONNX export path into the C++ engine;
  pytest for the fuzzing/correctness harness; Jupyter for roofline and
  scaling plots.

## Priority note

The inference-engine project currently carries `[X]%`, `[Y]`, `[Z]`
placeholders. Filling those with measured numbers is a higher-ROI move than
any new language — do it in parallel with the Python learning track.

## Market findings (entry/mid listings, fetched 2026-07)

### Quant dev (HFT): HRT, Optiver, Tower, Point72, Axq

- HRT posts *"Software Engineer (C++ or Python)"* — Python is an accepted
  alternative; C++ is the house language for low-latency work.
- Optiver Grad Quant Dev: strong C++, data structures/algorithms, ML used in
  live HFT strategies, trading infrastructure.
- Tower: "low latency, high throughput production trading and research
  systems," high-performance computing, market data access, risk tools.
- Common thread: C++/systems primary; Python at the data/research interface;
  ML knowledge increasingly expected even on dev roles.

### AI performance/infra: Baseten, DeepInfra, NVIDIA

- Baseten GPU Kernel Engineer (1–5 yrs): CUDA, C++, GPU memory hierarchy,
  tensor cores, quantization (FP8/FP4), GEMM tuning, Nsight profiling,
  Cutlass/Triton, Flash Attention. Near 1:1 map of the inference-engine
  project — exactly where the `[X]%` placeholders hurt.
- DeepInfra Early Career (0–2 yrs): verbatim requirement — "Proficiency in
  Python, including experience with AI/ML libraries and frameworks (e.g.,
  NumPy, pandas, SciPy, TensorFlow, PyTorch)"; plus C++, CUDA, NCCL,
  distributed inference.
- NVIDIA DGX new grad: distributed GPU workloads, cluster bring-up,
  benchmarking, root-cause analysis.

### Implications

1. The narrow Python scope (NumPy/pandas/PyTorch at the boundary) is exactly
   what target listings name — not general Python.
2. Two role families with different weightings: quant-dev = C++-first, Python
   at the data edge; AI-infra = Python+C+++CUDA roughly equal.
3. Highest-ROI gap: measured kernel numbers (Baseten's listing reads like the
   project's missing `[X]` values).
4. New signals to fold in: ML literacy on quant roles (Optiver),
   NCCL/distributed (DeepInfra/NVIDIA), profiling tools by name (Nsight,
   perf) — the orderbook story has perf; the inference story needs Nsight.

### Sources

- HRT Software Engineer (C++ or Python) 2026 Grads — builtin.com/job/software-engineer-c-or-python-2026-grads/6833743
- HRT Algorithm Developer 2027 Grads — hudsonrivertrading.com/careers/job/?gh_jid=8052050
- Optiver Grad Quant Dev — optiver.com (greenhouse 8632528002, PDG)
- Tower Research Quant Dev — tower-research.com/open-positions/?gh_jid=7102180
- Point72 Quant Software Developer — boards.greenhouse.io/point72/jobs/8036928002
- Baseten GPU Kernel Engineer — jobright.ai/jobs/info/69246380c0cefa13343de6e4
- DeepInfra Early Career — ainfradev.dev / jobleads (Palo Alto, 0–2 yrs)
- NVIDIA DGX Cloud AI Infra new grad — builtin.com/job/software-engineer-dgx-cloud-ai-infrastructure-new-college-grad-2026/11431452

Note: greenhouse pages 403 direct fetch; requirements taken from indexed
copies/snippets, may omit nuance.

## Open items

- [x] Learning resources / project plan for the core (NumPy, pandas,
      pybind11, PyTorch export, pytest/Jupyter) — folded into `docs/00-master.md`
      §11 as **step P — the Python bridge** (per-repo pybind11 wrapper, pytest
      harness, pandas reference for step H, notebooks over committed bench JSON).
      PyTorch-at-export-boundary already lives in step C.
- [x] Decide where Python artifacts live per repo — `py/` (wrapper + tests)
      and `notebooks/` at the repo root, per step P's placement convention.
