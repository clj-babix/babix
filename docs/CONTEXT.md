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
A capability supplied by a Package whose outputs reside in the Store. A Builder is itself the output of a realized Derivation (i.e., an ordinary package in the Package Collection that depends on whatever tools and prior packages it needs, such as a Rust compiler or a generic phased build framework). When a downstream derivation needs build logic, it includes the specific builder derivation as one of its Specification Inputs under `:inputs` and references it by namespaced keyword (e.g. `:babix.builders/python`). The core contains no language-specific build knowledge; it only knows how to discover the capability manifest from a realized store artifact and invoke it according to the stable contract. The seed (the ultimate host-environment assumption from which the first builder package is itself built) is not a Builder.
_Avoid_: builtin builder, stdenv in core, in-process symbols, manifests that do not correspond to store artifacts

**Activator**:
A capability supplied by a Package whose outputs reside in the Store. An Activator is itself the output of a realized Derivation (i.e., an ordinary package in the Package Collection). A derivation declares its chosen activator by namespaced keyword into its Specification Inputs exactly as it does for a Builder. The core discovers the activator manifest from the realized store artifact and invokes it to produce roots, activation scripts, and environment setup. Activation is first-class on every derivation.
_Avoid_: special activation section, embedded activation, in-process symbols, manifests that do not correspond to store artifacts

**Derivation (Derivation Description)**:
The fundamental unit of work. A primarily data (EDN map) that describes how to produce one or more outputs from a set of inputs. A derivation declares what it produces, its inputs (categorized), which builder and/or activator to use, sandbox requirements, and runtime expectations. Every package in the Store is the result of realizing one Derivation Description (plus its locked inputs). The derivation hash that identifies a particular immutable plan is computed over the locked form of the description (direct references resolved to stable identities in the manner described under Stable Identifier and Lock). Changes to the chosen packages for any referenced logical name (including builders and activators supplied by a package namespace) produce a new locked form and therefore a new derivation hash for the consuming derivation. Transitive changes deep in a referenced package's closure change that package's identity, which then appears as a new concrete identity in the direct inputs of anything that references it; the identity change percolates without requiring embedding of transitive closures in every consumer's locked form.
_Relationship_: Common substrate for packages, development environments, and collector targets.

**Specification Inputs**:
The closed scope of concrete package instances (i.e., specific realized store artifacts, identified by derivation hash or equivalent stable identity) passed to a derivation under the `:inputs` map. This is the set from which the derivation may reference builders, activators, and other dependencies by keyword. Because builders and activators are themselves packages, they appear here as first-class members of the closed input set exactly like any other dependency.
_Avoid_: ambient discovery, global registry

**Categorized Inputs**:
References into the specification inputs that describe the role of a package: `:build-inputs` (perform the build), `:host-inputs` (result depends on at runtime), `:propagated-host-inputs` (transitive runtime propagation).
_Avoid_: nativeBuildInputs / buildInputs (legacy confusing names)

**Stable Identifier**:
A small map (or equivalent structure) that pins a reference to an immutable identity, typically containing the original logical name and the derivation hash or store identity.

At locking / materialization time, only the `:inputs` section of a Derivation Description is replaced in-place with stable identifiers. Every entry under `:inputs` becomes a dereferenced form (shape `{:ref <keyword> :id <store-identity>}` or equivalent) that carries both the original reference name and the concrete stable identity. References that appear elsewhere in the derivation description — in categorized input lists (`:build-inputs`, etc.), in builder or activator arguments, inside strings, or in any other body data — remain as keywords that point into the now-pinned `:inputs` map. The keyword that appears as the value of `:builder` or `:activator` itself is never replaced; only the corresponding entry under `:inputs` is turned into the dereferenced form. This keeps the capability selection visible as a named choice while still pinning the exact package chosen.

The derivation hash is computed over the Canonical Locked Form (the locked form after inputs-only materialization, serialized deterministically so that logically identical content always produces the same hash bytes). The consuming derivation's hash includes the concrete identity of the chosen builder/activator package (via the dereferenced inputs entry), but does not embed that package's own transitive closure.

