# Project Structure

Project structure principles covering src layout, public interface design, and package organization.

Organize Python projects with the `src/` layout, explicit public interfaces through `__init__.py`, and private implementation behind underscore-prefixed modules. Structure packages around single cohesive domains, splitting when conceptual boundaries emerge rather than when arbitrary size limits are reached.

## The src Layout

The `src/` layout places all package source under a `src/` directory, preventing accidental imports from the project root during testing. Without it, `import mypackage` might resolve to the local directory rather than the installed package, masking import errors and missing dependencies that surface only in production.

```
project/
  src/
    my_package/
      __init__.py
      client.py
      config.py
      _internal/
        resolver.py
        cache.py
  tests/
    conftest.py
    test_client.py
    test_config.py
  pyproject.toml
  justfile
```

uv and modern build backends (hatchling, setuptools with `pyproject.toml`) support the src layout natively. Configure the package source in `pyproject.toml`:

```toml
[tool.setuptools.packages.find]
where = ["src"]
```

## Public Interface Design

`__init__.py` serves as the package's public API contract. Import and re-export only the symbols that consumers should use. Every other symbol is implementation detail.

```python
# src/mypackage/__init__.py
from mypackage.client import Client
from mypackage.config import Config
from mypackage.errors import PackageError, ValidationError

__all__ = ["Client", "Config", "PackageError", "ValidationError"]
```

`__all__` makes the public API explicit and controls `from mypackage import *` behavior. Consumers who import names not listed in `__all__` accept the risk of breakage across versions.

### Visibility Conventions

| Convention | Meaning | Example |
|------------|---------|---------|
| Listed in `__all__` | Public API, stable across minor versions | `Client`, `Config` |
| Module without underscore | Accessible but not guaranteed stable | `mypackage.client` |
| Underscore-prefixed module | Private implementation | `mypackage._internal` |
| Underscore-prefixed name | Private to containing module | `_resolve_config()` |
| Double-underscore prefix | Name-mangled, class-private | `self.__cache` |

Prefer single-underscore modules (`_internal/`, `_utils.py`) over double-underscore name mangling. Module-level privacy communicates intent clearly without the complexity of Python's name mangling mechanism.

## Package Organization

Structure packages around single cohesive domains. A package whose purpose requires "and" to describe (e.g., "parsing AND validation AND serialization") is a decomposition candidate.

### Thresholds

| Metric | Target | Warning | Restructure |
|--------|--------|---------|-------------|
| Public symbols in `__all__` | <30 | 30-60 | >60 |
| Modules per package | <15 | 15-25 | >25 |
| Lines per module | 200-500 | 500-1,000 | >1,000 |
| Conceptual domains | 1 | 2 | >2 |
| Parameters per `__init__` | <8 | 8-12 | >12 |

### Decomposition Patterns

**Sub-packages for distinct domains**: When a package covers multiple concepts, split into sub-packages with their own `__init__.py`:

```
src/mypackage/
  __init__.py           # Re-exports from sub-packages
  client/
    __init__.py         # Client public API
    _http.py
    _retry.py
  config/
    __init__.py         # Config public API
    _loader.py
    _validation.py
```

**Private implementation packages**: Group internal helpers in `_internal/` to keep the top-level package directory focused on public modules:

```
src/my_package/
  __init__.py
  client.py
  _internal/
    cache.py            # Shared internal utilities
    serialization.py
```

**Namespace packages**: For large systems distributed across multiple repositories, use PEP 420 implicit namespace packages. Omit `__init__.py` from the namespace root so multiple distributions can contribute sub-packages under the same namespace.

## Module Design

Each module focuses on a single concern. Group related classes and functions in one module rather than creating one-class-per-file. A module with one 20-line class is too granular; a module with 1,000 lines of mixed concerns is too coarse.

**Import organization**: Use absolute imports from the package root. Relative imports (`from . import utils`) are acceptable within `_internal/` packages but absolute imports provide clearer stack traces and easier navigation.

