# Qwen3.8 worker-model facts

Collected 2026-08-23. Family shipped 2026-08-12.

Qwen3.8-Max: 2.4T total, ~95B active (512 experts, 10 routed + 1
shared), 1M context, custom license, no published benchmarks. Self-host
is another K3-sized node (FP8 ~2.4 TB, int4-class on 8xB300); the ladder
routes it hosted-api at $2 / $0.25 cached / $6 per MTok.

Qwen3.8-27B: dense, natively multimodal, 262k native context (1M via
YaRN), Apache 2.0, ~28 GB at FP8. Qwen3.5-family hybrid (Gated DeltaNet
plus gated attention), so per-token KV is small. Vendor claims it beats
Qwen3.7-Plus on real-world coding; no published SWE-bench table. The
claim is unproven; management-plane#756 tests it against qwen3.6-27b on
real delegated briefs.

Worker economics (demand model in docs/design/cohort.md): dense 27B FP8
decode reads ~27 GB/step, so one H200 (~4.8 TB/s) supports batched
aggregate conservatively 3-6k output tok/s. A 50-member cohort's worker
tier (75% of agent-hours, ~30% decode duty) needs 1-2.7B output
tokens/month; one or two H200/B200 GPUs (~$3-6k/month) cover it. Planner
tier (25%) runs on the K3 node or hosted Qwen3.8-Max.

Tool-call reality: most agent wall-clock is tool execution, not decode.
Consequences: slots exceed active streams by ~3x at ~30% duty; felt
latency is TTFT after each tool result, so prefix-cache hit rate is the
user experience; tool sandboxes are CPU capacity someone must host.
