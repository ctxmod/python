# Python Toolchain

Modern Python toolchain rationale covering uv, basedpyright, ruff, pytest, and Just.

The recommended Python toolchain consolidates around Rust-based tools from Astral, delivering 10-100x performance improvements over legacy alternatives while maintaining production stability. Each tool was selected based on performance benchmarks, ecosystem maturity, and long-term viability.

## Tool Selection Rationale

| Tool | Role | Replaces | Performance Gain |
|------|------|----------|------------------|
| uv | Package management | pip, Poetry, PDM | 10-100x faster resolution and install |
| basedpyright | Type checking | mypy | 3-5x faster analysis, stricter inference |
| ruff | Linting + formatting | Flake8 + Black + isort + Pylint | 10-1000x faster across operations |
| pytest | Testing | unittest, nose2 | Standard; 800+ plugin ecosystem |
| Just | Task running | Make, tox | Simpler syntax, cross-platform |

## Package Management: uv

uv reached ThoughtWorks "Adopt" rating in April 2025. Lock time measures ~2 seconds versus 43 seconds for Poetry and 67 seconds for PDM. Install speeds exceed pip by 10-80x. Beyond speed, uv provides Python version management (replacing pyenv), ephemeral environments via `uvx` (replacing pipx), workspace support for monorepos, and built-in dependency confusion protection.

A single `uv.lock` file ensures cross-platform reproducibility. CI pipeline savings of 30-60 seconds per run translate to measurable cloud cost reduction for teams running hundreds of daily builds.

Current uv surfaces this toolchain relies on (verified against uv 0.13.9, 2026-11):

| Surface | Current form | Replaces |
|---------|--------------|----------|
| Development dependencies | `[dependency-groups] dev = [...]` (PEP 735); `uv add --dev` writes here | `[tool.uv] dev-dependencies`, which uv now warns is deprecated and slated for removal |
| Project scaffold | `uv init` creates a packaged src layout (`src/<name>/__init__.py`) with the `uv_build` backend; `--no-package` gives the flat, unpackaged form; `--lib` adds `py.typed` | Hand-built src trees and a separately configured setuptools backend |
| Build backend | `[build-system] requires = ["uv_build>=X.Y,<X.(Y+1)"]`, `build-backend = "uv_build"` — bound the upper version, as uv's docs direct | setuptools/hatchling for pure-Python packages (still valid where their features are needed) |
| Builds | `uv build` (sdist + wheel), `uv publish` | `python -m build`, twine |
| Interoperable lock export | `uv export --format pylock.toml` (PEP 751), or `requirements.txt` / CycloneDX for other consumers | Hand-maintained requirements files |

`uv.lock` stays the source of truth; `pylock.toml` is the standardized export for installers and auditors that do not read `uv.lock`, generated, never hand-edited.

**When to consider Pixi**: Data science projects requiring CUDA, MKL, or GPU libraries from conda-forge. Pixi supports both conda and PyPI ecosystems. All other projects use uv exclusively.

Monorepo composition — uv workspaces, per-package manifests, sibling dependency edges, and script-tool packaging — is owned by the packaging reference ([Python Packaging](packaging.md)).

## Type Checking: basedpyright

basedpyright extends Microsoft's Pyright with 10+ additional diagnostic rules, aggressive type inference, and built-in LSP support for non-VSCode editors. It installs from PyPI without a system Node.js (the wheel pulls a bundled Node runtime) and runs 3-5x faster than mypy on large codebases through lazy evaluation architecture.

Strict mode catches type errors that mypy's conservative inference misses. When basedpyright reports zero errors in strict mode, the codebase has stronger type safety guarantees than mypy strict can provide. The mode is selected with `typeCheckingMode = "strict"` — the config schema and its traps are owned by the type-checking reference ([Type Checking Validation](../../typing/reference/type_checking.md)).

### The type-checker landscape (2026-11)

| Checker | Status | Position here |
|---------|--------|---------------|
| basedpyright (1.40.x) | Stable, Pyright-derived | The configured checker |
| Pyrefly (Meta, 1.x) | Stable since its 1.0; Google adopted it for its monorepo (2026-10), and it type-checks PyTorch, NumPy and Polars | The credible alternative; evaluate it where its speed or IDE behavior is decisive |
| ty (Astral, 0.0.x) | Still labelled beta by its maintainers | Monitor; do not gate CI on it yet |
| mypy (2.x) | Stable, conservative inference | Legacy target of the migration table below |

Run exactly one checker as the CI gate. A second checker in advisory mode is a deliberate, time-boxed evaluation, never a permanent second gate: two checkers disagree on edge semantics, and a suite that must satisfy both encodes the intersection of their bugs.

## Linting and Formatting: ruff

