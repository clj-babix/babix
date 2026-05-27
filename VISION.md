# Babix Vision

**One-line thesis**  
Babix brings the strongest ideas from Nix and Guix — content-addressed storage, reproducible derivations, explicit dependency graphs, and atomic environments — into a system whose surface is data-first, explainable by default, and built around Clojure-shaped composition rather than a custom language or accumulated folklore.

---

## The Problem

Nix and Guix solved real, hard problems:

- Reproducible builds
- Content-addressed, immutable artifacts
- Declarative dependency graphs
- The ability to roll back, share, and reason about entire environments

Yet the experience of using these systems remains painful for most people, especially when trying to do the simplest valuable thing: enter a reproducible development environment or build your own software against a locked set of dependencies.

The dominant pain is not the core model. It is the surface:

- A language whose ergonomics and error messages feel hostile to newcomers and exhausting to experts.
- Flakes that made some things better while introducing new ceremony, implicit Git behavior, and confusing boundaries.
- A module system whose power is real but whose complexity is rarely justified for development work.
- Accumulated cruft (deep wrapper layers, special cases, hidden behavior) that forces ever-more-complex tooling (dream2nix, flake-parts, etc.) just to make the system usable.
- Poor causality: when something rebuilds or breaks, it is often difficult to know why.

The result is a powerful kernel surrounded by a thick layer of accidental complexity that new users must fight through and experienced users must constantly work around.

## The Opportunity

There is a better path.

We can keep the hard guarantees (the store, derivations, hermetic builds, strong provenance) while replacing the painful surfaces with something that feels like a natural, data-oriented tool rather than a foreign cathedral.

Clojure (and Babashka's pragmatic execution model) offers an unusually good substrate for this:

- Plain data as the primary representation.
- Small, pure functions as the composition mechanism.
- Excellent support for introspection, transformation, and error explanation.
- A culture that values simplicity and explicitness.

Babix is the attempt to do this properly — not as a thin wrapper or syntax port, but as a first-principles re-implementation of the valuable core with a radically simpler and more honest user model.

## Core Thesis

> Reproducible systems should compose like data.

A package, a build plan, a development environment, and a lockfile should primarily be data — readable, diffable, queryable, and transformable by ordinary functions. The language should be an escape hatch for when you genuinely need power, not the default medium in which you live.

Babix should feel like the tool you reach for when you want a development environment you can trust, explain, and reproduce — without requiring you to learn a new programming language or accept hidden complexity as the price of reproducibility.

## References, Identifiers, and Surfaces

Babix maintains a deliberate separation between the surfaces humans use to describe and reason about systems and the internal representations required for correctness and reproducibility.

Humans work effectively with references — short, contextual names that are easy to remember, discuss, and evolve. Authoring descriptions, project configuration, and day-to-day mental models live in this reference-oriented layer. People are not good at managing long, opaque identifiers at scale; they are good at reasoning with names that carry local meaning.

Once a description is locked, references are resolved and replaced in place with stable identifiers. The resulting locked forms are fully explicit, self-contained data structures. Their primary consumers are programs: the evaluator, the hasher, the realization layer, provenance tools, and other machines that must make reliable, reproducible decisions across time and contexts. This separation — ergonomic references for humans, durable identifiers for machines — is not a compromise. It is the mechanism that lets Babix deliver both excellent local ergonomics and strong guarantees without forcing either side to do the other's job.

Data remains the primary medium throughout. The difference is which layer is allowed to stay in the reference form and which layer must commit to identifiers.

## Guiding Principles

- **Data first, functions when needed, macros last.** The lowest useful representation is structured data (EDN). Functions are the tool for transformation and abstraction. Hidden magic is a bug.
- **Explainability is not optional.** Every decision the system makes must be traceable. `babix explain` is a first-class primitive, not a debugging afterthought.
- **Keep the hard parts, remove the accidental ones.** Store semantics, derivations, hermetic builds, locking, and provenance are valuable. Deep wrapper layers, implicit Git behavior, overlay confusion, and poor error messages are not.
- **The core must stay small.** Language-specific builders, environment setup logic, and most policy belong in the package collection, not the core tool. This keeps the trusted base minimal and the ecosystem evolvable.
- **Interfaces over implementations.** Sandboxing, storage, and realization strategies should be pluggable behind stable contracts. This enables portability, experimentation, and future projects (including potential OS-level work) without rewriting the foundation.
- **Users learn Babix, not Clojure.** The primary experience must feel like using a purpose-built tool. The fact that the implementation substrate is Clojure and the data format is EDN should be an implementation detail for most users, not a prerequisite.
- **Explicitness over ceremony.** Source modes are explicit. Runtime environment requirements are explicit. What will change a build is explicit. Hidden behavior is minimized.

## What Babix Is

- A foundation for reproducible development environments and package builds.
- A system in which derivations are primarily data, with controlled escape hatches into code.
- A platform designed so that higher-level tools and collections (the equivalent of flake-parts or smart builders) can be built *on top* rather than having to work around fundamental rough edges.
- A long-term bet on a cleaner, more explainable substrate for reproducible systems work.

## What Babix Is Not (in the first major horizon)

- A full replacement for NixOS or nix-darwin system configuration (modules and privileged system activation are out of scope initially).
- Another skin over the existing Nix daemon or Nix language.
- A general-purpose configuration management tool.
- A project that requires users to become functional programmers or Clojure experts to be productive.

## Scope and Horizons

**First horizon (the current focus):**  
A real, production-grade replacement for the combination of `nix develop` + flakes + the ability to define and build your own packages, with strong provenance and a dramatically better user experience. This is the "clone of nix develop that includes the better parts of flakes and derivations" — done from first principles.

**Later horizons:**  
- Unified targets (dev, home, project, system) under one model.
- A powerful but simpler module system.
- First-class support for system-level targets (BabixOS ambitions).
- Rich higher-level frameworks built on the solid core.

We do not version the system. We have Babix. It grows and improves, but we do not ship "Babix 0.1" as a temporary placeholder we intend to replace.

## Success Criteria

We will know Babix is working when:

- A competent developer can define a locked, reproducible development environment for a non-trivial project in minutes, not hours, and understand why it behaves the way it does.
- Adding a new package to the collection does not require fighting the core tool or learning hidden conventions.
- `babix explain` can answer meaningful questions about why something built or changed, without requiring deep system archaeology.
- People begin building higher-level tools and collections on top of Babix because the core contracts are stable and the accidental complexity is low.
- The system feels like it was designed, not accreted.

---

Babix exists because the hardest parts of reproducible systems are worth preserving, but the current experience of using them is not. We can do better. This is the attempt to prove it.