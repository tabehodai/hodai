# GPU cohort design: adversarial review (codex gpt-5.6-sol/high, 2026-08-23)

Review of `gpu-cohort-design.md` as drafted 2026-08-23, before any corrections. Prices checked live 2026-08-23.

Verdict: **kill** the 50-member, $1,000/month, single-node service. Reshape the proxy and benchmark work into a hosted-API club or short-lived compute experiment.

Market and technical claims below were checked live on 2026-08-23 unless marked as an inference. I used no unverified memory. Prices exclude tax.

## 1. Are we rebuilding a business that already exists?

Yes. The plan combines four mature products: hosted inference, a coding subscription, a GPU rental, and an API management proxy. The member-financed structure is novel. The service is not.

### Hosted K3 already has a competitive market

The comparison below uses the draft's 20 input tokens per output token and a 90% input cache-hit rate. The last column is the cost of 100M combined tokens at that mix.

| Offering | Price per MTok, input/cached/output | $/100M | What it adds over the cohort | What the cohort adds |
|---|---:|---:|---|---|
| [Moonshot API](https://www.kimi.com/blog/kimi-k3) | $3/$0.30/$15 | $126 | First-party operation and a claimed 90%+ coding cache-hit rate | No trust in Moonshot's application layer; custom policy |
| [Together](https://www.together.ai/models/kimi-k3) | $3/$0.30/$15 | $126 | Serverless, dedicated, and provisioned deployment; 1M context | Published infrastructure source and member-controlled policy |
| [Fireworks](https://fireworks.ai/models/fireworks/kimi-k3) | $3/$0.30/$15 | $126 | Zero retention by default, fast/priority/US tiers, dedicated endpoints, fine-tuning | A specific operator relationship and fixed share of one node |
| [DeepInfra](https://deepinfra.com/) | $2.85/$0.285/$14.25 | $119 | Zero retention, SOC 2, ISO 27001, US data centers, 1M context | Source visibility and custom admission rules |
| [OpenRouter](https://openrouter.ai/moonshotai/kimi-k3-20260715) | $2.60/$0.29/$13 | $112 | Twelve K3 providers, automatic failover, routing, one API | Dedicated capacity if the cohort can actually deliver it |
| [Groq](https://console.groq.com/docs/models) | No K3 listing | n/a | Very fast managed inference for other open models | K3 itself, but Groq is not a current K3 comparator |

OpenRouter's current upstream list includes Moonshot, Together, Fireworks, DeepInfra, Baseten, DigitalOcean, Modal, Chutes, Phala, Morph, Sail Research, and Wafer. This is already an inference reseller with failover. The cohort replaces twelve failure domains with one.

### Coding subscriptions are much cheaper

| Offering | Member price | What it includes that the cohort does not |
|---|---:|---|
| [Kimi Code](https://www.kimi.com/code/docs/en/) | ¥99/month for K3 256K; ¥199 (about $28) for 1M coding context; ¥699 top tier | K3 client, vendor operations, automatic upgrades, shared product features |
| [Claude Max](https://support.anthropic.com/en/articles/11049762-choosing-a-claude-ai-plan) | $100 or $200/month | Claude Code, multiple models, priority access, vendor support |
| [ChatGPT Pro with Codex](https://help.openai.com/en/articles/9793128) | $100 or $200/month | Codex, other agent products, model upgrades, managed execution surfaces |
| [Cursor](https://cursor.com/docs/models-and-pricing) | $20/$60/$200; $20/$70/$400 of third-party usage included | Editor, multiple providers, cloud agents, completions, usage UI, support |

These subscriptions have quotas and rate limits. They are not unlimited capacity. That does not help the cohort much: its five slots, resident-token budget, and queue are also limits, and its price is five times the common $200 power-user tier.

Cursor's own usage data says daily agent users typically consume $60-$100/month and power users often consume $200+/month. That is the best checked public proxy for realistic individual coding demand. A $1,000 seat assumes usage far beyond an ordinary power user.

### Direct GPU and shared-compute substitutes

- [RunPod](https://www.runpod.io/product/cloud-gpus) rents B300s directly. The checked secure price is $7.89/GPU-hour. Eight GPUs running for 730 hours cost $46,078/month, or $922 per cohort member before payment fees, storage, support, or labor. Community pricing is cheaper but has weaker supply and trust characteristics. Spot capacity can be evicted.
- [Fireworks](https://fireworks.ai/pricing) lists on-demand B300 deployment at $12/GPU-hour through 2026-08-31 and $15 afterward. That is $70,080 or $87,600 per 8-GPU month, but it includes a managed serving product rather than raw rental.
- Together, Fireworks, DeepInfra, Hugging Face, and other inference vendors sell dedicated endpoints or clusters. They already sell the single-tenant version of this idea, with support and contractual accountability.
- [co/core](https://www.cocore.dev/) is an actual member-owned inference experiment. It meters jobs and returns compute revenue to contributing members. It does not offer K3, a fixed seat, or a production service commitment.
- I found no mature retail cooperative offering this exact K3 club. That is not evidence of a moat. It is evidence that cloud rental, multi-tenant inference, payment risk, and volunteer operations fit together poorly.

### API break-even

Let `U`, `H`, and `O` be millions of uncached-input, cached-input, and output tokens. Hosted cost is:

`$3U + $0.30H + $15O`

At 20 input tokens per output token:

| Input cache hit | Combined tokens bought by $1,000 | Input/output split |
|---:|---:|---:|
| 0% | 280M | 266.7M input, 13.3M output |
| 90% | 795M | 757.6M input, 37.9M output |
| 100% | 1.0B | 952.4M input, 47.6M output |

[Fireworks reports](https://fireworks.ai/blog/kimik3-fable) roughly 1.3M tokens for a K3 SWE task. Under the same 20:1 assumption, $1,000 buys about 215 such tasks with no cache or 612 at 90% cache. That is 7 or 20 substantial tasks every day of a 30-day month. Most individual developers will not cross the break-even. The cohort also loses any unused monthly allocation.

This break-even is generous to self-hosting. It assumes the member can consume the node share at useful latency, the cluster is never down, cache accounting matches the API, and the operator's unpaid work costs zero.

## 2. Is it viable?

Not as a paid service in its current shape.

### The margin is missing

At current checked prices:

| Monthly item | Amount |
|---|---:|
| Gross receipts | $50,000 |
| [Stripe domestic-card fee](https://stripe.com/pricing), 2.9% + $0.30 x 50 | -$1,465 |
| RunPod secure 8xB300, $7.89 x 8 x 730 | -$46,078 |
| Remainder | **$2,457** |

That $2,457 must cover storage, networking, database, monitoring, accounting, tax administration, failed collections, refunds, disputes, security work, support, and operator labor. International cards add 1.5%; currency conversion adds another 1%. Stripe does not return processing fees on refunds. A launch at 48 members leaves about $516 before every omitted cost.

The draft cannot simultaneously claim a $34k spot node and zero margin against a $50k cluster. If the node really costs $34k, there is about $14.5k after domestic Stripe fees. If the cluster really costs $50k, the operation starts $1,465 negative. Current secure rental lands between those stories and leaves no serious reserve.

Spot is the wrong capacity class for a paid interactive service. Reclaim loses every live request and all HBM/host-RAM state. Reserved capacity reduces reclaim risk but raises price and usually adds a commitment. A quote is not enough; launch needs a binding capacity reservation and explicit service terms.

### One node is one failure domain

The GPU baseboard, NVLink fabric, host, local storage, serving process, provider, and region can each take out every member. Kubernetes on one node does not create failover. Upgrades also require an outage.

The cited [13-minute startup](https://huggingface.co/moonshotai/Kimi-K3/discussions/136) was measured on 16 H200s with weights on a shared filesystem and a warm filesystem cache. It is not a measured B300 spot-recovery time. Reacquiring eight B300s and making 1.56 TB of weights local can take much longer. There is no spare capacity during recovery.

### The capacity promise is speculative

- The [111-370 tokens/s figures](https://vllm-project.github.io/2026/07/27/k3.html) are batch-size-1 results on GB300. The 2,000+ tokens/GPU-second point is a normalized Pareto result with unstated traffic conditions. Neither proves 500 simultaneous B300 streams.
- Multiplying 2,000 by eight gives a 16,000 output-token/s ceiling. Five hundred streams at 30 tokens/s consume 15,000 of it before prefill, KDA work, attention, communication, skew, or failures. At the benchmark target of 20 tokens/s and a 20:1 prefill:decode ratio, 500 streams require 10,000 output tokens/s plus 200,000 input tokens/s.
- The cited H200 deployment measured 147 aggregate tokens/s at 54 requests. It used different hardware and was still scaling, so it is not a B300 forecast. It does show how far the draft extrapolates beyond observed multi-request data.
- vLLM's `--max-num-seqs 512` is a scheduler setting, not evidence that 512 useful streams fit.
- Five published slots times 50 members is 250 slots, not 500. The draft's 500-request sweep is therefore not the launch product, while its later warning discusses 10 slots per member.
- A 19M-token pool gives 380K resident tokens per member or 76K per published slot. Five 128K contexts require 640K. The advertised slot count and context cap cannot both be used fully.
- Five hundred 48K prompts need 24M resident tokens, above the stated 19M pool. The benchmark conflicts with the capacity model unless FP8 KV changes the pool, but the draft derives capacity from BF16 measurements and never reconciles the two.
- The product advertises K3's 1M context, then caps the shared tier at 128K-256K. Current K3 APIs and Kimi Code already host 1M. The cohort removes its supposed differentiator.
- Data-parallel attention plus expert parallelism is mandatory for the 19M aggregate pool. The draft has no cited 8xB300 result for that topology. If KV is tensor-parallel and replicated, its own estimate divides usable capacity by eight.

The single fatal assumption is that one 8xB300 node can sustain at least 250 agent-shaped streams at useful latency under prefill load. If that is false, adding price, terms, encryption, or more members cannot fix the product inside the budget.

### KV tiering is a research item, not a service feature

[vLLM documents](https://vllm-project.github.io/2026/07/27/k3.html) hybrid KDA/MLA transfer and prefix retention, but that does not prove the draft's interchangeable SGLang HiCache, LMCache, Mooncake, encrypted disk, and idle-session path. The exact topology needs destructive pressure tests on the real engine version.

The transfer math assumes sustained 50-60 GB/s PCIe, suitable NUMA placement, enough host memory, and fast local NVMe. One 700K session may copy quickly in isolation. Many simultaneous evictions contend for the same links and storage. A 2 TB host can barely hold 100 claimed 18 GB sessions before OS, page cache, serving overhead, and weight staging.

The encryption design is not 200 lines of policy glue. It needs authenticated encryption, nonce management, key separation, rotation, crash consistency, deletion, backup rules, and recovery tests. Deriving the data key from the API key couples authentication compromise to cache decryption. Rotation either loses cached state or requires an unstated rewrap path. Revoking a database row does not prove every blob and key copy is unrecoverable.

Tenant-salted prefix hashes also conflict with the claimed cross-tenant shared public preamble. The design needs two cache namespaces or it gets only one of those properties.

### Legal and trust exposure is understated

A one-page terms document is not enough for a $600k/year multi-tenant service. The operator needs an entity, tax treatment, recurring-payment consent, refund and outage rules, privacy notice, data-processing terms, breach response, acceptable-use and abuse response, sanctions/export review, insurance decisions, and counsel. The categorical claim that one specific disclosure is “a legal requirement” is jurisdiction-dependent and unsupported.

The [K3 license](https://huggingface.co/moonshotai/Kimi-K3/blob/main/LICENSE) expressly defines model-as-a-service. This cohort is below its $20M trailing-12-month affiliate-revenue threshold for a separate Moonshot agreement, but the license still needs to be included and reviewed.

Publishing source does not prove what is deployed. The draft admits image attestation is unresolved. The operator and cloud provider can read plaintext prompts, host RAM, crash dumps, and live process memory. The privacy gain is only freedom from a specific inference vendor. Fireworks and DeepInfra already advertise zero retention and carry compliance programs. A member must trust this one operator more than those organizations for the plan to improve privacy.

One person would own 24x7 incidents, security patches, capacity, upgrades, abuse, billing, disputes, support, and member communication. The budget pays that person zero and provides no backup operator.

## 3. What is the actual value proposition?

The honest value is a club for a small group that trusts one operator, wants to experiment with serving internals, and accepts outages. It is not cheaper K3 for a normal developer.

- **Privacy from one vendor:** real but narrow. Prompts avoid Moonshot/Fireworks/Together application systems. They remain visible to the operator and infrastructure provider.
- **Predictable cost:** weak. A spend cap on hosted APIs is equally predictable. The cohort makes cost fixed by making unused capacity expire and peak demand queue.
- **1M context:** none. Multiple checked providers already host it, and the shared tier proposes a 256K maximum.
- **Owning the router and policy stack:** real. It does not require owning inference. The same proxy can route hosted K3 providers and enforce keys, budgets, analytics, privacy preferences, and failover.
- **A club:** real if members value governance and experimentation. That is not a $1,000/month compute case.

No current value proposition justifies the financial and operational risk for 50 unrelated buyers. A defensible exception would require a known group already spending more than the dedicated endpoint cost on K3, a shared trust requirement, and written acceptance of downtime.

## 4. Minimum honest reshaping

1. Publish the policy proxy as open source. Put Moonshot, Fireworks, Together, DeepInfra, and OpenRouter behind it. Let members bring keys or pay exact usage plus a disclosed operating fee. This preserves routing, quotas, analytics, and policy ownership without a $50k fixed burn.
2. Run an hourly compute club for 5-10 named participants. Pre-fund specific 8xB300 benchmark windows. Promise no SLA, persistent KV, or monthly availability. At the current secure rate, one 24-hour run is about $1,515 total.
3. If isolation matters, negotiate a dedicated endpoint and sell seats only if the provider contract permits resale. Pass through the invoice. Require a reserve and pay the operator. This moves hardware recovery and serving support to a vendor.
4. Collect 60-90 days of hosted-API telemetry first. Reconsider self-hosting only if real aggregate K3 spend stays above the full dedicated cost and the burst distribution fits one node.
5. Benchmark the exact paid topology before any signup. Test 250 streams, cold and warm cache, 128K/256K/700K contexts, the observed prefill ratio, P50/P95/P99 queue and token latency, member fairness, tier eviction, key rotation, engine crash, node loss, and full recovery. A 16K/48K throughput sweep is insufficient.
6. If a monthly service survives that evidence, price from a binding reserved quote. Hold at least one month of refund exposure, add payment costs, storage, insurance/accounting, and paid on-call labor, then divide by the measured admission cap. Do not choose member count and price first.

Fewer members reduce contention but raise the seat price. More members improve accounting but worsen the unproven capacity problem. Price cannot rescue a failed node-capacity assumption.

## 5. Factual errors and internal inconsistencies

1. “Margin is zero after Stripe fees” is wrong. A literal $50k cluster makes margin negative. A $34k cluster leaves about $14.5k after domestic Stripe fees. The draft uses both cost bases.
2. Launching at 48 members does not fund a $50k cluster. At current secure B300 pricing it leaves about $516 after Stripe and compute.
3. The $34k spot estimate is not current secure or reserved pricing. Current checked RunPod secure pricing is about $46.1k/month for eight B300s.
4. The 13-minute cold start is an H200 warm-filesystem measurement, not a B300 reclaim-recovery measurement.
5. “93 total: 69 KDA, 24 gated MLA, 1 dense” reads as 94 layers. The dense/MoE distinction describes the feed-forward block inside the same 93 layers.
6. KDA state size is no longer simply unpublished. [AMD](https://www.amd.com/en/developer/resources/technical-articles/2026/kimi-k3-on-amd-instinct-gpus.html) and [SGLang](https://www.lmsys.org/blog/2026-07-27-kimi-k3-day0-support) publish enough detail to estimate about 54 MB per sequence under TP8, with different placement under other parallel layouts.
7. The draft derives 26 KB/token from BF16 KV, while the cited vLLM reproduction uses FP8 KV. Capacity, quality, and backend support must use one explicit format.
8. The 500-request, 48K benchmark requires 24M resident tokens, but the stated pool holds 19M.
9. Five slots at the minimum 128K cap require 640K tokens per member, but the member budget is 380K.
10. K3's 1M context is advertised as value but denied by the proposed 128K-256K shared cap. The test rung also sets 262K.
11. Per-tenant salted prefix hashes prevent the cross-tenant public-prefix hit claimed in the next section unless the implementation adds a separate shared namespace.
12. Key revocation does not necessarily make blobs unreadable. That depends on whether old keys, derived keys, backups, crash dumps, or KMS versions remain.
13. “Roughly 200 lines” for the encrypted store has no evidence and omits key lifecycle, recovery, integrity, concurrency, and deletion.
14. The 50-60 GB/s PCIe and 1-3 second NVMe timings are isolated arithmetic, not measurements of the proposed B300 host under concurrent load.
15. A SetupIntent stores a payment method but does not reserve funds or validate that the later $1,000 charge will succeed. The 5-10% failure rate is asserted without cohort data.
16. The current repo's `gpu-runner/routing.yaml` labels a route `kimi-k3` but still records $0.60/$2.50 and 262K, which are stale K2-era values. The ladder also says “594GB weight loads,” conflicting with this draft's 1.56 TB K3 checkpoint.
17. The draft says Terraform state is “checked in.” `terraform/gpu-runner/terraform.tfstate` is present locally but ignored; `git ls-files terraform/gpu-runner` does not include it.
18. Shared infrastructure source provides auditability only if deployment attestation exists. The draft sells audit access in section 1 while leaving proof of the deployed image open in section 13.
19. The benchmark decides capacity but not commercial viability. It has no reserved quote, uptime test, recovery objective, support model, legal review, or refund reserve criterion.
20. The plan gives no model-refresh or exit rule. A recurring cohort can be obsolete one model release after members commit.

---

Post-review note (2026-08-23, Eric): section 1's break-even anchored on
Cursor's average-user spend. The intended cohort is multi-subscription
power users who exhaust several $200/month plans; recomputed demand for
that population is $1.1k-$4k/month at K3 list rates. See "Demand model
correction" in `gpu-cohort-design.md`. The supply-side findings stand.
