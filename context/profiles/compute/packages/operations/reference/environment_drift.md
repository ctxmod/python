# Managed-Environment Drift

Managed-environment lessons from rented sessions — don't fight the preinstalled numeric stack, colab exec's green-lies exit code, upload pre-flight mkdir, PEP 668, and the economics of bulk inference.

Rented sessions run managed images you don't control, on catalogs that move. The lessons below generalize across both tools; each cost a real failure somewhere before it became a rule.

## Don't fight the preinstalled numeric stack

On managed images, the preinstalled numpy/torch stack is paired with the image's Python. Installing a different numpy version can break it — from the 2026-10/11 pilot: a numpy version pin hit a **py3.13 wheel gap** and broke a transformers stack that the image had shipped working.

Rule: on a managed image, **work with the preinstalled numeric stack**. Pin versions only in your own bring-your-own environments (venv, container). If a library genuinely needs a different numpy, use an isolated environment, not `pip install numpy==X` into the system interpreter. Record the image's actual versions in the run's provenance block (see [rented-GPU job shape](../../discipline/reference/rented_gpu_job.md)) so results remain interpretable later.

## `colab exec` exits 0 even when cells fail

`colab exec` streams a script or notebook into the kernel and exits 0 regardless of cell errors — the CLI's exit code says "delivery succeeded", not "execution succeeded". Green requires checking the output notebook: for notebook runs, scan `<basename>_output.ipynb` for error outputs before treating the run as passed; for script runs, require an explicit in-script success marker or use `colab run`, which does propagate exit codes. The [verification procedure](../../discipline/reference/teardown_and_verification.md) makes this a checklist step.

## `colab upload` 500s unless the parent exists

`colab upload -s NAME LOCAL REMOTE` fails with a 500 when the remote parent directory doesn't exist. Pre-flight the directory from the kernel:

```bash
echo "import pathlib; pathlib.Path('/content/out').mkdir(parents=True, exist_ok=True)" | colab exec -s NAME
colab upload -s NAME results.bin /content/out/results.bin
```

## PEP 668 and tokenizer dependencies

Externally-managed images (PEP 668) reject bare `pip install` into the system interpreter. Where the image's own tooling can't be used:

```bash
PIP_BREAK_SYSTEM_PACKAGES=1 pip install <pkg>
```

…as a last resort, sparingly. And bring your own tokenizer dependencies: stacks that depend on them often ship without them — `tiktoken`, `protobuf`, `sentencepiece` are the recurring three. Install them with the model's requirements or the first tokenize call fails.

## The economics that justify all of this

Bulk inference on rented hardware beats per-token serving APIs whenever the workload is large, batch, and latency-tolerant. Reference point from the pilot: **fp16 CLS-pooled encoder embeddings with chunked top-k retrieval, at millions-of-documents × thousands-of-queries scale — 7 minutes, ~$0.11 per run.** At that rate, embedding a corpus or batch-generating content with open models costs pennies where an API path costs dollars. The pattern is the [rented-GPU job shape](../../discipline/reference/rented_gpu_job.md): benchmark, embed or generate, pull artifacts down, tear down.

## Evidence

All five lessons from the 2026-10/11 rented-GPU pilot runs; each stated as experienced, with the general rule it produced. Nothing here is tool documentation — check `colab skill`, `colab readme`, and `runpodctl --help` for that.

## Related

- [colab CLI — Operational Shape](colab_cli.md)
- [runpodctl — Operational Shape](runpodctl.md)
