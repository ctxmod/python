# Rented-GPU Job Shape

Worked job shape for rented sessions — setup, gated benchmark, embed/generate with cached intermediate artifacts, run_meta.json provenance, download, teardown.

A generic, adaptable skeleton for one rented session doing bulk work: benchmark → embed (or generate) → retrieve/download → teardown. It assumes the [operator contract](operator_contract.md) — budget cap stated, benchmark gate armed, teardown verified. The example drives `colab`; the same phases map onto a RunPod pod with `runpodctl` swapped in per [its reference](../../operations/reference/runpodctl.md).

## Phase 0 — Setup

One script, one `exec`: verify hardware, record versions, create output directories (upload 500s otherwise), install only what the image lacks.

```python
# setup.py
import json, pathlib, torch, sys

out = pathlib.Path("/content/out"); out.mkdir(parents=True, exist_ok=True)
meta = {
    "gpu": torch.cuda.get_device_name(0) if torch.cuda.is_available() else "cpu",
    "torch": torch.__version__,
    "cuda": torch.version.cuda,
    "python": sys.version,
}
pathlib.Path("/content/out/host_meta.json").write_text(json.dumps(meta, indent=2))
print(json.dumps(meta))   # agent reads this from exec output and compares to the smoke-tested versions
```

```bash
colab exec -s job -f setup.py
colab install -s job -r requirements.txt    # image-lacking deps only — see environment-drift on the numeric stack
```

## Phase 1 — Gated benchmark

Run a small representative slice, measure throughput, project, and **stop the job in-script if the projection blows the cap**. The gate lives in the job, not in the operator's head:

```python
# bench.py — slice = a fixed, representative 1% of the workload
import time
t0 = time.time()
items = run_slice(workload_slice)                 # job-specific
rate = items / (time.time() - t0)                 # items/sec
projected_minutes = total_items / rate / 60
if projected_minutes > 2 * budget_minutes:        # hard gate: 2x estimate
    print(f"GATE: projected {projected_minutes:.0f} min > 2 x {budget_minutes} min"); sys.exit(2)
print(f"OK: {rate:.1f} items/s, projected {projected_minutes:.0f} min")
```

Exit code 2 stops the run; `colab run` propagates it. If the gate fires, report measured numbers and renegotiate the cap — don't continue.

## Phase 2 — Embed or generate, with cached intermediates

Write each stage's output as a checkpointed artifact so a crashed stage 3 doesn't redo stage 2. Embeddings and logits go to `.npy`; anything JSON-shaped goes through `gzip` first — they compress hard and the download is over a websocket.

```python
import numpy as np, gzip, json
emb = embed_all(corpus)                                  # fp16 halves VRAM and file size; CLS pooling if the model supports it
np.save("/content/out/embeddings.npy", emb.astype("float16"))
with gzip.open("/content/out/counts.json.gz", "wt") as f:
    json.dump({"chunks": len(emb), "dim": emb.shape[1]}, f)
```

Retrieval (top-k against a query set) or generation runs from the cached `.npy`, chunked to fit VRAM — chunk size goes into the provenance block.

## Phase 3 — Provenance block

Every run writes `run_meta.json` alongside the artifacts. It is the difference between "some embeddings" and a reproducible result:

```json
{
  "model": "org/model-name",
  "model_revision": "<commit-hash-pinned>",
  "host": {"gpu": "...", "torch": "...", "cuda": "...", "python": "..."},
  "timing": {"benchmark_s": 41.2, "total_s": 431.7},
  "counts": {"documents": 2103934, "queries": 5120, "chunks": 8415736},
  "cost": {"currency": "compute-units", "cap": 0.5, "projected": 0.18},
  "versions": {"transformers": "...", "numpy": "..."}
}
```

Pin the model revision (commit hash, not tag) — templates and model repos both drift.

## Phase 4 — Download and teardown

```bash
colab exec -s job -f makedirs.py            # or the mkdir from Phase 0, if not already done
colab download -s job /content/out/run_meta.json ./out/run_meta.json
colab download -s job /content/out/embeddings.npy ./out/embeddings.npy
colab stop -s job                            # before this, the run is not done
```

Then run the [teardown and output verification](teardown_and_verification.md): sha-verify downloaded artifacts, scan the output notebook for errors, confirm `colab sessions` is clean (or `pod list` / `network-volume list` on RunPod).

## Evidence

Shape distilled from the 2026-10/11 pilot bulk-embedding runs (millions × thousands scale, ~$0.11/run); every phase corresponds to a failure mode observed once — missing mkdir (upload 500), ungated benchmark (projection surprises), unpinned revision (irreproducible run), unverified teardown (billable leftovers).

## Related

- [colab CLI — Operational Shape](../../operations/reference/colab_cli.md)
