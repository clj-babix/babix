# Babix Design

**Status:** Initial draft — intended to be iterated on during the design process.  
**Scope:** The architecture for the scoped problem (reproducible development environments + derivations + locking), with clear extension points for later horizons.

---

## Overview

Babix is architected around a small, stable core that defines interfaces and evaluation semantics, with almost all policy, builder logic, and higher-level abstractions living in the package collection as ordinary derivations.

The system has two primary goals that must not be in tension:

1. Deliver real Nix/Guix-grade guarantees for reproducibility, provenance, and hermeticity.
2. Make the experience of using those guarantees feel like a well-designed, data-oriented tool rather than a foreign system with accumulated cruft.

## Architectural Layers

```
User Experience (CLI + data files)
          │
          ▼
Core Evaluation & Orchestration
  - Derivation Description Interface
  - Input Locking & Resolution
  - Environment Presentation
  - (cross-cutting provenance recording)
          │
          ▼
Pluggable Backends (behind stable interfaces)
  - Realization (sandboxing)
  - Store
          │
          ▼
Package Collection (the vast majority of logic)
  - Language builders (Cargo, Python, etc.)
  - Environment setup tools
  - Source fetchers, patchers, wrappers (as packages)
  - Higher-level templates and frameworks
```

### Core (Minimal Trusted Base)

The core is deliberately small. Its responsibilities are:

- Define and enforce the contracts (the five interfaces), including the Builder Provider resolution mechanism.
- Provide a minimal evaluation engine for derivation descriptions (data + limited escape hatches into pure functions).
- Orchestrate the flow between locking → realization → store → environment presentation.
- Guarantee provenance and explainability across the system.

The core does **not** contain:
- Language-specific build knowledge (these are supplied by Builder Provider packages).
- Most environment setup logic.
- Concrete sandbox implementations (only the interface).
- Any notion of "standard" phases or stdenv (explicit future discussion topic).

Input category names in the Derivation Description Interface use clear terms (`:build-inputs`, `:host-inputs`, `:propagated-host-inputs`) chosen early to be better than the legacy confusing names, even though full cross-compilation is a later concern.

### Standard Library (Tiny)

A small set of pure, stable helper functions (the "nixpkgs lib" equivalent) for common data transformations. This lives in the core and changes extremely slowly.

Because real Clojure is available in the places where code is written, the standard library does not need to re-implement a large builtin ecosystem.

### Package Collection (The Growth Surface)

This is where the system actually becomes useful and evolvable:

- All language builders are packages.
- All "makeWrapper" equivalents and environment injection tools are packages.
- Higher-level abstractions (the things that would be flake-parts or dream2nix today) are packages or small collections.
- Even many source fetchers, unpackers, and patchers can be expressed as packages following the interfaces.

This design is deliberate: it allows the collection to grow rapidly without requiring changes to the core or its interfaces.

## The Five Core Interfaces

These interfaces are the primary architectural boundaries. See the individual documents in `design/interfaces/` for current thinking.

1. **Derivation Description Interface**  
   The data shape of a build plan. Primarily EDN. Explicit about inputs, builder, sandbox requirements, and runtime environment expectations. Higher-level builders produce instances of this shape.

2. **Store Interface**  
   Content-addressed, immutable storage with reference tracking and roots. The single source of truth for artifacts.

3. **Realization Interface**  
   The contract for sandboxed execution. The main pluggability point for different isolation mechanisms (landlock, bubblewrap, OS-specific, future).

4. **Input Locking & Resolution Interface**  
   Declaration and locking of external inputs with explicit source modes. The flakes equivalent, designed for honesty and local ergonomics.

5. **Environment Presentation Interface**  
   How store artifacts become runnable environments. Activation logic lives in activator packages (parallel to builders). A derivation can name an `:activator` (resolved via the input graph exactly like `:builder`). This keeps the core tiny while allowing very different activation strategies (dev shell, home-manager-style, system-style, etc.) to be supplied as ordinary packages.

**Provenance and Explainability** is treated as a cross-cutting requirement. Every interface must produce or preserve enough information for `babix explain` to be genuinely useful. There is no separate "provenance engine" — it emerges from the combination of derivation descriptions, realization records, and store metadata.

## Data Model Philosophy

- The default representation for almost everything is plain EDN data.
- Derivation descriptions are large, explicit maps (the spiritual successor to `mkDerivation` calls).
- Small pure functions are the mechanism for transformation, overriding, and creating higher-level abstractions.
- Real code (Babashka/Clojure scripts) appears primarily inside builder contexts and as escape hatches for complex logic.
- The system is designed so that a user can stay in the data layer for normal work and only drop into functions when they need power.

This is the opposite of "Nix but with Clojure syntax." The goal is Nix/Guix *semantics* with a data-oriented composition model.

## Extensibility and Pluggability

Babix has two primary extension mechanisms, kept deliberately separate:

- **Interface pluggability** (core concern): Different implementations of Realization, Store, etc. This is for fundamental mechanisms (sandboxing strategy, storage backend, future OS support).
- **Package-level extensibility** (ecosystem concern): New builders, new environment tools, new templates, new frameworks. This is where the vast majority of innovation is expected to happen.

The core's job is to make the second mechanism powerful and safe, not to compete with it.

## Key Flows (High Level)

### `babix develop` (or equivalent)
1. Read environment description + locked inputs.
2. Resolve inputs (Input Locking interface).
3. Ensure required derivations are realized (or realize them).
4. Present the environment via the Environment Presentation interface.
5. Record provenance for later explanation.

### Building a package
1. Derivation description is evaluated (data + any pure transforms).
2. Inputs are locked and materialized.
3. Realization backend is selected that satisfies the derivation's sandbox requirements.
4. Builder executes inside the sandbox.
5. Outputs are registered in the Store with full provenance.

### `babix explain`
Walks the graph of derivation descriptions, realization records, and store references to answer "why" questions. This must work even when different pluggable backends were used.

## Non-Goals in the Current Architecture

- Hiding the data model behind a "friendly" configuration language for normal users.
- Making the core understand Cargo, Python, npm, etc.
- Building a full module system or system activation model (explicitly deferred).
- Creating a large set of core builtins (Clojure provides the language power where needed).
- Optimizing for the "I just want to write a shell.nix equivalent in five lines" experience at the expense of the long-term model.

## Risks and Open Questions

- How far can the "users experience this as Babix, not Clojure" principle be taken when the primary representation is EDN data?
- Can we design the derivation description and builder contract such that common language builders remain low-friction without re-introducing hidden behavior?
- What is the minimal viable provenance model that still delivers on the "explain everything" promise?
- How do we handle the bootstrap problem (building the initial package collection)?
- What does the user-facing file format(s) actually look like for normal projects (the equivalent of `flake.nix` + `flake.lock`)?

These are expected to be refined through further design work and the per-interface grill sessions.

---

This design is intentionally minimal at the core and maximal at the edges. The bet is that a small set of well-chosen interfaces, combined with a data-first philosophy and a package collection that is allowed to grow freely, can deliver both the power of Nix/Guix and a dramatically better experience.