_Avoid_: assuming a full static walk replaces references throughout the entire derivation body; assuming the locked form contains the full transitive closure of its builder or activator packages; assuming the keyword in the `:builder` or `:activator` field is itself replaced during locking; assuming the dereferenced form for a builder/activator reference is inlined into the `:builder`/`:activator` field itself; assuming the transitive dependencies of a referenced builder or activator package participate in the consumer's derivation hash or appear in its locked specification; assuming the keyword in `:builder` or `:activator` is ever materialized during locking; assuming the keyword in `:builder` or `:activator` is ever replaced by its dereferenced form; treating materialization as a broad "replace every reference anywhere" operation rather than an inputs-section-only operation.

**Canonical Locked Form**:
The exact byte sequence that is fed to the derivation hash function. It is produced by taking the locked form of a Derivation Description (the authoring data after only its `:inputs` section has been replaced in-place with stable identifiers, while all other references remain as keywords into that now-pinned map) and serializing it with a deterministic serializer.

The serializer must guarantee that any two representations of logically identical content produce identical bytes, regardless of superficial differences in the input. Deterministic serialization requires sorted map keys, consistent representation of keywords versus strings, and removal of all insignificant whitespace, newlines, and comments (and any other formatting that does not affect the logical EDN structure). The derivation hash is always computed over the Canonical Locked Form, never over an arbitrary pretty-printed or author-supplied serialization.

Body keyword references that appear outside the `:inputs` section (in categorized inputs, builder or activator arguments, strings, and other data) are treated as opaque tokens by the deterministic serializer. The specific keyword names chosen in the body participate in the derivation hash. Two Canonical Locked Forms that differ only in the spelling of such a body keyword (even when both keywords resolve to the same stable identifier in the pinned `:inputs` map) will produce different derivation hashes. Semantic equivalence after pinning is not normalized at the hash level.

This ensures that the same logical plan always yields the same derivation hash even when the order of keys or whitespace have been changed between authoring, transmission, or storage.

**Collector Derivation**:
A derivation whose primary purpose is to aggregate other derivations (as inputs) and produce a combined activation. Collectors are ordinary derivations; they are not a separate core kind.
_Relationship_: Large roots can remain lazy at the graph level; per-derivation closure is required only for what is actually realized.

**Source Mode**:
An explicit declaration of how a source input should be materialized (e.g. `:worktree`, `:git-head`, `:archive`, `:path`). Source modes make reproducibility level and “dirtiness” first-class and visible in the lockfile.
_Avoid_: implicit Git behavior

For namespace inputs (remote package collections), the nature of the source (git repository, tarball, local path, etc.) is not declared via an explicit `:type` or `:mode` field. It is carried in the URL itself using standard URL schemes, query parameters, and path conventions. The system infers the concrete materialization strategy after fetching. This keeps the human surface minimal while the locked form records the exact resolved identity.

**Lock / Lockfile**:
The artifact that captures the exact, resolved state of all external inputs for a project or derivation, including reproducibility level. The lockfile is the primary mechanism for reproducible environments and builds. For references under `:inputs`, the lockfile records the concrete stable identities (including for builder and activator packages). References appearing as the value of `:builder` or `:activator` remain as logical keywords in the locked form; the corresponding input entry is turned into a dereferenced form (shape `{:ref <keyword> :id <store-identity>}` or equivalent) so the choice is fully pinned. The lockfile does not embed the full transitive specifications of builder or activator packages; those are consulted from the store when the builder/activator package itself must be realized or its environment prepared. The graph of dependencies is represented implicitly through these references in `:inputs`; there is no separate visualization layer at present.

A derivation hash is computed over the Canonical Locked Form of a Derivation Description (the locked form after only its `:inputs` section has been replaced with stable identifiers, with all other references remaining as keywords into the now-pinned inputs map, serialized deterministically so that logically identical content always produces the same hash bytes). The resulting hash identifies the immutable plan. Any change to the locked inputs — including a change in which concrete builder or activator package was chosen for a given logical keyword — produces a new derivation hash for that derivation. Transitive changes in a builder package's own dependencies change that builder package's identity, which then appears as a new concrete identity in the direct inputs of anything that references it, causing the identity change to percolate to all dependent derivations without requiring their locked forms to embed transitive closures.

The exact byte sequence produced by the deterministic serializer (the Canonical Locked Form bytes) is the durable artifact that is persisted — for example in lockfiles and alongside realization records in the store. The derivation hash is used as part of the stable identity and storage key (e.g., as part of filenames) for these artifacts. This enables direct byte-level caching and avoids re-serialization or re-hashing when the same locked description is encountered again.

