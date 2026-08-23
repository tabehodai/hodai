# ADR 002: monorepo, gpu-runner folds in

Date: 2026-08-23. Status: accepted.

hodai is one repository. gpu-runner moves in as `runner/` (extracted from
management-plane with history via git filter-repo on its path prefixes)
instead of getting its own repository, superseding the two-repo plan in
docs/design/cohort.md section 12. Members and self-hosters audit one repo,
and runner plus gateway changes land atomically. Cost: Eric's personal
"run this workload" use of gpu-runner now depends on this repository.

management-plane keeps the systemd units, Prometheus scrape config,
promoted secret names, and the NATS gpu_pool bucket, repointed at
~/git/hodai. Those interfaces are the operator contract, not part of
hodai.
