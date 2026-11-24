---
type: concept
tags: [provisioning, metering-billing]
sources: [2026-11-12-transformer-inference-arithmetic, 2026-11-21-what-is-llm-inference]
created: 2026-11-21
updated: 2026-11-24
---

# Prefill and decode

Generating a response takes the GPU through two phases. **Prefill** reads the entire prompt at once, fills the [[kv-cache]], and produces the first output token. **Decode** then produces the rest of the answer one token at a time, and each new token depends on all the tokens before it ("autoregressive"). Analogy: prefill is reading the whole exam question in one go. Decode is writing the answer word by word, glancing back at your notes before each word. The two phases stress the GPU differently, so they cost differently. (src: [[2026-11-12-transformer-inference-arithmetic]], [[2026-11-21-what-is-llm-inference]]; first token from prefill: general knowledge; verified: [Sarathi-Serve (Agrawal et al., 2024)](https://arxiv.org/abs/2403.02310))

## Details

- The prompt "can happen in parallel": all prompt tokens go through the model together. Sampling new tokens has to go "one by one". (src: [[2026-11-12-transformer-inference-arithmetic]])
- Parallel forward passes are "very cheap because of the parallelism," whereas in sampling "we're forced to read through all the weights for each token." (src: [[2026-11-12-transformer-inference-arithmetic]])
- In terms of the 208 FLOPs/byte threshold of an A100 40 GB in 16-bit ([[inference-batching]]): one pass costs the same time for 1 to ~208 tokens, because reading the weights dominates. A prompt of ~208 tokens or more fills that budget in one pass, so prefill tends to be **compute-bound**. A decode step handles one token per request, so with fewer than ~208 requests in the batch it is **memory-bandwidth-bound** ([[gpu-performance-regimes]]). (derived from src: [[2026-11-12-transformer-inference-arithmetic]])
- Measurements of real serving agree: prompt computation is "compute-intensive" and token generation "memory-intensive" and underuses compute. (general knowledge; verified: [Splitwise (Patel et al., 2023)](https://arxiv.org/abs/2311.18677), [Sarathi-Serve](https://arxiv.org/abs/2403.02310))
- The handbook defines inference as "tokens in, next-token distribution out, repeated through prefill and the decode loop." (src: [[2026-11-21-what-is-llm-inference]])
- User-visible latency splits the same way: **time to first token (TTFT)** is queueing plus prefill (plus tokenization and network); **time per output token (TPOT)**, the speed of the streamed text after that, is set by decode. (general knowledge; verified: [DistServe (Zhong et al., 2024)](https://arxiv.org/abs/2401.09670))

## Why it matters for a GPU cloud

- **Metering and billing:** per-token APIs price input tokens lower than output tokens; Anthropic's list prices put output at 5× input for every Claude model (details and other providers in [[per-token-pricing]]). (general knowledge; verified: [Anthropic pricing](https://platform.claude.com/docs/en/about-claude/pricing)) A plausible cost reason, which providers do not publish: prompt tokens share one parallel prefill pass, while each output token needs its own decode step over the weights. (general knowledge)
- **Provisioning / scheduling:** when a long prompt's prefill is batched with other requests' decode steps, those decodes pause until the prefill finishes, so their streamed output stutters. Two scheduler responses (general knowledge):
  - **Chunked prefill:** split a long prefill into chunks and mix them with decode steps, giving "stall-free" schedules that admit new requests "without pausing ongoing decodes". (verified: [Sarathi-Serve](https://arxiv.org/abs/2403.02310); supported in vLLM: [vLLM README](https://github.com/vllm-project/vllm))
  - **Disaggregated prefill/decode:** run the two phases on separate GPUs or machines and transfer the request's state (its KV cache) from the prefill machine to the decode machine, so each pool is sized for its own bottleneck. (verified: [Splitwise](https://arxiv.org/abs/2311.18677), [DistServe](https://arxiv.org/abs/2401.09670))

## Related

- [[kv-cache]]: written during prefill, read on every decode step.
- [[inference-batching]]: decode is cheap only when many requests share a step.
- [[inference-latency-model]]: the per-token decode latency equations.
- [[llm-inference]]: where these phases sit in a request's path.

## Open questions

- How do production frameworks decide chunk sizes, and when does disaggregation pay for the cost of moving the KV cache between GPUs? Needs a source.