When a derivation hash is known, loading the stored Canonical Locked Form gives both the exact plan bytes and (via the content-addressed store) the realized package outputs for that plan, because the derivation hash and the output content hash participate together in the CAS addressing. The Canonical Locked Form bytes literally contain the (dereferenced) `:inputs` map as part of the serialized structure; the `:id` values in that map are the derivation hashes of the direct parent inputs. These parent hashes are directly digested by the derivation hash function — precisely analogous to how a Git commit object contains `parent <hash>` lines. 

As part of producing the canonical bytes, inputs that were present in the original non-locked authoring form but whose references were never actually used inside the description are pruned; only inputs with positive reference count inside that same description survive into the serialized form. Because the `:id` values are (or contain) the actual parent hashes, the "direct embedding of parent hashes" view and the "input identity map whose values are the hashes" view are equivalent for hashing purposes. The resulting derivation hash therefore transitively captures the entire reachable dependency subtree in a Merkle-DAG, so that changing any ancestor necessarily changes all descendant hashes.

**Merkle Derivation Graph (Babix Merkle-DAG)**:
The cryptographic structure formed by derivation plans and their realized outputs. Each Canonical Locked Form (the exact bytes persisted for a derivation) contains direct references to its parent derivations by their derivation hashes; these parent hashes are part of the data that is hashed to produce the current derivation's identity. Unused inputs declared in the authoring form are pruned during canonicalization, so only actually referenced parents appear. The derivation hash therefore transitively binds the entire reachable history of ancestors.

The seed (the ~229-byte hex0-seed or equivalent host capability) is a precondition outside the DAG; its hash is the ultimate stable identifier and the leaf from which the entire graph grows. The seed is a pure assembler: it reads hex digits, labels, and comments from a source file and emits the corresponding flat machine code. It is the only pre-compiled artifact in the bootstrap chain; everything above it (the full toolchain and the first Babix packages) is built from source in a fully auditable ladder of 15 phases (hex0-seed → hex0 → hex1 → hex2-0 → M0 → M2-Planet → blood-elf → kaem → M1 → hex2 → mes → mescc → tcc → gcc and beyond). Verification uses content-hash pinning: the seed is executed as the builder of a derivation whose output hash is required to match the seed's own hash exactly. Any tampering produces a detectable mismatch. The seed is a host-environment assumption (analogous to the kernel and syscall ABI), not a derivation and not a Builder package.

Both the Canonical Locked Form (the plan) and the actual binary contents written to each output (the "blobs") are content-addressed by cryptographic hash. This combination — a Merkle-DAG of plans with parent pointers plus full content-addressing of binary artifacts — is a deliberate architectural commitment. It delivers strong tamper-evidence, verifiable inclusion of any ancestor, and resistance to certain supply-chain attacks as emergent properties, without requiring additional mechanisms in the core.

Foundational references:
- Ralph Merkle. "A Certified Digital Signature." (1979 / 1989)
- The Merkle tree / Merkle DAG construction as presented in R. Merkle's work (see e.g. the readings at https://people.eecs.berkeley.edu/~raluca/cs261-f15/readings/merkle.pdf and https://people.eecs.berkeley.edu/~raluca/cs261-f15/readings/merkleodb.pdf).

**Realization**:
The process of turning a locked derivation description into actual store artifacts under sandboxed conditions. This is the primary pluggable point for different isolation mechanisms. When realizing a derivation that names a builder or activator package, realization first resolves the chosen builder/activator from the dereferenced entry in the locked `:inputs` (the `{:ref <keyword> :id <store-identity>}` form), ensures that package is available in the store, reads that package's own locked specification from the store, and uses it to prepare the environment in which the builder or activator logic will execute. The consuming derivation's locked form identifies *which* builder/activator package was chosen (keyword stays in `:builder`/`:activator`; concrete identity lives in the dereferenced inputs entry) but does not duplicate that package's own closure or transitive dependencies. The builder/activator package is unlikely to appear in the consuming derivation's categorized inputs (`:build-inputs`, `:host-inputs`, etc.); it is referenced primarily via the `:builder` or `:activator` field. Transitive dependencies of the builder/activator are discovered by reading that package's own specification from the store at the time its environment must be prepared; they do not appear in the consuming derivation's specification at any stage.

