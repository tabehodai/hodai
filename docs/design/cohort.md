# GPU cohort: shared Kimi K3 cluster design (draft 2026-08-23)

> Reviewed 2026-08-23 by codex gpt-5.6-sol/high: verdict **kill / reshape**. See `gpu-cohort-design-review.md`. Factual corrections from that review are marked "Correction (review 2026-08-23)" below; the reshape decision is Eric's and is not applied here.

Status: design draft, not an implementation plan. No tasks are checked off
here. Facts and numbers dated 2026-08-23.

## 1. Goal

50 people each pre-authorize a $1,000 card payment. When the cohort fills,
everyone is charged. The pooled amount, about $50k/month, rents a GPU
cluster running the Kimi K3 open-weight model. Each member gets an API key
with a fixed number of concurrent sessions.

Buying in also grants access to the infrastructure-as-code source. Members
can audit what the operator can see and log.

## 2. Commerce

Card pre-authorizations expire in about 7 days. A signup window that runs
longer than that makes auth-and-hold unworkable: a card authorized on day 1
would lapse before a slow cohort fills on day 20.

Use Stripe SetupIntents instead. SetupIntents save the card without holding
funds, and the charge fires once the signup threshold is hit.

Expect 5-10% of charges to fail at settlement time (expired cards, declined
charges, insufficient funds). Two ways to cover this: collect roughly 55
signups and run a waitlist that backfills failures, or launch once 48 or
more people have signed up and backfill from there.

Margin is tight. At $1,000 x 50 members against a $50k cluster, margin is
zero after Stripe fees (about $1.5k) and failed cards. Three options close
the gap: price at $1,200 per member, raise the cohort to 60 members, or
target a $40k cluster instead of $50k.

The cohort needs a one-page terms document. It must cover: what happens if
a member's payment is late or fails, what happens if the cluster provider
reclaims capacity, refund conditions, and the mechanics of recurring
monthly charging. The terms must also disclose that the operator can read
plaintext prompts in the serving process. That disclosure is a legal
requirement, not an optional detail.

## 3. Model facts (Kimi K3)

Sources: the Kimi K3 Hugging Face model card, the vLLM day-0 blog post
(2026-07-27), a 16xH200 deployment report on Hugging Face discussion #136,
and a Spheron sizing post (2026-07-30).

| Property | Value |
|---|---|
| Total parameters | 2.8T |
| Active parameters per token | 104B |
| Experts | 16 of 896 active per token |
| Layers | 93 total: 69 KDA, 24 gated MLA, 1 dense |
| Context length | 1M tokens |
| Weight format | MXFP4, quantization-aware trained |
| Checkpoint size on disk | 1.56 TB (not the theoretical 1.4 TB) |

The 1.56 TB checkpoint does not fit on an 8xB200 node, which has 1.5 TB of
HBM.

Two attention mechanisms carry different memory costs. KDA (Kimi Delta
Attention) is linear attention with a fixed-size recurrent state per
request; the 69 KDA layers do not grow their per-request memory with
context length. Gated MLA carries per-token KV cache; the 24 MLA layers do
grow with context.

MLA KV cost is about 26 KB per token. This is derived from the H200 report:
a 7.95 GB KV pool held 308,608 tokens, and 7.95 GB / 308,608 tokens matches
24 layers x 576 latent dimensions x 2 bytes per value.

KDA state per request is fixed-size but its exact size is unpublished.
Budget hundreds of megabytes per live request until measured directly.

vLLM supports KDA state snapshots for prefix caching. Snapshots are taken
at intervals (for example every 32K tokens, controlled by
`VLLM_PREFIX_CACHE_RETENTION_INTERVAL`) and at prompt boundaries. vLLM also
offers a Marconi-style selective snapshot mode. Prefix caching is off by
default for K3 and requires the `--enable-prefix-caching` flag.

vLLM ties K3's agentic KV caching to Mooncake, which supports multi-tier
partial-prefix reuse.

