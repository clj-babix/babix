# Derivation Description Interface

**Status:** Initial draft — subject to deep review and revision.  
**Owner:** Babix design process  
**Last updated:** 2025 (via grill-me session)

## Purpose

The Derivation Description Interface defines the fundamental data contract for "how to build a thing from declared inputs in a reproducible way."

This is the Babix equivalent of a Nix derivation (the result of `mkDerivation` and friends) and the core of what makes `nix develop` + `nix build` + flakes-style locking possible.

Everything else in the system (locking, realization, store, environments, provenance) is built on top of or in service of this interface.

## Core Principle

A derivation description must be:

- **Primarily data** (EDN maps and vectors).
- **Self-describing** for provenance and explainability.
- **Explicit** about inputs, outputs, builder, and environment expectations.
- **Free of hidden policy** — the core does not know about "Cargo builds" or "Python builds". All such knowledge lives in packages that produce or consume these descriptions.

## Required Shape (Initial Sketch)

A derivation is a map with the following responsibilities. Exact field names and nesting are open for discussion.

```clojure
{:name "ripgrep-14.1.0"
 :version "14.1.0"

 ;; What this derivation produces.
 ;; Output paths are promised (or deterministically computable) before realization.
 ;; The sandbox preparation step makes the final immutable store locations visible
 ;; inside the sandbox as stable absolute paths. The builder writes into them
 ;; (typically via $out and similar environment variables). After successful build,
 ;; the contents are registered in the Store under content-addressed names.
 :outputs {:out  {:path "/store/...-ripgrep-14.1.0"}
           :man  {:path "/store/...-ripgrep-14.1.0-man"}}
 ;; (In practice the concrete paths or the exact inputs needed to compute the
 ;; derivation hash + output names are carried or derivable from the locked inputs.)

 ;; Inputs, categorized for clarity around platform and runtime concerns.
 ;; See "Input Categories and Platform Concerns" section below.
 :inputs
 {:build-inputs             [gcc-for-build cmake ...]
  :host-inputs              [openssl zlib ...]     ; things the output links against / needs at runtime
  :propagated-host-inputs   [...]}

 ;; Source(s) this derivation consumes
 :sources
 [{:type :git
   :url "..."
   :rev "..."
   :sha256 "..."}
  ...]

 ;; Builder to use (resolved via Builder Provider mechanism)
 ;; The value is a namespaced keyword. The actual builder logic comes
 ;; from a package in the input graph that provides a matching capability.
 :builder :babix.builders/python

 ;; Sandbox / realization requirements (high-level)
 ;; The derivation can declare high-level isolation needs and output names.
 ;; Detailed policy (network, exact features, etc.) is often contributed by the
 ;; chosen builder package. The core + sandbox preparation package combine the
 ;; declaration here with requirements coming from the builder package.
 :sandbox
 {:required-features [:landlock]
  :outputs ["out"]}

 ;; Note: Low-level sandbox mechanics and the exact shape of the prepared
 ;; environment are the concern of the sandbox preparation package (invoked
 ;; by realization before the builder) and the builder package itself.

 ;; How the resulting artifacts should find their dependencies at runtime
 ;; (This is where we try to make wrapper/ rpath / LD_LIBRARY_PATH concerns explicit)
 :runtime-environment
 {:type :explicit-rpath
  :extra-paths [...]
  :library-paths [...]}

 ;; Optional activator package (parallel to :builder).
 ;; An activator package supplies the logic and policy for turning the
 ;; outputs of this derivation into an activated environment (root,
 ;; activation script, symlinks, etc.).
 ;;
 ;; A derivation may have a builder, an activator, both, or neither.
 ;; Activator packages are the preferred way to express activation behavior.

 ;; Arbitrary but structured metadata for provenance, licensing, etc.
 :meta
 {:homepage "..."
  :license "MIT"
  :maintainers [...]}}
```

## What Must Be Explicit

- All inputs that affect the output hash (build inputs, sources, patches, builder logic).
- The mechanism by which runtime dependencies are made available to the produced artifacts.
- The isolation/sandbox requirements the realization backend must satisfy.
- (When an activator is used) The activator package must be resolvable from the input graph.
- Provenance information sufficient for `babix explain` to answer "why does this exist and what would change it?"

## What the Core Must Not Know

- Language-specific build conventions (Cargo, setuptools, npm, Go modules, etc.).
- Any notion of "standard" phases (configure, build, install, fixup, etc.).
- Phase runners, hook systems, setup-hook sourcing, or genericBuild-style orchestration logic.
- Wrapper tools, environment setup helpers, or "stdenv"-like constructs.
- Any blessed or default build model that all other builders are expected to inherit from or extend.

All of the above must be expressible as ordinary packages (builder packages) that are supplied as inputs to derivations using this interface. A derivation names one such package via the `:builder` keyword. Builder packages may themselves depend on other builder packages for shared logic if they wish, or they may implement everything from scratch. The core does not privilege any of them.

