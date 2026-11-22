---
paths:
  - "pyproject.toml"
  - "*/pyproject.toml"
  - "*/*/pyproject.toml"
  - "*/*/*/pyproject.toml"
  - "*/*/*/*/pyproject.toml"
  - "*/*/*/*/*/pyproject.toml"
  - "justfile"
  - "*/justfile"
  - "*/*/justfile"
  - "*/*/*/justfile"
  - "*/*/*/*/justfile"
  - "*/*/*/*/*/justfile"
---

# Python Toolchain

Use uv, basedpyright, ruff, and pytest as the standard toolchain.

Use uv for package management, basedpyright for type checking, ruff for linting and formatting, and pytest for testing. Configure all tools through `pyproject.toml`. Declare development tooling in one canonical surface — `[dependency-groups]` — and never mirror it by hand into a second list. In a repository with several packages, each member carries its own manifest and the workspace resolves to one lockfile.

## Rationale

The Astral-based toolchain (uv, ruff) delivers 10-100x performance improvements over legacy alternatives (pip, Poetry, Flake8, Black). A single `pyproject.toml` configures all tools, eliminating fragmented configuration across setup.cfg, tox.ini, and .flake8 files. This consolidation reduces onboarding friction and CI pipeline time while maintaining production stability. One dev surface matters for the same reason one manifest does: a hand-mirrored second list drifts silently, and the drift is invisible until a tool runs at a different revision in CI than on a developer machine.

- [Python Toolchain](../reference/tooling.md)
- [Python Packaging](../reference/packaging.md)

## Examples

**Correct**:
```toml
# pyproject.toml
[dependency-groups]
dev = [
    "basedpyright>=0.1.0",
    "ruff>=0.9.0",
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
]

[tool.basedpyright]
strict = true

[tool.ruff]
select = ["ALL"]
```

**Incorrect**:
```toml
# pyproject.toml using legacy tools
[tool.poetry.dev-dependencies]
mypy = "^1.0"
flake8 = "^7.0"
black = "^24.0"
isort = "^5.0"
```

## Exceptions

Projects with complex binary dependencies (CUDA, MKL) may use Pixi alongside uv for conda-forge package resolution. The rest of the toolchain remains unchanged. Heavy optional dependency sets may live in named extras that share the same lockfile rather than in the dev group.

## Tooling

Run `ruff check --fix .` for linting, `ruff format .` for formatting, and `basedpyright` for type checking. Use `just` or `uv run` for task orchestration. Contract-generation tools belong in the same dev surface and run through `uv run` so the lockfile — not the local environment — selects which revision executes; the toolchain reference owns the tool list and its pinning discipline.
