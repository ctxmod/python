# Project Setup

Project initialization workflow using uv, pyproject.toml configuration, and src layout creation.

Procedure for initializing a Python project with the modern toolchain: uv for package management, basedpyright for type checking, ruff for linting and formatting, pytest for testing, and Just for task running. The result is a project structure ready for development with all tools configured through `pyproject.toml`.

## Prerequisites

Gather before starting:

- **Project name**: PEP 503-compliant (lowercase, hyphens). Package name uses underscores.
- **Target Python version**: 3.12+ recommended for inline type parameter syntax and modern features.
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

uv creates `pyproject.toml`, `.python-version`, and a basic directory structure. The `--lib` flag additionally creates the `src/` layout and `py.typed` marker.

### 2. Create the src Layout

Application projects require manual src layout setup. Create the package directory structure:

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

Configure the package source location in `pyproject.toml`:

```toml
[tool.setuptools.packages.find]
where = ["src"]
```

[Project Structure](../../structure/reference/structure.md)

### 3. Configure pyproject.toml

Add development dependencies and tool configuration:

```toml
[project]
name = "project-name"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = []

[tool.uv]
dev-dependencies = [
    "basedpyright>=0.1.0",
    "ruff>=0.9.0",
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "pytest-cov>=6.0",
]

[tool.basedpyright]
strict = true
venvPath = "."
venv = ".venv"

[tool.ruff]
select = ["ALL"]
src = ["src"]

[tool.ruff.lint.isort]
known-first-party = ["project_name"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

[Python Toolchain](tooling.md)

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

[Type Checking Validation](../../typing/reference/type_checking.md)

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

[Contract-First Architecture](../../contracts/reference/contracts.md)

### 8. Pin the Codegen Tools in the Lockfile

Add the code generation tools as development dependencies so `uv.lock` records their exact versions:

```bash
uv add --dev redocly datamodel-code-generator buf
```

`uv add` updates the manifest's development group and the lockfile in one step — the manifest carries the requirement, the lockfile the exact version. Keep this as the one development-dependency surface; never hand-mirror it into extras or a second table.

### 9. Wire the Codegen Tasks

Add contract recipes to the justfile, invoking the tools through `uv run` so the pinned versions execute:

```just
contract-lint:
    uv run redocly lint contracts/api/openapi.yaml
    uv run buf lint contracts/domain-events

contract-generate:
    uv run redocly bundle contracts/api/openapi.yaml --output gen/api/bundle/openapi.yaml
    uv run datamodel-codegen --input gen/api/bundle/openapi.yaml --input-file-type openapi --output gen/api/python/models.py --output-model-type pydantic_v2.BaseModel
    uv run buf generate contracts/domain-events
```

Extend the `ci` recipe to depend on `contract-lint`, then run `just contract-generate` once to land the initial committed output under `gen/`. Do not add `gen/` to `.gitignore` — generated output is committed and drift-checked, never ignored.

### 10. Add the Drift Check

CI must regenerate every output and fail on any difference against the committed `gen/` tree:

```just
contract-drift-check:
    #!/usr/bin/env bash
    set -euo pipefail
    tmp=$(mktemp -d)
    uv run redocly bundle contracts/api/openapi.yaml --output "$tmp/openapi.yaml"
    diff "$tmp/openapi.yaml" gen/api/bundle/openapi.yaml
```

Repeat the regenerate-and-compare pair for every generation target — the Pydantic DTOs and the Domain Event bindings as well as the bundle. Any difference is a build failure, not a warning: a contributor who edits generated output by hand, or forgets to regenerate after a contract change, is caught here.

[Python Toolchain](tooling.md)

## Decision Support

### Application vs Library

| Criterion | Application | Library |
|-----------|-------------|---------|
| uv init flag | `uv init` | `uv init --lib` |
| Entry point | `src/project_name/__main__.py` | None |
| Build system | Optional | Required (hatchling, setuptools) |
| Version pinning | Exact (`uv.lock`) | Compatible ranges |
| py.typed | Optional | Required |

### Python Version Selection

| Version | Status | Key Features |
|---------|--------|-------------|
| 3.12+ | Recommended | Inline type params, f-string improvements |
| 3.11 | Acceptable | Exception groups, `tomllib` |
| 3.10 | Legacy support only | Pattern matching |

## Troubleshooting

**uv sync fails on platform-specific packages**: Use `uv sync --all-extras` or add platform markers to dependencies.

**basedpyright cannot find venv**: Verify `venvPath` and `venv` settings in `pyproject.toml` point to the correct location. Run `uv sync` first to create the virtual environment.

**ruff conflicts with formatter**: Add `COM812` and `ISC001` to the ignore list. These rules conflict with ruff's own formatter.

## Related

- [Python Packaging](packaging.md)
- [Module Creation](../../structure/reference/module_creation.md)
