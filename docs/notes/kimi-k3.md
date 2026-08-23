# Kimi K3 serving facts

Collected 2026-08-23. Weights published 2026-07-26.

| Fact | Value |
|---|---|
| Parameters | 2.8T total, 104B active, 16 of 896 experts |
| Layers | 93: 69 KDA, 24 gated MLA, 1 with a dense FFN block |
| Context | 1,048,576 tokens |
| Weights | MXFP4 (quantization-aware trained), 1.56 TB checkpoint on disk |
| MLA KV | ~26 KB/token (7.95 GB pool held 308,608 tokens on 16xH200; 24 x 576 x 2 B) |
| KDA state | fixed size per request; ~54 MB/sequence under TP8 per AMD and SGLang posts |
| License | "Kimi K3 License"; model-as-a-service defined; separate agreement above $20M TTM affiliate revenue |

KDA is linear attention: no per-token cache, a fixed recurrent state per
request. Multi-turn prefix reuse needs that state checkpointed. vLLM
snapshots KDA state at intervals (VLLM_PREFIX_CACHE_RETENTION_INTERVAL,
e.g. every 32k tokens, plus prompt boundaries) and supports a
Marconi-style selective mode. Prefix caching is off by default for K3;
pass --enable-prefix-caching. Agentic KV offload is wired to Mooncake
with multi-tier partial-prefix reuse. KDA-state offload under load is
unverified.

Measured throughput (all Hopper, dequant-bound, no native MXFP4):
16xH200 SGLang TP16+EP16: 16.8 tok/s single stream, 147 tok/s aggregate
at 54 concurrent and still scaling, TTFT 0.37-0.54 s short prompts,
13 min cold start (warm shared filesystem). 32xH100: 5.8 tok/s single
stream. Blackwell: vLLM reports 111-370 tok/s single user (config and
speculative decoding dependent) and a Pareto range from 2k+ tokens per
GPU-second to 100+ tok/s per user, conditions unstated.

Node options (spot 2026-07-30; checked secure RunPod 2026-08-23 was
$7.89/GPU-hr, i.e. $46.1k/mo for 8xB300): 8xB300 2.3 TB is the only
single-node fit for the 1.56 TB checkpoint; 8xB200 (1.5 TB) does not fit.

Hosted list prices 2026-08: $3 / $0.30 cached / $15 per MTok (Moonshot,
Together, Fireworks); DeepInfra $2.85/$0.285/$14.25; OpenRouter twelve
upstream providers.

Sources: huggingface.co/moonshotai/Kimi-K3 (card and discussion 136),
vllm-project.github.io/2026/07/27/k3.html, vllm.ai/blog/2026-07-22
preview, spheron.network and ecorpit.com sizing posts, amd.com Kimi K3 on
Instinct article, lmsys.org SGLang day-0 post.
