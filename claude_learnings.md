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
pip install mooncake          # or build from source in this repo
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
