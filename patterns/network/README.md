# Network Patterns

Patterns for how traffic enters and flows through the system. These sit in front of your [topology](../topology/) — routing, aggregating, and shaping requests between external clients and internal services.

## Patterns

| Ref | Pattern | One-liner |
|:----------|:-------------------------------|:------|
| [`NET001`](NET001%20-%20api-gateway.md) | API Gateway | Single entry point — routing, auth, rate limiting, request aggregation |
| [`NET002`](NET002%20-%20backend-for-frontend.md) | Backend for Frontend | Per-client-type API layer with tailored data shaping and orchestration |
