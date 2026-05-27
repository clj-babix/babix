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

- Define and enforce the contracts (the five interfaces).
- Provide a minimal evaluation engine for derivation descriptions (data + limited escape hatches into pure functions).
- Orchestrate the flow: locking + full materialization of references (via in-place stable identifier replacement) → sandbox preparation (via package or backend) → builder invocation inside the prepared hermetic view → store registration.
- Guarantee provenance and explainability across the system.

See `docs/CONTEXT.md` for the current authoritative terminology and distilled decisions. The core changes only when the stable interfaces prove insufficient; almost all policy, builder logic, and higher-level abstractions live in the package collection as ordinary derivations.

The core does **not** contain:
- Language-specific build knowledge, phases, hooks, or any centralized build model (these live in ordinary builder packages in the collection).
- Detailed sandbox policy or output conventions (primarily supplied by the chosen builder package).
- Most environment setup logic inside the sandbox.
- Concrete sandbox implementations (only the interface and orchestration).

All build policy and conventions are supplied as ordinary packages. A derivation names its builder via keyword into its specification inputs. The core remains agnostic to any particular build model.

The vast majority of capability and policy lives in the package collection (builders, activators, fetchers, environment composers, higher-level frameworks). The core changes only when the stable interfaces prove insufficient. No builder is forced to inherit from a privileged parent; shared logic is ordinary dependencies.

Multi-step workflows (e.g., fetch source with expected hash → fetch dependencies → build) are expressed as small graphs of ordinary derivations that can live together in one authoring file for convenience. The locked forms are independent and fully explicit. Authoring sugar (whole namespaces, compact forms, local names) must normalize to explicit, identifier-pinned descriptions before locking and hashing.

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

These interfaces are the primary architectural boundaries. See the individual documents in `docs/interfaces/` for the current contracts. All five are subject to the cross-cutting provenance and explainability requirements. The core changes only when these stable interfaces prove insufficient for the package collection.

1. **Derivation Description Interface**  
   The data contract for a build plan. Primarily plain EDN. Explicit about inputs (specification vs. categorized), builder and activator (referenced by keyword into the closed input set), sandbox requirements, and runtime environment expectations. Higher-level builders produce instances of this shape. No language-specific build knowledge or phase models live here.

2. **Store Interface**  
   Content-addressed, immutable storage with reference tracking and roots. The single source of truth for artifacts. Uses two-layer hashing (derivation hash for the plan, output hash for the result) and supports promised paths known before realization.

3. **Realization Interface**  
   The contract for sandboxed execution. The main pluggability point for different isolation mechanisms (landlock, bubblewrap, OS-specific, future). Realization first invokes sandbox preparation (to create the hermetic view and promised output paths) and only then the builder inside that environment.

4. **Input Locking & Resolution Interface**  
   Declaration and locking of external inputs with explicit source modes (`:worktree`, `:git-head`, `:archive`, `:path`, etc.). The flakes equivalent, designed for honesty, local ergonomics, and visible reproducibility level. No implicit Git behavior.

5. **Environment Presentation Interface**  
   How store artifacts become runnable environments. Activation logic lives in activator packages (parallel to builders). A derivation can name an `:activator` (resolved via the input graph exactly like `:builder`). Activation is a first-class concern on *every* derivation, not a special case. This keeps the core tiny while allowing very different activation strategies (dev shell, home-manager-style, system-style, etc.) to be supplied as ordinary packages. Collector-style targets are ordinary derivations that compose activations from their inputs.

**Provenance and Explainability** is treated as a cross-cutting requirement. Every interface must produce or preserve enough information for `babix explain` to be genuinely useful. There is no separate "provenance engine" — it emerges from the combination of derivation descriptions (with explicit closed specification inputs), realization records (including which sandbox preparation package and builder package were used), and store metadata.

**Locked Derivation Descriptions** are fully explicit and self-contained. A full static walk over the description discovers all references (including inside strings and builder-private data). Before the derivation hash is computed, every reference is replaced *in-place* with a stable identifier (a small map preserving the original logical name plus the derivation hash or store identity). This follows the Spec-ulation reference/identifier distinction: authoring forms may use convenient names and whole namespaces; the locked form pins everything to immutable identities while preserving surrounding data shape. The locked artifact is the ground truth for hashing, realization, provenance, and `babix explain`.

