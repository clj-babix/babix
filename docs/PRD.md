# Babix Product Requirements Document (Draft)

**Status:** Draft v0.1 — intended for internal iteration and RFC  
**Date:** 2025  
**Audience:** Nix-using Clojurians and early design reviewers  
**Related Documents:**
- [VISION.md](../VISION.md)
- [DESIGN.md](DESIGN.md)
- `docs/interfaces/` (core contracts)

---

## 1. Overview & Goals

**Core Goal**  
Create a reproducible, declarative, functional systems tool that delivers the strongest guarantees of Nix/Guix (content-addressed store, derivations, hermetic builds, generations/rollback, strong provenance) while using Clojure-shaped composition and data-first ergonomics.

**Positioning**  
Babix is not "Nix with Clojure syntax." It is a first-principles re-implementation that keeps the hard semantic guarantees while replacing the painful surfaces (Nix language ergonomics, flake ceremony, overlay confusion, rebuild opacity, split ontologies, accumulated wrapper cruft).

**Primary Success Statement**  
A competent developer should be able to define, lock, build, and enter a reproducible development environment (and build their own software inside it) with dramatically better explainability, lower ceremony, and clearer mental models than current Nix/Guix tooling, while still achieving equivalent or stronger reproducibility guarantees.

---

## 2. Scope (Current Horizon)

### In Scope
- Derivation model for building and packaging software (data-first, EDN primary representation)
- Builder and Activator providers as ordinary packages (pluggable via namespaced keywords)
- Flake-like concerns: explicit source modes, locking, input resolution, reproducibility levels
- Environment / target activation (dev shells, user environments, collector-style targets)
- Content-addressed local store with roots and provenance
- Unified target model (different roots and activation policies, not separate ontologies)
- Strong, first-class explainability (`babix explain`)
- Lean core + tiny stable standard library + package collection as the growth surface

### Explicitly Out of Scope (for this horizon)
- Full module system and service configuration (NixOS / Home Manager equivalent)
- Privileged system-level activation as primary focus (BabixOS ambitions are later horizon)
- Complete cross-compilation support (naming and basic architecture prepared, implementation deferred)
- Remote binary caches and distributed trust model (minimal local-first story first)
- Tight integration with or dependence on `bb.edn` / `deps.edn` as user concepts

---

## 3. User Personas & Primary Workflows

**Primary Persona: "Alex" — Experienced Nix user who is tired of the surface pain**  
Wants reproducible dev environments and to package their own software without fighting the language or hidden behavior.

**Secondary Persona: "Sam" — Clojurian or data-oriented developer**  
Wants to use a powerful reproducible tool without having to deeply learn a new language or accept folklore.

**Key Workflows (MVP-relevant)**

1. **Initialize and enter a locked dev environment**
2. **Add / override a package** (including version + source changes)
3. **Build a package** from source with full provenance
4. **Explain why something built or changed**
5. **Create a collector-style target** (home-like or project environment composed of many packages)

---

## 4. Functional Requirements

### 4.1 Derivation Description (Core Data Contract)
- Derivations are primarily plain EDN data.
- Must explicitly declare:
  - Inputs (categorized as `:build-inputs`, `:host-inputs`, `:propagated-host-inputs`)
  - Sources (with explicit source modes)
  - Builder (an ordinary package supplied as one of the specification inputs under `:inputs`; referenced by keyword inside the derivation). The core must not know language build conventions or phase models.
  - Activator (optional, via namespaced keyword)
  - Sandbox requirements
  - Runtime environment expectations
