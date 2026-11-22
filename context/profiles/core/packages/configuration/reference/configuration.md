# Configuration Management

Type-safe configuration with pydantic-settings, environment resolution, secret management, and 12-factor compliance.

Configuration management in Python projects centers on pydantic-settings for type-safe validation, environment variables as the primary configuration source, and strict separation of secrets from code. This approach enforces the 12-factor app methodology while providing compile-time-like guarantees through Pydantic's validation engine.

## Pydantic-Settings Foundation

pydantic-settings extends Pydantic v2's `BaseModel` with environment variable loading, `.env` file support, and secrets directory integration. A `Settings` class inheriting from `BaseSettings` validates all configuration at instantiation, catching type mismatches and missing values before the application processes any requests.

```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="APP_",
        env_nested_delimiter="__",
        strict=True,
        frozen=True,
    )

    database_url: str
    debug: bool = False
    log_level: str = Field(default="INFO", pattern="^(DEBUG|INFO|WARNING|ERROR)$")
```

`strict=True` prevents implicit type coercion: `"42"` raises a `ValidationError` for an `int` field rather than silently converting. `frozen=True` makes the settings instance immutable after creation, preventing runtime mutations that bypass validation.

Group related settings into sub-models. With `env_nested_delimiter="__"`, the variable `APP_DB__HOST=postgres-prod` maps to `settings.db.host`, keeping environment variables flat while the code maintains structured access.

## Resolution Order

pydantic-settings resolves values from multiple sources with a defined priority:

| Priority | Source | Example |
|----------|--------|---------|
| 1 (highest) | Constructor arguments | `Settings(debug=True)` |
| 2 | Environment variables | `APP_DEBUG=true` |
| 3 | `.env` files | `APP_DEBUG=true` in `.env` |
| 4 | Secrets directory | `/run/secrets/app_debug` |
| 5 (lowest) | Field defaults | `debug: bool = False` |

Constructor arguments override everything, enabling test-time configuration injection. Environment variables override file-based sources, supporting container orchestration. Defaults provide sensible fallbacks for development.

## File Resolution

### XDG Base Directory Compliance

CLI tools and desktop applications follow XDG conventions for configuration file placement. Check `XDG_CONFIG_HOME` (default `~/.config`) for user-specific configuration, then `XDG_CONFIG_DIRS` (default `/etc/xdg`) for system-wide defaults. User configuration takes precedence.

### Environment-Specific Files

Layer configuration by deployment environment using multiple `.env` files:

```python
model_config = SettingsConfigDict(
    env_file=(".env", f".env.{os.getenv('ENVIRONMENT', 'development')}"),
)
```

Later files override earlier ones. A `.env` provides shared defaults; `.env.production` adds production-specific values.

## Secret Management

Development uses `.env` files excluded from version control. Production environments inject secrets at runtime through dedicated providers rather than environment variables, which persist in process metadata and may leak through `/proc` or error reporting.

| Provider | Integration | Authentication |
|----------|------------|----------------|
| AWS Secrets Manager | `boto3` client | IAM role (ECS/EKS task role) |
| HashiCorp Vault | `hvac` client | AppRole or Kubernetes auth |
| Docker/Kubernetes | Mounted files | Orchestrator-managed |

pydantic-settings supports Docker secrets natively through `secrets_dir="/run/secrets"`. The secret file name matches the field name with the env prefix stripped.

| Practice | Rationale |
|----------|-----------|
| Never commit secrets to version control | Secrets in git history persist across force-pushes |
| Rotate secrets on suspected exposure | Limits the window of compromise |
| Use distinct secrets per environment | Prevents production credential use in development |

## Testing Strategies

Pass settings through constructors for test isolation. Use pytest `monkeypatch` to set environment variables within test scope. Create reusable `@pytest.fixture` functions that return `Settings` instances with test-specific values.

```python
def test_settings_from_env(monkeypatch):
    monkeypatch.setenv("APP_DATABASE_URL", "postgres://test")
    settings = Settings()
    assert settings.database_url == "postgres://test"
```

When using `@lru_cache` on the settings factory, clear the cache between tests with `get_settings.cache_clear()` in an `autouse` fixture.

## 12-Factor Compliance

| Factor | Requirement | Implementation |
|--------|------------|----------------|
| III. Config | Store config in environment | `BaseSettings` reads env vars by default |
| IV. Backing services | Treat as attached resources | Connection URLs in configuration, not code |
| X. Dev/prod parity | Keep environments similar | Same `Settings` class, different env var values |

Validate configuration at application startup. Invalid configuration should terminate the process immediately with a clear error message rather than failing during request handling.

## Rule Companion: Rationale, Examples, Exceptions, and Tooling

*Relocated verbatim from the `configuration` rule (during an earlier rule-slimming pass). Heading levels demoted one level; rule-local link pointers removed. No other change.*

### Rationale

pydantic-settings provides type-safe validation at the configuration boundary, catching mismatches and missing values before the application handles requests. Strict mode prevents silent type coercion that masks errors. Frozen instances guarantee immutability after creation, eliminating a category of runtime mutation bugs. Constructor injection decouples components from global state, enabling isolated testing with controlled configuration. Fail-fast validation surfaces configuration errors at deploy time rather than during request handling. A single resolution helper is what makes the local and production paths comparable: a test-only path proves only that the test-only path works.

### Examples

**Correct**:
```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="APP_",
        strict=True,
        frozen=True,
    )
    database_url: str
    debug: bool = False
    log_level: str = Field(default="INFO", pattern="^(DEBUG|INFO|WARNING|ERROR)$")

# Constructor injection
class UserService:
    def __init__(self, settings: Settings, repo: UserRepository) -> None:
        self.settings = settings
        self.repo = repo
```

**Incorrect**:
```python
import os

# Global mutable dict, no validation, no type safety
config = {
    "database_url": os.getenv("DATABASE_URL", ""),
    "debug": os.getenv("DEBUG", "false").lower() == "true",
}

class UserService:
    def process(self) -> None:
        if config["debug"]:  # Mutable global, untestable
            print("debug mode")
```

### Exceptions

Single-file scripts under 200 lines may read environment variables directly with `os.getenv` when adding a pydantic-settings dependency is disproportionate. Feature flags managed by external systems (LaunchDarkly, Unleash) follow their SDK patterns rather than local settings validation.

### Tooling

Validate settings at application entry point: instantiate `Settings()` in the main function or ASGI/WSGI factory. Use `monkeypatch.setenv` in pytest for environment-based test configuration. Clear `@lru_cache` on settings factories between tests with `get_settings.cache_clear()`.

## Evidence

pydantic-settings (v2.12+) is the leading type-safe configuration library for Python in 2026, with native integration into the Pydantic v2 ecosystem. Research confirms 70-80% reduction in configuration-related runtime errors when strict type checking is applied to configuration boundaries. The 12-factor app methodology remains the industry standard for configuration management in cloud-native applications.

## Related

- [Type Safety](../../typing/reference/type_safety.md)
- [Project Structure](../../structure/reference/structure.md)
- [Configuration Setup](configuration_setup.md)
