# 6 · Serving and inference optimization

Inference cost and latency are architecture problems, not procurement problems. Before comparing GPU quotes, understand the three numbers that govern everything: time-to-first-token (TTFT), inter-token latency, and tokens per second per dollar.

## API or self-host?

| Factor | Managed API | Self-hosted open weights |
|---|---|---|
| Time to production | days | weeks |
| Marginal cost at low volume | lower | higher (idle GPUs bill too) |
| Marginal cost at high, steady volume | higher | lower — the crossover is real but further out than enthusiasm suggests |
| Data control / residency | contractual | physical |
| Model choice | frontier quality | best open weights + your fine-tunes |
| Ops burden | theirs | yours: CUDA, drivers, autoscaling, on-call |

The honest default: start on APIs, instrument per-request cost, and revisit when the monthly bill rivals an engineer — or when residency/fine-tuning requirements force the issue earlier.

## The anatomy of a generation

Two phases with opposite profiles: **prefill** (process the prompt; compute-bound, parallel, determines TTFT) and **decode** (one token at a time; memory-bandwidth-bound, determines tokens/sec). Long prompts tax prefill; long outputs tax decode. This asymmetry explains most latency surprises — a 100-token answer to a 20k-token prompt is slow *before* the first token, not during.

## The serving stack that matters

- **[vLLM](https://docs.vllm.ai/en/latest/)** is the de-facto open-source serving baseline. Its core idea, PagedAttention, treats the KV cache like virtual memory — allocated in blocks, no fragmentation — which enables **continuous batching**: new requests join the running batch at token granularity instead of waiting for the batch to drain. Throughput gains vs. naive serving are routinely an order of magnitude.
- **KV cache is your scarcest resource.** Its size grows with batch × context length; it, not weights, is usually what caps concurrency. Prefix caching (sharing the KV of common prompt prefixes — system prompts, few-shot blocks) is the same trick as provider-side prompt caching in chapter 1, and it's why prompt *ordering* is a performance decision.
- **[llama.cpp](https://github.com/ggml-org/llama.cpp)** owns the CPU/edge niche (GGUF formats) — the right tool for local development, small-scale private deployments, and laptops; the wrong tool for datacenter throughput.

## Quantization

Weights in fewer bits: less memory, more bandwidth, faster decode.

- **8-bit** (INT8/FP8): effectively free quality-wise for most models; the default for serving.
- **4-bit** (AWQ/GPTQ-class, GGUF K-quants): large savings, small but *measurable* degradation — acceptable for many tasks, not for all. The only defensible policy: run **your** eval suite (chapter 5) on the quantized artifact, not someone's perplexity table. Quantization hits reasoning-heavy and low-resource-language tasks hardest, which is exactly what generic benchmarks underweight.
- Quantize weights first; KV-cache quantization (8-bit) buys concurrency when memory-capped.

## Latency engineering

- **Stream to humans** — perceived latency is TTFT, and TTFT is prefill: trim the prompt, cache the prefix.
- **Speculative decoding**: a small draft model proposes tokens, the large model verifies in parallel — 2–3× decode speedups when the draft model fits the distribution; free lunch only if acceptance rates hold on *your* traffic.
- **Right-size the model per route** (chapter 1's routing): the cheapest optimization is not running the big model.
- Set `max_tokens` from product requirements; unbounded generation is unbounded tail latency.

## Cost arithmetic (do it before the meeting)

```
tokens/month  = requests × (prompt_tokens + output_tokens)
API cost      = tokens × per-token price (input/output priced separately; batch tier ≈ half)
GPU cost      = GPUs × hourly rate × 730h  — divide by measured tokens/sec to get $/1M tokens
```

Two numbers decide the argument: your **utilization** (idle GPUs destroy self-hosting economics; bursty traffic favors APIs) and your **output length** (output tokens are the expensive ones on APIs and the slow ones everywhere).

## Production checklist

- Continuous batching on; prefix caching on; metrics for TTFT, tokens/sec, queue depth, KV-cache utilization.
- Load-shedding policy: degrade to a smaller model under pressure instead of queueing into timeout.
- Quantized artifact passed the same eval suite as the FP16 one, on your tasks.
- Capacity math re-run when prompt templates change — a 2× prompt is a 2× prefill bill nobody announced.
