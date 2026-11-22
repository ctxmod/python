# Type Checking Validation

Type checking validation workflow using basedpyright strict mode with error resolution and coverage analysis.

Procedure for running basedpyright in strict mode, resolving type errors, and maintaining type coverage. Type checking validates that the codebase's type annotations are consistent and that static analysis detects no type safety violations. Run this procedure during development, before commits, and in CI.

## Prerequisites

- basedpyright installed as dev dependency (`uv add --dev basedpyright`)
- `pyproject.toml` configured with `[tool.basedpyright]` section
- Virtual environment created (`uv sync`)

## Validation Workflow

### 1. Run basedpyright

Execute the type checker against the project:

```bash
uv run basedpyright
```

basedpyright analyzes all Python files in the project, following import chains to verify type consistency. Output reports each error with file path, line number, and diagnostic code.

For targeted checking of specific files or directories:

```bash
uv run basedpyright src/mypackage/client.py
uv run basedpyright src/mypackage/
```

### 2. Analyze Error Categories

Group errors by diagnostic category to prioritize resolution:

| Category | Diagnostic Prefix | Priority | Approach |
|----------|-------------------|----------|----------|
| Missing annotations | `reportMissingParameterType` | High | Add type hints to all signatures |
| Incompatible types | `reportArgumentType`, `reportReturnType` | High | Fix type mismatches |
| Missing stubs | `reportMissingTypeStubs` | Medium | Install stubs or add override |
| Unknown imports | `reportMissingImports` | High | Fix import paths or install package |
| Unnecessary casts | `reportUnnecessaryCast` | Low | Remove redundant type assertions |
| Type narrowing | `reportUnnecessaryIsInstance` | Low | Simplify conditionals |

### 3. Resolve Common Errors

**Missing type annotations**: Add return types and parameter types to all function definitions. basedpyright strict mode requires explicit annotations rather than relying on inference.

```python
# Error: reportMissingParameterType
def process(data):  # Missing type
    return data.strip()

# Fixed
def process(data: str) -> str:
    return data.strip()
```

**Incompatible return types**: Verify that all code paths return the declared type. Use `None` union for optional returns.

```python
# Error: reportReturnType
def find(items: list[str], key: str) -> str:
    for item in items:
        if item == key:
            return item
    # Missing return path

# Fixed
def find(items: list[str], key: str) -> str | None:
    for item in items:
        if item == key:
            return item
    return None
```

**Third-party stubs missing**: First install the published stubs (`uv add --dev pandas-stubs`, `types-*` packages). For a package with none, wrap it in one typed adapter module and relax rules for that path only, instead of disabling strict mode globally. Pyright-family checkers have no per-module `overrides` table; path scoping is `executionEnvironments`:

```toml
[tool.basedpyright]
typeCheckingMode = "strict"
executionEnvironments = [
  { root = "src/project_name/adapters/legacy_lib", reportUnknownMemberType = "warning", reportUnknownVariableType = "warning" },
  { root = "src" },
]
```

The first matching `root` wins, so list narrow roots before broad ones. Check the configuration took effect: basedpyright prints `Config contains unrecognized setting` or `"strict" entry must contain an array` on stale keys and then runs in its default mode — a green run under a misspelled config is not a strict run.

- [Type Safety](type_safety.md)

### 4. Handle Suppression Comments

When type errors represent genuine edge cases that cannot be resolved through better typing, use `# pyright: ignore[<rule>]` naming the basedpyright rule, plus an explanation:

```python
# Acceptable: third-party library returns untyped data
result = legacy_lib.compute(data)  # pyright: ignore[reportUnknownVariableType] -- legacy API, typed wrapper planned
```

Do not use mypy error codes: under `typeCheckingMode = "strict"` basedpyright honors `# type: ignore`, but the bracketed mypy code means nothing to it, so `# type: ignore[no-any-return]` acts as a blanket ignore that suppresses every diagnostic on the line (verified with basedpyright 1.40.2; its `recommended` default mode ignores `# type: ignore` entirely). Make the discipline mechanical:

```toml
[tool.basedpyright]
typeCheckingMode = "strict"
enableTypeIgnoreComments = false              # only rule-named pyright: ignore comments suppress
reportIgnoreCommentWithoutRule = "error"      # a suppression must name its rule
reportUnnecessaryTypeIgnoreComment = "error"  # stale suppressions fail the run
```

Guidelines for suppression comments:

| Use | Avoid |
|-----|-------|
| Third-party libraries without stubs | Suppressing errors in your own code |
| Dynamic metaprogramming patterns | Hiding type design problems |
| Named rules (`# pyright: ignore[reportX]`) | `# type: ignore` with or without mypy codes |
| Accompanying explanation | Silent suppression |

### 5. Check Type Coverage

Assess the percentage of code with complete type annotations. basedpyright reports untyped functions and parameters in strict mode -- zero errors means full coverage.

Track type coverage over time:

| Metric | Pass | Warning | Fail |
|--------|------|---------|------|
| basedpyright errors | 0 | 1-5 | >5 |
| Suppression comment count | <5 per 1,000 LOC | 5-10 per 1,000 LOC | >10 per 1,000 LOC |
| Untyped function signatures | 0 | 1-3 | >3 |
| `Any` in public API | 0 | 1-2 | >2 |

### 6. Validate CI Integration

Confirm type checking runs in CI and fails the build on errors:

```yaml
# GitHub Actions
- name: Type check
  run: uv run basedpyright
```

Type checking runs before tests in the CI pipeline. A type error is a build failure, not a warning.

## Incremental Adoption

For existing codebases migrating to strict mode, adopt in phases:

| Phase | Scope | Configuration |
|-------|-------|---------------|
| 1 | New modules only | `typeCheckingMode = "strict"`, legacy paths relaxed via `executionEnvironments` |
| 2 | Critical modules | Remove relaxed environments, fix errors |
| 3 | Full codebase | Global strict, zero relaxed environments |

Use path-scoped `executionEnvironments` to relax legacy code. Remove each environment as its tree receives annotations. The goal is zero relaxations. For a large legacy tree, basedpyright's baseline (`basedpyright --writebaseline`, stored in `.basedpyright/baseline.json`) is the alternative: it records existing errors so only new ones fail, and shrinks as they are fixed.

## Troubleshooting

**Cannot resolve imports**: Verify `venvPath` and `venv` in `pyproject.toml`. Run `uv sync` to ensure dependencies are installed.

**Protocol satisfaction errors**: Implementing classes must match Protocol method signatures exactly, including parameter names and return types.

**Slow analysis**: For monorepos, run type checking per-package rather than globally.

- [Python Toolchain](../../toolchain/reference/tooling.md)

## Related

- [Project Setup](../../toolchain/reference/project_setup.md)

## Sources

- https://docs.basedpyright.com/latest/configuration/config-files/ — config keys, `executionEnvironments`, `reportIgnoreCommentWithoutRule`, basedpyright 1.40 (accessed 2026-11-22)
- https://docs.basedpyright.com/latest/configuration/comments/ — `# pyright: ignore[rule]` and `# type: ignore` handling (accessed 2026-11-22)
- https://docs.basedpyright.com/latest/benefits-over-pyright/baseline/ — baseline file for incremental adoption (accessed 2026-11-22)
