# Python Packaging

Python packaging principles covering uv workspaces and per-package manifests, sibling dependency edges, dependency declaration discipline, uv-run script tools, and container packaging.

How Python code composes into a repository: one manifest per package, uv as
the single resolver, disciplined dependency declaration, script tools for
operational work, and containers that consume lock-derived artifacts. Tool
selection and pinning are owned by the toolchain reference
([Python Toolchain](tooling.md)); generated-output
authority and the contract-generation pipeline are owned by the
contract-first reference
([Contract-First Architecture](../../contracts/reference/contracts.md)); the src layout is
owned by the project-structure reference. This file owns the composition
rules between them.

## Monorepo Composition: uv Workspaces

A multi-package repository is one uv workspace. The root `pyproject.toml`
declares the members and nothing else:

```toml
[tool.uv.workspace]
members = ["apps/*", "packages/*"]
```

- **The root is an orchestrator, not a package.** It carries no product
  code: only the workspace declaration, the shared toolchain development
  dependencies, and orchestration scripts. Anything a package can own lives
  in the package; a root script that would duplicate a workspace-level
  tool's entry point points at that tool instead.
- **One lockfile for the whole workspace.** A single `uv.lock` at the root
  records one resolution of the full dependency graph, and `uv sync` at the
  root materializes every member's environment. Per-package lockfiles or
  per-package re-resolution resolve the same graph twice — the exact drift
  the lockfile exists to prevent.
- **Every member is a fully standalone installable package**: its own
  manifest, README, src tree, and tests tree. Workspace membership changes
  resolution, not shape — a member keeps the src layout and the
  hyphen-distribution-name / underscore-import-name pair like any package.
- **Non-obvious members are enumerated explicitly.** A tool directory
  nested outside the glob roots is listed by name rather than pulling the
  glob wider; widening a glob to reach one member silently adopts
  everything else the pattern matches.

The workspace composes with the contract-first layout (owned by the
contract-first reference): `apps/<service>` holds one deployable
application per entry, `packages/` holds shared source packages, and `gen/`
holds generated output.

An alternative posture — a language tier of sibling packages with no
workspace umbrella, each consumer mapping its siblings onto editable
relative paths — delivers the same edges with more per-consumer machinery
(see sibling dependency edges below). Pick one posture per tier and apply
it uniformly; mixing the two inside one tier leaves two resolution stories
for the same edge.

## Per-Package Manifests

- **All tool configuration is collocated in the per-package manifest**
  (lint, types, test) — there is no tier-level or repository-level config
  file for them. Siblings carry a near-identical base block and diverge
  only where annotated with an inline rationale for the delta. Anything
  two packages share verbatim is a candidate for a shared template or
  generated config, not a copy — copied config blocks drift (a dead rule
  exclusion inherited from a fuller template; a rule family silently
  dropped by one sibling).
- **Derived tool configurations that must stay uniform across packages are
  generated from one source and committed**, and any CI step that mirrors
  a generated file carries a must-change-with-it comment, so the mirror
  cannot silently diverge from its source.
- **Entry points are declared in the manifest.** A package that is both an
  operator CLI and a deployed service declares two console entry points
  rather than relying on documented `python -m` incantations.
- **The published dependency list is a deliberately light install floor.**
  Heavyweight execution sets ride named extras or environment features
  instead, keeping the package installable without pulling them.
- **Version stamping has one source or one check.** The version is stamped
  in `[project].version`; stamping it again in `__init__` is a known drift
  point — derive one from the other, or gate the pair in CI. Package
  families published in lockstep may instead share one version from a root
  file with a checker script (and a sync mode) as the single source.
- **Task menu.** Under this profile's toolchain, task running is the
  justfile (the toolchain reference owns it). Where an environment manager
  supports manifest-embedded tasks, the same shape applies — short verbs,
  each mapping to exactly one tool invocation, plus one composite `check`
  verb depending on the verification verbs. Under plain uv with no task
  runner, the documented `uv run` commands are the task menu; document
  them once.