## Builder Providers (Current Model)

Higher-level builders are **not** special cases in the core. A derivation declares its builder using a namespaced keyword (e.g. `:babix.builders/python`). The actual build logic and conventions are supplied by a separate package in the derivation's input graph that registers itself as a `Builder Provider` for that keyword.

This keeps the core extremely small and allows builders to evolve on their own release cadence, independently of Babix itself. Versioning of builders happens at the package level (like any other dependency), not as a field inside the derivation.

Builder packages are ordinary derivations. They may themselves declare other builder packages as inputs if they wish to compose on top of lower-level build logic. The core never assumes or requires any such composition.

## Activators (as Packages)

Activation behavior is supplied by **activator packages**, parallel to how builders are supplied by builder packages.

A derivation can name an activator using a namespaced keyword (e.g. `:babix.activators/home-environment`), resolved the same way as builders — via a package in its input graph that registers as an `Activator Provider`.

This design keeps the core extremely small and allows different activation strategies (dev shell, home-manager-style, system-style, etc.) to evolve independently as packages, often written in Clojure rather than bash.

A derivation may declare a `:builder`, an `:activator`, both, or neither. When an activator is present, it is the source of truth for how the derivation should be activated.

## Input Categories and Platform Concerns

When designing the input model for derivations we explicitly considered the real reasons Nix and Guix maintain multiple input categories (`nativeBuildInputs` / `buildInputs` / `propagatedBuildInputs` and the Guix equivalents).

The three primary concerns are:

1. **Architecture separation for cross-compilation**  
   Some inputs (compilers, build tools, code generators) must run on the *build* machine. Others (libraries, resources) must be built for the *host/target* machine where the final artifact will execute. Without a way to express this distinction, correct cross-compilation is impossible.

2. **Runtime closure hygiene**  
   Not every input used during the build should become a runtime dependency of the output. Build tools should ideally not pollute the runtime closure of the final package.

3. **Transitive runtime propagation**  
   Some packages (especially for interpreted languages, header-only libraries, or pkg-config style metadata) need their dependencies to be automatically visible to anything that depends on them at runtime. This is the role of propagated inputs.

### Naming Decision (Made Early)

We deliberately reject the legacy confusing names (`nativeBuildInputs`, `buildInputs`) because they have caused decades of confusion.

Instead, the Derivation Description Interface uses clearer names from the beginning:

- `:build-inputs` — packages and tools required to *perform the build*. These execute on the build platform.
- `:host-inputs` — packages that the *resulting artifact* depends on (libraries, data files, etc.). These are for the host/target platform.
- `:propagated-host-inputs` — transitive runtime dependencies that should be made available to consumers of the output.

These names were chosen now, while the interface is still fluid, precisely so that Babix can be clearer than its predecessors even before full cross-compilation support is implemented.

Full cross-compilation (with a possible future `:target-inputs` distinction when host ≠ target) is acknowledged as a future requirement. The current names are designed to extend cleanly in that direction without renaming pain later.

### Builder Packages and Build Conventions

Any "stdenv-like" construct — a set of default phases, a phase runner, a hook registration mechanism, common environment setup logic, etc. — must be implemented as one or more ordinary builder packages in the collection. The base Derivation Description Interface imposes no such model and remains usable even if no such package is ever written. Language-specific or domain-specific builders are free to depend on a generic builder package, to extend or fork its logic, or to ignore it completely. The core never assumes any such composition or convention exists.

## Open Questions (for later grill-me per interface)

- How does the core resolve a `:builder` keyword (e.g. `:babix.builders/python`) to a concrete provider capability from the input package graph?
- What is the minimal contract a package must satisfy to be recognized as a Builder Provider?
- What is the minimal set of fields required vs. what can be convention in the package collection?
- How do we represent "this derivation is a cross-compiled package for architecture X"?
- How does the data model support very large lazy package sets without forcing full evaluation?
- What is the exact relationship between this interface and the Input Locking interface?

### Unified Capability Contracts (direction from design session)

All pluggable capabilities that the core invokes directly (builders, sandbox/isolator providers, activators, etc.) must present a **unified, small, stable, data-driven contract**.

The goal is consistency across different kinds of pluggable packages so that:
- The core has a predictable, minimal way to discover and invoke them.
- Tooling, provenance, and `babix explain` can treat them uniformly.
- New capability kinds can be added over time without changing the core invocation model (Open-Closed Principle).

A capability package publishes a small, static manifest (EDN) in its outputs at a conventional location. The manifest includes at minimum:
- `:kind` (e.g. `:builder`, `:sandbox`, `:activator`)
- `:namespace` + `:name` (so the keyword in the derivation can be matched, e.g. `:babix.builders/rust`)
- Entry point information (how the core starts the package)
- A version for the capability contract itself (to support evolution)

Each `:kind` then has its own additional contract on top of this common envelope (what data is passed on invocation, what the package is expected to do, how results/outputs are described, etc.). The per-kind contracts must also be small, data-oriented, and stable.