- **Specification inputs vs. categorized inputs** are deliberately distinct. Specification inputs (the `:inputs` map) form the closed scope of concrete package instances available to the derivation. Categorized inputs describe architectural and runtime roles within that scope.
- **In-place materialization with stable identifiers before hashing**: A full static walk discovers all references (including inside strings and builder-private data). Before the derivation hash is computed, every reference is replaced *in-place* with a stable identifier (a small map preserving the original logical name plus the derivation hash or store identity). Authoring forms may use convenient names and whole namespaces; the locked form is fully explicit and self-contained.
- **Two-layer hashing**: Derivation hash (over the normalized description plus locked input identities) identifies the plan and determines promised output paths. Output hash (content hash of bytes written to a promised location) identifies the actual result. Fixed-output steps (source fetches, vendored trees, etc.) are explicit derivations carrying a declared expected content hash.
- **Single primary output with FHS-style layout**: Derivations declare one primary `out` (plus logical siblings only when genuinely required). Inside `out` the package follows a conventional FHS-like layout. Composition across packages into roots or environments happens at activation time.
- Higher-level language builders (Cargo, Python, etc.) are packages that produce valid derivations.
- Stdenv-like conventions, if any, are packages, not core concepts.
- Multi-step workflows (source fetch with expected hash → dependency fetch → build) can be expressed as small graphs of ordinary derivations that may live together in one authoring file for convenience. The locked forms are independent and fully explicit. Authoring sugar (local names, whole namespaces, compact forms) must expand to fully explicit, identifier-pinned descriptions before locking and hashing.

### 4.2 Builders as Packages
- Builder logic is supplied by ordinary packages, not special-cased in the core.
- A derivation receives its builder (and any other builder-like packages it needs) explicitly as specification inputs under `:inputs`.
- The derivation references the desired builder by keyword into that closed input set.
- Builder packages evolve on their own cadence. There is no central registry or `providers.edn`-style discovery; the referencing derivation simply lists the specific builder package instance it wants to use.
- **Sandbox preparation is a distinct pre-builder step.** Realization first ensures a hermetic view with promised stable output paths (via a sandbox preparation package or backend). Only then is the chosen builder package invoked inside that prepared environment. Detailed sandbox policy and conventional environment setup (PATH, wrappers, etc.) are supplied primarily by the builder package or its dependencies.

No builder is required to inherit from a privileged parent or generic builder. There is no privileged `genericBuild`, phase runner, or hook system in the core. All phase models, hook systems, and language-specific conventions live in the package collection. The core must not know Cargo, setuptools, npm, CMake, Autotools, Python paths, or similar details. It must not source hooks or understand phase models. A generic phased builder may exist as an ordinary package that other builders can depend on, but it has no special status.

### 4.3 Activators as Packages
- Activation behavior (root layout, activation script, environment contributions) is supplied by activator packages.
- A derivation may declare zero or one activator.
- Collector derivations compose activations from their inputs.
- Activation packages can be written in Clojure/Babashka.

### 4.4 Input Locking & Resolution
- Explicit source modes (`:worktree`, `:git-head`, `:archive`, `:path`, etc.).
- Lockfile records reproducibility level and dirty state.
- No implicit Git behavior.

### 4.5 Store & Realization
- Content-addressed, immutable store with reference tracking and roots.
- Sandboxed realization (pluggable backends: landlock, bubblewrap, etc.).
- Realization is the only way to write to the store.

### 4.6 Environment Presentation & Activation
- Activation is a first-class concern on *every* derivation, not a special case.
- Activation behavior is supplied by activator packages (parallel to builders). A derivation may declare zero or one activator, resolved by keyword into its specification inputs.
- Collector derivations (home-style roots, project environments, etc.) are ordinary derivations that compose activations from their inputs. They have the same shape and are not a separate core kind.
- `develop` / `run` / profile-style targets are expressed as derivations using activators. Large roots can remain lazy at the graph level; per-derivation closure is required only for the derivations that are actually realized.

### 4.7 Explainability & Provenance
- Every important decision and transformation must be traceable.
- `babix explain` must be able to answer meaningful "why" and "what would change this" questions across derivations, builds, locks, and activations.

### 4.8 CLI (Minimal Surface)
Target commands (exact surface still evolving):
- `babix init`
- `babix update`
- `babix build [target]`
- `babix develop [target]`
- `babix run [target] -- <command>`
- `babix switch [target]`
- `babix explain [topic]`

Project-specific workflows belong in `bb.edn` (not part of Babix core commands).