```python
# Preferred: absolute import
from mypackage.config import Config

# Acceptable within _internal/: relative import
from .cache import LRUCache
```

## Testing Directory Structure

Mirror the source structure in the test directory. Place `conftest.py` files at directory boundaries where fixtures change scope:

```
tests/
  conftest.py              # Session-scoped fixtures (db, clients)
  test_client.py           # Tests for src/mypackage/client.py
  test_config.py           # Tests for src/mypackage/config.py
  _internal/
    conftest.py            # Fixtures specific to internal tests
    test_resolver.py
    test_cache.py
```

This structure makes test-to-source mapping immediate and keeps fixture scope visible through directory hierarchy.

## Monorepo and Contract-First Composition

The src/ layout above remains the authoritative shape for every Python package and application. A repository that hosts multiple deployables or a shared contract surface composes additional top-level directories around it:

```
<repo>/
  contracts/           # shared contract sources (OpenAPI + Protobuf)
  gen/                 # generated output; committed, never hand-edited
  apps/<service>/      # deployable applications
  packages/<package>/  # shared source packages
  tests/
```

- Every application and source package keeps the src/ layout internally: `apps/<service>/src/<package>/...` and `packages/<package>/src/<package>/...`. The added directories organize deployables and shared code; they do not replace the per-package layout.
- Single-package projects remain plain src/ projects — no `contracts/`, `gen/`, or `apps/` directories are introduced without a second deployable or a shared contract surface.
- The contract-first rules that govern these directories — contract ownership, generated-code authority, and drift discipline — live in the contract-first architecture reference ([Contract-First Architecture](../../contracts/reference/contracts.md)).

## Trade-offs

The src layout adds one directory level and requires build tool configuration. This cost is minor compared to the import isolation it provides. Projects under 500 lines (single-file scripts, CLI utilities) may omit the src layout without risk.

Explicit `__init__.py` exports require maintenance when the public API evolves. This overhead is intentional: every public API change should be deliberate. Implicit exports lead to consumers depending on implementation details that break across releases.

## Rule Companion: Rationale, Examples, Exceptions, and Tooling

*Relocated verbatim from the `project-structure` rule (during an earlier rule-slimming pass). Heading levels demoted one level; rule-local link pointers removed. No other change.*

### Rationale

The src layout prevents accidental imports from the project root during testing, ensuring tests validate the installed package rather than local source. Explicit `__init__.py` exports establish a clear public API surface, allowing internal refactoring without breaking consumers. Private modules (prefixed with `_`) signal implementation details that external code should not depend on. The contract-first top level is a composition around the src layout, not a replacement for it — a reader who knows only one of the two shapes will file generated output or contract source in the wrong place, so the layout reference is mandatory before applying this rule in a multi-deployable repository.

### Examples

**Correct**:
```
src/
  my_package/
    __init__.py       # Public API: from mypackage import Client, Config
    client.py         # Public module
    config.py         # Public module
    _internal/        # Private implementation
      resolver.py
      cache.py
tests/
  test_client.py
  test_config.py
pyproject.toml
```

```python
# src/mypackage/__init__.py
from mypackage.client import Client
from mypackage.config import Config

__all__ = ["Client", "Config"]
```

**Incorrect**:
```
mypackage/
  __init__.py         # No src/ layout
  client.py
  resolver.py         # Internal module exposed alongside public API
tests/
  test_client.py
```

### Exceptions

Single-file scripts and CLI tools under 500 lines may omit the src layout. Namespace packages distributed across multiple repositories may omit `__init__.py` when using implicit namespace packages (PEP 420).

## Related

- KISS (engineering module, `principles` profile: `design/reference/kiss.md`)
- SSOT (engineering module, `principles` profile: `design/reference/ssot.md`)
- [Python Toolchain](../../toolchain/reference/tooling.md)
