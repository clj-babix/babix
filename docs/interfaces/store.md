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
- **Content addressing**: Store paths are derived from the hash of their inputs + build process (or equivalent).
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

### Two-Layer Hashing Model (captured from design sessions)

Store paths use two distinct hashes for clarity and caching:

- **Derivation hash**: A deterministic hash over the normalized Derivation Description + the hashes/identities of all its locked specification inputs and sources. This identifies the *plan* to produce something. Any change to the description or any input produces a new derivation hash (and thus new output paths).

- **Output hash**: The content hash of the actual bytes written to a particular promised output location during realization. This is what ends up in the final immutable store path for that output.

Fixed-output derivations (sources, vendored dependency trees, etc.) carry a pre-declared expected content hash. Their output hash can be that declared hash. They participate in the derivation hash of anything that consumes them, while still being independently cacheable by content.

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

---

This interface is deliberately minimal on the surface but carries enormous weight. Getting the contracts for roots, references, and atomicity right will determine whether higher layers can deliver on the "reproducible systems" promise.