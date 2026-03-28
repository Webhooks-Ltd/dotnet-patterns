# Structural Patterns

How to organise a .NET solution at the project and folder level. Each pattern answers: "where do files go, what references what, and where does business logic live?"

## Patterns

| Ref | Pattern | One-liner |
|:----------|:-------------------------------|:------|
| [`STR001`](STR001%20-%20n-tier.md) | N-Tier | Multi-project, Controllers → Services → Data Access |
| [`STR002`](STR002%20-%20clean-architecture-lite.md) | Clean Architecture Lite | Single project with domain/application/infrastructure boundaries |
| [`STR003`](STR003%20-%20full-clean-architecture.md) | Full Clean Architecture | Multi-project, compiler-enforced dependency inversion |
| [`STR004`](STR004%20-%20vertical-slice.md) | Vertical Slice | Feature-per-file, each slice owns its full stack |
| [`STR005`](STR005%20-%20modular-monolith.md) | Modular Monolith | Independent modules within one deployable, own data, explicit contracts |
| [`STR006`](STR006%20-%20hexagonal.md) | Hexagonal (Ports & Adapters) | Domain at centre, ports define all external interaction |
| [`STR008`](STR008%20-%20clean-vertical-slice.md) | Clean Architecture + Feature Folders | Multi-project Clean Architecture with CQRS, Application organised by feature |
| [`STR009`](STR009%20-%20minimal-api.md) | Minimal API | Endpoint-focused, no controllers, explicit route registration |
| [`STR010`](STR010%20-%20worker-service.md) | Worker Service | Background processing — queue consumers, scheduled jobs, long-running services |

## Decision Matrix

| Ref | Pattern | Team | Domain complexity | Deploy targets | Key signal |
|:----------|:------|:-----|:------------------|:---------------|:-----------|
| [`STR001`](STR001%20-%20n-tier.md) | N-Tier | 1–4 | Low (CRUD) | 1 | "It's basically database operations with validation" |
| [`STR002`](STR002%20-%20clean-architecture-lite.md) | Clean Lite | 2–5 | Medium | 1 | Business rules exist but don't justify multiple projects |
| [`STR003`](STR003%20-%20full-clean-architecture.md) | Full Clean | 4+ | High | 1+ | Rich domain model worth protecting with compiler enforcement |
| [`STR004`](STR004%20-%20vertical-slice.md) | Vertical Slice | 2–8 | Low–Medium | 1 | Features are independent; tired of cross-layer changes |
| [`STR005`](STR005%20-%20modular-monolith.md) | Modular Monolith | 5–20 | High | 1 | Multiple bounded contexts, want service-like autonomy without distributed overhead |
| [`STR006`](STR006%20-%20hexagonal.md) | Hexagonal | 3–10 | High | 1+ | Multiple entry points (API, queue, CLI) to the same domain |
| [`STR008`](STR008%20-%20clean-vertical-slice.md) | Clean + Features | 3–8 | Medium–High | 1+ | Want STR003's rigour with feature-level co-location of commands, queries, and DTOs |
| [`STR009`](STR009%20-%20minimal-api.md) | Minimal API | 1–4 | Low–Medium | 1 | Lightweight API, no MVC overhead, endpoint-focused |
| [`STR010`](STR010%20-%20worker-service.md) | Worker Service | 1–4 | Any | 1 | Background processing — no HTTP pipeline |

### How to read the matrix

1. **Start with team size and domain complexity.** These two factors narrow the field to 2–3 options.
2. **Check the "key signal."** This is the deciding factor when multiple patterns seem viable.
3. **When in doubt, pick the simpler pattern.** You can always graduate to a more complex one later. Going the other direction is harder.

### Common combinations

Patterns compose well:

- **STR005 + STR003:** Modular monolith where each module internally uses Clean Architecture
- **STR005 + STR004:** Modular monolith where each module internally uses Vertical Slices
- **TOP003 + any:** Each microservice picks its own internal architecture based on that service's complexity
- **STR006 + STR005:** Domain-centric module boundaries with hexagonal architecture inside each module
