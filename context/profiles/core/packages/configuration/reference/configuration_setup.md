# Configuration Setup

Step-by-step procedure for setting up type-safe configuration with pydantic-settings.

Procedure for implementing type-safe configuration with pydantic-settings. The result is a validated, immutable settings object loaded from environment variables, `.env` files, and secret directories with fail-fast behavior at startup.

## Prerequisites

- **Project structure**: Existing Python project with `pyproject.toml` and `src/` layout.
- **Configuration fields**: Required settings (database URLs, API keys, feature flags).

## Setup Workflow

### 1. Install Dependencies

```bash
uv add pydantic-settings
```

### 2. Create the Settings Module

Create `src/{package}/config.py`:

```python
"""Application configuration with type-safe validation."""

from __future__ import annotations

from functools import lru_cache

from pydantic import BaseModel, ConfigDict, Field, ValidationError
from pydantic_settings import BaseSettings, SettingsConfigDict


class DatabaseConfig(BaseModel):
    model_config = ConfigDict(strict=True)
    host: str = "localhost"
    port: int = Field(default=5432, ge=1, le=65535)
    name: str = "app"
    user: str = "app"
    password: str = ""
    pool_size: int = Field(default=10, ge=1, le=100)

    @property
    def url(self) -> str:
        return (
            f"postgresql+asyncpg://{self.user}:{self.password}"
            f"@{self.host}:{self.port}/{self.name}"
        )


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="APP_", env_nested_delimiter="__",
        env_file=(".env",), env_file_encoding="utf-8",
        strict=True, frozen=True,
    )
    environment: str = Field(default="development")
    debug: bool = False
    log_level: str = Field(default="INFO", pattern="^(DEBUG|INFO|WARNING|ERROR)$")
    db: DatabaseConfig = DatabaseConfig()
    host: str = "0.0.0.0"
    port: int = Field(default=8000, ge=1, le=65535)


@lru_cache(maxsize=1)
def get_settings() -> Settings:
    try:
        return Settings()
    except ValidationError as e:
        import sys
        print(f"Configuration error:\n{e}", file=sys.stderr)
        raise SystemExit(1)
```

[Configuration Management](configuration.md)

### 3. Create Environment Files

Create `.env` for local development (add to `.gitignore`):

```bash
APP_ENVIRONMENT=development
APP_DEBUG=true
APP_DB__HOST=localhost
APP_DB__NAME=app_dev
APP_DB__USER=dev
APP_DB__PASSWORD=dev_password
```

Create `.env.example` as a committed template showing required variables without values.

### 4. Integrate with Application Startup

```python
from my_package.config import get_settings

def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(title="My Service", debug=settings.debug)
    return app
```

Validation runs at `get_settings()`. Missing or invalid fields terminate the process before request handling.

### 5. Configure Secret Directory Support

For container deployments with Docker or Kubernetes secrets, add `secrets_dir="/run/secrets"` to `SettingsConfigDict`. Secret file names match field names with the env prefix stripped.

### 6. Create Test Fixtures

Add to `tests/conftest.py`:

```python
import pytest
from my_package.config import Settings, get_settings

@pytest.fixture(autouse=True)
def clear_settings_cache():
    get_settings.cache_clear()
    yield
    get_settings.cache_clear()

@pytest.fixture
def test_settings() -> Settings:
    return Settings(
        environment="testing", debug=True,
        db={"host": "localhost", "name": "test_db"},
    )
```

Use `monkeypatch.setenv()` for tests that verify environment variable loading.

[Type Safety](../../typing/reference/type_safety.md)

### 7. Verify the Setup

| Verification | Method |
|-------------|--------|
| Default values | Instantiate `Settings()` without env vars |
| Environment override | Set `APP_DEBUG=true`, verify `settings.debug is True` |
| Nested config | Set `APP_DB__HOST=myhost`, verify `settings.db.host` |
| Validation failure | Set `APP_DB__PORT=99999`, verify `ValidationError` |
| Strict mode | Set `APP_DB__PORT=abc`, verify rejection |
| Frozen enforcement | Attempt `settings.debug = False`, verify `TypeError` |

## Troubleshooting

**Validation errors on startup**: Check variable names match the expected format. With `env_prefix="APP_"`, the field `database_url` expects `APP_DATABASE_URL`.

**Nested fields not loading**: Verify `env_nested_delimiter` is set.

**Cache stale in tests**: Clear `get_settings.cache_clear()` in test fixtures.

## Related

- [Project Setup](../../toolchain/reference/project_setup.md)
