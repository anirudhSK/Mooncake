# Claude Learnings: Reproducing FAST 2025 Figure 1

## What Figure 1 Shows

Figure 1 (Introduction, backed by Section 5.2.2 "Effective Request Capacity") shows
**effective request capacity** (% of requests meeting SLO) vs. **TBT threshold (ms)**
for four systems under the real-world Kimi conversation workload:

- **Mooncake** (the paper's system)
- **vLLM** (baseline v0.5.1)
- **vLLM + Prefix Caching**
- **vLLM + Chunked Prefill**

A request is "effective" only if it meets both its TTFT and TBT thresholds.
Requests arrive via a **Poisson process**. The key takeaway is that Mooncake
achieves ~48–59% higher effective request capacity than baselines at strict SLO targets.

---

## Original Hardware (Paper)

| Resource | Spec |
|---|---|
| Nodes | 16 |
| GPUs per node | 8 × NVIDIA A800-SXM4-80GB |
| Total GPUs | 128 |
| NIC | 4 × 200 Gbps RDMA per node |
| Model | LLaMA3-70B |
| KVCache block size | 256 tokens |
| vLLM version | v0.5.1 |

---

## Scaled-Down Reproduction on a 4-Node Cluster

### Key constraint
LLaMA3-70B needs ~140 GB VRAM. With 4 nodes:
- If each node has 8×80 GB GPUs: run with `--tensor-parallel-size 2`
- Easier alternative: use **LLaMA3-8B**, which fits on a single GPU and still
  demonstrates the relative SLO behavior correctly

### Step 1: Install prerequisites (all nodes)

```bash
pip install vllm==0.5.1
pip install mooncake-transfer-engine   # or build from source in this repo (NOT "pip install mooncake" — that is a different unrelated PyPI package)
pip install quart httpx aiohttp datasets

# Start etcd (required by Mooncake metadata server)
docker run -d -p 2379:2379 bitnami/etcd
```

### Step 2: Configure Mooncake

Edit `benchmarks/xypd_benchmarks/vllm-benchmarks/mooncake.config` with your
cluster's RDMA device names and etcd endpoint. Auto-generate the cluster topology:

```bash
python3 scripts/generate_cluster_topology.py --hosts node1,node2,node3,node4
```

### Step 3: Launch the system

```bash
# Start Mooncake master
mooncake_master --port 10001 &

# Prefill instance (nodes 1-2)
MOONCAKE_CONFIG_PATH=mooncake.config \
python3 -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Meta-Llama-3-8B-Instruct \
  --port 8100 \
  --kv-cache-block-size 256 \
  --kv-transfer-config '{"kv_connector":"MooncakeStoreConnector","kv_role":"kv_producer"}'

# Decode instance (nodes 3-4)
MOONCAKE_CONFIG_PATH=mooncake.config \
python3 -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Meta-Llama-3-8B-Instruct \
  --port 8200 \
  --kv-transfer-config '{"kv_connector":"MooncakeStoreConnector","kv_role":"kv_consumer"}'

# Disaggregated proxy
python3 benchmarks/xypd_benchmarks/proxy_demo.py \
  --prefill localhost:8100 --decode localhost:8200 --port 8000
```

### Step 4: Replay the trace

The conversation trace from the paper is already in the repo:

```
FAST25-release/traces/conversation_trace.jsonl
```

Each line has the format:
```json
{"timestamp": 0, "input_length": 6758, "output_length": 500, "hash_ids": [...]}
```

Use vLLM's `benchmark_serving.py` to send requests at varying rates and measure
TTFT/TBT. Sweep `--request-rate` to find the saturation point at each SLO threshold:

```bash
# Clone vLLM source to get benchmark script
git clone https://github.com/vllm-project/vllm.git vllm-src

python3 vllm-src/benchmarks/benchmark_serving.py \
  --backend vllm \
  --model meta-llama/Meta-Llama-3-8B-Instruct \
  --dataset-name sharegpt \
  --num-prompts 1000 \
  --request-rate 5 \          # sweep this to find SLO saturation
  --port 8000 \
  --percentile-metrics="ttft,tpot,itl,e2el" \
  --save-result \
  --result-filename results_mooncake.json
```

Parse results with the included script:
```bash
python3 benchmarks/xypd_benchmarks/vllm-benchmarks/parse_results.py \
  results/ results/summary.xlsx
```

Repeat for all four system configurations and for TBT thresholds: 100, 200, 300, 500, 1000 ms.

---

## What Is and Isn't Ready-Made in the Repo

| Component | Status |
|---|---|
| Mooncake master + transfer engine | Available (build from source) |
| vLLM connector (MooncakeStoreConnector) | Available |
| Disaggregated proxy | `benchmarks/xypd_benchmarks/proxy_demo.py` |
| Benchmark orchestration script | `benchmarks/xypd_benchmarks/vllm-benchmarks/benchmarks.sh` |
| Result parser | `benchmarks/xypd_benchmarks/vllm-benchmarks/parse_results.py` |
| Conversation trace (from paper) | `FAST25-release/traces/conversation_trace.jsonl` |
| Tool+Agent trace | `FAST25-release/traces/toolagent_trace.jsonl` |
| Synthetic trace | `FAST25-release/traces/synthetic_trace.jsonl` |
| **Trace-replay client with Poisson arrivals** | **Not pre-built — needs ~50–100 lines of Python** |

The main gap is a benchmark client that replays `conversation_trace.jsonl` with
correct Poisson inter-arrival times, tracks per-request TTFT/TBT, and sweeps
request rates to find the SLO saturation point. This should be built on top of
`vllm-src/benchmarks/benchmark_serving.py`.

---

## Expected Outcome

The **qualitative result** — Mooncake having substantially higher effective request
capacity at strict TBT SLOs — should reproduce on a 4-node cluster with a smaller
model. Absolute numbers will differ from the paper due to scale and model size.

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

# Cache Hierarchy Setup for the FAST 2025 Paper

## Storage Tiers in the Architecture

Mooncake Store is designed around a three-tier hierarchy (as shown in Figure 2):

1. **GPU VRAM (HBM)** — The "hot" working KV cache used during active inference
   ("Paged KVCache" in Figure 2). Holds only the working set for requests currently
   being processed.
2. **CPU DRAM** — The primary persistent cache tier ("Distributed KVCache Pool" in
   Figure 2). All KVCache in Mooncake Store is stored as paged blocks here, spread
   across nodes and accessed via RDMA.
3. **SSD** — Listed as an additional overflow tier in the architecture diagram
   (`CPU/DRAM/SSD`).

## What Is Actually Used in the Experiments

**SSD is not used in any experiment in the paper.** The evaluation setup (Section 5.1)
only describes GPU VRAM and CPU DRAM. All cache capacity figures in Section 5.3 are
expressed purely in terms of DRAM:

> *"reserving approximately 1 TB of DRAM for local caching, this setup only supports
> storage for about 3 million tokens"* (Section 5.3.1)

Figure 9, Figure 10, Figure 11, and the transfer benchmarks in Figure 12 are all
framed around DRAM capacity and DRAM-to-DRAM RDMA transfers. SSD appears only as a
future/optional overflow tier in the architecture design — it plays no role in the
reported results.

## KVCache Movement During Inference (Figure 3)

The data path for a single request is:

| Step | Action |
|---|---|
| s1: KVCache Reuse | Prefix KVCache loaded from remote **CPU DRAM → GPU HBM** on prefill node |
| s2: Incremental Prefill | Newly generated KVCache stored from **GPU HBM → CPU DRAM** on prefill node |
| s3: KVCache Transfer | KVCache streamed layer-by-layer over **RDMA** from prefill node CPU DRAM → decode node CPU DRAM |
| s4: Decoding | KVCache async-loaded from **CPU DRAM → GPU HBM** on decode node before joining batch |

# What Is mooncake_master?

`mooncake_master` is the **central coordinator (control plane)** for the Mooncake
distributed KV cache store. It is a C++ binary (built from `mooncake-store/src/`)
and must be started before any vLLM prefill/decode instances, since those instances
register their DRAM segments with it at startup.

## Responsibilities

- **Memory segment tracking** — maintains a registry of which nodes have registered
  DRAM segments and how much free space each has
- **Replica placement** — decides where KV cache blocks are stored across the cluster
  (`random` or `free_ratio_first` allocation strategy)
- **Eviction policy** — enforces high-watermark thresholds and evicts soft-pinned
  objects when memory pressure is high
- **Lease/TTL management** — tracks KV object lifetimes via
  `--default_kv_lease_ttl` (default 5 s) and `--default_kv_soft_pin_ttl` (default 30 min)
- **Task management** — queues and retries async RDMA transfer tasks across nodes
- **Metrics endpoint** — exposes `/metrics` over HTTP (default port 9003)
- **Optional embedded HTTP metadata server** — serves cluster topology to the
  transfer engine (port 8080, off by default)
- **Optional HA** — can use etcd for high-availability snapshotting and restore

## Key Flags

| Flag | Default | Purpose |
|---|---|---|
| `--rpc_port` | 50051 | RPC listen port |
| `--metrics_port` | 9003 | HTTP metrics port |
| `--enable_http_metadata_server` | false | Embedded metadata server for transfer engine |
| `--http_metadata_server_port` | 8080 | Metadata server port |
| `--allocation_strategy` | `random` | Replica placement strategy |
| `--eviction_high_watermark_ratio` | 0.95 | Usage ratio that triggers eviction |
| `--enable_ha` | false | Enable HA via etcd |

## Python Entry Point

The `mooncake_master` command installed by the Python wheel
(`mooncake-wheel/mooncake/cli.py`) is a thin wrapper that locates and `exec`s the
compiled C++ binary with all arguments passed through.
