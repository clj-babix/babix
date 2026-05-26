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

 ;; What this derivation produces
 :outputs {:out {...}
           :man {...}}

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

 ;; Sandbox / realization requirements
 :sandbox
 {:required-features [:landlock :user-namespace]
  :network false
  :writable-dirs [...]}

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
- "Standard" phases (configure, make, make install, etc.).
- Wrapper tools or environment setup helpers.
- Any notion of "stdenv".

All of the above must be expressible as ordinary packages that produce or act as inputs to derivations using this interface.

## Builder Providers (Current Model)

Higher-level builders are **not** special cases in the core. A derivation declares its builder using a namespaced keyword (e.g. `:babix.builders/python`). The actual build logic and conventions are supplied by a separate package in the derivation's input graph that registers itself as a `Builder Provider` for that keyword.

This keeps the core extremely small and allows builders to evolve on their own release cadence, independently of Babix itself. Versioning of builders happens at the package level (like any other dependency), not as a field inside the derivation.

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

### Stdenv and Build Conventions

The role of something like "stdenv" (a blessed set of build tools, phases, and environment setup) is explicitly left as a future discussion. The base Derivation Description Interface should remain usable without assuming any particular stdenv-like construct; such conventions, if desired, can be expressed via builder packages and the package collection.

## Open Questions (for later grill-me per interface)

- How does the core resolve a `:builder` keyword (e.g. `:babix.builders/python`) to a concrete provider capability from the input package graph?
- What is the minimal contract a package must satisfy to be recognized as a Builder Provider?
- What is the minimal set of fields required vs. what can be convention in the package collection?
- How do we represent "this derivation is a cross-compiled package for architecture X"?
- How does the data model support very large lazy package sets without forcing full evaluation?
- What is the exact relationship between this interface and the Input Locking interface?

**Future discussion topics (not urgent for the base interface):**
- The role and design of something like "stdenv" (blessed build environment, phases, setup hooks). It should be treated as a package (or set of packages) that builders can depend on, not as special core machinery.
- Refinement of input categories once serious cross-compilation work begins.
- The exact contract and discovery mechanism for Activator Providers (symmetric to Builder Providers).

## Relationship to Other Interfaces

- **Store Interface**: Derivations compute content-addressed store paths.
- **Realization Interface**: This description is the primary input to realization.
- **Input Locking & Resolution**: Sources and input derivations are locked and resolved before realization.
- **Environment Presentation**: Activation is performed by activator packages. The Environment Presentation interface is responsible for discovering and invoking the chosen activator against a derivation's outputs.
- **Provenance**: This structure (plus realization records) is the primary source of truth for explainability. Which activator (if any) was used, and with which inputs, must be traceable.

## Non-Goals (for this interface)

- Defining *how* a derivation is written by end users (raw data vs. template sugar).
- Service configuration or system state (modules are out of scope for the current focus).
- Full system activation and generations for root-owned targets.

---

This document is intentionally incomplete and opinionated in places. Its purpose is to give us a concrete artifact to grill, refine, and then implement against.

Next interfaces to document (proposed order):
1. Store Interface
2. Realization Interface
3. Input Locking & Resolution Interface
4. Environment Presentation Interface

Provenance is currently treated as cross-cutting.