Single-user decode throughput ranges from 111 to 370 tokens/second
depending on configuration and whether speculative decoding is enabled.
The throughput-versus-latency tradeoff runs along a Pareto curve: at the
throughput end, a GPU can do 2,000+ tokens per GPU-second; at the latency
end, a single user can get 100+ tokens/second. At the throughput end, 2,000
tokens per GPU-second across 8 GPUs is about 16,000 tokens/second
aggregate. Spread across 500 concurrent users that is about 30
tokens/second each. The conditions behind that number (context length,
batch composition) are not stated in the source.

## 4. Cluster options and cost

Spot prices are from 2026-07-30. On-demand or reserved pricing runs
20-50% higher than spot.

| Option | Nodes | HBM total | Headroom after weights | Spot cost/mo |
|---|---|---|---|---|
| 8xB300 | 1 | 2.3 TB | ~740 GB | ~$34k |
| 16xB200 | 2 | 3.1 TB | ~1.5 TB | ~$62k |
| 32xH100 | 4 | 2.6 TB | n/a | ~$68k |
| 32xH200 | 4 | 4.5 TB | n/a | ~$77k |

The 8xB300 node is the only option that fits a $50k budget. It has native
MXFP4 support.

Hopper GPUs (H100, H200) lack native MXFP4 support and are dequantization-
bound: weights must be converted to a format Hopper can run, which costs
throughput. Measured Hopper numbers: a 16xH200 setup running SGLang got
16.8 tokens/second on a single stream, 147 tokens/second aggregate at 54
concurrent requests (still scaling at that point), and a 13-minute cold
start. A 32xH100 setup got 5.8 tokens/second on a single stream.

The plan of record is one 8xB300 node, one replica, no failover.

## 5. Capacity model

Decode on this MoE model is bound by memory bandwidth on expert weights.
Past a batch size of about 60, each decode step touches nearly all 896
experts, so step cost stays flat as batch size grows further. On 8xB300,
aggregate memory bandwidth of about 64 TB/s gives a step-time floor of
about 25 ms. Per-stream decode should hold around 30-40 tokens/second until
two other costs start to dominate: attention reads, which scale as batch
size times context length times 26 KB per token, and the expert-parallel
all-to-all communication between GPUs. The concurrency level where these
costs bite (the "knee" in the curve) is guessed at 200-300 concurrent
requests and needs to be measured.

A session is a concurrent request slot with a per-key token rate. It is
not a guarantee of a specific throughput number.

Three things cause churn in the KV cache and hurt other members' latency:

1. **KV pool overcommit.** When the pool fills, something must be evicted
   and later re-prefilled from scratch. vLLM's default behavior under
   pressure is recompute-preemption: evict and recompute. SGLang instead
   queues new admissions rather than evicting live sessions.
2. **Prefill starving decode.** Agentic loops resend 30,000-60,000 token
   contexts on every turn. Chunked prefill, with a per-step prefill budget
   of about 4,000-8,000 tokens, keeps the prefill queue moving without
   stalling decode for other sessions.
3. **One key bursting.** Ten agents run in parallel under one API key,
   each sending a 60,000-token prompt: that is 600,000 prefill tokens from
   a single key in one burst.

Controls to hold the line:

- `max_total_tokens`, computed from the real KV pool size at boot. With
  data-parallel attention, roughly 500 GB of pool at 26 KB/token gives
  about 19M tokens of capacity. If attention is instead tensor-parallel,
  MLA KV is replicated across all 8 ranks and usable capacity divides by 8.
  Data-parallel attention plus expert parallelism is required to get the
  full 19M-token pool.
- A per-request context cap on the shared tier, 128k-256k tokens.
- A chunked-prefill budget per step.
- A per-key prompt-token rate limit plus a slot count.
- A per-key resident-token budget, about 380k tokens (19M tokens / 50
  members).
- A rule that a session with an in-flight request is never evicted from
  KV.

Slots per member come from the concurrency benchmark (section 11): publish
5 slots per member with a queue at first, and raise the number later if the
measured knee allows more.

## 6. Long contexts and KV tiering

A single 700,000-token session holds about 18 GB of MLA KV. Per-step cost
for that session is about 2 ms on one GPU, which is not a problem by
itself. The problem is capacity: 27 sessions of that size fill the entire
KV pool. When a long session's KV is evicted and then reused, the miss
costs 30-90 seconds of whole-node prefill on the next turn. That
recompute cost, not the steady-state per-step cost, is the real hazard for
long-context users.

