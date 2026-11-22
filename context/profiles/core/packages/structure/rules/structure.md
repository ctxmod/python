# Project Structure

Use src layout with explicit public interfaces via __init__.py.

Use the `src/` layout for all Python projects. Define public interfaces explicitly through `__init__.py` exports. Keep implementation details in private modules prefixed with underscore. In a repository that hosts several deployables or a shared contract surface, the src layout stays authoritative inside each package and application, and composes underneath the contract-first top level (`contracts/`, `gen/`, `apps/`, `packages/`, `tests/`) — that top level is a composition around the src layout, not a replacement for it.

Single-file scripts and CLI tools under 500 lines may omit the src layout. Namespace packages distributed across multiple repositories may omit `__init__.py` when using implicit namespace packages (PEP 420).

- src-layout mechanics, visibility conventions, package-organization thresholds and decomposition patterns, module design, the test directory shape, the monorepo/contract-first composition, and the worked correct/incorrect trees — [Project Structure](../reference/structure.md)
- the step-by-step project scaffold that produces this layout — [Project Setup](../../toolchain/reference/project_setup.md)
- where contract source and generated output belong in the top level this layout composes under — [Contract-First Architecture](../../contracts/reference/contracts.md)
- distributing a src-layout package — [Python Packaging](../../toolchain/reference/packaging.md)
