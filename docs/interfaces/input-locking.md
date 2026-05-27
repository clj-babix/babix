# Input Locking & Resolution Interface

**Status:** Initial draft.  
**Scope:** Mandatory for reproducible "flake-like" input management without the pain of flakes.

## Purpose

This interface governs how external inputs to a derivation graph are declared, locked to specific versions/revisions, and materialized into usable forms (sources in the store, or references to other derivations).

It is the Babix equivalent of `flake.nix` + `flake.lock` + the fetching mechanism, but designed from the start to be explicit, data-driven, and explainable.

## Core Responsibilities

- Allow derivations (and user environments) to declare their external dependencies (git repositories, tarballs, URLs, other Babix flakes/projects, etc.).
- Produce a lockfile that captures the exact resolved state with enough information for perfect reproduction later.
- Materialize the locked inputs (fetch into the store, verify hashes, etc.).
- Support different "source modes" for the same input (worktree for development, git HEAD, specific revision, archive, local path).

## Key Concepts (from earlier discussion)

- Explicit source modes: `:worktree`, `:git-index`, `:git-head`, `:archive`, `:path`.
- Lockfile records not just the revision, but the **reproducibility level** and dirty state.
- No implicit Git behavior.

## Open Questions

- What is the exact shape of an input declaration and the resulting lock entry?
- How do we handle "this input is itself a Babix project with its own lockfile"?
- What is the story for private repositories and authenticated fetches?
- How do we support "floating" inputs when the user explicitly wants latest (while still recording what "latest" meant at lock time)?
- How does resolution interact with the Store (do we always materialize sources into the store, or are there cases where we can reference them directly)?
- How do we make the lockfile itself human-reviewable and diffable while still being machine-authoritative?

## Relationship to Other Interfaces

- **Derivation Description**: Sources and input derivations are declared here. All references are materialized in-place with stable identifiers before the derivation hash is computed.
- **Store**: Resolved sources usually end up as store paths.
- **Realization**: Uses the resolved inputs.
- **Environment Presentation**: User environments can also declare inputs (for `develop`).
- **Provenance**: Lock records (including reproducibility level and dirty state) are essential for `babix explain`.

## Scope Note

This interface is central to the "start with nix develop + flakes strengths" goal. Getting the source modes + lockfile ergonomics + explainability right is one of the highest-leverage areas for making Babix feel like a strict improvement over flakes rather than a port.

---

The goal is to keep the power of flakes (reproducible inputs) while removing the ceremony, surprise behavior, and poor local development experience.