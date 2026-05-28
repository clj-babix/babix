# Store Interface

**Status:** Initial draft — subject to deep review.  
**Scope:** Mandatory core interface for the scoped Babix problem (derivations + develop + locking).

## Purpose

The Store is the single source of truth for all built artifacts and their metadata. It is the "sacred" component that gives Nix/Guix their strongest guarantees: immutability, content addressing, reference tracking, safe sharing, and the ability to roll back or GC safely.

Babix must have a real store (not just an artifact cache) if it wants to deliver on provenance, reproducibility claims, and trustworthy environments.

## Core Responsibilities

- Content-addressed storage of derivation outputs.
- Maintaining reference graphs (what depends on what) for GC and provenance.
- Managing roots (including the equivalents of profiles and generations for environments).
- Providing atomicity and consistency guarantees during realization.
- Enabling safe sharing and deduplication across environments and machines.

## Key Properties (Non-Negotiable)

- **Immutability**: Once a store path exists, its contents never change.
- **Content addressing as primary model**: Babix adopts content-addressed storage as the primary and foundational model from the outset. The derivation hash identifies the immutable plan (the locked derivation description plus all input identities). The output identity is the cryptographic content hash of the actual bytes written to the promised location during realization. Realisation records (or the Babix equivalent) map plan identities to concrete content-addressed output paths and their dependency closures. Purely input-addressed (derivation-hash-only) paths are not used as a foundational path. This decision is recorded in the "Decisions from RFC Review (2026)" section of [docs/CONTEXT.md](docs/CONTEXT.md). Babix adopts the approach cleanly from day one.
- **Reference tracking**: The store knows which paths reference which other paths.
- **Roots as first-class concept**: Garbage collection only collects paths that are unreachable from active roots.
- **Atomic realization**: A derivation either fully succeeds into the store or leaves no partial state.

## Initial Scope Notes

For the current focus (nix-develop-like + derivations + locking):

- We need enough of the store to support development environments and building user packages.
- Full multi-user / privileged system store semantics can be deferred, but the interface should not make them impossible later.
- Binary cache / remote store behavior can initially be handled via the Input Resolution interface or as a simple fetch-into-local-store mechanism.

## Open Questions (for dedicated grill-me)

- What is the exact addressing scheme? (pure content hash, derivation hash + output name, something else?)
- How are roots represented and persisted? (symlinks in a roots directory? a database? explicit root objects?)
- What are the minimum durability and consistency requirements for a local store in v1?
- How does the store interface accommodate alternative backends (local filesystem, content-addressable object storage, future distributed stores)?
- How much of the "generation" concept lives in the Store vs. the Environment Presentation layer?
- Trust model: how do we validate that a store path really was built from the claimed inputs when it comes from a remote source?

### Two-Layer Hashing Model and Content-Addressed Storage as Primary Model (captured from design sessions)

Store paths use two distinct hashes for clarity and caching:

- **Derivation hash**: A deterministic hash over the normalized Derivation Description + the hashes/identities of all its locked specification inputs and sources. This identifies the *immutable plan*. Any change to the description or any input produces a new derivation hash (and thus new output paths).

- **Output hash**: The cryptographic content hash of the actual bytes written to a particular promised output location during realization. This is what ends up in the final immutable store path for that output and serves as the primary identity of the artifact.

Babix adopts content-addressed storage as the primary model from the outset. Both the locked derivation description (the plan) and the actual produced content (the result) are identified by cryptographic hashes. Realisation records map plan identities to concrete content-addressed output paths and their dependency closures. Purely input-addressed (derivation-hash-only) paths are not used as a foundational path. This decision is recorded in the "Decisions from RFC Review (2026)" section of [docs/CONTEXT.md](docs/CONTEXT.md). Babix adopts the approach cleanly from day one.

Fixed-output-style steps (sources, vendored dependency trees, etc.) are expressed as ordinary derivations that declare an expected content hash up front and receive controlled sandbox privileges for the necessary impurity (network). Their output is pinned by that declared content hash. They participate in the derivation hash of anything that consumes them, while still being independently cacheable by content.

The sandbox preparation step makes the final store paths (derived from derivation hash + output name) visible as stable absolute paths before the builder runs. The builder writes into them; the Store later registers the contents under the output-hash-based name (or re-uses the path if it matches).

This supports the reproducibility pillar and Spec-ulation-style stability: output paths are knowable and stable once the locked inputs + description exist.

## Relationship to Other Interfaces

- **Derivation Description**: Derivations compute the store paths they will produce (via two-layer hashing: derivation hash for the plan, output hash for the result). Promised paths are known before realization.
- **Realization**: Realization is the only operation that is allowed to write into the store (under strict controls). Sandbox preparation happens first; the builder writes into promised paths inside the prepared environment.
- **Input Locking & Resolution**: Resolved inputs are materialized into the store.
- **Environment Presentation**: Environments are built by creating roots into specific store paths or sets of paths.
- **Provenance**: The store + realization records are the ground truth for explainability. All references in a locked derivation are already pinned to stable identifiers.

## Non-Goals (Current Scope)

- Implementing a full NixOS-style system profile with generations and atomic switch for `/`.
- Sophisticated distributed or multi-tenant store semantics.
- Built-in binary cache serving (can be a separate package or service later).
- Encoding specific on-disk permission or visibility policies (e.g., group readability, discovery resistance via `o-r`, 1735-style defaults) into the core Store contract. Permission and visibility policy is explicitly deferred and is the concern of the concrete store implementation or a higher policy layer (see the "Decisions from RFC Review (2026)" section in [docs/CONTEXT.md](docs/CONTEXT.md)).

## The Stable Store Interface Library

The concrete implementation of this interface (and related store-interaction contracts) is provided by the **Stable Store Interface Library**. This is a stable, versioned library whose sole responsibility is to implement the interactions required by the Babix Store Interface. Any Babix implementation, alternate runtime, higher-level tool, or even a tool written in another language can depend on this library for all store interaction while providing its own evaluation, realization orchestration, or user-experience layers. The library is the engineering boundary that keeps the Store Interface contract independent of any one runtime or language (the framing was informed by store-layering work in the ecosystem; see the "Decisions from RFC Review (2026)" section of [docs/CONTEXT.md](docs/CONTEXT.md)).

---

This interface is deliberately minimal on the surface but carries enormous weight. Getting the contracts for roots, references, atomicity, and content-addressed realisation records right will determine whether higher layers can deliver on the "reproducible systems" promise.