Ruff implements 800+ lint rules covering Flake8 plugins, isort, and more. It formats code with >99% Black compatibility. On the TensorFlow codebase, Ruff runs 16x faster than Flake8 (cached) and 110x faster than Pylint. It provides auto-fix for most rules and native `pyproject.toml` configuration.

Using Ruff eliminates the need to install, configure, or maintain Flake8, Black, isort, Pylint, or any of their plugins. One tool handles all linting and formatting concerns.

## Testing: pytest

pytest remains the dominant Python testing framework with no viable replacement. Its fixture system, 800+ plugins (pytest-cov, pytest-xdist, pytest-mock, pytest-benchmark), automatic test discovery, and readable assertion syntax make it the only reasonable choice. pytest-asyncio adds async test support with `@pytest.mark.asyncio` and scoped event loop management.

## Task Running: Just

Just provides a simpler alternative to Make with spaces instead of tabs, no `.PHONY` declarations, built-in parameter support, automatic `.env` loading, and cross-platform compatibility. For Python-only projects, `uv run` serves as a lighter alternative when complex task orchestration is unnecessary.

## Unified Configuration

All tools configure through `pyproject.toml`, establishing a single configuration source:

```toml
[project]
name = "project-name"
requires-python = ">=3.12"

[dependency-groups]
dev = [
    "basedpyright>=1.40",
    "ruff>=0.17",
    "pytest>=9.0",
    "pytest-asyncio>=1.0",
]

[tool.basedpyright]
typeCheckingMode = "strict"

[tool.ruff.lint]
select = ["ALL"]
ignore = ["COM812", "D203", "D213"]  # COM812 conflicts with the formatter; D203/D211 and D212/D213 are mutually exclusive pairs

[tool.pytest]
strict = true
asyncio_mode = "auto"
```

Each key above is the current spelling, and each stale spelling fails differently: `[tool.uv] dev-dependencies` warns and is slated for removal; top-level `[tool.ruff] select` warns and must move under `[tool.ruff.lint]`; basedpyright's `strict = true` is rejected (`strict` takes a list of paths) and falls back to the default mode silently in all but a log line; `[tool.pytest]` native TOML needs pytest 9 — older suites keep `[tool.pytest.ini_options]` with string values.

This replaces the fragmented configuration previously spread across `setup.cfg`, `tox.ini`, `.flake8`, `mypy.ini`, and `.isort.cfg`.

## Migration Priorities

| Legacy Tool | Replacement | Migration Effort |
|-------------|-------------|-----------------|
| pip/pip-tools | uv | Low -- `uv pip sync` is 100% pip-compatible |
| Poetry | uv | Low -- export requirements, `uv init`, `uv add` |
| PDM | uv | Low -- similar workflow |
| mypy | basedpyright | Low -- change config section, update CI; per-module `[[tool.mypy.overrides]]` become basedpyright `executionEnvironments` |
| Flake8 + Black + isort | ruff | Low -- drop-in rule compatibility |
| Pylint | ruff | Medium -- some deep analysis rules lack equivalents |
| Make | Just | Low -- rewrite recipes in simpler syntax |

Migrate new projects immediately. Prioritize existing projects by CI pipeline frequency -- high-frequency pipelines capture the largest time savings from the performance improvements.

## Development Workflow

The standard workflow integrates all tools through a justfile:

```just
bootstrap:
    uv sync

test:
    uv run pytest tests/

lint:
    uv run ruff check --fix .
    uv run ruff format .

typecheck:
    uv run basedpyright

ci: lint typecheck test
```

Setup requires `just bootstrap`. Daily development uses `just test`, `just lint`, and `just typecheck`. CI runs `just ci` for the complete verification pipeline. Package management uses `uv add <package>` and `uv remove <package>` directly.

## Language and Runtime Boundaries

Minimize language and runtime boundaries in the backend. Iteration speed comes less from any single tool choice than from having one contract-generation pipeline, one deployment model, and one set of domain DTOs: a second backend language adds a new inter-process contract, and every item on its cost list — failure handling, correlation, observability, deployment, versioning, and duplicate types across the boundary — recurs for the life of the system.

Before splitting a component out into a separate service in a different language, require a reason that survives that cost. An in-process typed component in the existing language, invoking shared services through generated types, is almost always faster to iterate than a new boundary. The deliberate exception is the browser: a frontend language boundary is standard, and it is crossed only by generated clients from the same contract source, never by hand-duplicated types (the contract-first reference owns the generation discipline).

### Time Sinks

- Do not carve a component into a second-language service for aesthetics or for a single library — the boundary cost outweighs the library every time.
- Do not hand-duplicate contract types in a second language; generate them from the single contract source.

Transport, realtime-infrastructure, and capability-admission time sinks are owned by the cloud-platform and browser-client doctrine; they are link-only here.

Bottom line: the smallest number of language/runtime boundaries, the highest reuse of generated contracts, and validation at every untrusted boundary is the strongest combination of development velocity and reliability available at the stack-selection layer.

