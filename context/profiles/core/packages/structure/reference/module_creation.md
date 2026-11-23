# Module Creation

Module creation workflow covering new modules, __init__.py patterns, and public interface design.

Procedure for adding new modules to an existing Python package. Modules follow the single-concern principle, expose a controlled public interface through `__init__.py`, and integrate with the project's type checking and testing infrastructure from creation.

## Prerequisites

Gather before starting:

- **Module purpose**: One sentence describing what the module does. An "and" connecting unrelated concepts signals two modules.
- **Consumer inventory**: Which modules import this one and what operations they need.
- **Visibility**: Public (listed in `__init__.py`) or private (`_` prefix).
- **Dependencies**: Standard library, third-party, and internal package imports.

## Creation Workflow

### 1. Determine Module Placement

Choose placement based on visibility and scope:

| Content | Placement | Example |
|---------|-----------|---------|
| Public API component | `src/package/module.py` | `client.py`, `config.py` |
| Private implementation | `src/package/_module.py` | `_resolver.py`, `_cache.py` |
| Private helper group | `src/package/_internal/module.py` | `_internal/serialization.py` |
| Sub-domain package | `src/package/subdomain/__init__.py` | `auth/__init__.py` |

Private modules use the underscore prefix convention. Group related private modules under `_internal/` when there are three or more.

- [Project Structure](structure.md)

### 2. Create the Module File

Create the file with the module docstring and initial imports:

```python
"""Short description of the module's purpose.

Extended description if necessary. Explains the module's role
within the package and its primary abstractions.
"""

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from collections.abc import Sequence
```

On a Python 3.14+ floor, annotations are evaluated lazily by default (PEP 649/749), so forward references and `TYPE_CHECKING`-only imports work in annotations without any future import. While `requires-python` still admits 3.13 or older, add `from __future__ import annotations` (PEP 563) as the module's first statement after the docstring to get the same behavior. Place type-only imports inside `TYPE_CHECKING` blocks to prevent circular imports at runtime — except in modules whose annotations a library evaluates at runtime (Pydantic models, FastAPI route signatures, SQLAlchemy `Mapped[...]` columns): there every annotated name must be a real runtime import. The type-safety reference owns the evaluation model ([Type Safety](../../typing/reference/type_safety.md)).

### 3. Design the Public Interface

Define the module's exported symbols. Each module exposes the minimum surface needed by its consumers.

```python
__all__ = ["Client", "ClientConfig", "ClientError"]
```

Guidelines for public interface design:

| Guideline | Rationale |
|-----------|-----------|
| Export types consumers construct directly | Constructors and configuration are the primary API |
| Export error types consumers catch | Callers need to handle specific failure modes |
| Keep Protocol definitions in the consumer | Consumer declares the narrowest contract it needs |
| Omit internal helpers from exports | Implementation details change without notice |

### 4. Define Core Types

Create the primary types the module provides. Use type hints on all function signatures and class attributes.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ClientConfig:
    """Configuration for Client instances."""
    base_url: str
    timeout: float = 30.0
    max_retries: int = 3

class Client:
    """HTTP client with retry and timeout support."""

    def __init__(self, config: ClientConfig) -> None:
        self._config = config

    def get(self, path: str) -> Response:
        """Fetch a resource at the given path."""
        ...
```

Prefer `dataclass(frozen=True)` or Pydantic `BaseModel` for value types. Use regular classes for stateful objects with behavior.

- [Type Safety](../../typing/reference/type_safety.md)

### 5. Define Error Types

Create one base exception per module or sub-package. Specific exceptions inherit from the base and include context attributes callers need. Avoid generic `Exception` or `ValueError` for domain-specific failures.

```python
class ClientError(Exception):
    """Base error for Client operations."""

class ConnectionError(ClientError):
    """Raised when a connection attempt fails."""
    def __init__(self, url: str, cause: Exception) -> None:
        self.url = url
        self.cause = cause
        super().__init__(f"Connection to {url} failed: {cause}")
```

### 6. Update Package __init__.py

Add the new module's public symbols to the package's `__init__.py`:

```python
# src/mypackage/__init__.py
from mypackage.client import Client, ClientConfig, ClientError

__all__ = [
    "Client",
    "ClientConfig",
    "ClientError",
    # ... existing exports
]
```

Only add symbols that consumers import directly from the package root. Module-level access (`from mypackage.client import Client`) remains available for all public symbols regardless of `__init__.py` contents.

### 7. Create Test File and Verify

Create a test file mirroring the source path (`tests/test_client.py` for `src/mypackage/client.py`). For sub-packages, mirror the directory structure: `tests/auth/test_session.py` for `src/mypackage/auth/session.py`.

Run `just ci` to verify linting, type checking, and tests pass. Common verification failures:

| Failure | Cause | Resolution |
|---------|-------|------------|
| Import error in basedpyright | Missing `__init__.py` in sub-package | Create `__init__.py` with exports |
| Circular import | Module A imports B, B imports A | Use `TYPE_CHECKING` block or restructure |
| ruff import order | New import breaks sorting | Run `just lint` to auto-fix |
| Missing `__all__` entry | Public symbol not exported | Add to `__all__` in both module and package |

## Decision Support

### When to Create a Sub-package

| Condition | Action |
|-----------|--------|
| 3+ related modules sharing a concept | Create sub-package with `__init__.py` |
| Module exceeds 1,000 lines | Split into sub-package with focused modules |
| Distinct domain boundary | Separate sub-package with own public API |
| Single module under 500 lines | Keep as module, no sub-package needed |

### Class vs Function Module

| Content | Module Style |
|---------|-------------|
| Stateful object with methods | Class-based module |
| Pure transformations, no shared state | Function-based module |
| Configuration + behavior | Dataclass + class module |
| Utility operations | Function-based, underscore-prefixed module |

## Related

- [Project Setup](../../toolchain/reference/project_setup.md)

## Sources

- https://peps.python.org/pep-0749/ — deferred annotation evaluation in Python 3.14 and the planned `from __future__ import annotations` deprecation path (accessed 2026-11-22)
- https://peps.python.org/pep-0563/ — postponed evaluation via the future import, for floors below 3.14 (accessed 2026-11-22)
