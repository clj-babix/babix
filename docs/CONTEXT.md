# Babix — Context and Terminology

**Purpose**  
This document defines the core concepts and terminology used in the Babix design. Its goal is to ensure that everyone working on (or reviewing) Babix is using the same words to mean the same things.

This is a living document. As design decisions are made or refined, this file should be updated.

---

## Core Principles (Current Stance)

- **Data first**: The primary representation for almost everything is plain EDN data. Logic and behavior are expressed as small pure functions that transform that data, or as packages that provide capabilities.
- **Lean core**: The trusted core is deliberately small. It defines stable interfaces and provides minimal evaluation/orchestration. Almost all policy, builder logic, and higher-level behavior lives in the package collection.
- **Builders and activators are packages**: Language-specific build logic and activation behavior are supplied by ordinary packages (resolved via namespaced keywords), not special-cased in the core.
- **Unified model**: There is one primary concept (the derivation) and one primary way to extend behavior (packages providing capabilities). Different use cases (dev environments, user environments, future system-level targets) are achieved by varying inputs, builders, and activators rather than by creating separate systems.
- **Explainability is non-negotiable**: Every important decision and transformation must be traceable.

### Key Decisions from Design Sessions (distilled)

These decisions were reached through extended grill-me sessions on the core interfaces and are now treated as current stance:

- **Builders and activators are ordinary packages supplied as specification inputs** under the `:inputs` map of a Derivation Description. The `:builder` (or `:activator`) field is a keyword reference *into* that closed set. There is no separate global registry or `providers.edn`-style discovery mechanism (D42).
- **Specification Inputs vs. Categorized Inputs** are deliberately distinct (D47). The former is the closed scope passed to the derivation; the latter describes roles within the build.
- Logical package names under `:inputs` do **not** include versions (except deliberate parallel cases such as python313 alongside python311). Versions live in the concrete resolved value, not the key (user preference for override ergonomics).
- A file may contain multiple derivation descriptions for authoring convenience (e.g., source-fetch → deps-fetch → build). These are treated as *virtually separate* descriptions. References between them are plain keywords with local-first lookup (file-local derivations first, then the package collection). At locking time they resolve to independent, stable, hashable descriptions (D56 and related).
- **In-place materialization before hashing**: A full static walk over the entire Derivation Description discovers references (even inside strings or builder-private data). All discovered references are replaced *in-place* with stable identifiers *before* the derivation hash is computed. The locked form must be fully explicit and self-contained.
- **Stable Identifier shape**: The recommended form is a small map that preserves the original reference name for readability and to keep surrounding structure intact:

  ```clojure
  {:name "my-package-source"
   :derivation-hash "sha256-..."}
  ```
  This follows Spec-ulation principles: references are pinned to stable identifiers while the data shape remains stable.
- **Sandbox preparation is a distinct pre-builder step**: The core invokes a sandbox preparation package (or backend) *before* the actual builder. The builder receives an already-prepared hermetic environment with promised stable output paths. Sandbox policy and output conventions are primarily supplied by the builder package (or a parent it extends), not repeated in every derivation.
- **Two-layer hashing**: Derivation hash (plan + inputs) vs. output hash (result). Promised paths are known before the builder runs.
- **Single primary output**: Derivations use one primary `out` (FHS layout inside). Composition across packages happens at activation/root time.
- **No centralized build monolith**: There is no privileged `genericBuild` / phase runner / hook system in the core. All such logic lives in ordinary builder packages (D18, D19, D62, D63).
- **Unified small contracts for pluggables**: Builders, sandbox/isolator providers, and activators present small, stable, data-driven manifests (common envelope + per-kind payload) so the core can invoke them uniformly.
- **Full static walk for discovery**: For now, the system walks the entire description to find references (keys, values, strings, blobs). Results are materialized explicitly in the locked form.
- **Collectors are ordinary derivations**: No special syntax or core treatment.
- **Laziness at graph level, closure per derivation**: Large roots/collectors can stay demand-driven. Any specific derivation must have its closed, resolved form before realization.
- **Inside the prepared sandbox**, the builder (or its supporting packages) owns conventional environment setup (PATH, etc.). The sandbox preparation layer focuses on isolation and promised paths.
- **Materialization before hash**: All reference discovery and replacement must be complete before the derivation hash is computed.

These decisions take precedence over earlier or vaguer language in the documents. The files have been updated to reflect them.

### Builder Architecture Lessons (from prior art)

