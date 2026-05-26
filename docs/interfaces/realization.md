# Realization Interface

**Status:** Initial draft.  
**Scope:** Mandatory for turning derivation descriptions into real, reproducible store artifacts under sandboxing.

## Purpose

The Realization Interface is the contract between "I have a description of what to build" and "here are the bytes in the store."

This is where sandboxing, hermeticity, and reproducibility are actually enforced. It is the primary pluggable point for different isolation mechanisms (landlock, bubblewrap, user namespaces, future mechanisms, OS-specific jails, etc.).

## Core Responsibilities

- Take a Derivation Description + resolved inputs.
- Execute the builder inside a controlled, minimal environment.
- Produce outputs that match the declared store paths (or fail atomically).
- Record enough information for provenance and reproducibility verification.

## Key Requirements

- **Sandboxed by default**: The builder must not have uncontrolled access to the host filesystem, network, or other processes unless explicitly declared in the derivation.
- **Hermetic inputs**: Only the inputs declared in the derivation description may be visible to the build (modulo what the sandbox mechanism itself provides).
- **Deterministic output**: For the same inputs + same builder logic + same sandbox policy, the output should be bit-for-bit identical (or we must be able to explain why it is not).
- **Observable & explainable**: Realization must produce structured records that feed the provenance/explainability system.

## Pluggability Intent

Different realizations may offer different guarantees:

- Strong sandbox (landlock + user namespaces)
- Weaker but faster (overlayfs + chroot for trusted builders)
- OS-specific (jails on BSD, whatever Darwin offers)
- Future mechanisms (gVisor, Firejail successors, hardware-backed, etc.)

The interface must allow the derivation to declare its minimum requirements and allow the system (or user) to choose a suitable backend.

## Open Questions

- What is the exact protocol for invoking a builder? (executable + args + environment map + working directory + output path promises?)
- How do we handle "impure" but declared escape hatches (network access for some fetchers, specific devices, etc.) without weakening the whole model?
- How much of the "environment setup" (PATH, library paths, etc.) happens inside the sandbox vs. being prepared by the realization layer before entering the sandbox?
- Can realization backends be mixed in one store (some derivations realized with strong sandbox, others with a faster trusted path)?
- What happens when no available backend satisfies the derivation's sandbox requirements?

## Relationship to Other Interfaces

- **Derivation Description**: Primary input (especially the `:builder` and `:sandbox` sections).
- **Store**: Only successful realization writes to the store.
- **Input Locking**: Resolved inputs are presented to the sandbox.
- **Provenance**: Realization is the moment when most provenance data is generated.

## Current Scope Limitation

For the initial focus, we do not need remote or distributed realization. Local sandboxed execution (with at least one strong backend) is sufficient. The interface should be designed so that remote or delegated realization is not ruled out later.

---

This interface is where the "remove the cruft while keeping the hard guarantees" battle is largely won or lost. The design of the builder invocation contract and the sandbox policy language will heavily influence how painful or pleasant it is to write and maintain packages.