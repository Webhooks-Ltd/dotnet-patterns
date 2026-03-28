# Topology Patterns

System-level patterns that define how deployable units are organised, how they communicate, and where boundaries sit. Each service within a topology picks its own [structural pattern](../structural/) for internal code organisation.

## Patterns

| Ref | Pattern | One-liner |
|:----------|:-------------------------------|:------|
| [`TOP001`](TOP001%20-%20monolith.md) | Monolith | Single deployable — the default topology for most projects |
| [`TOP002`](TOP002%20-%20service-oriented-architecture.md) | Service-Oriented Architecture | Few coarse-grained services, shared contracts, enterprise-scale integration |
| [`TOP003`](TOP003%20-%20microservices.md) | Microservices | Many fine-grained independently deployable services, own databases, async messaging |