**Specification inputs vs. categorized inputs** are deliberately distinct. Specification inputs (under `:inputs`) form the closed scope of concrete package instances available to the derivation. Categorized inputs (`:build-inputs`, `:host-inputs`, `:propagated-host-inputs`) are references into that scope describing architectural and runtime roles. This separation keeps the input graph honest and supports future cross-compilation without collapsing concerns.

**Two-layer hashing** is used throughout: a derivation hash (over the normalized description plus the identities of all locked specification inputs and sources) identifies the plan; an output hash (content hash of the bytes written to a promised location) identifies the actual result. Promised output paths are known before the builder runs. Fixed-output steps (source fetches, vendored dependency trees, etc.) are explicit derivations carrying a declared expected content hash.

**Sandbox preparation is a distinct pre-builder step.** The core (or realization backend) first invokes a sandbox preparation package (or backend) to establish the hermetic view and make stable absolute output paths visible inside the sandbox. Only then is the builder package (named via `:builder`) invoked inside that prepared environment. Sandbox policy and conventional environment setup (PATH, wrappers, etc.) are supplied primarily by the chosen builder package or its dependencies, not by the core or repeated in every derivation. This separation is a deliberate guardrail against core accretion.

**Single primary output with FHS-style layout.** Derivations declare one primary `out` (plus logical siblings only when genuinely required). Inside `out` the package follows a conventional FHS-like layout (`bin/`, `lib/`, `include/`, `share/man/`, etc.). Composition across packages into coherent roots or environments happens at activation time via activator packages, not by proliferating outputs on every derivation.

**Collectors are ordinary derivations.** A collector (home-style root, project environment, etc.) has the same shape as any other derivation: inputs, categories, optional builder and/or activator, outputs. Large roots and package namespaces can remain lazy at the graph level; per-derivation closure (full materialization and hashing) is required only for the specific derivations that are realized. A whole package collection may be supplied as a single specification input for ergonomics; only actually referenced members are materialized for a given derivation.

**Unified small data-driven contracts** govern all pluggable capabilities the core invokes directly (builders, sandbox/isolator providers, activators). Each publishes a minimal static manifest (kind, namespace/name, entrypoint, contract version) plus a small per-kind payload. This keeps the core's invocation model tiny and uniform while allowing the package collection to evolve new capabilities without core changes.

### The Five Core Interfaces

These interfaces are the primary architectural boundaries. See the individual documents in `docs/interfaces/` for the current contracts. Provenance and explainability requirements apply to all of them.

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

The per-interface grill sessions have been conducted and the major mechanisms (in-place stable identifier materialization, specification vs. categorized inputs, two-layer hashing, sandbox preparation separation, single primary output, ordinary collectors, unified pluggable contracts, and the absence of any privileged genericBuild or hook system in the core) are now recorded in this document, `docs/CONTEXT.md`, and the five interface specifications.

Remaining higher-level risks and open questions include:

- How far can the "users experience this as Babix, not Clojure" principle be taken when the primary representation is EDN data?
- Can we design the derivation description and builder contract such that common language builders remain low-friction without re-introducing hidden behavior?
- What is the minimal viable provenance model that still delivers on the "explain everything" promise?
- How do we handle the bootstrap problem (building the initial package collection) and the initial package collection itself?
- What does the user-facing project configuration and target declaration format actually look like for normal projects (the equivalent of `flake.nix` + `flake.lock` plus `babix init` output)?
- Trust and distribution model for locked inputs and pre-built artifacts (even a minimal local-first story).
- Detailed CLI surface, `babix explain` question catalog, and exact activator/collector composition contracts.

See `docs/CONTEXT.md` (Key Decisions and Key Concepts sections) for the current authoritative stance and terminology. The interface documents in `docs/interfaces/` contain the detailed contracts.

---

This design is intentionally minimal at the core and maximal at the edges. The bet is that a small set of well-chosen interfaces, combined with a data-first philosophy and a package collection that is allowed to grow freely, can deliver both the power of Nix/Guix and a dramatically better experience.