---

## 5. Non-Functional Requirements

- **Explainability first**: Every mechanism must produce data usable by `babix explain`.
- **Data transparency**: The vast majority of important state must be readable, diffable EDN.
- **Minimal core**: The trusted base must stay small. Policy and language-specific logic live in packages.
- **Pluggability**: Sandboxing, storage, and realization strategies must be replaceable behind stable interfaces.
- **Reproducibility**: Same inputs + same derivation description + same realization policy must produce bit-identical or explainably-different results.
- **User model clarity**: Users should feel they are learning "Babix," not "Clojure + a new build system."

---

## 6. Architecture Alignment

See [DESIGN.md](../DESIGN.md), [CONTEXT.md](../CONTEXT.md) (the authoritative glossary and distilled decisions), and the five core interface documents in `docs/interfaces/`:

1. Derivation Description Interface
2. Store Interface
3. Realization Interface
4. Input Locking & Resolution Interface
5. Environment Presentation Interface

**Key Layering**:
- Core = interfaces + minimal evaluation + orchestration + provenance
- Tiny standard library = stable pure helpers
- Package collection = builders, activators, language support, stdenv-like constructs, higher-level frameworks

The core mechanisms (in-place stable identifier materialization before hashing, specification inputs vs. categorized inputs, two-layer hashing, sandbox preparation as a distinct step, single primary `out` with FHS layout, ordinary collectors, and unified small data-driven contracts for pluggable capabilities) are first-class architectural commitments. See DESIGN.md for the current detailed stance.

---

## 7. Known Open Questions & Risks

The per-interface grill sessions have been conducted. The major mechanisms (in-place stable identifier materialization, specification vs. categorized inputs, two-layer hashing, sandbox preparation separation, single primary output, ordinary collectors, unified pluggable contracts, and the absence of any privileged genericBuild or hook system in the core) are now recorded in DESIGN.md, CONTEXT.md, and the interface specifications.

Remaining higher-level open questions and risks (largely corresponding to the U* items from the design sessions):

- Concrete shape and ergonomics of the project configuration file(s) (`babix init` output, target declarations, environment definitions) and the local authoring surface.
- Bootstrap story for the first working Babix binary + initial package collection, and eventual self-hosting.
- Exact depth, question catalog, and data requirements for `babix explain`.
- Detailed CLI surface, command semantics, and target declaration syntax.
- Trust and distribution model for locked inputs and pre-built artifacts (even a minimal local-first story).
- Precise contract, privilege declarations, and composition rules for activator packages (especially collectors and large roots).
- How builder-provided defaults (sandbox policy, output conventions, environment setup) interact with explicit declarations and overrides in individual derivations.
- Performance characteristics of evaluation and realization for large graphs (laziness at collector/root level vs. per-derivation closure) and the exact algorithms for materialization.
- Documentation and onboarding story that delivers on "users learn Babix, not Clojure."
- Store path naming, identifier formats, and exact builder/sandbox/activator invocation manifest shapes.

---

## 8. Success Criteria (for this horizon)

- A developer can create a locked, reproducible development environment for a non-trivial project with clear provenance and low ceremony.
- Adding or overriding a package feels like editing data with good tooling, not fighting a language.
- `babix explain` routinely answers questions that are difficult or impossible in current Nix/Guix workflows.
- The core interfaces are stable enough that higher-level frameworks and custom builders can be developed against them without forking the core.
- Early reviewers (Nix-using Clojurians) see a coherent, principled system rather than "Nix with different syntax."

---

## 9. Next Steps

1. Iterate on this PRD via targeted grill-me sessions on the open questions above.
2. Produce more detailed interface specifications as decisions stabilize.
3. Develop a thin implementation spike (evaluation + one builder + one activator + local store) to validate the model.
4. Prepare RFC package (VISION + DESIGN + this PRD + key interface docs).

---

*This document is intentionally a living draft. Its purpose is to create a stable enough artifact for review and to surface the remaining design work that must be completed before serious implementation planning.*
