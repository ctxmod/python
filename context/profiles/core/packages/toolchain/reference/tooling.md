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

**When to consider Pixi**: Data science projects requiring CUDA, MKL, or GPU libraries from conda-forge. Pixi supports both conda and PyPI ecosystems. All other projects use uv exclusively.

Monorepo composition — uv workspaces, per-package manifests, sibling dependency edges, and script-tool packaging — is owned by the packaging reference ([Python Packaging](packaging.md)).

## Type Checking: basedpyright

basedpyright extends Microsoft's Pyright with 10+ additional diagnostic rules, aggressive type inference, and built-in LSP support for non-VSCode editors. It installs from PyPI without Node.js and runs 3-5x faster than mypy on large codebases through lazy evaluation architecture.

Strict mode catches type errors that mypy's conservative inference misses. When basedpyright reports zero errors in strict mode, the codebase has stronger type safety guarantees than mypy strict can provide.

**Future migration**: Astral's ty type checker (formerly Red Knot) projects 10-60x speed over mypy and pyright. It remains in beta for 2026. basedpyright is the production choice today; monitor ty for a stable release that will become the long-term standard.

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

[tool.uv]
dev-dependencies = [
    "basedpyright>=0.1.0",
    "ruff>=0.9.0",
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
]

[tool.basedpyright]
strict = true

[tool.ruff]
select = ["ALL"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

This replaces the fragmented configuration previously spread across `setup.cfg`, `tox.ini`, `.flake8`, `mypy.ini`, and `.isort.cfg`.

## Migration Priorities

| Legacy Tool | Replacement | Migration Effort |
|-------------|-------------|-----------------|
| pip/pip-tools | uv | Low -- `uv pip sync` is 100% pip-compatible |
| Poetry | uv | Low -- export requirements, `uv init`, `uv add` |
| PDM | uv | Low -- similar workflow |
| mypy | basedpyright | Low -- change config section, update CI |
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

| Tool | Role | Replaces |
|------|------|----------|
| Redocly CLI | OpenAPI contract lint + bundle: validates the modular contract source and resolves it into one portable, reviewed artifact | Hand-merged flat OpenAPI files and generic YAML linting |
| datamodel-code-generator | Pydantic v2 DTO generation from the bundled OpenAPI specification | Handwritten request/response models duplicating the contract |
| buf | Domain Event binding generation: lint, breaking-change detection, and Python code generation from the Protobuf sources | Hand-rolled message serialization and unenforced schema evolution |

Generation commands, generalized (`uvx` runs an ephemeral tool environment; when a tool is pinned in the lockfile as a development dependency, `uv run <tool>` executes the pinned version):

```bash
uvx redocly lint contracts/api/openapi.yaml
uvx redocly bundle contracts/api/openapi.yaml --output gen/api/bundle/openapi.yaml

uvx datamodel-codegen \
  --input gen/api/bundle/openapi.yaml \
  --input-file-type openapi \
  --output gen/api/python/models.py \
  --output-model-type pydantic_v2.BaseModel

uvx buf generate contracts/domain-events
```

CI executes these same pinned recipes. What the CI check must prove — that regeneration into a temporary location differs in nothing from the committed `gen/` tree, and that the resolved bundle is retained as a reviewed build artifact — is generated-artifact authority, owned by the contract-first reference ([Contract-First Architecture](../../contracts/reference/contracts.md)); this section applies that discipline to concrete tools rather than restating it. Tool selection and pinning are what this section owns. The browser-client doctrine (the typescript module) owns the browser-client generation from the same bundle.

## Evidence

Performance benchmarks from Astral (2024-2025), ThoughtWorks Technology Radar adoption signals (April 2025), production usage by FastAPI, Pandas, Hugging Face, LangChain, and enterprise teams. Community consensus from Python developer surveys and tooling discussions confirms this stack as the 2026 standard.

## Related

- KISS (engineering module, `principles` profile: `design/reference/kiss.md`)
- SSOT (engineering module, `principles` profile: `design/reference/ssot.md`)
