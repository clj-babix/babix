# Babix

**Babix** is a new Linux distribution / system configuration tool inspired by Nix, NixOS, and Guix, but built around Clojure, data-first design, and a radically simpler user model.

## Current Status

This repository currently contains the **design artifacts** for Babix. We are in the early architectural and requirements phase.

### Core Design Documents

- [VISION.md](VISION.md) — The "why" and long-term ambition
- [docs/DESIGN.md](DESIGN.md) — High-level architecture and key decisions
- [docs/PRD.md](PRD.md) — Product requirements and scope (draft)

### Detailed Interface Specifications

- [docs/interfaces/](docs/interfaces/) — The five core interfaces that define the mandatory kernel:
  - Derivation Description
  - Store
  - Realization
  - Input Locking & Resolution
  - Environment Presentation

## Philosophy (in brief)

- Keep the hard guarantees of Nix/Guix (store, derivations, provenance, atomicity)
- Replace the painful surfaces with data-first, explainable, Clojure-shaped composition
- Builders and activators are ordinary packages
- A tiny core with stable interfaces; almost everything else lives in the package collection
- Users should feel like they are learning *Babix*, not a new language or a port of Nix

## Next Steps

The design is still evolving. Major open areas include:
- Concrete project configuration format
- Bootstrap story
- Detailed activation composition and provenance requirements

We welcome thoughtful review and discussion from people who care deeply about reproducible systems tooling.

## Repository Layout

```
.
├── README.md
├── VISION.md
└── docs/
     ├── interfaces/
     ├── DESIGN.md
     └── PRD.md
```

---

*This is a design-first repository. Code will come once the core contracts and scope are solid.*
