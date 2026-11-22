# Contract-First Development

Change the contract before the implementation, never hand-edit generated output, and never deserialize untyped JSON at a process boundary.

Change the contract before the implementation. API and Domain Event contracts are reviewed source code: edit the contract, regenerate every derived artifact, then write the handwritten code the change requires. Never hand-edit generated output. Never deserialize untyped JSON at a process boundary — payloads cross boundaries as generated types or validated models.

Generated artifacts carry exactly four properties: reproducible from their contract source, never authoritative, committable for build ergonomics, and always regenerable under a CI drift check. A generator defect is resolved by re-pinning or replacing the generator in the lockfile, never by patching its output. Never add `gen/` to the ignore file — the drift check needs the committed tree to compare against. A single-package project with no external contract surface has no `contracts/` tree, and this rule does not apply to it.

- contract technology per boundary, Python-side boundary ownership, generated-code authority, handwritten-versus-generated, the repository layout, modular contract source, drift semantics, definition of done, and the worked correct/incorrect examples plus the pinned `just` recipe set — [Contract-First Architecture](../reference/contracts.md)
- the contract-conformance suite and the CI drift gate — [Contract Testing Validation](../reference/contract_testing.md)