## Sibling Dependency Edges

- Declare a sibling as a **bare distribution name** in the normal
  dependency list — the list stays index-publishable, with no path or
  local constraint in it.
- Inside a workspace, uv resolves the bare name to the member
  automatically. Outside a workspace (the no-umbrella posture), map the
  name onto an editable relative-path source in one clearly separated
  table (`[tool.uv.sources]`) — the mapping mechanism lives in exactly one
  place.
- **Editable always; no version constraints between siblings.** The
  repository checkout is the version. A constraint between siblings would
  re-pin a graph the lockfile already resolves.
- The dependency graph between siblings is **acyclic by construction**:
  one shared-infrastructure package that depends on no sibling, consumers
  that depend on it, and no consumer-to-consumer edge (next section).
- The edge is test-pinned: the consuming package carries a guard test that
  imports every public symbol it expects from the shared package, so a
  broken edge fails as a test, not at deployment. The testing reference
  owns the guard-test discipline.

## Shared-Infrastructure Package

One member owns every cross-cutting concern the others need — structured
logging setup, configuration models, persistence helpers, and the shared
domain data models:

- exactly one subpackage per concern, each exporting its public surface
  from its own `__init__` — no deep module trees for their own sake;
- the dependency edge is strictly one-directional: shared ← consumers,
  never the reverse;
- cross-cutting code lives at the cross-cutting level; a domain package
  owns only its domain specializations.

Which concerns land in the shared slot is a per-repository design
decision; the slot, the one-way edge, and the `__init__`-level public
surface are the transferable rules.

## Dependency Declaration Discipline

- **One canonical development-dependency surface per repository** — the
  dependency-groups table — with tooling reading or generating any other
  surface. Never hand-mirror a development-dependency list across
  surfaces: extras, environment-manager features, and dependency-groups
  carrying the same names drift apart mechanically (the same type-checker
  observed floored at two different minimums in two surfaces of one
  manifest).
- **Every nontrivial version cap carries its failure-mode rationale
  inline** — the upstream behavior that motivated the cap — plus a
  provenance pointer to the investigation that documented it. Pins that
  exist only to satisfy one library's transitive expectations are promoted
  to top-level with the reason written beside them. The rationale travels
  with the pin into generated exports; a cap without its failure mode is
  unmaintainable.
- **Mutually exclusive or heavy dependency sets live in named extras**
  composed into environments, never in the published floor; environments
  that must agree share one lock.
- **Container builds consume the lock, not the manifest.** The build takes
  a requirements export generated from the lockfile for the target
  platform — runtime dependencies only, with a provenance header recording
  the lock, the platform, and the regeneration command. Workspace members
  are excluded; they are provided by the deployment mechanism. Never
  hand-edit the export; never resolve the graph a second time.

## Script Tools and uv run

Operational tooling grows along a ladder of three shapes. Choose the
smallest that works; promote when it outgrows.

### Single-file scripts

A script of a few dozen lines carries its dependencies as inline script
metadata (PEP 723 form) and runs with no manifest, lockfile, or install
step:

```python
# scripts/check.py
# /// script
# requires-python = ">=3.12"
# dependencies = [
#     "httpx>=0.27",
#     "structlog>=24.0",
# ]
# ///
```

```bash
uv run scripts/check.py
```

uv resolves an ephemeral environment from the inline metadata.
Single-file scripts live inside the project-structure rule's script
exemption. Inline metadata is not pinned by a lockfile — once a script is
reused across machines or people, promote it to a standalone tool.

### Standalone tool projects

A recurring operational tool is one tool = one directory = one manifest +
one committed lockfile + a src-layout package + a README that is the
tool's contract document:

```toml
[project]
name = "<repo>-<app>"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["httpx>=0.27", "structlog>=24.0", "typer>=1.0"]

[project.scripts]
<app> = "<repo>_<app>.cli:app"
```

