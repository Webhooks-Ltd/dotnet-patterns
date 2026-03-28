# Design Patterns

Cross-cutting design patterns that layer on top of [structural patterns](../structural/). These don't define project structure — they define how data flows through a system, how state is managed, and how reads and writes are separated.

## Patterns

| Ref | Pattern | One-liner |
|:----------|:-------------------------------|:------|
| [`DSG001`](DSG001%20-%20cqrs.md) | CQRS | Separate read/write models — applies within any structural pattern |
| [`DSG002`](DSG002%20-%20event-sourcing.md) | Event Sourcing + CQRS | Events as source of truth, projections for reads — builds on DSG001 |

## How These Relate to Structural Patterns

Design patterns are not standalone architectures. They modify how you handle specific concerns within a structural pattern:

| Design Pattern | Commonly paired with | What it changes |
|:---------------|:---------------------|:----------------|
| CQRS (DSG001) | STR003, STR004, STR008 | Splits read and write code paths; query handlers bypass the domain model |
| Event Sourcing (DSG002) | STR003, STR006 | Replaces current-state persistence with append-only event streams; requires CQRS for reads |

**Start with a structural pattern.** Add a design pattern only when you have a concrete problem it solves.
