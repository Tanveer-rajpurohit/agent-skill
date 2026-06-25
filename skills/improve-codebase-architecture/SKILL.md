---
name: improve-codebase-architecture
description: >
  Find deepening opportunities in a codebase, informed by the domain language in
  CONTEXT.md and the decisions in docs/adr/. Use when the user wants to improve
  architecture, find refactoring opportunities, consolidate tightly-coupled
  modules, or make a codebase more testable and AI-navigable.
---

# /improve-codebase-architecture

Surface architectural friction and propose **deepening opportunities** — refactors that turn shallow modules into deep ones. The aim is testability and AI-navigability.

## Architectural Vocabulary

Use these terms exactly in every suggestion. Full definitions in [LANGUAGE.md](./LANGUAGE.md).

- **Module**: Anything with an interface and an implementation.
- **Interface**: Everything a caller must know to use the module correctly.
- **Implementation**: The code inside a module.
- **Depth**: Leverage at the interface (large behavior behind a small interface).
- **Seam**: Where an interface lives; a place behavior can be altered without editing in place.
- **Adapter**: A concrete thing satisfying an interface at a seam.
- **Leverage**: What callers get from depth (more capability per unit of interface).
- **Locality**: What maintainers get from depth (change/bugs concentrated in one place).

## Process

1. **Explore**: Read `CONTEXT.md` and ADRs. Walk the codebase to find friction (shallow modules, tight coupling, untested areas).
2. **Present Candidates**: List numbered deepening opportunities with Files, Problem, Solution, and Benefits. Use architectural and domain vocabulary.
3. **Grilling Loop**: Once a candidate is picked, drop into a conversation to walk the design tree. Update `CONTEXT.md` or create ADRs as needed.

## Supporting Documentation

### [LANGUAGE.md](./LANGUAGE.md)
Full architectural language and principles.

### [INTERFACE-DESIGN.md](./INTERFACE-DESIGN.md)
Pattern for exploring alternative interface designs.