This approach keeps the core's knowledge of "how to talk to a capability package" tiny and uniform, while still allowing very different responsibilities for builders vs. sandbox providers vs. activators.

### Decisions Captured (from design sessions)

**No centralized build monolith in the core**

There is no privileged `genericBuild`, phase runner, or hook system in the core or as a special construct. All such logic lives in ordinary builder packages in the collection. A derivation may name a generic phased builder if it wishes, but other builders are free to ignore it and implement their own model from scratch. The core Derivation Description and Realization interfaces remain agnostic to any particular build model.

**In-place materialization of references with stable identifiers**

Authoring forms (including multi-derivation files and references inside builder data) may use convenient references (plain keywords, namespaced maps, whole namespaces under `:inputs`, etc.).

Before the derivation hash is computed, a full static walk discovers all references. In the locked/closed Derivation Description:

- Every reference is replaced *in-place* with a stable identifier.
- The recommended shape for the identifier is a small map that includes the original reference name for readability and to allow the surrounding structure (categorized lists, builder data, etc.) to remain unchanged in shape:

  ```clojure
  {:name "my-package-source"           ; the original logical/reference name
   :derivation-hash "sha256-..."       ; primary stable identity
   ;; optionally :output "out"
   ;; optionally :store-path "/store/..." (derived, not required for identity)
   }
  ```

- This preserves the data shape between authoring and locked forms.
- The locked form is fully explicit and self-contained; the hash is computed over it.
- Higher-level tooling and language builders provide ergonomic authoring surfaces; the core only ever sees the explicit locked artifacts.

**Sandbox preparation and output promises**

Detailed sandbox policy and output conventions are primarily supplied by the chosen builder package (or a parent "core" builder it extends). The Derivation Description carries high-level requirements and logical output names. The core + sandbox preparation package combine builder-supplied policy with any derivation constraints. The sandbox preparation package (invoked by realization before the builder) is responsible for creating the hermetic view and making promised stable absolute output paths visible.

**Two-layer hashing and fixed-output**

- Derivation hash: deterministic over normalized description + hashes of all locked specification inputs/sources.
- Output hash: content hash of the actual result written to a promised output location.
- Fixed-output cases (sources, vendored deps, etc.) are explicit derivation steps (or lightweight source entries) carrying an expected content hash + controlled-impurity allowance in their sandbox policy. They produce normal content-addressed inputs for downstream derivations.

**Single primary output + FHS layout**

Derivations declare a single primary `out` (plus logical siblings only when truly needed). Packages follow FHS-style layout inside `out` (`bin/`, `lib/`, `share/man/`, etc.). Composition of multiple packages into coherent roots/environments happens at activation time via activator packages, not by multiple outputs on every derivation.

**Builders and build policy are ordinary packages**

There is no centralized phase model, hook system, or genericBuild equivalent in the core. All such logic lives in ordinary builder packages in the collection. A derivation names its builder via keyword into its specification inputs. The core is agnostic to any particular build model.

**Future discussion topics (not urgent for the base interface):**
- The role and design of something like "stdenv" (blessed build environment, phases, setup hooks). It should be treated as a package (or set of packages) that builders can depend on, not as special core machinery.
- Refinement of input categories once serious cross-compilation work begins.
- The exact contract and discovery mechanism for Activator Providers (symmetric to Builder Providers).

## Relationship to Other Interfaces

- **Store Interface**: Derivations compute content-addressed store paths (via two-layer hashing: derivation hash for the plan, output hash for the result).
- **Realization Interface**: This description is the primary input to realization. Sandbox preparation occurs before the builder is invoked; the builder receives an already-prepared hermetic environment with promised stable output paths.
- **Input Locking & Resolution**: Sources and input derivations are locked and resolved before realization. All references are materialized in-place with stable identifiers before the derivation hash is computed.
- **Environment Presentation**: Activation is performed by activator packages (first-class on every derivation). The Environment Presentation interface is responsible for discovering and invoking the chosen activator against a derivation's outputs. Collectors are ordinary derivations.
- **Provenance**: This structure (plus realization records) is the primary source of truth for explainability. Which activator (if any) was used, and with which inputs, must be traceable.

## Non-Goals (for this interface)

- Defining *how* a derivation is written by end users (raw data vs. template sugar).
- Service configuration or system state (modules are out of scope for the current focus).
- Full system activation and generations for root-owned targets.

See `docs/CONTEXT.md` for the current authoritative stance on in-place materialization, stable identifiers, specification vs. categorized inputs, two-layer hashing, single primary `out`, and the absence of any privileged genericBuild or hook system in the core.

---

This document is intentionally incomplete and opinionated in places. Its purpose is to give us a concrete artifact to grill, refine, and then implement against.

Next interfaces to document (proposed order):
1. Store Interface
2. Realization Interface
3. Input Locking & Resolution Interface
4. Environment Presentation Interface

Provenance is currently treated as cross-cutting.