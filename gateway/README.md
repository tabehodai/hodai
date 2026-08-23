# gateway

Inference data plane. OpenAI-compatible proxy over a LiteLLM-class base:
per-key slot semaphore, prompt-token rate, resident-token budget, cohort
preamble injection, metadata-only usage metering (never content). Reads
keys and quotas written by control. Design: docs/design/cohort.md
sections 5 and 8.
