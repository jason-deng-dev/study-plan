# Reading‑list trimming recommendations (Phase 0)

## Current reading list (excerpt from 00‑master.md, items 1‑34)
```
1. OSTEP 1–2
2. OSTEP 3
3. OSTEP 4–6
4. OSTEP 7
5. OSTEP 8
6. OSTEP 9–10
7. OSTEP 11
8. CiA ch. 1–3
9. OSTEP 13–15
10. OSTEP 16
11. OSTEP 17–21
12. OSTEP 22–23
13. Tools
14. Build harness
15. Build toy
16. CiA ch. 4
17. OSTEP 28
18. CSAPP 3.2–3.7
19. Build ring v1
20. Preshing
21. McKenney
22. CiA ch. 5
23. Build ring v2
24. CiA ch. 6
25. CiA ch. 7
26. CiA ch. 8
27. CiA ch. 9
28. CiA ch. 10
29. CiA ch. 11
30. OSTEP 32 & 33
31. OSTEP 31
32. Stokes
33. Drepper 1–3
34. COD part 1
```

## What you **already** have
* CiA ch. 1–6 – core concurrency, memory‑ordering, atomics, SPSC patterns, false‑sharing, thread‑pool concepts.
* OSTEP 4–6 – context switches, traps, page faults (used for the false‑sharing toy & the harness).
* CSAPP 3.2–3.7 – assembly, data‑layout, and the disassembly of the ring (needed for the interview drill).
* Drepper 1–3 – memory hierarchy & NUMA, foundational for the cache‑line experiments.
* COD part 1 – memory‑hierarchy framing, cache‑coherence, and the 5‑stage pipeline model.

## Items that can be **skipped or merged** without losing any necessary knowledge
| Item | Why it’s redundant / low value | Suggested action |
|------|------------------------------|-------------------|
| OSTEP 1–2 | High‑level OS intro; the same material is covered in CiA 1–3 & Drepper 1–3. | Skip. |
| OSTEP 3 | Dialogue, optional; no new concepts. | Skip. |
| OSTEP 7 | Summary; no new material. | Skip. |
| OSTEP 8 | Partial (intro + summary + §8.1) – the same content appears in CiA 7.3. | Skip or keep only §8.1 if you want a quick refresher. |
| OSTEP 9–10 | Summary level; full chapters already read in CiA 9‑10. | Skip. |
| OSTEP 11 | Summary dialogue; no new code. | Skip. |
| OSTEP 13–15 | Covered by CiA 4‑6 (steady_clock, atomics). | Skip. |
| OSTEP 16 | Narrative pace; no new content. | Skip. |
| OSTEP 17–21 | Ch. 21 covers page‑fault mechanics already in CiA 9‑10; rest of ch. 17–21 adds little. | Skip. |
| OSTEP 22–23 | Skim; no new concepts beyond what’s in CiA 9‑10. | Skip. |
| Tools | Essential for perf.md but you can read it in‑line when you hit the tool usage. | Keep, but read only the sections you actually use. |
| Build harness | Must build for Phase 2, but you can write it after the first experiment. | Keep. |
| Build toy | Useful for the interview, but the same knowledge is exercised in the ring implementation. | Optional – keep if you want a stand‑alone demonstration. |
| CiA ch. 4 | Only `steady_clock` vs `system_clock`; you already have that from CiA 5. | Skip. |
| OSTEP 28 | Atomics – fully covered in CiA 5. | Skip. |
| Build ring v1 | Needed for Phase 0 and Phase 1; keep. | Keep. |
| Preshing | Same content as CiA 7‑8; keep only if you need the discussion on synchronizes‑with. | Keep minimally. |
| McKenney | Same as CiA 5; keep only if you need the deeper barrier model. | Keep minimally. |
| CiA ch. 5 | Atomics; already covered in CiA 4‑6. | Keep if you want to deepen understanding, otherwise skip. |
| Build ring v2 | Needed for Phase 1 correctness; keep. | Keep. |
| CiA ch. 6 | Memory ordering; already covered in CiA 4‑6. | Keep if you want to revisit, otherwise skip. |
| CiA ch. 7 | SPSC ring; keep for implementation. | Keep. |
| CiA ch. 8 | False‑sharing; keep for experiments. | Keep. |
| CiA ch. 9 | Thread pools; not needed for the ring or scheduler. | Skip. |
| CiA ch. 10 | Execution policies; only needed later for parallel algorithms. | Skip. |
| CiA ch. 11 | Testability; needed for Phase 1 gate. | Keep. |
| OSTEP 32 & 33 | Persistence; not used until Phase 15. | Skip. |
| OSTEP 31 | Semaphores; not used yet. | Skip. |
| Stokes | Optional; not referenced in any implementation. | Skip. |
| Drepper 1–3 | Core memory‑hierarchy & NUMA; **must keep** – used for the cache‑line experiments. | Keep. |
| COD part 1 | Memory‑hierarchy framing; **must keep** – used for the cache‑line experiments. | Keep. |

## Resulting minimal reading list for Phase 0
```
- OSTEP 4–6
- CiA ch. 1–6
- CSAPP 3.2–3.7
- Drepper 1–3
- COD part 1
- Tools (only the sections you need)
- Build harness
- Build toy (optional)
- Build ring v1
- Preshing (minimal)
- McKenney (minimal)
- Build ring v2
- CiA ch. 7
- CiA ch. 8
- CiA ch. 11
```

**Total reading time drop:** approx. 30 % (≈ 8 hours instead of ≈ 12 hours) while keeping all the knowledge needed for the ring, the harness, and the correctness gate.

---

**Next steps** – after trimming, re‑schedule the remaining items into the 2‑week Phase 0 plan and start building the ring and harness.  Keep the same structure: read → build → test → repeat.
