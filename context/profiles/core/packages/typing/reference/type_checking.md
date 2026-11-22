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

**Third-party stubs missing**: For packages without type stubs, configure per-module overrides instead of disabling strict mode globally.

```toml
[tool.basedpyright]
strict = true

[[tool.basedpyright.overrides]]
module = ["pandas.*", "scipy.*"]
reportMissingTypeStubs = false
```

[Type Safety](type_safety.md)

### 4. Handle Type Ignore Comments

When type errors represent genuine edge cases that cannot be resolved through better typing, use `# type: ignore[code]` with the specific diagnostic code and an explanation:

```python
# Acceptable: third-party library returns untyped data
result = legacy_lib.compute(data)  # type: ignore[no-any-return] -- legacy API, typed wrapper planned
```

Guidelines for `# type: ignore`:

| Use | Avoid |
|-----|-------|
| Third-party libraries without stubs | Suppressing errors in your own code |
| Dynamic metaprogramming patterns | Hiding type design problems |
| Specific diagnostic codes (`[code]`) | Bare `# type: ignore` without code |
| Accompanying explanation | Silent suppression |

### 5. Check Type Coverage

Assess the percentage of code with complete type annotations. basedpyright reports untyped functions and parameters in strict mode -- zero errors means full coverage.

Track type coverage over time:

| Metric | Pass | Warning | Fail |
|--------|------|---------|------|
| basedpyright errors | 0 | 1-5 | >5 |
| `# type: ignore` count | <5 per 1,000 LOC | 5-10 per 1,000 LOC | >10 per 1,000 LOC |
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
| 1 | New modules only | `strict = true`, override legacy modules |
| 2 | Critical modules | Remove overrides, fix errors |
| 3 | Full codebase | Global strict, zero overrides |

Use per-module overrides to exclude legacy code. Remove overrides as modules receive annotations. The goal is zero overrides.

## Troubleshooting

**Cannot resolve imports**: Verify `venvPath` and `venv` in `pyproject.toml`. Run `uv sync` to ensure dependencies are installed.

**Protocol satisfaction errors**: Implementing classes must match Protocol method signatures exactly, including parameter names and return types.

**Slow analysis**: For monorepos, run type checking per-package rather than globally.

[Python Toolchain](../../toolchain/reference/tooling.md)

## Related

- [Project Setup](../../toolchain/reference/project_setup.md)