**Store**:
The globally unique, content-addressed, immutable storage layer shared by everyone participating in the Babix ecosystem (the equivalent of /nix/store). It is the single source of truth for all built artifacts (packages), their provenance, and the capability manifests they publish. Anyone can pull existing packages from the store and push newly realized ones into it. All dependencies — including builders, activators, and other capability providers — must be drawn from the store as realized packages. The store enables safe sharing, deduplication, reference tracking, and provenance across machines and time.

Babix adopts content-addressed storage as the primary model from the outset: the derivation hash identifies the immutable plan (locked description + input identities), while the output identity is the cryptographic content hash of the actual bytes produced. Realisation records (or their Babix equivalent) map plan identities to concrete content-addressed paths and their dependency closures. Input-only (purely derivation-hash-based) addressing is not used as a foundational path.

**Stable Store Interface Library**:
A stable, versioned library (initially implemented in Clojure) whose sole responsibility is to provide the concrete implementations of the Babix Store Interface (and related contracts) for interacting with any store that satisfies the interface. Alternate Babix implementations, higher-level tools, or even implementations in other languages can depend on this library for all store interaction while providing their own evaluation, realization orchestration, or user experience layers. The library is the engineering boundary that keeps the Store Interface contract independent of any particular Babix runtime.

**Package**:
A realized output (or set of outputs) of a Derivation that has been materialized into the Store under a content address. Packages are the things that participate in the dependency graph. A package may provide capabilities (as a Builder, Activator, or other kind) by publishing a manifest in its store outputs. When a downstream derivation depends on a package, it references a specific store identity (ultimately a derivation hash plus output name) under its Specification Inputs. The Package Collection produces packages; the Store holds them.

**Package Collection**:
The vast majority of Babix’s functionality lives here: language builders, activators, stdenv-like constructs, environment tools, higher-level frameworks, etc. The collection is expected to evolve much faster than the core. Every package in the collection is itself the output of one or more Derivations that were realized into the Store.

**Core**:
The minimal trusted base that defines the five stable interfaces, performs evaluation, orchestrates flows between the interfaces, and guarantees provenance.

**Standard Library**:
A small set of stable, pure helper functions (the “lib” equivalent) for common data transformations. Changes extremely slowly. Everything else should be expressible as packages.

**Purely Functional Deployment Model**:
The foundational architecture (Dolstra 2006) in which software components and system configurations are built by pure functions from explicit inputs to immutable, content-addressed outputs. The store path of every artifact cryptographically embeds the hashes of all inputs that produced it. This single mechanism simultaneously prevents undeclared dependencies, enables safe side-by-side installation, provides isolation, makes upgrades atomic, and supports O(1) rollbacks via roots. Babix adopts this model as its semantic core while replacing the original language surface with data-first EDN + ordinary packages.

**Bootstrap Seed** (also called Trust Anchor or Seed):
The minimal, hand-verifiable pre-compiled binary that serves as the ultimate host-environment assumption and root of trust for a Babix installation. In the current best-known architecture (the stage0-posix / hex0 chain from the bootstrappable community), this is a single ~229-byte ELF (181 bytes of machine code plus a 48-byte header) whose only job is to read hex digits from a source file and emit the corresponding machine code. It is a pure assembler; the compiler is built later, by the assembler, from source. Everything above the seed (the full toolchain and the first Babix packages) is built from source in a fully auditable 15-phase ladder: hex0-seed → hex0 → hex1 → hex2-0 → M0 → M2-Planet → blood-elf → kaem → M1 → hex2 → mes → mescc → tcc → gcc and onward.

The seed is verified by content-hash pinning: it is executed as the builder of a derivation whose output hash is required to match the seed's own hash exactly. Any tampering (or any deviation in behavior) produces a detectable hash mismatch. The seed is *not* a derivation, *not* a Builder package, and lies *outside* the Merkle-DAG; it is a precondition for the first store contents and the first realization (analogous to the kernel and syscall ABI). Its hash is the ultimate stable identifier and the leaf from which the entire graph grows.

A pragmatic cross-build from an existing Nix/Guix system is acceptable for the first horizon (producing the initial working `babix` binary and package collection). A true stage0-style self-hosting bootstrap (a `babix-seed` that can itself be built entirely inside a Babix store) is a later-horizon deliverable required for Babix to be a functional package manager in the strong sense. The design must not preclude it.

