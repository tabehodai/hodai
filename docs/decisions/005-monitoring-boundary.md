# ADR 005: retain the gpu-runner metrics path

Date: 2026-08-23. Status: accepted.

The runner agent keeps writing pod scrape targets as file_sd JSON to the
directory the operator's Prometheus mounts (management-plane mounts
/var/lib/prometheus-file-sd/gpu-runner today). control and gateway expose
/metrics and become ordinary scrape jobs. Dashboards and alert rules live
in monitoring/ as provisioning data. tabehodai does not re-implement
management-plane; a self-hoster points any Prometheus at the same
endpoints.
