# runner

gpu-runner, pending extraction from management-plane with history
(git filter-repo over gpu-runner/, scripts/gpu-run*, scripts/gpu_runner_*,
images/gpu-pod-bootstrap/, terraform/gpu-runner/, agent-skills/gpu-run).
Rents GPU pods via SkyPilot (RunPod Secure Cloud first), serves ladder
rungs with vLLM, joins pods to the operator's tailnet, reaps on TTL.
Terraform state moves to a remote backend before extraction.
