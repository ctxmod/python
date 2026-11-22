# Teardown and Output Verification

Validation procedure for rented-session jobs — prove the run actually passed, sha-verify downloaded artifacts, and confirm zero billable leftovers.

A rented-session job produces three claims that must each be verified before the task reports success: the run passed, the artifacts are intact, and nothing billable remains. Tool exit codes prove none of these on their own — `colab exec` exits 0 even when cells fail, and a stopped pod can still carry a billing volume.

## 1. Prove the run passed

- **Script runs:** prefer `colab run`, whose exit code propagates CPython semantics. Treat exit 0 as necessary, not sufficient — check the script's own success marker in stdout.
- **Notebook runs:** `colab exec -f nb.ipynb` always exits 0. Open `<basename>_output.ipynb` and scan cell outputs for error blocks (tracebacks, `ERROR:` lines) before claiming green.
- **When a step fails anyway:** `colab log -s NAME -n 20` for structured events; `colab restart-kernel -s NAME` to clear a wedged kernel without losing the VM.

## 2. Verify artifacts

Compute the hash on the remote, download, recompute locally, compare:

```bash
echo "import hashlib,sys; print(hashlib.sha256(open('/content/out/embeddings.npy','rb').read()).hexdigest())" | colab exec -s job
colab download -s job /content/out/embeddings.npy ./out/embeddings.npy
shasum -a 256 ./out/embeddings.npy    # or sha256sum; must match the printed digest
```

Also sanity-check `run_meta.json` against the download: counts in the provenance block should match the artifact shapes (`embeddings.npy` rows == `counts.chunks`). A mismatch means a stage reran partially — do not ship the artifact.

## 3. Verify teardown — inventory must be empty

**Colab** — every session the job touched is gone:

```bash
colab sessions        # no entry for the job's session names
```

**RunPod** — both listings come back empty; persistent storage bills while a pod is stopped:

```bash
( set -a; . ./.env.local; set +a; runpodctl pod list )
( set -a; . ./.env.local; set +a; runpodctl network-volume list )
```

Terminate the pod **and** delete any volumes before this check — `pod list` empty alone is not enough. Any leftover entry is a failed teardown: fix it, then re-run the listing.

## Passing

All three checks green: output notebook clean (or propagated exit 0 with a success marker), digests match, both inventories empty. Only then report the run's results — with the `run_meta.json` numbers (timing, counts, cost against the cap) in the report.

## Evidence

Checklist assembled from the 2026-10/11 pilot: the exit-0 trap and volume-billing leftovers each happened once before becoming steps.

## Related

- [Operator Contract for Rented Compute](operator_contract.md)
- [colab CLI — Operational Shape](../../operations/reference/colab_cli.md)
- [runpodctl — Operational Shape](../../operations/reference/runpodctl.md)
- [Rented-GPU Job Shape](rented_gpu_job.md)
