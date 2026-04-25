# Substrates API Specification

This directory contains the language-neutral specification for Substrates — a deterministic signal
circulation infrastructure.

## Contents

- **[SPEC.md](SPEC.md)** — The Substrates Specification (Version 2.0 — Draft)
- **[RATIONALE.md](RATIONALE.md)** — Design Rationale (non-normative companion to SPEC.md)

## Purpose

The specification defines structural primitives, behavioral contracts, and lifecycle semantics for
constructing computational networks in which typed values flow through governed topologies with
precise ordering guarantees.

It is independent of any programming language, runtime, or transport mechanism. A conformant
implementation may be realized as an in-process library, a networked service, or any combination
thereof.

## Scope

The specification covers:

- **Core abstractions** — Cortex, Circuit, Conduit, Pool, Pipe, Receptor, Flow, and their
  relationships
- **Behavioral contracts** — Deterministic ordering, emission semantics, cascading priority,
  temporal validity
- **Lifecycle management** — Scopes, resource cleanup, circuit shutdown sequences
- **Identity system** — Names, subjects, extents, and canonical identity guarantees
- **Tenure model** — Instance retention semantics (Interned, Ephemeral, Anchored, Context-scoped)
- **Conformance requirements** — Required types, operations, and behavioral invariants

## Reference Projection

The Java projection of this specification can be
found [here](https://github.com/humainary-io/substrates-api-java). Appendix A.2 of the spec
documents Java-specific binding decisions that are non-normative for other language projections.

## License

Copyright © 2025 William David Louth. Licensed under the Apache License, Version 2.0.
