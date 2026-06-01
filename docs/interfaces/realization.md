# Realization Interface

**Status:** Initial draft.  
**Scope:** Mandatory for turning derivation descriptions into real, reproducible store artifacts under sandboxing.

## Purpose

The Realization Interface is the contract between "I have a description of what to build" and "here are the bytes in the store."

This is where sandboxing, hermeticity, and reproducibility are actually enforced. It is the primary pluggable point for different isolation mechanisms (landlock, bubblewrap, user namespaces, future mechanisms, OS-specific jails, etc.).

The core orchestrates a clear sequence:
1. Sandbox preparation package (or backend) is invoked first to create the hermetic view with promised stable output paths visible.
2. The actual builder package (named via `:builder`) is then executed inside that prepared environment.

All build logic, phase models, and conventional environment setup live in the builder package (or packages it depends on), not in the core or the sandbox preparation layer.

## Core Responsibilities

- Take a Derivation Description + resolved inputs.
- Invoke the chosen sandbox preparation package (if any) or a suitable realization backend to create a hermetic, isolated environment that satisfies the derivation's `:sandbox` requirements. This step prepares stable absolute paths for all declared outputs (the equivalents of `$out`, `$man`, etc.) so the build process sees them as immutable destinations from the start.
- Execute the builder (the package named via `:builder`) inside that prepared sandbox.
- Capture the contents written to the promised output locations.
- Register the results in the Store under content-addressed, immutable paths (a derivation hash capturing the description + all inputs, plus an output hash for the actual result).
- Record enough information for provenance and reproducibility verification.

The build is intended to behave like a pure function: same locked inputs + same derivation description + same sandbox policy must produce bit-identical results. Side effects are confined to the prepared sandbox.

In practice we distinguish *rebuildability* (re-executing the plan succeeds and produces a usable result) from *bitwise reproducibility* (the bytes are identical). Large-scale empirical studies of functional package managers show rebuildability >99% even when bitwise identity is 69–91%. The sandbox + exact input identities are what deliver the high rebuildability; bitwise identity additionally requires toolchain discipline (SOURCE_DATE_EPOCH, path stripping, deterministic code generation). Realization records must capture enough information for `babix explain` to diagnose the difference.

## Key Requirements

- **Sandboxed by default**: The builder must not have uncontrolled access to the host filesystem, network, or other processes unless explicitly declared in the derivation.
- **Hermetic inputs**: Only the inputs declared in the derivation description may be visible to the build (modulo what the sandbox mechanism itself provides).
- **Stable output paths ("$out" convention)**: The sandbox preparation step must make the final immutable output locations visible to the builder process as stable absolute paths before the builder runs. The builder writes its results into these promised locations (classically via an environment variable `$out` and similar for other outputs). Nothing outside the sandbox may mutate them afterward.
- **Deterministic output**: For the same inputs + same builder logic + same sandbox policy, the output should be bit-for-bit identical (or we must be able to explain why it is not).
- **Observable & explainable**: Realization must produce structured records that feed the provenance/explainability system.

**Bootstrap / host-environment precondition**

Realization assumes the presence of the host kernel (with its syscall ABI) and a minimal bootstrap seed (or a previously-bootstrapped `babix` binary + builder). The first realization in any new Babix installation is special: it ultimately depends on the seed as the only pre-compiled artifact in the chain. The seed is a host-environment assumption (analogous to the kernel and syscall ABI), not a derivation and not a Builder package; it lies outside the Merkle-DAG and its hash is the ultimate stable identifier. The seed is a pure assembler (hex digits, labels, and comments in; flat machine code out). Everything above it is built from source in a fully auditable 15-phase ladder and is verified by content-hash pinning (the seed is executed as the builder of a derivation whose output hash must match the seed's own hash exactly). Subsequent realizations are ordinary and use Builder packages from the store.

**Controlled impurity for source-fetch steps (the escape hatch)**

Source-fetch (and similar) derivations are the only place where the sandbox deliberately grants impurity (network, non-deterministic clocks, etc.). These are ordinary derivations — not a special "fixed-output" kind — that declare an expected content hash; their sandbox policy (or the builder package they name) permits the necessary escape for that step only. After the fetch step completes, all downstream realization is again pure with respect to the pinned identity. This is the minimal, auditable exception that makes the overall model practical. Babix expresses this uniformly via the multi-derivation graph and per-derivation sandbox policy, rather than via a distinguished derivation variant in the core (an improvement over the classic mechanism described in Dolstra 2006). See the Derivation Description Interface for the data shape and the Store Interface for how these pinned artifacts participate in two-layer hashing.

## Pluggability Intent

Different realizations may offer different guarantees:

- Strong sandbox (landlock + user namespaces)
- Weaker but faster (overlayfs + chroot for trusted builders)
- OS-specific (jails on BSD, whatever Darwin offers)
- Future mechanisms (gVisor, Firejail successors, hardware-backed, etc.)

The interface must allow the derivation to declare its minimum requirements and allow the system (or user) to choose a suitable backend.

## Open Questions

- What is the exact protocol for invoking the sandbox preparation step and then the builder? (How are promised output paths communicated? What does the sandbox package receive vs. what the builder receives?)
- How do we handle "impure" but declared escape hatches (network access for some fetchers, specific devices, etc.) without weakening the whole model?
- How much of the "environment setup" (PATH, library paths, wrappers, etc.) is the responsibility of the builder package versus any supporting setup packages it depends on?
- Can realization backends (different sandbox mechanisms) be mixed in one store (some derivations realized with strong sandbox, others with a faster trusted path)?
- What happens when no available backend satisfies the derivation's sandbox requirements?
- How are "fixed-output" style derivations (e.g., source downloads whose content hash is known in advance) represented so they participate cleanly in the two-layer hashing (derivation hash + output hash) while remaining cacheable?

## Relationship to Other Interfaces

- **Derivation Description**: Primary input (especially the `:builder` and `:sandbox` sections). Sandbox preparation (creating the hermetic view and promised output paths) occurs before the builder is invoked; detailed policy lives with the builder package.
- **Store**: Only successful realization writes to the store. Uses two-layer hashing (derivation hash identifies the plan and promised paths; output hash identifies the actual result).
- **Input Locking**: Resolved inputs are presented to the sandbox. All references have already been materialized in-place with stable identifiers.
- **Provenance**: Realization is the moment when most provenance data is generated. Records must capture which sandbox preparation package/backend and which builder package were used.

## Current Scope Limitation

For the initial focus, we do not need remote or distributed realization. Local sandboxed execution (with at least one strong backend) is sufficient. The interface should be designed so that remote or delegated realization is not ruled out later.

---

This interface is where the "remove the cruft while keeping the hard guarantees" battle is largely won or lost. The design of the handoff between core, sandbox preparation package, and builder package — plus the data-driven contracts for invocation and output capture — will determine how much accidental complexity leaks into the daily experience of using the system.