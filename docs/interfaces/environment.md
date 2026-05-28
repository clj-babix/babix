# Environment Presentation / Activation Interface

**Status:** Initial draft.  
**Scope:** Mandatory for delivering the "nix develop" and "enter a reproducible environment" experience.

## Purpose

This interface defines how artifacts that exist in the store (outputs of derivations) are turned into usable, runnable environments for development, testing, running commands, or long-lived user environments.

Activation is a first-class concern on *every* derivation, not a special case. A derivation may declare zero or one activator (referenced by keyword into its specification inputs and resolved the same way as builders). Collector-style targets (home-like roots, project environments, etc.) are ordinary derivations whose primary purpose is to aggregate other derivations and compose activations from their inputs.

It is the reason `babix develop`, `babix run`, and `babix switch` (for non-system targets) can exist.

## Core Responsibilities

- Take a set of store paths (or a derivation + build plan) and produce an environment description.
- Materialize that environment so that processes can be launched inside it (setting PATH, library paths, environment variables, wrappers, etc.).
- Support different "activation" styles: temporary shell, persistent profile, one-shot command execution, etc.
- Record enough information for provenance (`babix explain` on an active environment).
- Activation packages declare their own privilege requirements (home-level, system-level, dev-shell, etc.). The core does not hardcode privilege rules.

Environments are ultimately backed by *roots* into the store. A root keeps the transitive runtime closure (the actual referenced store paths discovered via reference scanning) alive. Activator packages are responsible for producing a coherent root layout and activation script from the chosen derivation outputs; the Environment Presentation interface orchestrates the creation or switching of the root. This is the mechanism that makes "enter a reproducible development environment" and "switch to a new home profile" atomic, roll-backable, and explainable. See the Store Interface for roots and generations, and the Derivation Description Interface for runtime closure hygiene requirements.

## Key Tensions to Resolve in This Interface

- How much of the "how binaries find their libraries" problem is solved at environment activation time vs. at build time (rpath, patchelf, wrappers, LD_LIBRARY_PATH, etc.).
- The desire for minimal wrapper depth vs. the reality of dynamic linking on Linux.
- Supporting both "pure" environments (very close to the store artifacts) and "practical" environments (with user overrides, additional tools, etc.).

## Open Questions

- What is the data shape of an "environment descriptor"?
- How do we represent the difference between a temporary `develop` shell and a longer-lived activated profile?
- What is the activation model for non-root targets (home, dev, project-local)? How does it interact with existing shells and direnv-like tools?
- How do we handle the case where an environment needs setuid helpers or other privileged mechanisms (future system targets)?
- How does environment presentation compose with the Runtime Environment section declared inside individual derivations?

## Relationship to Other Interfaces

- **Derivation Description**: Derivations can declare expectations about their runtime environment. Every derivation may name an activator (or none). Collectors are ordinary derivations.
- **Store**: Environments are almost always roots or references into store paths.
- **Realization**: The outputs that environments consume come from realization.
- **Input Locking**: Environments can have their own locked inputs.
- **Provenance**: Which activator (if any) was used, and with which inputs, must be traceable through realization records and store metadata.

## Scope Note (Current Focus)

For the initial phase we are primarily concerned with development and user-level environments. System-level activation (the equivalent of `nixos-rebuild switch` or `darwin-rebuild`) is explicitly out of scope for now, but the interface should not make it impossible or require a total redesign later.

See `docs/CONTEXT.md` (Key Decisions and Key Concepts) for the current stance on activation as universal, activator packages, collectors as ordinary derivations, and the distinction between root construction and live mutation.

---

This interface is where the user actually feels the quality of the system. A beautiful derivation and store model can still produce a miserable daily experience if environment presentation is clunky, slow, or surprising.