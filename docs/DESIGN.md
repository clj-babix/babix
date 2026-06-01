# Babix Design

**Status:** Initial draft — intended to be iterated on during the design process.  
**Scope:** The architecture for the scoped problem (reproducible development environments + derivations + locking), with clear extension points for later horizons.

---

## Overview

Babix is architected around a small, stable core that defines interfaces and evaluation semantics, with almost all policy, builder logic, and higher-level abstractions living in the package collection as ordinary derivations.

The system has two primary goals that must not be in tension:

1. Deliver real Nix/Guix-grade guarantees for reproducibility, provenance, and hermeticity.
2. Make the experience of using those guarantees feel like a well-designed, data-oriented tool rather than a foreign system with accumulated cruft.

### Intellectual Foundations

Babix is not a new invention. It is a deliberate re-imagining of the *purely functional software deployment model* developed by Eelco Dolstra in his 2006 PhD thesis and refined through the Nix and NixOS projects. That model treats software components (and later entire system configurations) as pure functions from declared inputs to immutable, content-addressed outputs. The store path of every artifact embeds a cryptographic hash of *all* inputs that produced it; this single mechanism simultaneously prevents undeclared dependencies, enables side-by-side installation of conflicting versions, guarantees isolation (no "DLL hell"), and makes upgrades atomic and rollbacks trivial (O(1) via roots).

The original motivation was brutally practical: imperative package managers produce incomplete dependency specifications (nominal names rather than exact instances), allow components to interfere through shared mutable state, make upgrades non-atomic (leaving the system in an inconsistent state mid-upgrade), and provide no reliable way to reproduce a configuration or roll back. The functional model eliminates these problems at the architectural level by making the entire static part of a system (packages, configuration files, startup scripts, etc.) the output of pure functions stored under immutable, hashed paths.

Babix inherits this model wholesale — the store, derivations, two-layer hashing (plan vs. result), hermetic realization, and provenance requirements are direct descendants — while replacing the original surface (a lazy functional language whose ergonomics and error messages remain hostile even to experts, plus decades of accumulated wrapper layers) with plain EDN data, small pure functions for transformation, and ordinary packages for all policy and builder logic. The 2025 large-scale empirical study of nixpkgs (709k+ historical package builds across 17 snapshots) provides strong validation: the model delivers 69–91% bitwise reproducibility (with a clear upward trend) and >99% rebuildability at the scale of the largest cross-ecosystem FOSS distribution. Babix's bet is that a data-first surface plus a radically leaner core can preserve (and in some dimensions strengthen) those guarantees while dramatically lowering the activation energy.

Key references:
- Eelco Dolstra. *The Purely Functional Software Deployment Model*. PhD thesis, Utrecht University, 2006.
- Eelco Dolstra, Andres Löh, Nicolas Pierron. "NixOS: A Purely Functional Linux Distribution." *Journal of Functional Programming*, 2010/2011.
- Julien Malka, Stefano Zacchiroli, Théo Zimmermann. "Does Functional Package Management Enable Reproducible Builds at Scale? Yes." *MSR 2025* (arXiv:2501.15919).

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
Stable Store Interface Library
  - Concrete implementation of the Store Interface contract
  - Reusable by alternate Babix implementations or tools in other languages
          │
          ▼
Package Collection (the vast majority of logic)
  - Language builders (Cargo, Python, etc.)
  - Environment setup tools
  - Source fetchers, patchers, wrappers (as packages)
  - Higher-level templates and frameworks
