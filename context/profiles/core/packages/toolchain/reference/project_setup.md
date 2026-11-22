# Project Setup

Project initialization workflow using uv, pyproject.toml configuration, and src layout creation.

Procedure for initializing a Python project with the modern toolchain: uv for package management, basedpyright for type checking, ruff for linting and formatting, pytest for testing, and Just for task running. The result is a project structure ready for development with all tools configured through `pyproject.toml`.

## Prerequisites

Gather before starting:

- **Project name**: PEP 503-compliant (lowercase, hyphens). Package name uses underscores.
- **Target Python version**: 3.14 for new projects (deferred annotations, template strings); 3.12 as the floor for inline type parameter syntax — see Python Version Selection below.
- **Project type**: Application (single deployment) or library (published to PyPI).
- **Dependencies**: Known third-party packages for initial setup.

## Setup Workflow

### 1. Initialize the Project

Create the project directory and initialize with uv:

```bash
uv init project-name
cd project-name
```

For library projects that will be published to PyPI:

```bash
uv init --lib project-name
```

Current uv (0.13) scaffolds a packaged src layout by default: `pyproject.toml`, `.python-version`, `README.md`, `src/project_name/__init__.py`, a `[project.scripts]` entry point, and a `[build-system]` on the `uv_build` backend with a bounded version range. `--lib` additionally adds the `py.typed` marker; `--no-package` produces the flat, unpackaged form, which this profile does not use. Older uv releases created a flat `main.py` for applications — if the scaffold differs, upgrade uv rather than hand-building the tree.

### 2. Complete the src Layout

Add the pieces the scaffold does not create — the tests tree, the justfile, and (for applications) the `py.typed` marker:

```
project-name/
  src/
    project_name/
      __init__.py
      py.typed
  tests/
    conftest.py
  pyproject.toml
  justfile
```

The `py.typed` marker file (PEP 561) signals to type checkers that this package ships inline type information. Create it as an empty file in the package root.

The `uv_build` backend discovers `src/<package_name>` by convention, so no package-finding configuration is needed. Only a project that keeps setuptools or hatchling as its backend configures package discovery in that backend's own table.

- [Project Structure](../../structure/reference/structure.md)

### 3. Configure pyproject.toml

Add development dependencies and tool configuration:

```toml
[project]
name = "project-name"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = []

[build-system]
requires = ["uv_build>=0.13.9,<0.14"]
build-backend = "uv_build"

[dependency-groups]
dev = [
    "basedpyright>=1.40",
    "ruff>=0.17",
    "pytest>=9.0",
    "pytest-asyncio>=1.0",
    "pytest-cov>=7.0",
]

[tool.basedpyright]
typeCheckingMode = "strict"
venvPath = "."
venv = ".venv"

[tool.ruff]
src = ["src"]

[tool.ruff.lint]
select = ["ALL"]
ignore = ["COM812", "D203", "D213"]

[tool.ruff.lint.isort]
known-first-party = ["project_name"]

[tool.pytest]
strict = true
asyncio_mode = "auto"
testpaths = ["tests"]
```

Prefer `uv add --dev <tool>` over editing the group by hand: it writes `[dependency-groups] dev` and the lockfile in one step. `[tool.uv] dev-dependencies` is the deprecated pre-PEP 735 spelling; uv warns on it.

- [Python Toolchain](tooling.md)

### 4. Create the Justfile

Set up task running with a justfile:

```just
bootstrap:
    uv sync

test *args:
    uv run pytest {{args}}

lint:
    uv run ruff check --fix .
    uv run ruff format .

typecheck:
    uv run basedpyright

ci: lint typecheck test
```

### 5. Bootstrap and Verify

Install dependencies and verify the toolchain:

```bash
just bootstrap
just ci
```

All three checks (lint, typecheck, test) should pass on an empty project. If basedpyright reports errors from third-party stubs, add per-module overrides rather than weakening strict mode.

- [Type Checking Validation](../../typing/reference/type_checking.md)

### 6. Configure Version Control

Create `.gitignore` with entries for `.venv/`, `__pycache__/`, `.pytest_cache/`, `.ruff_cache/`, `*.egg-info/`, `dist/`, `build/`, `.coverage`, and `htmlcov/`. Initialize the repository and commit the project skeleton.

### 7. Initialize the Contract Source

Skip steps 7-10 for a single-package project that exposes no external boundary; a plain src/ project needs no contract pipeline. For a project that serves an HTTP API or publishes Domain Events, create the contract source beside the code:

```
contracts/
  api/
    openapi.yaml          # small root; imports path items
    paths/
    schemas/
  domain-events/          # Protobuf sources, one file per definition
gen/                      # generated output; committed, never hand-edited
```

- [Contract-First Architecture](../../contracts/reference/contracts.md)

### 8. Pin the Codegen Tools in Their Lockfiles

Only datamodel-code-generator is a Python package; add it to the development group so `uv.lock` records its exact version:

```bash
uv add --dev datamodel-code-generator
```

Redocly CLI and buf are not PyPI packages — the PyPI name `buf` is an unrelated chemistry library, and `redocly` does not exist there. Pin them through the Node toolchain instead, so a lockfile still records the exact versions:

```bash
npm install --save-dev --save-exact @redocly/cli @bufbuild/buf
```

Each lock is the single surface for its ecosystem; never hand-mirror either into extras or a second table. The toolchain reference owns the per-tool distribution table ([Python Toolchain](tooling.md)).

