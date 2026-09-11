# Multi-LoRA 混合流量压测

一个推理服务可以同时托管一个 Base Model 和多个 LoRA Adapter。在 vLLM 等 OpenAI-compatible 服务中，通常通过每个请求的 `model` 字段选择对应 Adapter。EvalScope 无需新增专用的 Multi-LoRA 模式即可压测这类混合流量：使用 `line_by_line` 数据集，并在 JSONL 的每一行直接提供完整请求体。

## 准备混合流量

创建 `multi_lora.jsonl`，在每条请求中指定目标 Adapter。文件中不同 Adapter 的出现比例就是压测的流量分布。

```jsonl
{"messages":[{"role":"user","content":"介绍一下杭州。"}],"model":"lora-A"}
{"messages":[{"role":"user","content":"写一首短诗。"}],"model":"lora-B"}
{"messages":[{"role":"user","content":"解释相对论。"}],"model":"lora-A"}
{"messages":[{"role":"user","content":"翻译这段话。"}],"model":"lora-C"}
```

例如，要模拟 80/20 的热点 Adapter 流量，可以让大约 80% 的行指定 `lora-A`，其余 20% 指定其他 Adapter。`line_by_line` 会保留每条请求中显式给出的 `model`；命令行的 `--model` 只作为未指定 `model` 行的默认值。

## 发起压测

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

这些请求会并发混合发送到同一个推理实例，因此汇总的 TTFT、时延、吞吐量等压测指标会反映该 JSONL 所描述工作负载中的 Adapter 切换和资源争用影响。

## 当前范围与限制

该方式面向 issue #1493 中确认的 **混合流量整体性能** 场景。目前不直接提供：

- 内置随机、加权或 Zipf Adapter 分配策略；请直接在 JSONL 输入中按期望比例编排请求；
- 按 Adapter 分组的 TTFT、吞吐量或 SLA 指标；当前结果对整组混合请求进行汇总。

如果需要 per-adapter 指标，可以暂时分别运行各 Adapter 的独立压测，或对保存的压测数据进行后处理，直到框架原生支持分组指标。
