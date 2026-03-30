# KV Cache Setup for FAST 2025 Figure 1

## KV Cache Size

**Per-token cost (LLaMA3-70B):** 320 KB/token (stated explicitly in Section 5.3.1).

**Per-node capacity:** Each node reserves ~1 TB of DRAM, supporting ~3M tokens of KV cache
(1 TB ÷ 320 KB ≈ 3.1M tokens). Confirmed in Section 5.3.2: *"each node also has a 3M token capacity."*

**Total for Figure 1 (16 nodes):**
- 16 × 3M tokens = ~48M tokens across the cluster
- 16 × 1 TB DRAM = ~16 TB total KV cache storage

---

## Is the Cache Pre-Populated?

No. The cache starts **cold** and fills organically as requests are served:

1. The conversation workload uses **real timestamps** from the trace (Table 2:
   "Arrival Pattern = Timestamp") — requests are replayed in arrival order with no
   separate warm-up phase.
2. The ~40% cache hit ratio (Table 2) is an emergent property of multi-turn
   conversation structure (later turns reuse KV blocks from earlier turns), not a
   pre-loaded cache.
3. Figure 9 (Section 5.3.1) studies cache hit rate as a function of cache capacity,
   showing it grows as the cache fills — only meaningful if the cache starts cold.

---

## Cache Saturation Caveat

Figure 9 shows that at 3M tokens per node (the local cache size used in Figure 1),
the hit rate does **not** reach 50% of the theoretical maximum. You need ~**50M tokens
total** (pooling ~20 nodes' DRAM) to approach the theoretical ceiling. So even in the
Figure 1 experiment, the 16-node global cache of ~48M tokens operates below its
theoretical maximum — the cache accumulates blocks during the run but never fully
saturates.
