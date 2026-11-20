---
type: concept
tags: [provisioning, metering-billing, career]
sources: [2026-11-12-transformer-inference-arithmetic, 2026-11-21-what-is-llm-inference]
created: 2026-11-21
updated: 2026-11-21
---

# Prefill and decode

Generating a response takes the GPU through two phases. **Prefill** reads the entire prompt at once and fills the [[kv-cache]]. **Decode** then produces the answer one token at a time, and each new token depends on all the tokens before it ("autoregressive"). Analogy: prefill is reading the whole exam question in one go. Decode is writing the answer word by word, glancing back at your notes before each word. The two phases stress the GPU differently, so they cost differently. (src: [[2026-11-12-transformer-inference-arithmetic]], [[2026-11-21-what-is-llm-inference]])

## Details

- The prompt "can happen in parallel": all prompt tokens go through the model together. Sampling new tokens has to go "one by one". (src: [[2026-11-12-transformer-inference-arithmetic]])
- Parallel forward passes are "very cheap because of the parallelism," whereas in sampling "we're forced to read through all the weights for each token." (src: [[2026-11-12-transformer-inference-arithmetic]])
- In terms of the 208 flops/byte threshold ([[inference-batching]]): a prompt of a few hundred tokens or more already puts that many tokens in flight, so prefill tends to be **compute-bound**. A decode step for a small batch is one token per request, so it is **memory-bandwidth-bound** ([[gpu-performance-regimes]]). (derived from src: [[2026-11-12-transformer-inference-arithmetic]]; the phase-by-phase labeling is general knowledge)
- The handbook defines inference as "tokens in, next-token distribution out, repeated through prefill and the decode loop." (src: [[2026-11-21-what-is-llm-inference]])
- User-visible latency splits the same way: time to first token is mostly queueing plus prefill, and the speed of the streamed text after that is decode. (general knowledge)

## Why it matters for a GPU cloud

- **Metering and billing:** per-token APIs commonly price input (prefill) tokens lower than output (decode) tokens. That matches the cost asymmetry above: prefill tokens share one parallel pass, while each output token needs its own pass over the weights. See [[per-token-pricing]]. (general knowledge)
- **Provisioning / scheduling:** a long-prompt request causes a compute spike that can stall other requests' decode steps when they share a batch. This is one reason serving schedulers treat the phases differently. (general knowledge; not in the sources so far)

## Related

- [[kv-cache]]: written during prefill, read on every decode step.
- [[inference-batching]]: decode is cheap only when many requests share a step.
- [[inference-latency-model]]: the per-token decode latency equations.
- [[llm-inference]]: where these phases sit in a request's path.

## Open questions

- How do serving frameworks schedule prefill and decode together (chunked prefill, disaggregated prefill/decode on separate GPUs)? Needs a source.