**Fixed-Output Derivation** (historical / background):
In the classic purely functional deployment model (Dolstra 2006), a special kind of derivation whose output content hash is declared in advance, allowing controlled impurity (network, etc.) only for that step while pinning the result by content hash. This was the escape hatch for source fetches and similar non-pure steps.

Babix improves on this by avoiding a distinguished "fixed-output derivation" kind in the core data model or evaluation. Instead, source fetching, vendoring, and building are expressed uniformly as ordinary derivations in a multi-derivation graph (see "Multi-derivation specifications" and the example dialogue below). A source-fetch derivation is just another derivation; it declares its expected content hash and is granted the necessary sandbox privileges (via its `:sandbox` policy or the builder package it names). The next derivation (e.g., dependency installation) simply receives its output as a normal specification input under `:inputs`. The final build derivation does the same. All steps share the exact same Derivation Description shape. The "controlled impurity + declared hash" concern is expressed through the uniform mechanisms rather than a special derivation variant. This keeps the core smaller and the mental model more regular. 

See also "Source Mode", "Two-layer hashing", and the example dialogue ("All three are ordinary derivations").

**Rebuildability vs Bitwise Reproducibility**:
Rebuildability: the ability to re-execute a historical locked derivation description with its exact inputs and obtain a result (empirically >99% in large-scale nixpkgs studies). Bitwise reproducibility: obtaining *bit-identical* artifacts on independent rebuilds. The purely functional model gives excellent rebuildability "for free"; bitwise reproducibility requires additional engineering discipline (SOURCE_DATE_EPOCH, stripping embedded build paths, avoiding uname leakage, deterministic toolchains). Babix provenance and `babix explain` should distinguish these two properties and surface the specific causes when bitwise identity fails (see diffoscope-style recursive diffs).

**Runtime Closure / Reference Scanning**:
The set of store paths that are actually reachable at runtime from a package's outputs (as opposed to the declared build-time inputs). In practice this is discovered by scanning ELF/Mach-O headers, rpaths, shebangs, wrapper scripts, etc., rather than relying solely on declared dependencies. Tight runtime closures are essential for correct garbage collection, minimal environment activation, and provenance queries. Builder and activator packages are responsible for producing outputs whose runtime references are clean; the store records the resulting reference graph.

**Roots and Generations**:
A root is an explicit reference that keeps a store path (or profile) alive against garbage collection. Generations are timestamped snapshots of a profile (user environment, system configuration, etc.). Switching to a new generation is an atomic root swap; old generations remain available for rollback. This mechanism, inherited from the original store model, gives users safe, instantaneous rollbacks and a clear audit trail of environment history. Collector derivations and activators ultimately produce roots.

**Nominal vs Exact Dependencies**:
Nominal dependencies name a package by string + optional version constraints ("python ≥ 3.11"). Exact dependencies name a specific immutable store artifact by its derivation hash (or content hash for fixed-output cases). The functional model requires exact dependencies for reproducibility and provenance; authoring surfaces may use nominal references for ergonomics, but locking materializes them to exact stable identifiers before hashing or realization.

**Builder Provenance / Provenance Log** (advanced / future direction):
Verifiable metadata attached to realization records or trace map entries that records which concrete builder (package instance + attested software stack + remote attestation evidence) performed a realization, plus evidence of what dependency resolution outcomes that builder actually chose for its own inputs. In decentralized or multi-builder settings this allows consumers and `babix explain` to apply user-defined trust policies at verification time rather than transitively trusting every upstream builder and cache. This builds on Babix's existing cross-cutting provenance requirement without changing first-horizon core interfaces.

**Binary Cache** (future / deferred concern):
A remote service or content-addressed store that serves pre-realized outputs (store artifacts) so that clients can fetch them instead of realizing derivations locally. In the current horizon Babix is intentionally local-first; remote binary caches and the full distributed trust model for them are explicitly out of scope (see PRD.md). When support is added later, the provenance, realization records, and (advanced) builder attestation mechanisms must allow consumers to verify that a cached artifact was produced under conditions they consider acceptable, rather than having to transitively trust the cache or its upstream builders. See also the Store Interface (initial local semantics) and the SCORED '24 paper on eliminating transitive trust in cloud build systems.

