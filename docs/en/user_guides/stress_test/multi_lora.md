# Multi-LoRA mixed-traffic stress testing

A single inference service may host one base model together with multiple LoRA adapters. In OpenAI-compatible serving stacks such as vLLM, the adapter is commonly selected per request through the `model` field. EvalScope can benchmark this mixed traffic without a dedicated Multi-LoRA mode: use the `line_by_line` dataset and provide a complete request body on each JSONL line.

## Prepare the traffic mix

Create `multi_lora.jsonl` and set the target adapter in each request. The frequency of each adapter in the file defines the traffic mix.

```jsonl
{"messages":[{"role":"user","content":"Introduce Hangzhou."}],"model":"lora-A"}
{"messages":[{"role":"user","content":"Write a short poem."}],"model":"lora-B"}
{"messages":[{"role":"user","content":"Explain relativity."}],"model":"lora-A"}
{"messages":[{"role":"user","content":"Translate this paragraph."}],"model":"lora-C"}
```

For example, to model an 80/20 hot-adapter split, make roughly 80% of the lines target `lora-A` and 20% target the other adapter(s). `line_by_line` preserves the per-request `model` value. The command-level `--model` remains the fallback for rows that do not provide one.

## Run the benchmark

```bash
evalscope perf \
  --url http://127.0.0.1:8000/v1/chat/completions \
  --api openai \
  --model lora-A \
  --dataset line_by_line \
  --dataset-path multi_lora.jsonl \
  --parallel 16 \
  --number 1000
```

The requests are mixed concurrently against the same serving instance, so the aggregate TTFT, latency, throughput, and other benchmark statistics include effects such as adapter switching and contention in the workload represented by the JSONL file.

## Current scope and limitations

This approach is intended for **aggregate mixed-traffic performance**, which is the use case discussed in issue #1493. It does not currently provide:

- built-in random, weighted, or Zipf adapter assignment; encode the desired distribution in the JSONL input instead;
- per-adapter TTFT, throughput, or SLA breakdowns; the reported metrics are aggregated over the full mixed workload.

If you need per-adapter metrics, run separate adapter-specific benchmarks or post-process the saved benchmark data until native grouped metrics are available.