- **Invocation contract:** `uv run --project scripts/<app> <app>
  <subcommand>`, from anywhere in the repository — the runner resolves the
  isolated manifest, environment, and lockfile, so the documented
  invocation contains no activation, installation, or path-setup step.
- **Isolation boundary:** the tool takes no dependency edge on any
  workspace package — its lockfile graph contains only third-party
  packages plus the tool itself. Conventions worth keeping (logging
  setup, CLI structure) are mirrored in a few dozen lines, with
  docstrings stating the mirroring, because importing a product package
  would pull its heavyweight dependency chain into a quick invocation.
- **One dependency surface only:** the runtime set. Development tools run
  through ephemeral environments — `uv run --no-project --with ruff ruff
  check src` — so the multi-surface dev-dependency drift above cannot
  arise.
- **When a script tool beats a package:** the tool is operational or
  procedural infrastructure for operators and agents rather than product
  code; recurring ad-hoc command-line improvisation is being consolidated
  into one place; or a package edge would pull a heavyweight dependency
  graph. **When a package wins:** product code must import the result, or
  the dependency graph is genuinely shared.
- **The data contract is machine-first.** Structured JSON is the default
  output mode on every subcommand, with a human mode behind one paired
  flag — never the reverse. Results go to stdout (cleanly pipeable);
  diagnostics and errors go to stderr through the structured logger.
  Canonical input and output paths are module-level constants resolved
  independently of the working directory, and every canonical default is
  overridable by flag. Errors use distinct exit codes per class (runtime
  failure versus usage error) and carry a remediation hint naming the
  subcommand that derives a missing input. A `doctor` subcommand reports
  environment health and degrades rather than fails when an optional
  accelerator is absent.
- **Tests are a decision, not an inheritance.** Decide explicitly whether
  a standalone tool carries its own suite; silence here inherits whatever
  the last similar tool did.

### Workspace packages

The top of the ladder is a full workspace member: shared dependency
graph, importable by product code, and every composition rule of a
package above.

## Container Packaging

Containers are dependency environments; the application source reaches
them through the deployment mechanism, and the cloud-platform doctrine owns
deployment wiring — this profile stops at the image contract.

- **Dependencies-only image posture.** The build context is the
  language-tier root (so sibling paths resolve), with one context-root
  ignore file (interpreter caches, the virtualenv, the VCS directory,
  tool caches — nothing else). Two variants reach a dependencies-only
  environment: install the workspace packages during the build and strip
  them from site-packages afterward, or install only the lock-derived
  requirements export (dependency-declaration section above) and never
  install workspace packages in the image at all.
- **Hardened runtime image.** A slim base carrying only the shared
  libraries the environment actually loads, with the reason stated
  inline; a non-root user with a fixed UID/GID chosen for mount-ownership
  compatibility, created before the environment copy so ownership lands
  in one step; the environment trimmed (bytecode caches, bundled test
  directories, compiled intermediates); declared mount points and a port
  matching the manifest's declared default.
- **Build-time verification waivers are recorded in the waiving
  artifact.** Where the execution environment admits only appliance-format
  images, keep both definitions with an automated export pipeline between
  them. A waived verification step states the exact failure reason and
  the post-build command that replaces it, inside the artifact that
  waives it — never in tribal memory.
- **Cross-architecture builds state the platform flag explicitly** where
  developer workstations and the execution cluster differ.

## Environment-Variable Template

The service package commits a fully-commented environment-variable
template, and setup is exactly: copy the template.

- every variable carries a purpose comment; values are placeholders,
  never real;
- externally-obtained credentials carry provenance — which authority
  issues them, via which channel;
- secret-type values carry a generation one-liner (the standard library's
  secrets module suffices);
- security-relevant defaults state their meaning and their safe local
  exception;
- tuning knobs state their units and effects.

The configuration reference owns the settings-model side of this
contract; the template is the packaging artifact that makes it operable.

## Related

- SSOT (engineering module, `principles` profile: `design/reference/ssot.md`)
- [Project Structure](../../structure/reference/structure.md)
- [Configuration Management](../../configuration/reference/configuration.md)
