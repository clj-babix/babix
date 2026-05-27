# Babix — Context and Terminology

**Purpose**  
This document defines the core concepts and terminology used in the Babix design. Its goal is to ensure that everyone working on (or reviewing) Babix is using the same words to mean the same things.

This is a living document. As design decisions are made or refined, this file should be updated.

---

## Language

**Data first**:
The primary representation for almost everything — derivations, packages, environments, lockfiles — is plain EDN data. Logic and behavior are expressed as small pure functions that transform that data, or as packages that provide capabilities.
_Avoid_: code-first, everything-is-functions

**Lean core**:
The trusted core is deliberately small. It defines stable interfaces and provides minimal evaluation and orchestration. Almost all policy, builder logic, and higher-level behavior lives in the package collection as ordinary packages.
_Avoid_: fat core, builtin-heavy

**Builder**:
A capability supplied by a package that knows how to turn a derivation description into real store artifacts. Builders are referenced by namespaced keyword (e.g. `:babix.builders/python`) and resolved from the derivation’s own specification inputs. The core contains no language-specific build knowledge.
_Avoid_: builtin builder, stdenv in core

**Activator**:
A capability supplied by a package that knows how to turn the outputs of a derivation into an activated environment (root + activation script + environment setup). Activation is first-class on every derivation.
_Avoid_: special activation section, embedded activation

**Derivation (Derivation Description)**:
The fundamental unit of work. A primarily data (EDN map) that describes how to produce one or more outputs from a set of inputs. A derivation declares what it produces, its inputs (categorized), which builder and/or activator to use, sandbox requirements, and runtime expectations.
_Relationship_: Common substrate for packages, development environments, and collector targets.

**Specification Inputs**:
The closed scope of concrete package instances passed to a derivation under the `:inputs` map. This is the set from which the derivation may reference builders, activators, and other dependencies by keyword.
_Avoid_: ambient discovery, global registry

**Categorized Inputs**:
References into the specification inputs that describe the role of a package: `:build-inputs` (perform the build), `:host-inputs` (result depends on at runtime), `:propagated-host-inputs` (transitive runtime propagation).
_Avoid_: nativeBuildInputs / buildInputs (legacy confusing names)

**Stable Identifier**:
A small map that pins a reference to an immutable identity, typically containing the original logical name and the derivation hash or store identity. References are replaced in-place with stable identifiers before the derivation hash is computed so the locked form is fully explicit and self-contained.
_Avoid_: adding new top-level sections like :closed-inputs everywhere

**Collector Derivation**:
A derivation whose primary purpose is to aggregate other derivations (as inputs) and produce a combined activation. Collectors are ordinary derivations; they are not a separate core kind.
_Relationship_: Large roots can remain lazy at the graph level; per-derivation closure is required only for what is actually realized.

**Source Mode**:
An explicit declaration of how a source input should be materialized (e.g. `:worktree`, `:git-head`, `:archive`, `:path`). Source modes make reproducibility level and “dirtiness” first-class and visible in the lockfile.
_Avoid_: implicit Git behavior

**Lock / Lockfile**:
The artifact that captures the exact, resolved state of all external inputs for a project or derivation, including reproducibility level. The lockfile is the primary mechanism for reproducible environments and builds.

**Realization**:
The process of turning a locked derivation description into actual store artifacts under sandboxed conditions. This is the primary pluggable point for different isolation mechanisms.

**Store**:
The content-addressed, immutable storage layer. The single source of truth for built artifacts and their provenance.

**Package Collection**:
The vast majority of Babix’s functionality lives here: language builders, activators, stdenv-like constructs, environment tools, higher-level frameworks, etc. The collection is expected to evolve much faster than the core.

**Core**:
The minimal trusted base that defines the five stable interfaces, performs evaluation, orchestrates flows between the interfaces, and guarantees provenance.

**Standard Library**:
A small set of stable, pure helper functions (the “lib” equivalent) for common data transformations. Changes extremely slowly. Everything else should be expressible as packages.

---

## Example Dialogue

A developer and a reviewer discussing a new package:

**Dev:** I want to add a new package for ripgrep. It needs to fetch the source, vendor its dependencies, then build.

**Reviewer:** Will you model that as one derivation or three?

**Dev:** Three — source fetch (fixed-output), dependency fetch (also fixed-output), then the actual build. All three are ordinary derivations. The build one will take the two fetch outputs as specification inputs under `:inputs`, reference them via keywords in `:build-inputs`, and name a Cargo builder package the same way.

**Reviewer:** How do you make sure only the actual references get materialized?

**Dev:** At locking time we do a full static walk. Every reference becomes a stable identifier in place — the locked form stays the same shape but everything is pinned. The big package namespace can still be supplied for ergonomics; only the bits that are actually named get closed over.

**Reviewer:** And activation?

**Dev:** The final derivation for the dev environment will name an activator package via `:activator`. That activator knows how to produce the root and the activation script. Collectors are just more derivations — same shape, different purpose.

This conversation uses the project’s canonical terms with the intended boundaries.