### 9. Wire the Codegen Tasks

Add contract recipes to the justfile, invoking each tool through its lock-aware runner so the pinned versions execute:

```just
contract-lint:
    npx --no-install redocly lint contracts/api/openapi.yaml
    npx --no-install buf lint contracts/domain-events

contract-generate:
    npx --no-install redocly bundle contracts/api/openapi.yaml --output gen/api/bundle/openapi.yaml
    uv run datamodel-codegen --input gen/api/bundle/openapi.yaml --input-file-type openapi --output gen/api/python/models.py --output-model-type pydantic_v2.BaseModel
    npx --no-install buf generate contracts/domain-events
```

Extend the `ci` recipe to depend on `contract-lint`, then run `just contract-generate` once to land the initial committed output under `gen/`. Do not add `gen/` to `.gitignore` — generated output is committed and drift-checked, never ignored.

### 10. Add the Drift Check

CI must regenerate every output and fail on any difference against the committed `gen/` tree:

```just
contract-drift-check:
    #!/usr/bin/env bash
    set -euo pipefail
    tmp=$(mktemp -d)
    npx --no-install redocly bundle contracts/api/openapi.yaml --output "$tmp/openapi.yaml"
    diff "$tmp/openapi.yaml" gen/api/bundle/openapi.yaml
```

Repeat the regenerate-and-compare pair for every generation target — the Pydantic DTOs and the Domain Event bindings as well as the bundle. Any difference is a build failure, not a warning: a contributor who edits generated output by hand, or forgets to regenerate after a contract change, is caught here.

- [Python Toolchain](tooling.md)

## Decision Support

### Application vs Library

| Criterion | Application | Library |
|-----------|-------------|---------|
| uv init flag | `uv init` | `uv init --lib` |
| Entry point | `src/project_name/__main__.py` | None |
| Build system | `uv_build` (scaffolded) | Required (`uv_build`, or hatchling/setuptools for features it lacks) |
| Version pinning | Exact (`uv.lock`) | Compatible ranges |
| py.typed | Optional | Required |

### Python Version Selection

| Version | Status (2026-11) | Key Features |
|---------|------------------|-------------|
| 3.15 | Newest (3.15.0 released 2026-10-09) | `lazy import` (PEP 810), `frozendict` (PEP 814), UTF-8 default I/O encoding (PEP 686), `profiling` package with a sampling profiler (PEP 799) |
| 3.14 | Recommended default for new projects | Deferred annotation evaluation (PEP 649/749), template strings (PEP 750), `concurrent.interpreters` (PEP 734), free-threaded build officially supported (PEP 779), `compression.zstd` |
| 3.12–3.13 | Acceptable floor | Inline type params (3.12); experimental free-threaded build (3.13) |
| 3.11 | Legacy | Exception groups, `tomllib` |
| 3.10 | End of life (2026-10); dropped by FastAPI and SQLAlchemy 2.1 | Pattern matching |

Choose 3.14 rather than the newest release when a project's heavy dependencies (CUDA stacks, compiled extensions) lag a new minor by months; move to 3.15 once the lockfile resolves wheels for every target platform. The free-threaded build (`python3.14t`) is a separate interpreter and wheel tag — adopt it per workload only after every compiled dependency ships `cp314t` wheels.

## Troubleshooting

**uv sync fails on platform-specific packages**: Use `uv sync --all-extras` or add platform markers to dependencies.

**basedpyright cannot find venv**: Verify `venvPath` and `venv` settings in `pyproject.toml` point to the correct location. Run `uv sync` first to create the virtual environment.

**ruff conflicts with formatter**: Add `COM812` to the ignore list — current ruff names it as the one selected rule that conflicts with its formatter (`ISC001` no longer warns). With `select = ["ALL"]`, also ignore one rule of each mutually exclusive docstring pair (`D203`/`D211`, `D212`/`D213`), or ruff picks one and warns on every run.

**basedpyright silently runs in the default mode**: `strict = true` is rejected ("`strict` entry must contain an array") and `[[tool.basedpyright.overrides]]` is an unrecognized setting; both are mypy-isms. Use `typeCheckingMode = "strict"` and `executionEnvironments` (owned by [Type Checking Validation](../../typing/reference/type_checking.md)).

## Related

- [Python Packaging](packaging.md)
- [Module Creation](../../structure/reference/module_creation.md)

## Sources

- https://docs.astral.sh/uv/concepts/projects/init/ — default packaged src layout, `--lib`, `--no-package`, uv 0.13.9 (accessed 2026-11-22)
- https://docs.astral.sh/uv/concepts/projects/dependencies/#development-dependencies — `[dependency-groups]` and the legacy `tool.uv.dev-dependencies` (accessed 2026-11-22)
- https://docs.python.org/3.15/whatsnew/3.15.html — Python 3.15.0 (2026-10-09) features (accessed 2026-11-22)
- https://docs.python.org/3.14/whatsnew/3.14.html — Python 3.14 features: PEP 649/749, 750, 734, 779 (accessed 2026-11-22)
- https://devguide.python.org/versions/ — supported-version and end-of-life table (accessed 2026-11-22)
- https://docs.astral.sh/ruff/formatter/#conflicting-lint-rules — formatter-conflicting rules, ruff 0.17.6 (accessed 2026-11-22)
- https://pypi.org/project/buf/ — unrelated package owning the `buf` PyPI name (accessed 2026-11-22)
