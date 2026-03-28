# Monolith

> **Ref:** `TOP001` | **Category:** Topology

Single deployable unit containing all application code. Every structural pattern (STR001–STR010) runs inside a monolith by default — this is the starting topology for almost every project.

## When to Use

- **Any team size** starting a new project — this is the default, not a choice to justify
- One deployment pipeline, one database, one process
- The system doesn't yet have evidence that independent deployment of subsystems is needed
- You're building an MVP, a line-of-business app, an internal tool, or anything where operational simplicity matters more than independent scaling

Every application starts as a monolith. The question is not "should we use a monolith?" but "when should we stop being one?"

## When NOT to Use

- Multiple teams need to deploy independently on different cadences — consider Service-Oriented Architecture ([TOP002](TOP002%20-%20service-oriented-architecture.md)) or Microservices ([TOP003](TOP003%20-%20microservices.md))
- Distinct subsystems have genuinely different scaling requirements (e.g., one component needs 50 instances while the rest need 2)
- Regulatory or security requirements demand process isolation between subsystems

## What Goes Inside

The monolith's internal structure is defined by a structural pattern:

| Complexity | Structural pattern |
|:-----------|:-------------------|
| Low (CRUD) | [STR001](../structural/STR001%20-%20n-tier.md) N-Tier or [STR009](../structural/STR009%20-%20minimal-api.md) Minimal API |
| Medium | [STR002](../structural/STR002%20-%20clean-architecture-lite.md) Clean Lite or [STR004](../structural/STR004%20-%20vertical-slice.md) Vertical Slice |
| High | [STR003](../structural/STR003%20-%20full-clean-architecture.md) Full Clean, [STR008](../structural/STR008%20-%20clean-vertical-slice.md) Clean + Features, or [STR006](../structural/STR006%20-%20hexagonal.md) Hexagonal |
| Multiple bounded contexts | [STR005](../structural/STR005%20-%20modular-monolith.md) Modular Monolith |

A modular monolith ([STR005](../structural/STR005%20-%20modular-monolith.md)) is still a monolith — one deployable. The "modular" part is about internal code boundaries, not deployment topology.

## Deployment

- Single CI/CD pipeline
- Single container / App Service / VM
- Single database (or multiple databases, but owned by one deployable)
- Scales by running multiple instances behind a load balancer

## How It Differs from Other Topologies

| | Monolith (TOP001) | Service-Oriented (TOP002) | Microservices (TOP003) |
|:---|:---|:---|:---|
| Deployable units | 1 | Few (3–10) | Many (10+) |
| Database | Shared | Shared or per-service | Always per-service |
| Communication | In-process method calls | Synchronous (SOAP/REST) + ESB | Async messaging + lightweight HTTP/gRPC |
| Team model | One team or feature teams | Teams per service area | Team per service |
| Operational overhead | Low | Medium | High |

## Common Mistakes

1. **Calling it a "legacy" pattern.** Monoliths are not legacy. A well-structured monolith is the correct architecture for most applications. The industry's bias toward distributed systems has produced countless over-engineered microservices that would be better served by a single deployable.

2. **No internal boundaries.** A monolith without module or layer boundaries becomes a big ball of mud. Use a structural pattern to maintain internal organisation. [STR005](../structural/STR005%20-%20modular-monolith.md) (Modular Monolith) is the strongest option for systems that might eventually need to extract services.

3. **Premature decomposition.** Splitting into services before you understand the domain boundaries. Get the boundaries right inside a monolith first — extracting a well-defined module into a service is straightforward. Merging two poorly-bounded services back together is painful.
