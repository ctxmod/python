# colab CLI — Operational Shape

Operational shape of the colab CLI — sessions as live kernels, exec/run discipline, accelerator gating, auth scopes, and TTY hazards for agent use.

The `colab` CLI drives Google Colab sessions from the terminal. A session is a live Jupyter kernel on a rented VM; `colab new` allocates a billable VM, `colab stop` releases it. Run `colab skill` and `colab readme` on the host for the authoritative text — flags drift between versions, and those two commands print it verbatim.

## Session model

- **Kernel state persists across `colab exec` calls in one session.** Each `exec` reattaches to the same kernel and only closes the websocket on exit. Imports, variables, and defined functions survive between separate `colab exec` commands — build state incrementally, don't re-install or re-import each call. `colab stop` and `colab restart-kernel` are what actually reset it.
- **Default working directory is `/content`.** Prefer absolute `/content/...` paths everywhere.
- **Always pass `-s NAME` to `colab new`.** An omitted name becomes a random 6-hex string, which makes every later command ambiguous.

## Provisioning and accelerator gating

```bash
colab new -s NAME              # CPU
colab new -s NAME --gpu T4     # T4, L4, G4, H100, A100
colab new -s NAME --tpu v6e1   # v5e1, v6e1
colab new -s NAME --high-mem   # high-RAM shape (Pro entitlement for most accelerators)
```

Two gates, both learned the hard way:

- **An unrecognized `--gpu` value silently falls back to A100**, which then usually fails the next step. Never trust a hand-typed accelerator string; copy it from `colab skill`.
- **A `400` on `colab new` with an accelerator means no quota/entitlement on this account.** Availability is tier-gated and most accounts only get CPU. Fall back to `--gpu T4` or omit the flag; don't retry the same tier.

## Execution

| Pattern | Command | Notes |
|---|---|---|
| Script | `colab exec -s NAME -f script.py` | Read locally, sent to the kernel — no upload step. Preferred. |
| Piped code | `echo 'print(1)' \| colab exec -s NAME` | Also `cat script.py \|` |
| Notebook | `colab exec -s NAME -f nb.ipynb` | Writes `<basename>_output.ipynb` next to the input |
| One-shot | `colab run [--gpu T4] [--keep] script.py args` | `new` + `exec` + `stop` in one command |
| Shell | `echo "cmd" \| colab console -s NAME` | tmux-wrapped; output carries terminal-control bytes — filter with `grep -a` |

`colab run` semantics an agent must know:

- **Exit codes propagate** — an uncaught exception or `sys.exit(N)` in the script makes `colab run` exit non-zero, with CPython semantics (`sys.exit("msg")` → 1).
- **Stream separation** — `[colab] ...` chatter goes to stderr, script output to stdout: `colab run job.py > out.txt` captures only the script's stdout.
- **`--keep` preserves the VM** after the run; without it the VM self-cleans even on script error.
- Works as a shebang: `#!/usr/bin/env -S colab run --gpu T4` turns a `chmod +x` script into a self-contained rent-run-clean job. A nonexistent script path exits non-zero **before** allocating a VM.
- Note the asymmetry: `colab run` propagates exit codes, **`colab exec` does not** — see [managed-environment drift](environment_drift.md).

## TTY hazards (agent rules)

Never run these interactively from an agent:

- `colab repl`, `colab console` — require a TTY; they accept piped stdin and exit on EOF, so only ever pipe.
- `colab auth`, `colab drivemount` — genuinely require a human at the terminal. Do not attempt, do not fake with `yes |`.

## Authentication

The CLI's global flag `--auth={adc,oauth2}` (default `adc`) must come **before** the subcommand: `colab --auth=adc new -s x`. ADC is the agent path; it must be re-minted with all four scopes, because gcloud rejects scope lists missing `cloud-platform` while the Colab backend needs `userinfo.email` and `colaboratory`:

```bash
gcloud auth application-default login \
  --scopes=openid,\
https://www.googleapis.com/auth/cloud-platform,\
https://www.googleapis.com/auth/userinfo.email,\
https://www.googleapis.com/auth/colaboratory
```

Diagnose 401/403 with `colab whoami` (prints active email, scopes, audience, expiry) or `colab sessions` (read-only). **`colab auth` is not CLI authentication** — it injects VM-side GCP credentials for in-notebook BigQuery/GCS calls. A CLI 401 is a scope problem, fixed by the `gcloud` command above, never by `colab auth`.

## Inspect, recover, stop

- `colab status -s NAME` — hardware, IDLE/BUSY, last execution. `colab log -s NAME -n 20` — structured events, invaluable when a step fails. `colab log -s NAME -o summary.ipynb` — export history as `.ipynb`/`.md`/`.txt`/`.jsonl`.
- `colab url -s NAME` — browser URL attaching the Colab web UI to the existing session (no new VM).
- "Session not found" / 404 on exec: the backend pruned the VM; the CLI cleans local state — re-check `colab sessions` and `colab new` again. Wedged kernel: `colab restart-kernel -s NAME` (keeps VM, resets kernel).
- **Always `colab stop -s NAME` when done.** Idle VMs burn compute units. Isolate parallel agent runs with `colab --config /tmp/agent.json ...` so local session state doesn't collide.

## Evidence

Grounded in the `colab skill` and `colab readme` output of `google-colab-cli` as installed on team hosts (captured verbatim 2026-11); the accelerator-fallback and tier-gating behaviors are from that skill's own gotcha notes plus operator experience in the 2026-10/11 pilot runs.

## Related

- [Operator Contract for Rented Compute](../../discipline/reference/operator_contract.md)
- [Rented-GPU Job Shape](../../discipline/reference/rented_gpu_job.md)
