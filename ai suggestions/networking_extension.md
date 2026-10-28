# Networking Extension — 3rd Quant Anchor (rationale + scope)

Status: AI suggestion (2026-10-26). Not yet in `00-master.md`. Supersedes "two quant
anchors" with a third: **low-latency network market-data ingestion feeding the existing
orderbook.** Engine is *triaged*, not dropped (see §5).

---

## 1. Rationale — why networking, why now

Entry-level data (see `career_research_2026.md`): quant dev is the highest-ROI *entry*
(deep junior pipeline, six-figure base); AI-perf demand is senior-biased (only 2-6%
junior-open). So the 3rd project must **maximize the door walked through now** while not
sacrificing the AI ceiling.

Networking is the only project that does both, because it is **dual-track**:

- **Quant side:** low-latency market-data ingestion is the core HFT QD skill. Current
  orderbook ingests from a *file* (mmap ITCH). Real HFT ingests from a *network feed*.
  This closes the one named quant gap in the profile (zero low-latency-networking
  experience) and turns the orderbook into a complete stack: **ingest → match → serve**.
- **AI-infra side:** the same fundamentals — kernel bypass, zero-copy RX, NIC tuning,
  NUMA, RDMA/RoCE, the NCCL/interconnect world — are **one of the most acute shortages
  in AI inference** (tensor parallelism, disaggregated serving over the network). So the
  project reinforces the AI track *without* being a pure-AI build.

Contrast: the Inference Engine (GEMM/GPT-2/batching) is *pure* AI — high ceiling, zero
quant-entry value. Networking serves both tracks; that's why it beats "build the full
engine now" for the 3rd slot.

**Decision:** 3rd project = AF_XDP + UDP-multicast ingestion feeding the orderbook.
Engine = triaged (cheap high-signal parts now, expensive parts post-landing).

## 2. How it affects the Plan

- **Anchors:** "two quant anchors (Orderbook + Backtester)" → **three** (Orderbook +
  Backtester + **NetworkIngest**). The engine pauses for the anchor, per the existing rule.
- **Compounds, not replaces:** NetworkIngest is an *extension* of the Orderbook repo —
  it reuses the existing SPSC FIFO (Phase 0) as the RX→match handoff. No new throwaway repo.
- **Engine triage (not a drop):**
  - **Now (cheap, high-signal resume lines):** Phase A GEMM ladder + 1 INT8 kernel;
    PyTorch reference decoder (bit-for-bit vs the C++ decoder).
  - **Post-landing (expensive, becomes a promotion asset):** full scheduler (continuous
    batching / KV budget), distributed TP, gateway hardening.
- **Time allocation:** NetworkIngest is the primary hours-sink for the next block; the
  engine's cheap parts proceed in parallel. Full engine is deferred, not abandoned —
  it stays a **planned/in-progress** resume line.

## 3. Project scope (NetworkIngest)

Goal: feed the existing matching engine from a **network** source with measured,
bracket-rule numbers, and prove the matched output is **bit-for-bit identical** to the
file-fed path.

### Phases (ordered)
- **N0 — Baseline (kernel UDP multicast):** receive an ITCH 5.0 / synthetic orderbook
  feed over UDP multicast via a plain kernel socket. Decode → existing SPSC FIFO →
  matching engine. Measure p50/p99 latency, msg/s, CPU. This is the "before."
- **N1 — AF_XDP path:** replace the kernel socket with AF_XDP (UMEM, fill/complete
  rings, zero-copy). Same decode → FIFO → engine. Measure the delta vs N0. This is the
  "after."
- **N2 — RX-path hardening:** CPU pinning + interrupt affinity, huge pages, NUMA
  placement of the RX thread, zero-alloc proof on the RX path (allocator interpose,
  same LD_PRELOAD technique as Phase 4).
- **N3 — Determinism/replay:** capture a network feed, replay it, and assert the match
  results are **bit-identical** to the file-fed orderbook run (and TSan-clean). This
  is the correctness anchor.
- **N4 — Proof pack:** README + design doc + ablation table (kernel-UDP vs AF_XDP ×
  {pinning, hugepages, NUMA}).

### Stretch (theory + cloud only — no local NIC)
- **RDMA / RoCE / InfiniBand:** learn the model, profile what's possible on a cloud
  instance with an RDMA NIC. *No local measured number* — state the boundary explicitly.
- **NCCL adjacency:** map the AF_XDP/zero-copy/NUMA lessons onto how NCCL moves tensors
  over the interconnect (feeds the deferred distributed-TP phase).

## 4. What it lets you learn (and where each maps)

| Skill | Quant value | AI-infra value |
|---|---|---|
| Kernel UDP path: softirq, NAPI, interrupt handling | core HFT ingest | — |
| Kernel bypass: AF_XDP (UMEM/rings/zero-copy), DPDK concepts | core HFT ingest | zero-copy RX pattern |
| NIC tuning: RSS, offloads, IRQ affinity, CPU pinning | latency work | NIC/interconnect tuning |
| Huge pages, NUMA placement | latency work | tensor-parallel placement |
| Multicast: group join, IGMP, loopback scope | market-data delivery | — |
| Network-path latency measurement (hw timestamps, cycle counts) | bracket-rule numbers | profiling vocab |
| Lock-free RX→handoff (extends Phase 0 SPSC) | concurrency | — |
| Feed capture + deterministic replay | correctness | replay/determinism culture |
| RDMA/RoCE/NCCL (theory+cloud) | — | **most acute AI-infra gap** |

**Resume lines this produces (bracket rule — each needs a measured number):**
- "Ingest a [N]M msg/s orderbook feed over **AF_XDP**, [X] ns lower p99 latency than
  kernel UDP multicast, zero hot-path allocations on the RX path (verified via
  LD_PRELOAD)."
- "Bit-for-bit identical match output, network-fed vs file-fed, across [M] replayed
  messages (TSan-clean)."
- (stretch) "Profiled RDMA/RoCE vs TCP for tensor all-reduce on [cloud] — [Y]% of
  linear scaling" *only if actually measured on cloud; else state as theory.*

## 5. Risks / boundaries (honesty rule)
- **AF_XDP needs XDP-capable NIC + driver.** Verify the 9800X3D desktop's NIC first.
  Fallback order: AF_XDP → DPDK → tuned kernel-UDP baseline (still a valid measured
  comparison, just not kernel-bypass).
- **No RDMA NIC locally** → RDMA is theory + cloud; no local number. Say so.
- **True multicast needs a switch.** Single-host loopback multicast (two processes) is
  fine for the measured delta; note the boundary, don't claim real network multicast.
- **Scope discipline:** N0→N3 is the shippable core. N4 is the proof. Stretch is
  optional and explicitly non-local. Shipped > ambitious.

## 6. Net
NetworkIngest is the door (quant entry, measured, compounds the orderbook) **and** the
bridge (the exact skills AI-inference infra is short on). The engine keeps its ceiling but
moves to "cheap parts now, full build post-landing." Primary search = quant; secondary =
the rare new-grad AI-perf slots; engine = the documented on-ramp once in.
