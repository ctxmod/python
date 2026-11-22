# Type Safety

Type safety principles covering strict checking, layered validation, model-class separation, Protocol patterns, and generics.

Python type safety operates in three layers: static checking at development time, runtime validation at data boundaries, and structural subtyping through Protocols. Strict static checking alone reduces production type-related failures by 70-80% in large codebases. Adding runtime validation at boundaries handles the remaining cases where external data enters the system.

## Model-Class Separation

Validation placement follows the model-class ladder: external contract DTOs, application DTOs, domain models, persistence models, and the model-facing decision class are distinct model classes, and the crossings between them are where data of unproven shape meets typed code. The canonical ladder statement, the per-class rules, and the mapping-not-mirroring discipline are owned by the model-boundaries reference ([Model Boundaries](model_boundaries.md)).

The three layers slot onto that ladder: basedpyright strict governs code within each model class and the typed mappings between classes; Pydantic fires wherever data of unproven shape crosses a class boundary (the Layered Validation table below expands that row); TypeGuard guards the narrow paths static analysis cannot prove. Protocol-based design keeps the classes decoupled — a consumer Protocol names domain types, never another class's DTOs, so structural subtyping cannot quietly weld two model classes together.

## Layered Validation

| Layer | Tool | Scope | Catches |
|-------|------|-------|---------|
| Static | basedpyright (strict) | All code, CI | Internal logic errors, missing annotations, incompatible assignments |
| Boundary | Pydantic v2 | API inputs, config, external data | Malformed payloads, wrong types from callers |
| Runtime | TypeGuard | Critical paths | Violations that static analysis cannot prove |

Static checking runs on every commit in CI with zero tolerance for errors. Boundary validation activates wherever untrusted data enters the system. Runtime guards apply selectively to paths where static analysis lacks sufficient information to verify correctness.

## Strict Mode Configuration

basedpyright strict mode raises the strict diagnostic set to errors. Configure it in `pyproject.toml`:

```toml
[tool.basedpyright]
typeCheckingMode = "strict"
venvPath = "."
```

