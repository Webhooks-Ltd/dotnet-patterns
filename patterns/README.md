# .NET Architecture Pattern Catalogue

A reference library of solution-level architecture patterns for .NET/C# projects. Each pattern is a self-contained descriptor that an AI coding agent (or developer) can follow to scaffold and maintain a project without further guidance.

## Quick Reference

| Ref | Pattern | One-liner |
|-----|---------|-----------|
| [`LAY-001`](LAY-001%20-%20n-tier.md) | N-Tier | Single project, Controllers → Services → Repositories |
| [`LAY-002`](LAY-002%20-%20clean-architecture-lite.md) | Clean Architecture Lite | Single project with domain/application/infrastructure boundaries |
| [`LAY-003`](LAY-003%20-%20full-clean-architecture.md) | Full Clean Architecture | Multi-project, compiler-enforced dependency inversion |
| [`FTR-001`](FTR-001%20-%20vertical-slice.md) | Vertical Slice | Feature-per-file, each slice owns its full stack |
| [`FTR-002`](FTR-002%20-%20modular-monolith.md) | Modular Monolith | Independent modules, own data, explicit contracts |
| [`DOM-001`](DOM-001%20-%20hexagonal.md) | Hexagonal (Ports & Adapters) | Domain at centre, ports define all external interaction |
| [`DST-001`](DST-001%20-%20microservices.md) | Microservices | Independent services, own databases, async messaging |

## Categories

- **`LAY` — Layered:** Horizontal separation of concerns. Code is organised by technical role (controllers, services, repositories).
- **`FTR` — Feature-oriented:** Vertical separation. Code is organised by business feature or bounded context.
- **`DOM` — Domain-centric:** The domain model is the architectural centrepiece. Everything else is secondary.
- **`DST` — Distributed:** Multiple independently deployable units.

## Decision Matrix

Use the first row where **all** conditions in the "When" column are true.

| Ref | Pattern | Team | Domain complexity | Deploy targets | Key signal |
|-----|---------|------|-------------------|----------------|------------|
| `LAY-001` | N-Tier | 1–3 | Low (CRUD) | 1 | "It's basically database operations with validation" |
| `LAY-002` | Clean Lite | 2–5 | Medium | 1 | Business rules exist but don't justify multiple projects |
| `LAY-003` | Full Clean | 4+ | High | 1+ | Rich domain model worth protecting with compiler enforcement |
| `FTR-001` | Vertical Slice | 2–8 | Low–Medium | 1 | Features are independent; tired of cross-layer changes |
| `FTR-002` | Modular Monolith | 5–20 | High | 1 | Multiple bounded contexts, want service-like autonomy without distributed overhead |
| `DOM-001` | Hexagonal | 3–10 | High | 1+ | Multiple entry points (API, queue, CLI) to the same domain |
| `DST-001` | Microservices | 15+ | High | Many | Teams need independent deployment; you've outgrown a monolith |

### How to read the matrix

1. **Start with team size and domain complexity.** These two factors narrow the field to 2–3 options.
2. **Check the "key signal."** This is the deciding factor when multiple patterns seem viable.
3. **When in doubt, pick the simpler pattern.** You can always graduate: `LAY-001` → `LAY-002` → `LAY-003`, or `FTR-001` → `FTR-002` → `DST-001`. Going the other direction (simplifying) is harder.

### Common combinations

Patterns aren't always exclusive. Some compose well:

- **FTR-002 + LAY-003:** Modular monolith where each module internally uses Clean Architecture
- **FTR-002 + FTR-001:** Modular monolith where each module internally uses Vertical Slices
- **DST-001 + any:** Each microservice picks its own internal architecture based on that service's complexity
- **DOM-001 + FTR-002:** Domain-centric module boundaries with hexagonal architecture inside each module

### Progression paths

```
LAY-001 ──→ LAY-002 ──→ LAY-003
  │                        │
  │                        ▼
  └──→ FTR-001 ──→ FTR-002 ──→ DST-001
                      │
                      ▼
                   DOM-001 (per module/service)
```

## Each Pattern Covers

1. **When to use / When NOT to use** — with concrete thresholds
2. **Solution structure** — exact folder/project layout
3. **Dependency rules** — what references what, with diagrams
4. **Naming conventions** — files, classes, interfaces, methods
5. **Key abstractions** — core interfaces with C# code
6. **Data flow** — request lifecycle from HTTP to database
7. **Where business logic lives** — the single most important rule
8. **Testing strategy** — what to test, how, and project layout
9. **Common mistakes** — what developers get wrong and how to fix it
