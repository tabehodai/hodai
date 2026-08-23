# ADR 004: Apache-2.0, operator as config

Date: 2026-08-23. Status: accepted.

tabehodai is open source under Apache-2.0 and self-hostable: anyone can run
their own instance and their own cohort. Consequences:

- No operator identity in code. One operator config (name, provider
  credentials, payment config, system-prompt preamble) defines an
  instance.
- GPU providers stay behind SkyPilot. Payments start with Stripe behind a
  thin interface in `control/`, not woven through it.
- Monitoring is bring-your-own: tabehodai ships /metrics endpoints, Grafana
  dashboard JSON, and alert rules as data. It ships no Prometheus,
  Grafana, or alert delivery.
- The license is decided before outside contributions exist, because
  relicensing later requires every contributor's consent.

The trust model follows: "audit the source" is the weak form, "run it
yourself" is the strong form, and a hosted cohort competes on operation.
