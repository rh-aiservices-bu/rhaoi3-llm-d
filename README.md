## Running llm-d Demo, comparing against vLLM for tail latency improvements

This demonstrates the performance benefits of llm-d's intelligent routing compared to vanilla vLLM deployments through three steps.

### Step 0: Deploy Monitoring Stack

**Objective**: Set up Prometheus and Grafana to visualize real-time metrics during benchmarks. Keep the Grafana dashboard open throughout to observe performance differences.

#### Deploy Monitoring

```bash
oc apply -k monitoring

# Wait for Grafana to be ready
oc wait --for=condition=ready pod -l app=grafana -n llm-d-monitoring --timeout=300s

# Get Grafana URL
export GRAFANA_URL=$(oc get route grafana-secure -n llm-d-monitoring -o jsonpath='{.spec.host}')
echo "Grafana URL: https://$GRAFANA_URL"
```

#### Access Grafana Dashboard

1. Open `https://$GRAFANA_URL` in your browser
2. Login with default credentials: `admin` / `admin`
3. Navigate to **Dashboards → LLM Performance Dashboard**
4. Keep this dashboard open during all benchmarks

#### Key Metrics to Watch

| Metric | What to Look For |
|--------|------------------|
| **KV Cache Hit Rate** | Higher is better - llm-d should show significantly higher cache hits |
| **Time to First Token (TTFT)** | Lower P95/P99 indicates better tail latency |
| **Requests per Second** | Overall throughput comparison |
| **GPU Utilization** | llm-d should show more balanced utilization across replicas |

---

### Step 1: Baseline vLLM Performance with GuideLLM

**Objective**: Demonstrate vLLM's raw throughput and ease of configuration. Show how easy it is to get blazingly fast inference from a single instance with tensor parallelism across multiple GPUs.

#### Deploy vLLM (4 replicas)

```bash
oc apply -k vllm

# Wait for all replicas to be ready
oc wait --for=condition=ready pod -l serving.kserve.io/inferenceservice=qwen-vllm -n demo-llm --timeout=300s

```

#### Run GuideLLM Benchmark

```bash
oc apply -k guidellm/overlays/vllm

# Watch the results
oc logs -f job/vllm-guidellm-benchmark -n demo-llm
```

> **Grafana**: Watch the dashboard during this benchmark. Note the baseline TTFT and throughput metrics for vLLM.

#### Key Takeaways

- vLLM provides excellent single-instance throughput
- Easy to configure and deploy
- Automatic prefix caching within each replica

**However**: Now that we've confirmed vLLM is the optimal inference server, let's examine how monolithic deployments and simplistic routing strategies can introduce bottlenecks that leave GPUs underutilized.

---

### Step 2: Reveal vLLM Scaling Limitations

**Objective**: Use the multi-turn benchmark to simulate scenarios where vLLM without llm-d demonstrates issues with tail latency. This shows **"what your most frustrated users see"**.

#### Run Multi-Turn Benchmark Against vLLM

```bash
# Run the benchmark job
oc apply -k benchmark-job/overlays/vllm

# Watch the results
oc logs -f job/vllm-multi-turn-benchmark -n demo-llm
```

> **Grafana**: Observe the **KV Cache Hit Rate** - with round-robin routing, cache hits will be low (~25%) as requests scatter across replicas. Watch **TTFT P95/P99** spike during multi-turn conversations.

#### Scenarios Demonstrated

**Scenario A: Multi-Turn Chat (KV Cache Re-use)**
- Simulates realistic conversations where efficient KV cache re-use is critical
- With round-robin routing, requests from the same conversation hit different replicas, missing cached prefixes

**Scenario B: Large Prompts (Prefill Bottlenecks)**
- Seed documents contain 4000+ token code files and research papers
- First turn latency varies wildly depending on which replica handles it
- No prefix sharing between replicas

#### Expected vLLM Results

```
Time to First Token (TTFT):
  P50:        123.22 ms
  P95:        744.71 ms    <-- High tail latency (frustrated users)
  P99:        840.95 ms

First Turn vs Subsequent Turns (Prefix Caching Indicator):
  First turn avg:      351.64 ms
  Later turns avg:     196.29 ms
  Speedup ratio:         1.79x   <-- Suboptimal cache reuse
```

#### Cleanup vLLM

```bash
oc delete job vllm-guidellm-benchmark vllm-multi-turn-benchmark -n demo-llm
oc delete -k vllm
```

#### Reset Monitoring Data

Restart the Prometheus pod to clear vLLM metrics before deploying llm-d:

```bash
oc delete pod -l app=prometheus -n llm-d-monitoring
oc wait --for=condition=ready pod -l app=prometheus -n llm-d-monitoring --timeout=120s
```

---

### Step 3: llm-d Intelligent Routing

**Objective**: Deploy llm-d and re-run the benchmark, demonstrating improved tail latency through prefix-aware routing.

#### Deploy llm-d (4 replicas)

```bash
oc apply -k llm-d

# Wait for all replicas to be ready
oc wait --for=condition=ready pod -l app.kubernetes.io/name=qwen -n demo-llm --timeout=300s
```

#### Run the Same Benchmark

```bash
# Run the benchmark job against llm-d
oc apply -k benchmark-job/overlays/llm-d

# Watch the results
oc logs -f job/llm-d-multi-turn-benchmark -n demo-llm
```

> **Grafana**: Compare with vLLM results. The **KV Cache Hit Rate** should jump to ~90%+ as llm-d routes requests to replicas with cached prefixes. **TTFT P95/P99** should be significantly lower and more consistent.


#### Expected llm-d Results

```
Time to First Token (TTFT):
  P50:         92.09 ms
  P95:        271.60 ms    <-- Significantly lower tail latency
  P99:        674.21 ms

First Turn vs Subsequent Turns (Prefix Caching Indicator):
  First turn avg:      361.79 ms
  Later turns avg:      94.22 ms
  Speedup ratio:         3.84x   <-- Excellent cache reuse
```

#### Why llm-d Performs Better

| Feature | vLLM (Round-Robin) | llm-d (Intelligent Routing) |
|---------|-------------------|----------------------------|
| **Routing Strategy** | Random/Round-robin | Prefix-aware scoring |
| **Cache Hits** | ~25% (1 in 4 replicas) | ~90%+ (routes to cached replica) |
| **P95 Latency** | High variance | Consistent, lower |
| **GPU Utilization** | Imbalanced | Balanced via KV-cache scoring |

---

### Results Comparison

| Metric | vLLM | llm-d | Improvement |
|--------|------|-------|-------------|
| P50 TTFT | 123 ms | 92 ms | 25% faster |
| P95 TTFT | 745 ms | 272 ms | **63% faster** |
| P99 TTFT | 841 ms | 674 ms | 20% faster |
| Cache Speedup | 1.79x | 3.84x | **2.1x better** |

### Key Messages for Customers

1. **Tail latency matters**: P95/P99 represents your most frustrated users
2. **Cache efficiency at scale**: Single-replica caching doesn't help when requests scatter across replicas
3. **Intelligent routing**: llm-d's prefix-aware routing ensures requests hit the replica with relevant cached data
4. **No application changes**: Same API, same model, better performance