## Contract Codegen Toolchain

Contract linting, bundling, and code generation run under the same pinned-toolchain discipline as the core tools: every codegen tool is a declared dependency with the lockfile recording its exact version, invoked through `uv run` or `uvx` — never as an unpinned global.

| Tool | Role | Distribution | Replaces |
|------|------|--------------|----------|
| Redocly CLI | OpenAPI contract lint + bundle: validates the modular contract source and resolves it into one portable, reviewed artifact | npm package `@redocly/cli` — **not on PyPI** | Hand-merged flat OpenAPI files and generic YAML linting |
| datamodel-code-generator | Pydantic v2 DTO generation from the bundled OpenAPI specification | PyPI `datamodel-code-generator` — the one tool here uv can lock | Handwritten request/response models duplicating the contract |
| buf | Domain Event binding generation: lint, breaking-change detection, and Python code generation from the Protobuf sources | Go binary: npm `@bufbuild/buf`, release binaries, or package managers — **not on PyPI** | Hand-rolled message serialization and unenforced schema evolution |

**Name-confusion hazard.** The PyPI name `buf` belongs to an unrelated chemistry library, and `redocly` is not a PyPI project at all. `uv add --dev redocly buf` therefore fails or — worse — locks a stranger's package under the expected name. Only datamodel-code-generator goes into `[dependency-groups]`; Redocly and buf are pinned by their own ecosystem's lock (a `package.json` + lockfile beside the contracts, run with `npx --no-install`), or by an exact-version binary with a recorded checksum. The pinning discipline is the same — an exact recorded version, never an unpinned global — only the lock that records it differs.

Generation commands, generalized (`uv run` executes the uv-locked version; `npx --no-install` executes the version the Node lockfile records):

```bash
npx --no-install redocly lint contracts/api/openapi.yaml
npx --no-install redocly bundle contracts/api/openapi.yaml --output gen/api/bundle/openapi.yaml

uv run datamodel-codegen \
  --input gen/api/bundle/openapi.yaml \
  --input-file-type openapi \
  --output gen/api/python/models.py \
  --output-model-type pydantic_v2.BaseModel

npx --no-install buf generate contracts/domain-events
```

CI executes these same pinned recipes. What the CI check must prove — that regeneration into a temporary location differs in nothing from the committed `gen/` tree, and that the resolved bundle is retained as a reviewed build artifact — is generated-artifact authority, owned by the contract-first reference ([Contract-First Architecture](../../contracts/reference/contracts.md)); this section applies that discipline to concrete tools rather than restating it. Tool selection and pinning are what this section owns. The browser-client doctrine (the typescript module) owns the browser-client generation from the same bundle.

## Evidence

Performance benchmarks from Astral (2024-2025), ThoughtWorks Technology Radar adoption signals (April 2025), production usage by FastAPI, Pandas, Hugging Face, LangChain, and enterprise teams. The configuration spellings above were re-verified 2026-11-22 by running uv 0.13.9, ruff 0.17.6, basedpyright 1.40.2 and pytest 9.2.0 against probe projects.

## Related

- KISS (engineering module, `principles` profile: `design/reference/kiss.md`)
- SSOT (engineering module, `principles` profile: `design/reference/ssot.md`)

## Sources

- https://docs.astral.sh/uv/concepts/projects/init/ — `uv init` src layout and `uv_build` default, uv 0.13.9 (accessed 2026-11-22)
- https://docs.astral.sh/uv/concepts/build-backend/ — `uv_build` backend and its bounded `requires` range, uv 0.13.9 (accessed 2026-11-22)
- https://peps.python.org/pep-0735/ — dependency groups (accessed 2026-11-22)
- https://peps.python.org/pep-0751/ — `pylock.toml` lock format, Final (accessed 2026-11-22)
- https://docs.astral.sh/ruff/settings/ — `lint.*` settings section; top-level linter keys deprecated, ruff 0.17.6 (accessed 2026-11-22)
- https://docs.basedpyright.com/latest/configuration/config-files/ — `typeCheckingMode`, `strict` path list, `executionEnvironments`, basedpyright 1.40 (accessed 2026-11-22)
- https://github.com/astral-sh/ty — ty status "beta", 0.0.108 (accessed 2026-11-22)
- https://github.com/facebook/pyrefly — Pyrefly 1.x, adopters including Google (2026-10) (accessed 2026-11-22)
- https://docs.pytest.org/en/stable/reference/customize.html — `[tool.pytest]` native TOML since pytest 9.0 (accessed 2026-11-22)
- https://pypi.org/project/buf/ — the unrelated chemistry package that owns the `buf` name on PyPI (accessed 2026-11-22)
- https://www.npmjs.com/package/@redocly/cli and https://www.npmjs.com/package/@bufbuild/buf — Redocly CLI 2.82.0 and buf 1.75.0 distribution (accessed 2026-11-22)