`typeCheckingMode` takes `"off"`, `"basic"`, `"standard"`, `"strict"`, `"recommended"` (basedpyright's default) or `"all"`. The pyright key `strict` is a list of paths forced to strict mode, not a boolean: `strict = true` is rejected and the run silently uses the default mode, which reports most strict findings as warnings.

Strict mode disallows untyped function definitions, reports unknown and `Any`-tainted types, and enforces strict optional checking. For code that cannot meet strict yet — a legacy tree, a wrapper around an untyped library — relax specific rules for that path with `executionEnvironments` rather than weakening the global mode (Pyright has no mypy-style per-module `overrides` table; basedpyright reports it as an unrecognized setting):

```toml
[tool.basedpyright]
typeCheckingMode = "strict"
executionEnvironments = [
  { root = "src/project_name/legacy", reportUnknownParameterType = "warning", reportMissingParameterType = "warning" },
  { root = "src" },
]
```

Missing third-party stubs are a dependency question first: install the published stubs package (`types-*`, `pandas-stubs`) as a development dependency before relaxing anything.

### Adoption Strategy

| Phase | Scope | Target |
|-------|-------|--------|
| 1 | New modules only | 100% type hints, strict mode |
| 2 | Critical existing modules | Add annotations, resolve errors |
| 3 | Full codebase | Strict mode globally, CI enforcement |

New projects start at Phase 3. Legacy codebases progress incrementally. Limit suppression comments to genuine edge cases, name the rule (`# pyright: ignore[reportUnknownMemberType]`), and always include an explanation; the type-checking reference owns the suppression syntax ([Type Checking Validation](type_checking.md)).

## Protocol-Based Design

`typing.Protocol` enables structural subtyping -- Python's equivalent of static duck typing. A class satisfies a Protocol if it implements the required methods, without explicit inheritance. This produces looser coupling than abstract base classes (ABCs).

```python
from typing import Protocol

class Repository(Protocol):
    def get(self, id: int) -> Model | None: ...
    def save(self, entity: Model) -> None: ...

class PostgresRepository:
    """Satisfies Repository through structural compatibility."""
    def get(self, id: int) -> Model | None:
        # Implementation
        ...

    def save(self, entity: Model) -> None:
        # Implementation
        ...
```

No registration, no inheritance, no metaclass. PostgresRepository satisfies Repository because it implements the required method signatures. basedpyright verifies this statically.

### Protocol Guidelines

| Guideline | Rationale |
|-----------|-----------|
| Prefer Protocol over ABC | No inheritance coupling, structural matching |
| Keep Protocols small (1-3 methods) | Larger Protocols reduce the set of satisfying types |
| Use `@runtime_checkable` sparingly | Only when `isinstance()` checks are genuinely needed |
| Combine via multiple inheritance | `class ReadWriter(Reader, Writer, Protocol): ...` |
| Define in the consumer package | Consumer declares only the methods it calls |

Consumer-defined Protocols produce the narrowest possible contract. When three or more consumers define identical Protocols, consolidate into a shared location.

## Pydantic for Boundary Validation

Use Pydantic v2 models wherever external data enters the system: API request bodies, configuration files, database query results, message queue payloads.

```python
from pydantic import BaseModel, ConfigDict

class CreateUserRequest(BaseModel):
    model_config = ConfigDict(strict=True)
    email: str
    name: str
    age: int
```

Strict mode (`strict=True`) prevents implicit type coercion. Without it, Pydantic silently converts `"42"` to `42` for an `int` field, a source of subtle bugs when data types matter.

### Validation Placement

| Boundary | Validation Method |
|----------|------------------|
| API endpoints | Pydantic request/response models |
| Configuration loading | Pydantic settings with environment variable support |
| External API responses | Pydantic models parsing JSON |
| Database results | Pydantic or dataclass mapping |
| Message queue payloads | Pydantic models for deserialization |

Internal function calls between trusted modules do not need runtime validation. Static type checking handles internal contracts.

## Type Narrowing

TypeGuard functions narrow types in contexts where static analysis cannot determine the type. Pattern matching (Python 3.10+) provides an alternative for structural patterns.

```python
from typing import TypeGuard

def is_valid_user(obj: object) -> TypeGuard[User]:
    return isinstance(obj, User) and obj.id > 0

def process(data: object) -> None:
    if is_valid_user(data):
        # basedpyright narrows data to User
        send_email(data.email)
```

## Generic Programming

Use `TypeVar` and `Generic` for reusable type-safe components. Python 3.12+ provides inline type parameter syntax:

```python
class Repository[T]:
    def add(self, item: T) -> None:
        self._items.append(item)

    def get(self, index: int) -> T | None:
        return self._items[index] if index < len(self._items) else None
```

Bounds and aliases use the same syntax: `class Repository[T: Entity]` restricts `T` to an `Entity` hierarchy when the generic code calls methods defined on the bound, and `type UserId = int` (PEP 695) declares an alias the checker treats as a distinct name. For a `requires-python` below 3.12, use explicit `TypeVar("T", bound=Entity)` with `Generic[T]`.

## Annotation Evaluation (Python 3.14+)

From Python 3.14, annotations are evaluated lazily by default (PEP 649/749): forward references to names defined later in the module work without quotes, and `annotationlib` reads annotations in value, forward-ref, or string form. Consequences:

- On a 3.14+ floor, drop `from __future__ import annotations`; it is redundant, and PEP 749 schedules its eventual deprecation. Keep it only while `requires-python` admits 3.13 or older and the module relies on unquoted forward references.
- Runtime-introspecting libraries — Pydantic models, FastAPI signatures, SQLAlchemy `Mapped[...]` columns, dataclass-based validators — evaluate annotations when the class or route is built. Every name such an annotation uses must be importable at runtime: an import placed under `if TYPE_CHECKING:` is invisible to them under either evaluation model, and fails at class construction or first request instead of in the checker.


## Rule Companion: Rationale, Examples, Exceptions, and Tooling

*Relocated verbatim from the `type-safety` rule (during an earlier rule-slimming pass). Heading levels demoted one level; rule-local link pointers removed. No other change.*

### Rationale

Strict static type checking catches type errors before runtime, reducing production failures by 70-80% in large codebases. Layered validation combines static checking (basedpyright in CI) with runtime validation (Pydantic at boundaries) to cover both internal logic and external data. Protocol-based interfaces enable structural subtyping with full static verification, replacing ABCs with less coupling. Which boundaries are untrusted, which model class owns each mapping, and where a validated model becomes a domain object are decided by the model-boundaries reference — this rule cannot be applied consistently without it, because "the boundary" is otherwise a matter of taste.

### Examples

**Correct**:
```python
from pydantic import BaseModel, ConfigDict
from typing import Protocol

class UserRepository(Protocol):
    def get(self, user_id: int) -> User | None: ...

class CreateUserRequest(BaseModel):
    model_config = ConfigDict(strict=True)
    email: str
    name: str

def create_user(repo: UserRepository, request: CreateUserRequest) -> User:
    user = User(email=request.email, name=request.name)
    repo.save(user)
    return user
```

**Incorrect**:
```python
def create_user(repo, request):  # No type hints
    user = User(email=request["email"], name=request["name"])  # Unvalidated dict
    repo.save(user)
    return user
```

### Exceptions

Dynamically generated code (e.g., ORM model metaclasses) may use rule-named `# pyright: ignore[<rule>]` comments with explanation. Third-party libraries lacking type stubs may require path-scoped `executionEnvironments` relaxations in basedpyright configuration.

### Tooling

Run `basedpyright` in CI with `typeCheckingMode = "strict"`. Configure path-scoped `executionEnvironments` relaxations for untyped dependencies rather than disabling strict mode globally.

## Evidence

Research on Python type safety (2024-2026) confirms 70-80% reduction in runtime type errors with strict static checking. Protocol adoption has grown as the community moves away from ABC-heavy designs. Pydantic v2 provides strict validation with improved performance over v1. basedpyright's aggressive inference catches issues that mypy's conservative approach misses.

## Related

- [Python Toolchain](../../toolchain/reference/tooling.md)
- [Project Structure](../../structure/reference/structure.md)
- [Testing Philosophy](../../testing/reference/testing.md)
- [Contract-First Architecture](../../contracts/reference/contracts.md)

## Sources

- https://docs.basedpyright.com/latest/configuration/config-files/ — `typeCheckingMode` values, `strict` as a path list, `executionEnvironments` diagnostic overrides, basedpyright 1.40 (accessed 2026-11-22; behavior re-probed with basedpyright 1.40.2)
- https://peps.python.org/pep-0649/ and https://peps.python.org/pep-0749/ — deferred evaluation of annotations, Final, Python 3.14 (accessed 2026-11-22)
- https://docs.python.org/3.14/library/annotationlib.html — annotation formats (accessed 2026-11-22)
- https://peps.python.org/pep-0695/ — type parameter syntax and `type` aliases, Python 3.12 (accessed 2026-11-22)
