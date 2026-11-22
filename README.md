# ctxmod/python

A [ctxmod](https://github.com/ludvigalden/ctxmod) context module for
contract-first Python: the uv, basedpyright, ruff and pytest toolchain,
src layout, strict typing and model boundaries, typed errors with RFC 9457
problem details, configuration, three-layer observability, and testing.

Module path: `github.com/ctxmod/python/context` (requires
`github.com/ctxmod/engineering/context`).

## Profiles

| Profile | Selected | Covers |
| --- | --- | --- |
| `core` | by default | Architecture overview and glossary, toolchain, structure, typing, contracts, errors, configuration, observability, testing |
| `api` | opt-in | FastAPI services: routing, async SQLAlchemy, request/response validation, API testing |
| `data` | opt-in | Data pipelines: durable jobs, migrations, processing and streaming, validation, database principles |
| `ml` | opt-in | ML training: experiments, training loops and checkpoints, PyTorch, evaluation |
| `compute` | opt-in | Rented GPU/CPU sessions: colab CLI, runpodctl, operator contract, teardown |

Every opt-in profile extends `core`. Select one by naming it, for example
`ctxmod render --profile github.com/ctxmod/python/context/profiles/api`,
or list it in a consuming module's profile `extends`.
