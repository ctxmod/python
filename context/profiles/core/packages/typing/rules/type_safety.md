# Type Safety

Enable strict type checking and validate data at boundaries with Pydantic.

Enable basedpyright strict mode for all code. Add type hints to every function signature, including return types. Use Pydantic models for runtime validation at data boundaries (API inputs, configuration, external data). Never pass `dict[str, Any]` — or any untyped mapping — as the interface between components: an untyped value is confined to the adapter that validates it.

Run `basedpyright` in CI with strict mode. Configure per-module overrides for untyped dependencies rather than disabling strict mode globally; dynamically generated code such as ORM model metaclasses may use targeted `# type: ignore` comments with explanation.

- layered validation, strict-mode configuration and adoption strategy, Protocol-based design, Pydantic boundary validation and placement, type narrowing, generics, and the worked correct/incorrect examples — [Type Safety](../reference/type_safety.md)
- which boundaries are untrusted, which model class owns each mapping, and where a validated model becomes a domain object — [Model Boundaries](../reference/model_boundaries.md)
