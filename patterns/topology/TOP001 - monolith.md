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

The monolith's internal structure is defined by a [structural pattern](../structural/). Use the structural decision matrix to pick one based on team size and domain complexity.

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

1. **Calling it a "legacy" pattern.** A monolith is a legitimate topology, not something you've failed to grow out of. Plenty of systems that moved to distributed architectures would have been better served by a well-structured single deployable.

2. **No internal boundaries.** A monolith without module or layer boundaries becomes a big ball of mud. Use a structural pattern to maintain internal organisation. [STR005](../structural/STR005%20-%20modular-monolith.md) (Modular Monolith) is the strongest option for systems that might eventually need to extract services.

3. **Premature decomposition.** Splitting into services before you understand the domain boundaries. Get the boundaries right inside a monolith first — extracting a well-defined module into a service is straightforward. Merging two poorly-bounded services back together is painful.