```

The **Stable Store Interface Library** is the concrete engineering boundary that reifies the Store Interface (and related contracts). Any Babix implementation, alternate runtime, or compatible tool can depend on this library for all store interaction while providing its own evaluation, realization orchestration, or user-experience layers. This keeps the interface contract independent of any one runtime or language.

### Core (Minimal Trusted Base)

The core is deliberately small. Its responsibilities are:

- Define and enforce the contracts (the five interfaces).
- Provide a minimal evaluation engine for derivation descriptions (data + limited escape hatches into pure functions).
- Orchestrate the flow: locking (with inputs-section-only materialization of references into stable identifiers under `:inputs`) → sandbox preparation (via package or backend) → builder invocation inside the prepared hermetic view → store registration. See the "Stable Identifier" entry in [docs/CONTEXT.md](docs/CONTEXT.md).
- Guarantee provenance and explainability across the system.

See `docs/CONTEXT.md` for the current authoritative terminology and distilled decisions. The core changes only when the stable interfaces prove insufficient; almost all policy, builder logic, and higher-level abstractions live in the package collection as ordinary derivations.

The core does **not** contain:
- Language-specific build knowledge, phases, hooks, or any centralized build model (these live in ordinary builder packages in the collection).
- Detailed sandbox policy or output conventions (primarily supplied by the chosen builder package).
- Most environment setup logic inside the sandbox.
- Concrete sandbox implementations (only the interface and orchestration).

All build policy and conventions are supplied as ordinary packages. A derivation names its builder via keyword into its specification inputs. The core remains agnostic to any particular build model.

The vast majority of capability and policy lives in the package collection (builders, activators, fetchers, environment composers, higher-level frameworks). The core changes only when the stable interfaces prove insufficient. No builder is forced to inherit from a privileged parent; shared logic is ordinary dependencies.

Builder packages contain code — they are the execution layer that actually turns a derivation description into artifacts. This is expected and healthy. What keeps the system predictable is that most real projects follow a small number of well-established conventions for their ecosystem (e.g., `cargo build` for Rust crates with a `Cargo.toml`, `go build` for Go modules, `uv` or `poetry` for Python projects with a `pyproject.toml`). When a builder is named and implemented to follow the dominant convention for its domain, the derivation data alone is usually sufficient to understand what will happen. Deviations from the common path exist and are supported, but they are the exception rather than the rule the core must optimize for.

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
   Content-addressed, immutable storage with reference tracking and roots. The single source of truth for artifacts. Uses two-layer hashing (derivation hash for the plan, output hash for the result) and supports promised paths known before realization. Content-addressed storage is the primary model from the outset; purely input-addressed paths are not foundational. Store permission and visibility policy (e.g., group readability, discovery resistance) is explicitly deferred and not part of the v1 core contract (see the "Decisions from RFC Review (2026)" section in [docs/CONTEXT.md](docs/CONTEXT.md)). The concrete implementation of this interface is provided by the Stable Store Interface Library.

3. **Realization Interface**  
   The contract for sandboxed execution. The main pluggability point for different isolation mechanisms (landlock, bubblewrap, OS-specific, future). Realization first invokes sandbox preparation (to create the hermetic view and promised output paths) and only then the builder inside that environment.

4. **Input Locking & Resolution Interface**  
   Declaration and locking of external inputs with explicit source modes (`:worktree`, `:git-head`, `:archive`, `:path`, etc.). The flakes equivalent, designed for honesty, local ergonomics, and visible reproducibility level. No implicit Git behavior. Babix is deliberately source-origin agnostic: the stable identity of any materialized source is the cryptographic content hash of what was fetched plus the locked derivation description that performed the fetch. Git-internal hashes are not part of the core identity model (see the "Decisions from RFC Review (2026)" section in [docs/CONTEXT.md](docs/CONTEXT.md)). Rev/tag resolution may occur at realization time as part of materialization, but the core does not privilege any particular upstream hashing scheme.

5. **Environment Presentation Interface**  
   How store artifacts become runnable environments. Activation logic lives in activator packages (parallel to builders). A derivation can name an `:activator` (resolved via the input graph exactly like `:builder`). Activation is a first-class concern on *every* derivation, not a special case. This keeps the core tiny while allowing very different activation strategies (dev shell, home-manager-style, system-style, etc.) to be supplied as ordinary packages. Collector-style targets are ordinary derivations that compose activations from their inputs.

**Provenance and Explainability** is treated as a cross-cutting requirement. Every interface must produce or preserve enough information for deep provenance questions to be answerable. While tooling can help surface this information (including `babix explain` for complex cases), the primary goal is that the data shapes themselves — explicit references in authoring forms, stable identifiers in locked forms, and clear separation between layers — make normal use and reasoning straightforward without requiring constant explanation.

**Bootstrap and the Trust Model**

The core model (builders and activators as ordinary packages, Merkle-DAG of plans with content-addressed outputs, Stable Store Interface Library, etc.) assumes a working Babix binary and at least one builder/activator package already exist in the store. The ultimate precondition for the first such artifacts is the *bootstrap seed* (also called a trust anchor): a minimal, hand-verifiable host-environment assumption outside the normal derivation / package model (analogous to the kernel and syscall ABI).

In the current best-known architecture this is the ~229-byte hex0-seed (181 bytes of machine code plus a 48-byte ELF header) from the stage0-posix / bootstrappable community — the only pre-compiled binary in the entire chain. It is a pure assembler: it reads hex digits, labels, and comments from a source file and emits the corresponding flat machine code. Everything above it is built from source in a fully auditable 15-phase ladder (hex0-seed → hex0 → hex1 → hex2-0 → M0 → M2-Planet → blood-elf → kaem → M1 → hex2 → mes → mescc → tcc → gcc and beyond). The seed is verified by content-hash pinning: it is executed as the builder of a derivation whose output hash must match the seed's own hash exactly. Any tampering is detected as a hash mismatch.

The seed is deliberately *not* modeled as a Builder package or as the smallest derivation in the DAG. The first "babix" binary, the first builder package, the first activator package, and the first lockfile are *produced by* the bootstrap chain using the seed + pinned sources. This is the concrete answer to the long-standing "elephant" open question (PRD §7). The design must keep the trusted base minimal and must not preclude a true stage0-style self-hosting bootstrap in a later horizon.

For the first horizon a pragmatic cross-build from an existing Nix/Guix system is acceptable and sufficient to produce the initial working `babix` binary and package collection. A true stage0-style self-hosting bootstrap (a `babix-seed` that can be built entirely inside a Babix store) is a later-horizon deliverable required for Babix to be a functional package manager in the strong sense. The Stable Store Interface Library and the first builder/activator packages must ultimately be realizable from a bootstrapped toolchain.

**Locked Derivation Descriptions** are fully explicit and self-contained. A full static walk over the description is used to discover which references are actually used (including inside strings and builder-private data), so that unused inputs can be pruned. At locking time, only the `:inputs` section of a Derivation Description is replaced in-place with stable identifiers (shape `{:ref <keyword> :id <store-identity>}` or equivalent). References that appear elsewhere — in categorized input lists (`:build-inputs`, etc.), as the value of `:builder` or `:activator`, inside strings, or in any other body data — remain as keywords that point into the now-pinned `:inputs` map. This follows the reference/identifier distinction: authoring forms may use convenient names and whole namespaces; the locked form pins exactly the chosen inputs while preserving surrounding data shape. See the "Stable Identifier" entry in [docs/CONTEXT.md](docs/CONTEXT.md) for the authoritative rule.

The locked form is the ground truth for the machine layer — hashing, realization, provenance tracking, and safe composition across time and contexts. Humans primarily interact with the reference-rich authoring layer (short names, ergonomic sugar, local conventions). The system resolves those references into stable identifiers only when crossing into the internal correctness layer. This separation keeps the human surface usable while giving machines the durable identities they require.

**Specification inputs vs. categorized inputs** are deliberately distinct. Specification inputs (under `:inputs`) form the closed scope of concrete package instances available to the derivation. Categorized inputs (`:build-inputs`, `:host-inputs`, `:propagated-host-inputs`) are references into that scope describing architectural and runtime roles. This separation keeps the input graph honest and supports future cross-compilation without collapsing concerns.

**Two-layer hashing and content-addressed storage as primary model** is used throughout: a derivation hash (over the normalized description plus the identities of all locked specification inputs and sources) identifies the immutable plan; an output hash (content hash of the bytes written to a promised location) identifies the actual result. Babix adopts content-addressed storage as the primary model from the outset. Both the locked derivation description (the plan) and the actual produced content (the result) are identified by cryptographic hashes. Realisation records map plan identities to concrete content-addressed output paths and their dependency closures. Purely input-addressed (derivation-hash-only) paths are not used as a foundational path. This decision was made after systematic review of content-addressed store designs (see the "Decisions from RFC Review (2026)" section in [docs/CONTEXT.md](docs/CONTEXT.md)). Babix adopts the approach cleanly from day one.

This is realized as a **Merkle Derivation Graph (Babix Merkle-DAG)**: each Canonical Locked Form (the exact persisted bytes of a derivation plan) contains direct references to its parent derivations by their derivation hashes; those parent hashes are part of the data hashed to produce the current plan's identity. Unused inputs are pruned during canonicalization. Both the plan bytes and the actual binary contents of outputs are content-addressed. The result is a Merkle-DAG of plans with parent pointers plus full content-addressing of binary artifacts. This is a deliberate first-class architectural commitment (see the "Merkle Derivation Graph (Babix Merkle-DAG)" entry in [docs/CONTEXT.md](docs/CONTEXT.md) and the foundational references cited there). It delivers strong tamper-evidence, verifiable inclusion of any ancestor, and resistance to certain supply-chain attacks as emergent properties.

**Sandbox preparation is a distinct pre-builder step.** The core (or realization backend) first invokes a sandbox preparation package (or backend) to establish the hermetic view and make stable absolute output paths visible inside the sandbox. Only then is the builder package (named via `:builder`) invoked inside that prepared environment. Sandbox policy and conventional environment setup (PATH, wrappers, etc.) are supplied primarily by the chosen builder package or its dependencies, not by the core or repeated in every derivation. This separation is a deliberate guardrail against core accretion.

**Single primary output with FHS-style layout.** Derivations declare one primary `out` (plus logical siblings only when genuinely required). Inside `out` the package follows a conventional FHS-like layout (`bin/`, `lib/`, `include/`, `share/man/`, etc.). Composition across packages into coherent roots or environments happens at activation time via activator packages, not by proliferating outputs on every derivation.

**Collectors are ordinary derivations.** A collector (home-style root, project environment, etc.) has the same shape as any other derivation: inputs, categories, optional builder and/or activator, outputs. Large roots and package namespaces can remain lazy at the graph level; per-derivation closure (full materialization and hashing) is required only for the specific derivations that are realized. A whole package collection may be supplied as a single specification input for ergonomics; only actually referenced members are materialized for a given derivation.

The overall graph of locked plans and their realized outputs forms the **Merkle Derivation Graph (Babix Merkle-DAG)** described in [docs/CONTEXT.md](docs/CONTEXT.md). Parent derivation hashes are directly embedded in each Canonical Locked Form and directly digested, with unused inputs pruned at canonicalization time. Both plans and full binary output contents are content-addressed. This is an intentional architectural choice, not an incidental property.

**Unified small data-driven contracts** govern all pluggable capabilities the core invokes directly (builders, sandbox/isolator providers, activators). Each publishes a minimal static manifest (kind, namespace/name, entrypoint, contract version) plus a small per-kind payload. This keeps the core's invocation model tiny and uniform while allowing the package collection to evolve new capabilities without core changes.

**Temporal advisories in derivation specifications**: Derivation descriptions carry an optional mechanism for temporal advisories (deprecation, removal, security notes, "no longer recommended," etc.). These are placed directly in the specification so that `babix explain`, activators, and environment presentation can surface them to users. This addresses the temporal aspect of package health in a data-first way without requiring external databases. (See the "Decisions from RFC Review (2026)" section in [docs/CONTEXT.md](docs/CONTEXT.md).)

### Detailed Decisions and Rationale from Design Sessions

The following records the substantive decisions and rationale that emerged from the interface grill sessions. They are presented here at full fidelity as the permanent home for this architectural record (no separate ADR folder was created). They expand on the high-level mechanisms listed above.

**Builders and activators are ordinary packages supplied as specification inputs** under the `:inputs` map of a Derivation Description. The `:builder` (or `:activator`) field is a keyword reference *into* that closed set. There is no separate global registry or `providers.edn`-style discovery mechanism.

**Specification Inputs vs. Categorized Inputs** are deliberately distinct. The former is the closed scope passed to the derivation; the latter describes roles within the build. Logical package names under `:inputs` do not include versions (except deliberate parallel cases such as python313 alongside python311). Versions live in the concrete resolved value, not the key.

**A file may contain multiple derivation descriptions** for authoring convenience (e.g., source-fetch → deps-fetch → build). These are treated as virtually separate descriptions. References between them are plain keywords with local-first lookup (file-local derivations first, then the package collection). At locking time they resolve to independent, stable, hashable descriptions.

**In-place materialization before hashing**: A full static walk over the Derivation Description is used to discover which references are actually used (even inside strings or builder-private data), so that unused inputs can be pruned. At locking time, only the `:inputs` section is replaced in-place with stable identifiers (shape `{:ref <keyword> :id <store-identity>}` or equivalent). References that appear elsewhere in the description — in categorized input lists, as the value of `:builder` or `:activator`, inside strings, or in any other body data — remain as keywords that point into the now-pinned `:inputs` map. The locked form must be fully explicit and self-contained for the inputs that are actually referenced. See the "Stable Identifier" entry in [docs/CONTEXT.md](docs/CONTEXT.md) for the authoritative rule.

**Stable Identifier shape**: The recommended form is a small map that preserves the original reference name for readability and to keep surrounding structure intact:

  ```clojure
  {:name "my-package-source"
   :derivation-hash "sha256-..."}
  ```
  This follows Spec-ulation principles: references are pinned to stable identifiers while the data shape remains stable.

**Sandbox preparation is a distinct pre-builder step**: The core invokes a sandbox preparation package (or backend) *before* the actual builder. The builder receives an already-prepared hermetic environment with promised stable output paths. Sandbox policy and output conventions are primarily supplied by the builder package (or a parent it extends), not repeated in every derivation.

**Two-layer hashing**: Derivation hash (plan) vs. output/content hash (result). The output identity in the CAS for a realized `out` is the cryptographic hash computed over *both* the Canonical Locked Form bytes of the producing derivation *and* the actual bytes that landed in that `out`. The hashes or regular store paths of the input dependencies are not folded into this output content hash; those relationships live exclusively in the Merkle-DAG of plans (parent derivation hashes inside the Canonical Locked Forms). Promised paths are known before the builder runs. Source-fetch derivations (and similar steps that must reach outside the sandbox) are ordinary derivations that declare an expected content hash and receive controlled impurity privileges; their output is pinned by the combined (plan + content) hash. All downstream derivations consume them as normal specification inputs under `:inputs`. This is expressed uniformly through the multi-derivation graph rather than a distinguished "fixed-output derivation" kind. The model enables both perfect reproducibility of plans and practical caching of expensive content hashing (when the Canonical Locked Form is already known, previously materialized local content with a matching hash can be trusted without re-computation, with an explicit bust-cache escape).

**Single primary output**: Derivations use one primary `out` (FHS layout inside). Composition across packages happens at activation/root time.

**No centralized build monolith**: There is no privileged `genericBuild` / phase runner / hook system in the core. All such logic lives in ordinary builder packages.

**Explicit rejection of plan dynamism / computed derivations**: Babix rejects the model in which a derivation can dynamically produce new derivation descriptions that then become new build goals. Instead, Babix uses a deliberate inversion: if a workflow would have required "a derivation that produces a derivation," the inner derivation is produced first as an ordinary step, and its output is supplied as a specification input to the subsequent ordinary derivation. Multi-derivation specifications (e.g., source-fetch → dependency-vendor → build) already solve this class of problems completely. The core never has to treat a derivation's output as a new plan to be scheduled. This keeps evaluation, hashing, and realization semantics simple and uniform. All steps remain ordinary derivations; there is no distinguished "plan-producing" kind. (See the "Decisions from RFC Review (2026)" section in [docs/CONTEXT.md](docs/CONTEXT.md).)

**Unified small contracts for pluggables**: Builders, sandbox/isolator providers, and activators present small, stable, data-driven manifests (common envelope + per-kind payload) so the core can invoke them uniformly.

**Discovery walk and pruning**: A full static walk over the Derivation Description is used to discover which references are actually used (keys, values, strings, blobs, etc.), so that inputs present in the authoring form but never referenced inside that same description can be pruned. Only inputs with positive reference count survive into the Canonical Locked Form. The results of this discovery determine what gets pinned under `:inputs`; they do not cause replacement of references that appear elsewhere in the description. See the "Stable Identifier" and "Canonical Locked Form" entries in [docs/CONTEXT.md](docs/CONTEXT.md) for the precise rule.

**Collectors are ordinary derivations**: No special syntax or core treatment. Large roots/collectors can stay demand-driven. Any specific derivation must have its closed, resolved form before realization.

**Inside the prepared sandbox**, the builder (or its supporting packages) owns conventional environment setup (PATH, etc.). The sandbox preparation layer focuses on isolation and promised paths.

**Materialization before hash**: All reference discovery and replacement must be complete before the derivation hash is computed.

**Builder Architecture Lessons (from prior art)**: Prior systems (notably Nixpkgs `stdenv` + `genericBuild` + its hook system, and Guix build systems) demonstrate that embedding a rich, centralized build model—phases, implicit hook mechanisms, generic runners, setup-hook sourcing, etc.—into the core or a single privileged package produces exactly the accumulated magic, hidden behavior, and maintenance burden that makes these tools painful. The history of Nix itself shows how a small, clean functional core gradually accreted policy as more language ecosystems and edge cases were supported inside privileged constructs; the result is a trusted computing base that is larger and more opaque than necessary.

Babix therefore keeps *all* build policy, phase models, hook systems, default build flows, and language-specific conventions strictly as ordinary packages in the collection. A generic phased builder may exist as a convenience that other builders can depend on, but:
- It has no special status or privileges in the core.
- Other builders may ignore it entirely and implement their own orchestration from scratch.
- The core Derivation Description and Realization interfaces remain completely agnostic to any particular build model or convention.

This is not merely an aesthetic preference for smallness; it is a direct lesson from the functional deployment literature: once policy lives in the core, it becomes nearly impossible to evolve or replace without forking, and every user pays the complexity tax even for use cases the policy was never intended to cover. By contrast, when builders, activators, and environment composers are ordinary packages, the collection can experiment aggressively while the trusted base stays minimal and stable.

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
- What is the minimal viable provenance model that still delivers on the "explain everything" promise? (The 2025 large-scale nixpkgs reproducibility study released full build logs and diffoscopes for 86k+ unreproducible artifacts; this is a concrete existence proof of the kind of data `babix explain` should surface.)
- How do we handle the bootstrap problem (building the initial package collection) and the initial package collection itself? The current best-known path uses a ~229-byte hex0-seed (the only pre-compiled artifact) and a 15-phase source-driven ladder from the stage0-posix / bootstrappable community. A pragmatic cross-build from Nix/Guix is acceptable for the first horizon; a true stage0-style self-hosting bootstrap (producing a babix-seed inside Babix) is the later-horizon goal for "strong sense" functional package management. The seed is a host-environment assumption (like the kernel and syscall ABI), not a Builder package and outside the Merkle-DAG; its hash is the ultimate stable identifier. The design must not preclude self-hosting.
- What does the user-facing project configuration and target declaration format actually look like for normal projects (the equivalent of `flake.nix` + `flake.lock` plus `babix init` output)?
- Trust and distribution model for locked inputs and pre-built artifacts (even a minimal local-first story).
- Detailed CLI surface, `babix explain` question catalog, and exact activator/collector composition contracts.

### Provenance and Reproducibility at Scale

The 2025 empirical study of nixpkgs (the largest cross-ecosystem FOSS distribution) rebuilt 709,816 packages from 17 historical snapshots spaced ~4 months apart (2017–2023). Key findings directly relevant to Babix:

- Bitwise reproducibility ranged from 69% to 91%, with a clear upward trend despite continuous growth in package count.
- Rebuildability (the ability to re-execute a historical plan and obtain a result) stayed above 99% across the entire period.
- Approximately 15% of unreproducible failures were caused by embedded build dates; other recurring causes include uname output leakage, embedded environment variables, and non-deterministic build IDs (e.g., Go).
- QA monitoring of a critical subset (the NixOS minimal ISO image) correlated with sustained >95% reproducibility rates — suggesting that active monitoring is an effective lever.
- Most reproducibility fixes were incidental (side-effects of routine package updates) rather than intentional reproducibility work.

These numbers validate the core claim of the purely functional deployment model: when every artifact's identity is a pure function of its exact inputs and the build is hermetically isolated, high reproducibility becomes an *emergent property at scale* rather than an unattainable ideal. Babix's provenance and `babix explain` ambitions are not speculative; they are the tooling layer on top of a model that has already been shown to work for >700k packages. The released dataset of logs and full recursive diffs (diffoscopes) is a model for the kind of explainable artifacts Babix should produce by default.

See `docs/CONTEXT.md` (Key Decisions and Key Concepts sections) for the current authoritative stance and terminology. The interface documents in `docs/interfaces/` contain the detailed contracts.

**Further reading** (foundational sources for the model Babix inherits):
- Dolstra 2006 (PhD thesis) — the original formalization of the purely functional deployment model, closures, the historical fixed-output derivation mechanism (for source fetches), and reference scanning. Babix adopts the model but solves the fetch/impurity problem uniformly via ordinary derivations in a multi-derivation graph rather than a distinguished derivation kind.
- Dolstra et al. JFP 2011 (NixOS paper) — the extension to full system configuration, the module system as a purely functional composition mechanism, and honest discussion of purity compromises required for real OS state.
- Malka et al. MSR 2025 — the first large-scale empirical validation that the model delivers high bitwise reproducibility and near-perfect rebuildability in practice.
- Schwaighofer, Roland, Mayrhofer (SCORED '24) — extensions for builder provenance, remote attestation, and dependency resolution outcome records to reduce transitive trust in decentralized/multi-builder settings (advanced provenance direction).

**Advanced provenance: Builder attestation and transitive trust**

Derivation-level provenance (locked inputs, chosen builder/activator packages, realization records) is powerful, but in a decentralized world with multiple independent builders, caches, or remote realization, consumers still face *transitive trust* in the builders themselves. Schwaighofer et al. (SCORED '24) show how to reduce this by attaching two kinds of verifiable metadata to the existing cryptographically secured trace map entries used by systems like Nix:

- Builder provenance data: which concrete builder (package + attested software stack + remote attestation / measured boot evidence) performed the realization.
- Record of dependency resolution outcome: not just the declared or pinned inputs, but evidence of what the builder actually resolved each of *its own* dependencies to at build time.

This lets downstream parties apply their own trust policies at consumption or explanation time ("only accept builds from builders whose attested configuration excludes known-vulnerable packages", "require N-of-M agreement from builders I designate", etc.) rather than blindly inheriting trust in every upstream builder and cache.

Babix's cross-cutting provenance requirement and the realization records produced by the Realization Interface should be designed to accommodate such richer attestation payloads as a future extension. This is particularly relevant for supply-chain security use cases and for any later horizon involving distributed or multi-builder realization. It does not change first-horizon core contracts, but it does argue for keeping realization provenance records extensible and for treating the choice of builder package as first-class observable data.

Further reading for this direction:
- Martin Schwaighofer, Michael Roland, René Mayrhofer. "Extending Cloud Build Systems to Eliminate Transitive Trust." *Proceedings of the 2024 Workshop on Software Supply Chain Offensive Research and Ecosystem Defenses (SCORED '24)*, ACM, 2024.

---

This design is intentionally minimal at the core and maximal at the edges. The bet is that a small set of well-chosen interfaces, combined with a data-first philosophy and a package collection that is allowed to grow freely, can deliver both the power of Nix/Guix and a dramatically better experience.