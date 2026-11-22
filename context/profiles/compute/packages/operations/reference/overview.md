# Rented Compute Overview

Reference profile for operating rented GPU/CPU compute sessions with the colab CLI and runpodctl — session discipline, spend gating, and teardown verification.

Rented-compute operations for Python workloads that exceed the workstation: bulk inference, batch generation with open models, and training-shaped jobs on GPUs or CPUs too expensive to own. The profile covers two operator tools — the `colab` CLI (Google Colab sessions) and `runpodctl` (RunPod pods) — and the operator contract that governs when and how billable resources may be allocated.

General-purpose by design: the patterns here are independent of any particular workload. Speech, music, and performance models with mixed licenses, corpus embedding, bulk content generation — anything cheaper run in bulk on rented hardware than per-token on a serving API.

## Purpose and scope

Applies whenever a task allocates or drives a rented session: provisioning (`colab new`, `runpodctl create pod`), execution (`colab exec`/`run`, pod exec), file transfer, and teardown. The colab CLI is documented by its own skill (`colab skill` on any host with the tool); that skill is the source of truth for its flags — this profile records the operational shape and the team's discipline around it.

The profile stops at the session boundary. Model authoring, experiment tracking, and training-loop code are owned by the ml profile; this profile specializes none of that, it moves the work to rented hardware and gets it (and its artifacts) back safely.

## Normative references

- **Tools** — [colab CLI operational shape](colab_cli.md) · [runpodctl operational shape](runpodctl.md)
- **Discipline** — [operator contract](../../discipline/reference/operator_contract.md) · [managed-environment drift](environment_drift.md)
- **Procedures** — [rented-GPU job shape](../../discipline/reference/rented_gpu_job.md) · [teardown and output verification](../../discipline/reference/teardown_and_verification.md)

## Placement

This profile is tool operations rather than Python doctrine: it sits in the python module because its workloads are Python jobs, and it may later move to the AI or infrastructure module.
