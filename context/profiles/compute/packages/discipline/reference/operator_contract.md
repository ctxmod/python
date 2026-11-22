# Operator Contract for Rented Compute

Team conventions for rented compute — secret handling for RUNPOD_API_KEY, who provisions, spend authorization with a budget cap, benchmark-first gating, and teardown verification.

Rented sessions bill real money per minute. The contract below is convention, not history: it applies to every task that touches a billable resource, whether the job is corpus embedding, batch generation with open models, or fine-tuning.

## Secrets

`RUNPOD_API_KEY` lives in the repo-root `.env.local` — operator-managed, gitignored, never committed. Load it per invocation with the subshell pattern:

```bash
( set -a; . ./.env.local; set +a; runpodctl pod list )
```

Rules that admit no exceptions:

- **Every** `runpodctl` invocation gets the subshell — no session-wide export, no `.env` sourcing outside the subshell.
- Never read, echo, trace, or log the key; never pass it as a command-line argument (arguments show up in `ps` and shell history).
- The subshell is the boundary: `set +a` before the tool call means the key exists only inside the parentheses.

## Provisioning roles

- **Colab sessions are provisioned by the operator on instruction.** Agents execute on existing sessions (`colab exec`, `colab run`, file transfer, install) — they do not allocate VMs on their own initiative.
- RunPod pod creation follows the task's explicit authorization; where a task names hardware or a template, re-verify it exists on the day (template drift — see [runpodctl](../../operations/reference/runpodctl.md)).

## Spend authorization

Before provisioning any billable resource:

1. **Confirm the task authorizes the spend.** A task that says "run X" authorizes X, not an open-ended exploration on bigger hardware.
2. **State a budget cap** in the run record before the first allocation — a number, in the billing unit of the platform, together with the projected duration that produced it.
3. **Benchmark first.** Run a small representative slice (e.g. 1% of the corpus or a fixed step count), measure throughput, extrapolate wall-time and cost.
4. **Hard gate.** STOP if the projection exceeds the cap — the working form is: stop if projected wall-time exceeds 2× the estimate. Report the measured numbers and ask for a new cap rather than continuing into one.

A benchmark that doesn't gate is decoration. The gate fires on the projection, before the money is spent, not on the invoice.

## Teardown is part of the job

A job is not done when its outputs are downloaded; it is done when the platform inventory is clean. `colab stop -s NAME` for sessions; terminate pod **and** delete volumes for RunPod — `pod list` and `network-volume list` must come back empty. Persistent storage bills while stopped. Verification procedure: [teardown and output verification](teardown_and_verification.md).

## Evidence

Conventions from team practice on rented-GPU work, 2026-10/11: benchmark-gated bulk jobs ran for minutes at cent-scale cost with zero runaway spend; the gating rule exists because extrapolation from a 1% slice has repeatedly differed from naive per-item estimates by more than 2×.

## Related

- [colab CLI — Operational Shape](../../operations/reference/colab_cli.md)
- [Rented-GPU Job Shape](rented_gpu_job.md)