---

## Decisions from RFC Review (2026)

These decisions were recorded after systematic review of the Nix RFC corpus. They codify stances that align with Babix's core principles. The review considered (among others):

- [RFC 62](https://github.com/NixOS/rfcs/blob/master/rfcs/0062-content-addressed-paths.md) — content-addressed paths
- [RFC 0134](https://github.com/NixOS/rfcs/blob/master/rfcs/0134-nix-store-layer.md) — store layering
- [RFC 0133](https://github.com/NixOS/rfcs/blob/master/rfcs/0133-git-hashing.md) — Git hashing
- [RFC 0097](https://github.com/NixOS/rfcs/blob/master/rfcs/0097-no-read-store-dir.md) — store permissions
- [RFC 0127](https://github.com/NixOS/rfcs/blob/master/rfcs/0127-issues-warnings.md) — structured problems
- [RFC 0032](https://github.com/NixOS/rfcs/blob/master/rfcs/0032-run-phase-changes-for-better-nix-shell-use.md) — phase UX for development environments
- [RFC 0092](https://github.com/NixOS/rfcs/blob/master/rfcs/0092-plan-dynamism.md) — plan dynamism
- [RFC 0140](https://github.com/NixOS/rfcs/blob/master/rfcs/0140-simple-package-paths.md) — package path ergonomics

Each decision below notes the relevant RFC(s) with a direct link.

**Content-addressed storage as primary model**:
Babix commits to content-addressed storage as the primary and foundational model. Both the locked derivation description (plan) and the actual produced content (result) are identified by cryptographic hashes. The derivation hash names the immutable plan; the output identity is the content hash of the bytes written to the promised location. Purely input-addressed (derivation-hash-only) paths are not used as a foundational path. Realisation records map plan identities to concrete content-addressed outputs and their closures.

The stance was informed by designs explored in the broader ecosystem, including [RFC 62](https://github.com/NixOS/rfcs/blob/master/rfcs/0062-content-addressed-paths.md) ("Simple content-addressed store paths"). Babix adopts the approach cleanly from day one without the compatibility tax that came from retrofitting it onto an input-addressed foundation.

**Stable Store Interface Library**:
The Store Interface (and related contracts) is reified as a stable, versioned library whose only job is to implement the interactions required by the interface. Any Babix implementation, alternate runtime, or even a tool written in another language can depend on this library to talk to any store that satisfies the Babix Store contract. This is a deliberate engineering boundary that keeps the interface independent of any one runtime or language. Concrete term: "Stable Store Interface Library."

The framing was informed by work on store layering in the ecosystem, including [RFC 0134](https://github.com/NixOS/rfcs/blob/master/rfcs/0134-nix-store-layer.md) ("nix-store-layer").

**Source origin agnosticism and content-hash primacy**:
Babix remains deliberately agnostic to the upstream origin of source material. When materializing content from Git (or any other source), the stable identity that matters is the cryptographic hash of the actual bytes materialized plus the locked derivation description that performed the fetch. Git-internal hashes (blob, tree, commit) are not part of the core identity model. Rev/tag resolution can occur at realization time as part of materialization, but the core does not privilege Git's hashing scheme. This follows directly from the content-addressed primary model and the desire to avoid baking any particular upstream format into the trusted base (see Input Locking & Resolution Interface).

**Store permission and visibility policy is deferred**:
Decisions about specific on-disk permissions for the store (e.g., 1735-style, group readability, discovery resistance via `o-r`, etc.) are explicitly deferred. The Store Interface does not encode a particular permission or visibility model in v1. Implementations may choose safe defaults on creation, but the policy is not part of the core contract. The topic remains open for later horizons.

The question was surfaced during review of [RFC 0097](https://github.com/NixOS/rfcs/blob/master/rfcs/0097-no-read-store-dir.md) ("nix-store-perms").

**Temporal advisories in derivation specifications**:
Derivation descriptions carry an optional mechanism for temporal advisories (deprecation, removal, security notes, "no longer recommended," etc.). These are placed directly in the specification so that `babix explain`, activators, and environment presentation can surface them to users without requiring external databases or side channels. This addresses the temporal aspect of package health in a data-first way.

The approach was informed by structured problem-reporting designs, including [RFC 0127](https://github.com/NixOS/rfcs/blob/master/rfcs/0127-issues-warnings.md) ("Nixpkgs 'problem' infrastructure").

**Explicit rejection of plan dynamism / computed derivations**:
Babix rejects the model in which a derivation can dynamically produce new derivation descriptions that then become new build goals. Instead, Babix uses an inversion: any step that would have required a "derivation that produces a derivation" is expressed as a prior ordinary derivation whose output is supplied as a specification input to a subsequent ordinary derivation. Multi-derivation graphs (e.g., source-fetch → dependency-vendor → build) already solve this class of problems. The core never has to treat a derivation's output as a new plan to be scheduled. This keeps evaluation, hashing, and realization semantics simple and uniform. All steps remain ordinary derivations; there is no distinguished "plan-producing" kind.

The inversion was chosen after considering dynamic plan models explored in the ecosystem, including [RFC 0092](https://github.com/NixOS/rfcs/blob/master/rfcs/0092-plan-dynamism.md) ("plan-dynamism-experiment").

---

## Example Dialogue

A developer and a reviewer discussing a new package:

**Dev:** I want to add a new package for ripgrep. It needs to fetch the source, vendor its dependencies, then build.

**Reviewer:** Will you model that as one derivation or three?

**Dev:** Three — source fetch (fixed-output), dependency fetch (also fixed-output), then the actual build. All three are ordinary derivations. The build one will take the two fetch outputs as specification inputs under `:inputs`, reference them via keywords in `:build-inputs`, and name a Cargo builder package the same way.

**Reviewer:** How do you make sure only the actual references get materialized?

**Dev:** At locking time we do a full static walk over the inputs. Ordinary references are turned into stable identifiers in place. However, the `:builder` and `:activator` keywords remain as logical names in the locked form; only their corresponding entries under `:inputs` become dereferenced forms (shape `{:ref <keyword> :id <store-identity>}` or equivalent) carrying both the reference name and the concrete stable identity from the package namespace. The locked form of the derivation pins exactly which builder and activator packages were chosen (keyword stays in `:builder`/`:activator`; concrete identity lives in the dereferenced inputs entry), but does not embed those packages' own transitive closures — those are discovered from the store when the builder or activator itself must be realized or its environment prepared. The big package namespace can still be supplied for ergonomics; only the bits that are actually named get closed over. The builder package is referenced primarily via the `:builder` field and the corresponding dereferenced entry under `:inputs`; it is unlikely to appear in the consuming derivation's categorized inputs. Transitive dependencies of the builder do not appear in the consuming derivation's specification at any stage.

**Reviewer:** And activation?

**Dev:** The final derivation for the dev environment will name an activator package via `:activator` (a keyword that stays as the logical name in the locked form). That activator is itself a realized package in the store; the derivation pins the choice by keeping the keyword in `:activator` and recording the concrete identity as a dereferenced entry (shape `{:ref <keyword> :id <store-identity>}`) under `:inputs`. The activator package knows how to produce the root and the activation script. Collectors are just more derivations — same shape, different purpose. When the activator (or builder) package must itself be realized or its environment prepared, its own specification is read from the store using the identity recorded in the dereferenced inputs entry of the consuming derivation. There is currently no separate visualization layer for the graph of dependencies; the graph is implicit through the references in `:inputs`. The builder/activator package is unlikely to appear in the consuming derivation's categorized inputs; it is referenced primarily via the `:builder` or `:activator` field. Transitive dependencies of the activator (or builder) do not appear in the consuming derivation's specification at any stage.

This conversation uses the project’s canonical terms with the intended boundaries.

**Further reading** (foundational sources for the model and empirical grounding)
- Dolstra 2006 (PhD thesis) — the original formalization of the purely functional deployment model, closures, the historical fixed-output derivation mechanism (special derivation kind for source fetches), reference scanning, roots, and the store-as-heap. Babix uses the same underlying model but expresses source fetching, vendoring, and building uniformly as ordinary derivations in a multi-derivation graph.
- Dolstra et al. JFP 2011 (NixOS paper) — extension to full system configuration, the module system as a purely functional composition mechanism, and honest discussion of purity compromises for real OS state.
- Malka et al. MSR 2025 — first large-scale empirical validation (709k+ builds) that the model delivers 69–91% bitwise reproducibility and >99% rebuildability at 100k-package scale.