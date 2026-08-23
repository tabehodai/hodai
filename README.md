# tabehodai

Unmetered agentic coding usage. tabehodai (食べ放題, "all you can eat") is
cohort infrastructure: a group pools a monthly payment, the pool rents GPU
capacity serving open-weight coding models, and each member gets an API
key with concurrent session slots. Anyone can host tabehodai, run their own
cohort, and set their own terms. Pronounced "tah-beh-hoe-DYE". The CLI
and package name stay `hodai`.

The system has four planes: `runner/` rents and serves GPU pods,
`gateway/` enforces keys, slots, and token budgets on the inference path,
`control/` handles members, payments, and key lifecycle, and `apps/`
carries the marketing site and the member portal. `kvstore/` holds the
encrypted per-key KV cache backend. `bench/` and `evals/` hold the
capacity and model evidence. Design history lives in `docs/`.

Operators configure an instance; the code carries no operator identity.
Monitoring is bring-your-own Prometheus: hodai ships metrics endpoints,
dashboards, and alert rules as data (`monitoring/`), not a stack.

Status (2026-08-23): pre-scaffold. The design draft and its adversarial
review are in `docs/design/`. The runner is being extracted from a private
repository. Nothing serves traffic yet.

License: Apache-2.0.