The fix is KV tiering: write live KV out to a slower tier instead of
discarding it, using SGLang HiCache or vLLM with LMCache/Mooncake.

- Write-through to host RAM while a session is live. PCIe Gen5 bandwidth
  is about 50-60 GB/s, so writing 18 GB takes about 0.3 seconds.
- Reload from host RAM takes about 0.3-0.5 seconds; reload from NVMe takes
  1-3 seconds. Both are far cheaper than the 30-90 second recompute.

8xB300 nodes carry 2-4 TB of host RAM, enough to hold 100-200 idle
700,000-token sessions.

KDA's recurrent state must be checkpointed alongside the KV cache when a
session is tiered out. vLLM's interval-based snapshots cover this in
principle. Whether that offload path actually works needs to be verified
on the benchmark run (section 10).

Prefix reuse in this scheme is block-granular and prefix-only. Editing an
early message in a conversation invalidates every cached block after that
point.

## 7. Encrypted per-key KV store (v1 scope)

Scope decided by Eric.

The host-RAM tier stays plaintext. It sits in the same trust domain as
HBM, so encrypting it adds cost without a corresponding gain.

The disk tier is encrypted. A per-key data-encryption key is derived from
the member's API key via the gateway or a KMS. The engine gets a
session-scoped DEK and encrypts blocks through a pluggable storage
backend; both HiCache and LMCache already expose a backend interface for
this, and the plugin is roughly 200 lines of code.

Revoking a member's key makes their disk-tier KV blobs unreadable, since
the DEK derivation depends on the key.

Prefix-cache hashes are salted per tenant. This stops one member's cached
prefix from being hit, or timing-probed, by another member's requests.

This design protects members from each other. It does not protect a
member from the operator: the operator still controls the serving process
and can read plaintext in HBM and host RAM (see section 2's disclosure
requirement).

## 8. Gateway

A thin OpenAI-compatible gateway sits in front of the cluster. Candidate
bases: LiteLLM proxy or Bifrost. A Kubernetes-native alternative is Envoy
AI Gateway (kgateway) with the Gateway API Inference Extension.

Keys live in Postgres. Revoking a key is a row update.

Per key, the gateway enforces: a slot semaphore, a prompt-token rate
limit, a resident-token budget, and usage metering.

The gateway injects a fixed system-prompt preamble on every request. This
does two things. It gives every request an identical public prefix, which
is safe to share in the cache across tenants since it carries no member
content. It also nudges client behavior: compact conversation history
around 128-200k tokens, avoid resending whole files, cap tool-call
fan-out, send diffs rather than full files. Enforcement of limits stays in
the gateway; the preamble only shapes model behavior, it does not enforce
anything.

Analytics are collected per key without reading request or response
content: request count, prompt tokens, cached tokens (from vLLM's
`usage.prompt_tokens_details`), completion tokens, the distribution of
context lengths used, slots in use, queue wait time, time to first token,
decode tokens/second, resident KV bytes, and cache-hit ratio. An optional
salted prefix hash supports cache debugging. Content itself is never
logged.

## 9. Serving stack candidates

Eric runs k3s, so the stack should be Kubernetes-native where practical.

| Candidate | What it provides |
|---|---|
| llm-d | Kubernetes-native vLLM, KV-cache-aware routing via the Gateway API Inference Extension, prefill/decode disaggregation |
| NVIDIA Dynamo | KV-aware router plus KVBM tiering to CPU/NVMe/object storage; works with vLLM and SGLang |
| AIBrix | Kubernetes-native LLM serving control plane |
| vLLM production-stack | Helm chart plus router plus LMCache |
| Mooncake / LMCache | KV store backends used by the above |

Recommendation: k3s plus llm-d or Dynamo for routing and tiering, a
LiteLLM-class gateway for keys, quotas, the preamble, and analytics.
Custom code is limited to policy logic, the encrypted-store plugin,
analytics views, and the infrastructure-as-code itself.

## 10. Test rung

