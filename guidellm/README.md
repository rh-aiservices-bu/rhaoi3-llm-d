# Multi-Turn Conversation Benchmark Tool

A Python-based benchmarking tool for testing LLM inference performance with realistic multi-turn conversations. Designed to evaluate prefix caching efficiency and KV cache behavior in LLM deployments like vLLM and llm-d.

## Features

- **Seed Document Based**: Uses real-world documents (code and text) as conversation starters
- **Document Type Detection**: Automatically classifies documents as CODE or TEXT and applies appropriate instructions
- **Multi-Turn Conversations**: Simulates realistic back-and-forth conversations with follow-up questions
- **Parallel Execution**: Multiple concurrent workers with random delays to simulate real user behavior
- **Prefix Caching Analysis**: Measures TTFT (Time to First Token) to evaluate cache hit rates
- **Comprehensive Statistics**: Per-turn analysis, document type breakdown, and speedup ratios

## Requirements

```bash
pip install httpx
```

## Usage

### Basic Usage

```bash
python multi-turn-benchmark.py http://localhost:8000/v1
```

### Full Options

```bash
python multi-turn-benchmark.py <URL> [OPTIONS]

Arguments:
  URL                     Base URL of the LLM API (e.g., http://localhost:8000/v1)

Options:
  -d, --seed-documents    Directory containing seed documents (default: ./seed-documents)
  -c, --conversations     Number of concurrent conversations (default: 11)
  -t, --turns             Number of turns per conversation (default: 10)
  -m, --max-tokens        Maximum tokens per response (default: 500)
  -p, --parallel          Number of parallel workers (default: 4)
  --min-delay             Minimum delay between requests in seconds (default: 0.5)
  --max-delay             Maximum delay between requests in seconds (default: 2.0)
  --timeout               Request timeout in seconds (default: 120)
  -v, --verbose           Show response previews
```

### Examples

```bash
# Basic run with defaults
python multi-turn-benchmark.py http://localhost:8000/v1

# High concurrency test
python multi-turn-benchmark.py $LLM_URL --parallel 8 --min-delay 0.1 --max-delay 0.5

# Quick test with fewer turns
python multi-turn-benchmark.py $LLM_URL --conversations 5 --turns 3

# Custom documents directory
python multi-turn-benchmark.py $LLM_URL --seed-documents /path/to/documents
```

## Seed Documents

The tool uses documents from the `seed-documents/` directory. Documents are classified by file extension:

### Code Documents (`.py`, `.go`, `.rs`, `.tsx`, `.ts`, `.js`, `.sql`, `.java`, `.c`, `.cpp`, `.rb`)

Initial prompts like:
- "Review this code and identify any bugs or issues"
- "Add detailed comments to explain what each function does"
- "Refactor this code to improve readability"
- "Identify potential security vulnerabilities"

Follow-up prompts like:
- "The code still has issues. Can you look more carefully?"
- "Can you show me what the fixed code would look like?"
- "What about edge cases - are those handled properly?"

### Text Documents (`.md`, `.txt`, `.rst`, `.html`)

Initial prompts like:
- "Summarize the main points of this document"
- "Create a bulleted list of the key takeaways"
- "What are the main arguments or findings presented?"

Follow-up prompts like:
- "Can you make that summary longer and more detailed?"
- "That's too long. Can you make it more concise?"
- "What are the practical implications of these findings?"

## Running on OpenShift

### 1. Create ConfigMaps

```bash
# Create ConfigMap for the benchmark script
oc create configmap benchmark-script -n demo-llm \
  --from-file=multi-turn-benchmark.py=guidellm/multi-turn-benchmark.py

# Create ConfigMap for seed documents
oc create configmap seed-documents -n demo-llm \
  --from-file=guidellm/seed-documents/
```

### 2. Run the Benchmark Pod

```bash
oc run benchmark-runner -n demo-llm -it --rm \
  --image=python:3.11-slim \
  --restart=Never \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "benchmark-runner",
      "image": "python:3.11-slim",
      "command": ["bash"],
      "stdin": true,
      "tty": true,
      "env": [
        {"name": "HOME", "value": "/tmp"},
        {"name": "PIP_CACHE_DIR", "value": "/tmp/pip-cache"}
      ],
      "volumeMounts": [
        {"name": "script", "mountPath": "/app"},
        {"name": "seed-documents", "mountPath": "/app/seed-documents"}
      ]
    }],
    "volumes": [
      {"name": "script", "configMap": {"name": "benchmark-script"}},
      {"name": "seed-documents", "configMap": {"name": "seed-documents"}}
    ]
  }
}'
```

### 3. Inside the Pod

```bash
# Install dependencies
pip install httpx

# Run against vLLM
python /app/multi-turn-benchmark.py http://qwen-vllm-lb.demo-llm.svc.cluster.local:8000/v1 \
  -d /app/seed-documents

# Run against llm-d
python /app/multi-turn-benchmark.py http://qwen-llmd-predictor.demo-llm.svc.cluster.local:8000/v1 \
  -d /app/seed-documents
```

## Understanding the Output

### Key Metrics

| Metric | Description |
|--------|-------------|
| **TTFT** | Time to First Token - measures how quickly the model starts responding |
| **Total Request Time** | End-to-end time for the complete response |
| **Speedup Ratio** | First turn TTFT / Later turns TTFT - indicates prefix caching effectiveness |

### Example Output

```
================================================================================
BENCHMARK SUMMARY
================================================================================

Total time: 242.75s
Total requests: 110
Completed conversations: 11/11
Requests per second: 0.45

Time to First Token (TTFT):
  Min:         51.19 ms
  Max:        804.54 ms
  Mean:       120.98 ms
  P50:         92.09 ms
  P95:        271.60 ms
  P99:        674.21 ms

TTFT by Turn Number:
  Turn  1:     361.79 ms avg (11 requests)
  Turn  2:      83.43 ms avg (11 requests)
  Turn  3:      86.10 ms avg (11 requests)
  ...

First Turn vs Subsequent Turns (Prefix Caching Indicator):
  First turn avg:      361.79 ms
  Later turns avg:      94.22 ms
  Speedup ratio:         3.84x
```

### Interpreting Results

- **High Speedup Ratio (>2x)**: Prefix caching is working effectively. Later turns reuse cached KV values from previous turns.
- **Low Speedup Ratio (~1x)**: Prefix caching may not be effective, or cache is being evicted between requests.
- **Consistent TTFT across turns**: Good cache retention throughout the conversation.
- **Increasing TTFT on later turns**: May indicate cache pressure or eviction.

## Comparing vLLM vs llm-d

When comparing deployments:

| Scenario | Expected llm-d Advantage |
|----------|-------------------------|
| Many unique prefixes across replicas | llm-d routes similar prefixes to same replica |
| Multi-turn conversations | Both benefit, but llm-d optimizes replica selection |
| High concurrency | llm-d balances cache utilization across replicas |

## Troubleshooting

### Error 400 Bad Request

Usually indicates context length exceeded. Solutions:
- Reduce `--max-tokens` (e.g., `--max-tokens 200`)
- Reduce `--turns` (e.g., `--turns 5`)
- Ensure model has sufficient `--max-model-len` configured

### Connection Errors

- Verify the service URL is correct
- Check if the model pods are running: `oc get pods -n demo-llm`
- Test connectivity: `curl $LLM_URL/models`

### Slow Performance

- Reduce `--parallel` if overwhelming the service
- Increase `--min-delay` and `--max-delay` for more realistic pacing
- Check GPU utilization on model pods
