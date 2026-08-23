# Capacity model summary

Dated 2026-08-23. Full derivation and controls: docs/design/cohort.md
sections 5, 6, and the demand model correction. Adversarial checks:
docs/design/cohort-review.md section 2.

- K3 decode is memory-bandwidth bound on expert weights. Past batch ~60 a
  step touches nearly all experts; 8xB300 (~64 TB/s) gives a ~25 ms step
  floor, ~30-40 tok/s per stream until attention reads and expert
  all-to-all dominate. Knee guessed at 200-300 active streams; at ~30%
  decode duty that supports roughly 2-3x as many slots. Unmeasured.
- The KV pool, not the slot count, is the admission unit. ~500 GB pool at
  26 KB/token is ~19M resident tokens, and only with data-parallel
  attention plus expert parallelism; tensor-parallel attention replicates
  MLA KV across 8 ranks and divides capacity by 8.
- Known inconsistencies to resolve in any v2 (review items 7-9): 5 slots
  x 128k cap exceeds a 380k per-member budget; a 500 x 48k sweep needs
  24M resident tokens against the 19M pool; BF16-derived KV size vs FP8
  serving must be reconciled.
- Churn causes: pool overcommit (vLLM recompute-preemption; SGLang queues
  admission instead), prefill starving decode (chunked prefill, 4-8k
  tokens/step budget), single-key bursts (per-key prompt-token rate).
- Long contexts: 700k tokens is ~18 GB MLA KV; fine resident, ruinous on
  cache miss (30-90 s whole-node prefill per turn). KV tiering with
  write-through to host RAM (~0.3 s per 18 GB over PCIe Gen5) makes
  eviction a storage event. Contention under many simultaneous evictions
  is unmeasured.
- Demand: the target population holds multiple $200/mo subscriptions and
  exhausts them; consumption is $1.1k-4k/month at K3 list rates
  (~$1.40-2.30 per agent-hour). Monthly node token capacity (~42B output)
  exceeds 50-member demand (2-7.5B) by 5-20x; peak-hour concurrency is
  the binding constraint, not tokens.
