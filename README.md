# .NET Architecture Pattern Catalogue

A reference library of architecture patterns for .NET/C# projects. Each pattern is a self-contained descriptor that an AI coding agent (or developer) can follow to scaffold and maintain a project without further guidance.

## Categories

- **[Topology](patterns/topology/)** — How many deployable units and how they communicate.
- **[Network](patterns/network/)** — How traffic enters and flows between clients and services.
- **[Structural](patterns/structural/)** — How to organise code inside each deployable.
- **[Design](patterns/design/)** — Cross-cutting patterns that layer on top of structural choices.

## Quick Reference

| Ref | Pattern | One-liner |
|:----------|:-------------------------------|:------|
| [`TOP001`](patterns/topology/TOP001%20-%20monolith.md) | Monolith | Single deployable — the default topology for most projects |
| [`TOP002`](patterns/topology/TOP002%20-%20service-oriented-architecture.md) | Service-Oriented Architecture | Few coarse-grained services, shared contracts, enterprise-scale integration |
| [`TOP003`](patterns/topology/TOP003%20-%20microservices.md) | Microservices | Many fine-grained independently deployable services, own databases, async messaging |
| [`TOP004`](patterns/topology/TOP004%20-%20serverless.md) | Serverless | Functions triggered by events, platform-managed scaling, no long-running processes |
| [`NET001`](patterns/network/NET001%20-%20api-gateway.md) | API Gateway | Single entry point — routing, auth, rate limiting, request aggregation |
| [`NET002`](patterns/network/NET002%20-%20backend-for-frontend.md) | Backend for Frontend | Per-client-type API layer with tailored data shaping and orchestration |
| [`STR001`](patterns/structural/STR001%20-%20n-tier.md) | N-Tier | Multi-project, Controllers → Services → Data Access |
| [`STR002`](patterns/structural/STR002%20-%20clean-architecture-lite.md) | Clean Architecture Lite | Single project with domain/application/infrastructure boundaries |
| [`STR003`](patterns/structural/STR003%20-%20full-clean-architecture.md) | Full Clean Architecture | Multi-project, compiler-enforced dependency inversion |
| [`STR004`](patterns/structural/STR004%20-%20vertical-slice.md) | Vertical Slice | Feature-per-file, each slice owns its full stack |
| [`STR005`](patterns/structural/STR005%20-%20modular-monolith.md) | Modular Monolith | Independent modules within one deployable, own data, explicit contracts |
| [`STR006`](patterns/structural/STR006%20-%20hexagonal.md) | Hexagonal (Ports & Adapters) | Domain at centre, ports define all external interaction |
| [`STR008`](patterns/structural/STR008%20-%20clean-architecture-feature-folders.md) | Clean Architecture + Feature Folders | Multi-project Clean Architecture with CQRS, Application organised by feature |
| [`STR009`](patterns/structural/STR009%20-%20minimal-api.md) | Minimal API | Endpoint-focused, no controllers, explicit route registration |
| [`STR010`](patterns/structural/STR010%20-%20worker-service.md) | Worker Service | Background processing — queue consumers, scheduled jobs, long-running services |
| [`DSG001`](patterns/design/DSG001%20-%20cqrs.md) | CQRS | Separate read/write models — applies within any structural pattern |
| [`DSG002`](patterns/design/DSG002%20-%20event-sourcing.md) | Event Sourcing + CQRS | Events as source of truth, projections for reads — builds on DSG001 |

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