Kimi-Linear-48B-A3B-Instruct, from the same team, is a smaller stand-in.
It combines KDA and MLA in a 3:1 ratio, supports 1M context, and runs at
about 96 GB in bf16. It fits on one H200 (about $3.5-4/hr on RunPod
secure cloud, to be confirmed against the current catalog) or one B200
($6.79/hr, already a rung shape in the gpu-runner ladder).

Cost to run it: a weekend of testing at 15-20 pod-hours costs $60-80 on
H200. A week running continuously costs about $650. A network volume to
hold the weights costs about $7/month.

This rung exercises: KDA state checkpointing, prefix caching, tiered
offload, chunked-prefill interaction with decode, token-budget admission,
gateway slot and rate logic, and analytics. It does not exercise: the
896-expert all-to-all communication pattern, MXFP4 kernels on Blackwell,
or multi-GPU data-parallel attention scaling. Those require the real
cluster to test.

Qwen3-Next-80B-A3B, a Gated DeltaNet hybrid model, is a second stand-in.
It checks that the tooling built for this rung is not specific to Kimi's
architecture.

Adding this rung to gpu-runner is a ladder entry: `hf_repo
moonshotai/Kimi-Linear-48B-A3B-Instruct`, `gpu_shape H200:1`, `vllm_args
--enable-prefix-caching --max-model-len=262144` plus offload flags, and a
weight-staging job.

## 11. Benchmark day

This benchmark decides whether the cohort proceeds. It runs before any
member's card is charged.

Rent one 8xB300 node for the run, at roughly $46+/hr spot, about $1.1k for
24 hours. Get a quoted reserved price first: spot pricing will not hold
long enough to plan a cohort launch against it.

Sweep concurrency at 50, 100, 200, 300, and 500 concurrent requests, at
16k and 48k token contexts, using an agent-shaped prefill-to-decode token
ratio of about 20:1. Run with data-parallel attention and expert
parallelism, chunked prefill on, prefix caching on with KDA snapshots
enabled, and KV tiering on.

Target: 20 tokens/second or better per stream while prefill is running.
The largest concurrency level that holds this target is the admission cap.
Slots per member equal that cap divided by 50, with some headroom left
unassigned.

The benchmark must also verify that KDA state offload actually works, not
just that it is configured.

If aggregate throughput comes in under about 1,500 tokens/second, the
product this cluster can support is 25 members at full slots, or 50
members at fewer slots each. It is not 50 members at 10 slots each.

## 12. Relationship to gpu-runner

Decided by Eric, 2026-08-23.

gpu-runner stays the general "run this workload" primitive: the ladder and
catalog, SkyPilot over RunPod Secure Cloud, tailnet networking, TTL and
reaper, and the bootstrap image. The cohort product is built on top of it
and calls it to launch pods and clusters.

gpu-runner moves to its own repository before signups open. Cohort
members receive the infrastructure-as-code source, and management-plane
cannot be shared: it is the whole homelab, and `terraform/gpu-runner/terraform.tfstate` sits in the working tree
(gitignored, not committed). Correction (review 2026-08-23): the earlier
wording "checked in" was wrong; the state still has to move to a remote
backend before the directory is shared.

What moves to the new repository: `gpu-runner/` (catalog, ladder, routing,
groups), the `scripts/gpu-run` and `gpu-runner-*` scripts and
`gpu_runner_jobs.py`, `images/gpu-pod-bootstrap/`, `terraform/gpu-runner/`
(its state moves to a remote backend first), the `agent-skills/gpu-run`
skill, the gpu-runner Grafana dashboard and alert JSON, and the
gpu-pod-bootstrap Woodpecker pipeline.

management-plane keeps the systemd units, timers, and pipelines that
invoke gpu-runner, pinned to a specific gpu-runner version.

Note: `kimi-k3` already exists in the gpu-runner ladder today, as a
hosted-api rung with the reason "needs 2x8xH200 or 8xB300".

Coupling points found 2026-08-23. No Python outside `scripts/gpu*` imports
gpu-runner code, so the code boundary is already clean. What crosses the
boundary is config and process supervision:

- The systemd units hardcode `%h/git/management-plane/scripts/...` and the
  `scripts/.venv` interpreter. They repoint at a checkout of the new
  repository or a pip-installed console script.