Prior systems (notably Nixpkgs `stdenv` + `genericBuild` + its hook system, and Guix build systems) demonstrate that embedding a rich, centralized build model—phases, implicit hook mechanisms, generic runners, setup-hook sourcing, etc.—into the core or a single privileged package produces exactly the accumulated magic, hidden behavior, and maintenance burden that makes these tools painful.

Babix therefore keeps *all* build policy, phase models, hook systems, default build flows, and language-specific conventions strictly as ordinary packages in the collection. A generic phased builder may exist as a convenience that other builders can depend on, but:
- It has no special status or privileges in the core.
- Other builders may ignore it entirely and implement their own orchestration from scratch.
- The core Derivation Description and Realization interfaces remain completely agnostic to any particular build model or convention.

---

## Key Concepts

### Derivation (Derivation Description)
The fundamental unit of work. A primarily data (EDN map) that describes how to produce one or more outputs from a set of inputs.

A derivation declares (at minimum):
- What it produces
- Its inputs (categorized)
- Which builder should be used
- Sandbox requirements
- Optionally, which activator should be used

Derivations are the common substrate for packages, development environments, collector targets, etc.

### Builder / Builder Provider
A capability supplied by a package that knows how to turn a derivation description into real store artifacts.

Builders are referenced by namespaced keyword (e.g. `:babix.builders/python`) and resolved from the derivation's input graph. The core does not contain language-specific build knowledge.

### Activator / Activator Provider
A capability supplied by a package that knows how to turn the outputs of a derivation into an activated environment (root + activation script + environment setup).

Like builders, activators are referenced by namespaced keyword and resolved from the input graph. Activation is a first-class concern on *every* derivation, not just special "environment" derivations.

### Input Categories
We deliberately use clearer names than the legacy `nativeBuildInputs` / `buildInputs` terminology:

- **`:build-inputs`** — Packages and tools required to *perform the build*. These execute on the build platform.
- **`:host-inputs`** — Packages that the *resulting artifact* depends on at runtime. These are for the host/target platform.
- **`:propagated-host-inputs`** — Transitive runtime dependencies that should be made available to consumers of the output.

These categories exist primarily to support:
- Architecture separation (for future cross-compilation)
- Runtime closure hygiene
- Transitive runtime propagation

### Target
A named configuration in a project that produces a specific environment or artifact set when built or developed. Different targets can have different roots and different activation policies (dev shell, user home, future system, etc.).

### Collector Derivation
A derivation whose primary purpose is to aggregate other derivations (as inputs) and produce a combined activation (e.g. a home-manager-style or project environment configuration).

### Source Mode
An explicit declaration of how a source input should be materialized (e.g. `:worktree`, `:git-head`, `:archive`, `:path`). Source modes make reproducibility level and "dirtiness" first-class and visible in the lockfile.

### Lock / Lockfile
The artifact that captures the exact, resolved state of all external inputs for a project or derivation, including reproducibility level. The lockfile is the primary mechanism for reproducible environments and builds.

### Realization
The process of turning a derivation description into actual store artifacts under sandboxed conditions. This is the primary pluggable point for different isolation mechanisms.

### Store
The content-addressed, immutable storage layer. The single source of truth for built artifacts and their provenance.

### Package Collection
The vast majority of Babix's functionality lives here: language builders, activators, stdenv-like constructs, environment tools, higher-level frameworks, etc. The collection is expected to evolve much faster than the core.

### Core vs. Standard Library
- **Core**: The minimal trusted base that defines the interfaces, performs evaluation, orchestrates flows between the five interfaces, and guarantees provenance.
- **Standard Library** (tiny): A small set of stable, pure helper functions (the "lib" equivalent). Changes extremely slowly.

Everything else should be expressible as packages.

---

## Current Stance on Data Formats (EDN)

- The primary user- and machine-facing format is **plain EDN**.
- We use **namespaced keywords** as the main extension mechanism (e.g. `:babix.builders/python`, `:build-inputs`).
- We have **not** committed to using EDN tagged literals (`#babix/...`) as part of the normal authoring surface. Custom tags will be introduced only when we have a clear, demonstrated need that cannot be met cleanly with plain maps, vectors, keywords, and sets.
- Derivations, lockfiles, and project configuration are expected to remain readable and useful even to generic EDN tools where possible.

---

## Notes on Terminology Evolution

- Many terms are intentionally borrowed from Nix/Guix but given clearer names or slightly different boundaries when it improves understanding.
- This document should be updated whenever a significant concept is introduced, renamed, or given new scope.
- When in doubt, prefer explicit, slightly longer names over short ambiguous ones (especially around inputs and activation).

---

*This document is the single source of truth for what words mean in the current Babix design.*