# runpodctl — Operational Shape

Operational shape of runpodctl — discover-and-verify against syntax drift, pod lifecycle, file transfer, template selection, and teardown that includes volumes.

`runpodctl` drives RunPod pods: rented GPU/CPU containers with optional persistent network storage. Unlike the colab CLI, there is no bundled skill — the tool's own help is the only trustworthy documentation, because **command syntax drifts between versions**.

## Discover and verify before use

Never write a `runpodctl` invocation from memory. On every new host or version:

```bash
runpodctl --help          # command inventory first
runpodctl <cmd> --help    # then the subcommand's actual flags
```

Confirm the exact subcommand name, flag spelling, and required arguments against the installed version before scripting around it. A plan written against a different `runpodctl` version is a guess, not a spec.

## Authentication

Every invocation that touches the API needs `RUNPOD_API_KEY`. On this team it lives in the repo-root `.env.local`, operator-managed and gitignored — see the [operator contract](../../discipline/reference/operator_contract.md) for the mandatory subshell loading pattern. Never export it into a long-lived shell, read it, echo it, trace it, or pass it as a command-line argument.

## Pod lifecycle

The create/attach/exec shape (verify names against `--help` on the host):

```bash
# Create — pick a template and GPU that currently exist (see Template selection)
( set -a; . ./.env.local; set +a; runpodctl create pod --templateName <t> --gpuType <g> ... )

# Exec — run commands in the running pod
( set -a; . ./.env.local; set +a; runpodctl pod exec <pod-id> -- cmd ... )

# File transfer
( set -a; . ./.env.local; set +a; runpodctl send <local>  )   # on the host side with a pod
( set -a; . ./.env.local; set +a; runpodctl receive <id>  )   # counterpart — verify actual pair in --help
```

Treat the pod as disposable and the artifacts as the only thing worth keeping: write outputs to a stable path, pull them down, then tear down.

## Template selection drift

Pod templates are a moving target. From the 2026-10/11 pilot: a plan specified **"RTX A5000"** hardware and a **"PyTorch 2.5.1+cu124"** template — within weeks, **both had vanished** from the catalog. The recovery pattern generalizes:

1. Substitute the **nearest shape** — comparable VRAM/compute class and a current template image.
2. **Verify the torch/CUDA versions inside the pod against what the job was smoke-tested on** before running the real workload (`python -c "import torch; print(torch.__version__, torch.cuda.version)"`).
3. If versions moved, re-run the benchmark gate — a different CUDA can change throughput enough to invalidate the projection.

A job spec that names exact templates or GPU SKUs is valid on the day it is written and must be re-verified on the day it is run.

## Teardown discipline

Stopping a pod is not enough: **persistent storage bills while the pod is stopped.** Termination is only complete when both listings come back empty:

```bash
( set -a; . ./.env.local; set +a; runpodctl pod list )
( set -a; . ./.env.local; set +a; runpodctl network-volume list )
```

Both commands must show nothing attributable to the job — terminate the pod **and** delete any volumes it created. See [teardown and output verification](../../discipline/reference/teardown_and_verification.md) for the full checklist.

## Evidence

Command shapes are illustrative and deliberately marked for re-verification; the template-drift and volume-billing lessons are from the 2026-10/11 rented-GPU pilot runs on team infrastructure.

## Related

- [colab CLI — Operational Shape](colab_cli.md)