- `monitoring/prometheus.yml` scrapes `gpu-runner-agent` and reads pod
  targets from `/var/lib/prometheus-file-sd/gpu-runner` (mounted in
  `docker-compose.yml`). The agent keeps writing file_sd JSON there; the
  directory is the contract.
- `scripts/load-tf-env.sh` sets the `TF_VAR_gpu_runner_tailscale_oauth_*`
  variables from the promoted `.env`, and `gpu-runner/routing.yaml` reads
  `GPU_RUNNER_HOSTED_<VENDOR>_API_KEY` from the same `.env`. Secret names
  and the Tailscale tag are part of the contract.
- `gpu-runner/groups.yaml` names the NATS `gpu_pool` KV bucket; that bucket
  is the contract.
- `.woodpecker/management-plane-validate.yml` runs `scripts/test-gpu-*`.
  Those tests move with the code and run in the new repository's CI.

Order of work: move the Terraform state to a remote backend; split the
paths above into `epsalmond/gpu-runner` with history (`git filter-repo
--path ...` or `git subtree split`); one management-plane PR deletes the
moved paths, repoints the units, and pins a gpu-runner tag; the
Kimi-Linear-48B rung lands as the new repository's first change. The
cohort repository starts after the 48B rung serves and the gateway
experiment has a shape.

## Demand model correction (2026-08-23)

The adversarial review anchored member demand on Cursor's published
average user ($60-$100/month, power users $200+). Eric's cohort is a
different population: engineers who hold multiple $200/month subscriptions,
exhaust all of them, and hit weekly limits. Recomputed from consumption:

- One agent decoding ~30 tokens/s at 50% duty for an hour produces ~54k
  output tokens and ~1.1M input tokens (20:1 mix). At K3 list rates with a
  90% cache hit that costs about $1.40 per agent-hour; at a 70% hit,
  about $2.30.
- A multi-subscription engineer (6-10 concurrent agents, 6-8 h/day,
  22 days) runs 800-1,750 agent-hours/month, which is $1.1k-$4k/month at
  API list rates. The review's 795M-token break-even bundle is one to two
  weeks of this profile, not an unreachable ceiling.
- 50 such members need 2-7.5B output tokens/month. The node's theoretical
  ceiling (16k tokens/s aggregate) is ~42B/month, so average utilization
  lands at 5-18%. A $1k seat nominally carries ~840M output tokens/month
  of continuous capacity, worth over $12k at list output pricing.

Consequences. Demand at $1k/seat is real for this population; the binding
constraint moves to peak concurrency: 50 members with 6-10 live agents in
overlapping work hours is 300-500 concurrent streams, on the unmeasured
knee. The benchmark must therefore run at 300-500 agent-shaped streams,
and time-of-day queueing policy is a first-class design item. Two review
findings survive untouched: supply risk (single node, $46k secure price,
unpaid ops, unverified KDA offload) and quality substitution (these
members are buying frontier models; a K3 seat only absorbs the agent-hours
where K3 is good enough, so the gateway routes hosted frontier and
self-hosted K3 side by side even in the cohort version).

## 13. Open questions

- Whether KDA state offload is actually supported on day 0, or needs a
  patch.
- The real reserved price for an 8xB300 node, not the spot estimate.
- Aggregate tokens/second at 200-500 concurrent requests, unmeasured until
  the benchmark runs.
- What legal entity runs this and what the terms document says.
- How members verify that the deployed image matches the published
  infrastructure-as-code, likely via reproducible image digests.
- Whether llm-d or Dynamo is the better router for this workload.

## Sources

- https://huggingface.co/moonshotai/Kimi-K3
- https://vllm-project.github.io/2026/07/27/k3.html
- https://huggingface.co/moonshotai/Kimi-K3/discussions/136
- https://www.spheron.network/blog/deploy-kimi-k3-gpu-cloud/
- https://huggingface.co/moonshotai/Kimi-Linear-48B-A3B-Instruct
- https://arxiv.org/pdf/2510.26692
- https://ecorpit.com/kimi-k3-self-host-gpu-cost-api-break-even-2026/
