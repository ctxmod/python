---
name: regenerate_contracts
description: "Recover from generated-output drift — read the diff, classify it as a hand edit or an unregenerated contract change, fix forward through the contract source, and re-run the whole chain before landing. Use when: drift check failed; regenerate contracts; generated output drift; gen out of date; contract drift; stale generated code."
---

# Regenerate Contracts

Recover from a failing generated-output drift check without silencing it.

## Purpose

Use when the drift check reports a difference between the committed generated tree and a fresh regeneration, when a contract change landed without its regenerated output, or when generated output was edited by hand. The drift check is a failure, never a warning, and it is always fixed forward — through the contract source and the generator — never by editing the generated file to match, and never by excluding the path from the check.

This skill sequences the recovery. The authority behind each step — why generated output is never authoritative, what the four generated-artifact properties are, what the gate must prove — lives in the reference material linked at the end and is not restated here.

## Workflow

**1. Read the diff before changing anything.** Regenerate into a temporary location and compare against the committed generated tree. The diff itself tells you which of the two failure modes you are in; guessing at this step is how a hand edit gets re-applied.

**2. Classify the failure mode.**

| What the diff shows | What happened | Fix |
|---|---|---|
| Fresh regeneration lacks something present in the committed tree | Generated output was edited by hand | Discard the hand edit; carry the intent into the contract source |
| Fresh regeneration has something the committed tree lacks | A contract change landed without regeneration | Regenerate and commit the output with the contract change |
| The whole tree differs on an untouched contract | The pinned toolchain moved | Re-pin the lockfile; the lockfile, not the local environment, selects which revision runs |

**3. Fix forward through the contract source.** If the hand edit expressed a real requirement, that requirement is a contract change: add it to the modular contract source, advance the `contract_revision` under the evolution rules, and let regeneration produce it. A hand-transformed artifact with no generation mechanism is two contracts claiming one boundary.

**4. Re-run the whole chain, in order.** Lint and bundle the contract source, run the breaking-change check for the asynchronous contracts, regenerate every output target, re-run the drift check, then run the conformance suite. A partial re-run is the most common way a second drift lands: the bundle, the DTOs, and the bindings are separate targets, and each one needs its own regenerate-and-compare pair.

**5. Land contract and generated output together.** One change set carries the contract change and its regenerated output. Splitting them leaves an intermediate revision where the two disagree, which is the state the check exists to make impossible.

## Escalation

If regeneration is not reproducible — the same inputs producing different output across runs or machines — the problem is the generator, not the contract. Re-pin or replace the generator in the lockfile and record the change with the toolchain material. Never stabilize a nondeterministic generator by committing one of its outputs and excluding the path from the check: that converts a build failure into a silent divergence.

If a needed change cannot be expressed in the contract at all, that is a contract-technology question for the boundary, not a case for hand editing.

## Context

### Generated-code authority and the drift semantics

[Contract-First Architecture](../../reference/contracts.md) (required reading)

### The contract gate, step by step

[Contract Testing Validation](../../reference/contract_testing.md) (required reading)

### Toolchain pinning and recipes

[Python Toolchain](../../../toolchain/reference/tooling.md)
[Project Setup](../../../toolchain/reference/project_setup.md)
[Python Packaging](../../../toolchain/reference/packaging.md)
