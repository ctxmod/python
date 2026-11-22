# Configuration Management

Use pydantic-settings with strict mode, environment variables, and constructor injection.

Use pydantic-settings with `strict=True` and `frozen=True` for all application configuration. Load configuration from environment variables as the primary source. Validate all settings at startup and fail immediately on invalid values. Inject settings through constructors for testability. Never use global mutable configuration. One helper resolves configuration for production and local runs alike — a test swaps the configuration source and injects a fake provider rather than patching a client module, so no test-only resolution path exists.

Single-file scripts under 200 lines may read environment variables directly with `os.getenv` when adding a pydantic-settings dependency is disproportionate. Feature flags managed by external systems (LaunchDarkly, Unleash) follow their SDK patterns rather than local settings validation.

- resolution order, XDG and environment-specific file resolution, secret management, 12-factor compliance, and the worked correct/incorrect settings examples plus the startup-validation and pytest env-var/cache-clearing tooling — [Configuration Management](../reference/configuration.md)
- fake-config and provider injection, and the same-helper local/production parity this rule depends on — [Testing Philosophy](../../testing/reference/testing.md)
- where settings modules sit in the distributed package — [Python Packaging](../../toolchain/reference/packaging.md)
