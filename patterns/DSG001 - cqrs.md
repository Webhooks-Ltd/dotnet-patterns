# CQRS (Command Query Responsibility Segregation)

> **Ref:** `DSG001` | **Category:** Design

Separate the read (query) and write (command) sides of the application into distinct models, each optimised for its purpose.

## When to Use

- Read and write workloads have **different shapes** — writes operate on rich domain aggregates, reads return flat, denormalised views
- Read and write workloads have **different scaling requirements** — reads outnumber writes 10:1 or more
- Query performance suffers because you're projecting complex aggregate graphs into flat DTOs on every request
- You need **different storage** for reads vs writes — e.g., normalised SQL for writes, denormalised SQL views / Elasticsearch / Redis for reads
- The domain model is rich enough that forcing queries through it adds complexity without benefit

CQRS is a **design pattern**, not a solution structure. It layers on top of any structural pattern: [STR003](STR003%20-%20full-clean-architecture.md), [STR004](STR004%20-%20vertical-slice.md), [STR008](STR008%20-%20clean-vertical-slice.md), etc.

## When NOT to Use

- Simple CRUD where the read and write models are essentially the same shape — CQRS adds two models where one would do
- You don't have performance issues on the read side — don't add a read model for reads that are already fast
- The team is unfamiliar with eventual consistency and you can't invest in learning it
- You're using CQRS "because it's modern" — if you can't articulate which specific problem it solves in your system, you don't need it

## The Two Levels

CQRS exists on a spectrum. Pick the level that matches your problem:

### Level 1: Same Database, Separate Models

Commands go through the domain model. Queries bypass it entirely and read directly from the database using optimised queries or views.

```mermaid
graph LR
    subgraph Write Side
        Cmd[Command] --> Handler[Command Handler] --> Domain[Domain Model] --> DB[(Database)]
    end
    subgraph Read Side
        Qry[Query] --> QHandler[Query Handler] --> DB
    end
```

- Write side: Command → Handler → Domain entities → Repository → DB
- Read side: Query → Handler → raw SQL / Dapper / EF projections → DTO
- **Same database**, different code paths
- No eventual consistency — reads see writes immediately
- **Start here.** This solves most CQRS use cases.

### Level 2: Separate Read Store

Commands write to the primary database. Domain events trigger projections that update a dedicated read store (denormalised tables, materialised views, or a completely different database).

```mermaid
graph LR
    subgraph Write Side
        Cmd[Command] --> Handler[Command Handler] --> Domain[Domain Model] --> WDB[(Write DB)]
    end
    WDB --> Events[Domain Events] --> Projections
    subgraph Read Side
        Projections --> RDB[(Read Store)]
        Qry[Query] --> QHandler[Query Handler] --> RDB
    end
```

- Write side: Command → Handler → Domain → Write DB → publish domain event
- Projection: Event handler updates Read Store (denormalised tables, Elasticsearch, Redis, etc.)
- Read side: Query → Handler → Read Store → DTO
- **Eventual consistency** — reads may lag behind writes
- Only use this when Level 1 read performance is genuinely insufficient

## Applying CQRS to Structural Patterns

### Within Clean Architecture ([STR003](STR003%20-%20full-clean-architecture.md) / [STR008](STR008%20-%20clean-vertical-slice.md))

The Application project already separates Commands and Queries:

```
MyApp.Application/
├── Orders/
│   ├── Commands/
│   │   ├── CreateOrder.cs          ← goes through domain model
│   │   └── CancelOrder.cs
│   └── Queries/
│       ├── GetOrderById.cs         ← bypasses domain, reads directly
│       └── ListOrders.cs
```

Command handler — uses domain model:

```csharp
public sealed class CreateOrder(
    string Street, string City, string PostCode,
    List<CreateOrder.LineItem> Items) : ICommand<Guid>
{
    internal sealed class Handler(
        IOrderRepository orders,
        IProductRepository products) : ICommandHandler<CreateOrder, Guid>
    {
        public async Task<Guid> HandleAsync(CreateOrder command, CancellationToken ct)
        {
            var address = new Address(command.Street, command.City, command.PostCode);
            var order = new Order(address);

            foreach (var item in command.Items)
            {
                var product = await products.GetByIdAsync(item.ProductId)
                    ?? throw new NotFoundException(nameof(Product), item.ProductId);
                order.AddItem(product, item.Quantity);
            }

            order.Submit();
            await orders.AddAsync(order);
            await orders.SaveChangesAsync(ct);

            return order.Id;
        }
    }
}
```

Query handler — bypasses domain, queries directly:

```csharp
public sealed record GetOrderById(Guid OrderId) : IQuery<OrderDto?>
{
    internal sealed class Handler(
        IReadDbConnection db) : IQueryHandler<GetOrderById, OrderDto?>
    {
        public async Task<OrderDto?> HandleAsync(GetOrderById query, CancellationToken ct)
        {
            return await db.QuerySingleOrDefaultAsync<OrderDto>(
                """
                SELECT o.Id, o.Status, o.Total,
                       o.Street, o.City, o.PostCode
                FROM Orders o
                WHERE o.Id = @OrderId
                """,
                new { query.OrderId });
        }
    }
}
```

The query handler doesn't load `Order` entities, doesn't go through `IOrderRepository`, and doesn't reconstruct the aggregate. It runs a flat query and returns a DTO directly. This is the core CQRS benefit — reads are fast and simple.

### Within Vertical Slice ([STR004](STR004%20-%20vertical-slice.md))

Each feature already owns its full stack. CQRS is natural — command slices use the domain, query slices use raw queries:

```
Features/
├── Orders/
│   ├── CreateOrder.cs        ← handler uses domain entities
│   ├── CancelOrder.cs        ← handler uses domain entities
│   ├── GetOrderById.cs       ← handler uses raw SQL/Dapper
│   └── ListOrders.cs         ← handler uses raw SQL/Dapper
```

No structural change needed — just a convention about how query handlers access data.

## Key Abstractions

Separate command and query interfaces:

```csharp
public interface ICommand<TResult> { }
public interface IQuery<TResult> { }

public interface ICommandHandler<in TCommand, TResult> where TCommand : ICommand<TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken ct = default);
}

public interface IQueryHandler<in TQuery, TResult> where TQuery : IQuery<TResult>
{
    Task<TResult> HandleAsync(TQuery query, CancellationToken ct = default);
}
```

Read-side data access interface (for Level 1):

```csharp
public interface IReadDbConnection
{
    Task<T?> QuerySingleOrDefaultAsync<T>(string sql, object? param = null);
    Task<IReadOnlyList<T>> QueryAsync<T>(string sql, object? param = null);
}
```

Implement with Dapper, raw ADO.NET, or EF Core's `FromSqlRaw`. The point is that the read side is not forced through the same ORM configuration as the write side.

For Level 2, add event-driven projections:

```csharp
public interface IProjection<in TEvent>
{
    Task ProjectAsync(TEvent @event, CancellationToken ct = default);
}

public sealed class OrderSummaryProjection(ReadDbContext readDb)
    : IProjection<OrderPlacedEvent>
{
    public async Task ProjectAsync(OrderPlacedEvent @event, CancellationToken ct)
    {
        readDb.OrderSummaries.Add(new OrderSummary
        {
            OrderId = @event.OrderId,
            CustomerName = @event.CustomerName,
            Total = @event.Total,
            Status = "Submitted",
            PlacedAt = @event.OccurredAt
        });
        await readDb.SaveChangesAsync(ct);
    }
}
```

## Data Flow

**Level 1 — Command:**

```
POST /api/orders
    │
    ▼
Controller maps request → CreateOrder command
    │
    ▼
CreateOrder.Handler
    │  loads aggregates via IOrderRepository
    │  calls domain methods (order.AddItem, order.Submit)
    │  persists via repository → EF Core → Database
    ▼
Guid returned → 201 Created
```

**Level 1 — Query:**

```
GET /api/orders/{id}
    │
    ▼
Controller maps request → GetOrderById query
    │
    ▼
GetOrderById.Handler
    │  runs optimised SQL via IReadDbConnection
    │  returns flat DTO directly — no domain model involved
    ▼
OrderDto returned → 200 OK
```

**Level 2 — Write with projection:**

```
POST /api/orders
    │
    ▼
CreateOrder.Handler
    │  domain model → write DB
    │  domain event raised: OrderPlacedEvent
    ▼
OrderPlacedEvent dispatched
    │
    ▼
OrderSummaryProjection
    │  updates denormalised read store
    ▼
Read store eventually consistent
```

## Where Business Logic Lives

**Write side: in the domain model.** Commands flow through domain entities that enforce invariants. No change from your chosen structural pattern.

**Read side: there is no business logic.** Queries are data retrieval — they transform stored data into DTOs. If you find business rules in a query handler, either it belongs on the write side or you're computing something that should be pre-computed by a projection.

## Testing Strategy

**Command handler tests** — same as your structural pattern. Mock the repository, verify domain methods are called, verify persistence.

**Query handler tests** — integration tests against a real database. Seed data, run the query, assert the DTO shape. These are fast because they're just SQL.

**Projection tests (Level 2)** — given an event, verify the read store is updated correctly. These are integration tests against the read store.

```csharp
[Fact]
public async Task GetOrderById_ReturnsFlat_Dto_Without_Loading_Aggregate()
{
    // Seed directly into DB — no domain model needed for read tests
    await SeedOrder(orderId: _testId, status: "Submitted", total: 59.98m);

    var result = await _handler.HandleAsync(new GetOrderById(_testId), CancellationToken.None);

    result.Should().NotBeNull();
    result!.Total.Should().Be(59.98m);
    result.Status.Should().Be("Submitted");
}
```

## Common Mistakes

1. **Using CQRS for everything.** Not every entity needs separate read/write models. Apply CQRS to the parts of the system where read and write shapes genuinely differ. A simple lookup table doesn't need CQRS.

2. **Query handlers that load domain entities and then map them.** `GetOrderById` loads `Order` via `IOrderRepository`, then maps to `OrderDto`. This defeats the purpose — you're paying the cost of aggregate reconstruction for a read. Query directly from the database.

3. **Jumping straight to Level 2.** Separate read stores, event-driven projections, eventual consistency — all for an app that gets 100 requests/minute. Start with Level 1 (same database, separate code paths). Only move to Level 2 when you have evidence that Level 1 reads are too slow.

4. **No strategy for read model staleness.** In Level 2, reads lag behind writes. If the UI shows stale data and users complain, you need a strategy: optimistic updates in the UI, polling, or websocket notifications. Don't ignore this.

5. **Command handlers returning complex objects.** A command changes state — it should return an ID or a simple confirmation, not a full DTO. If the caller needs data after a command, issue a separate query. This keeps the write path lean.

6. **Read side with write-side validation.** A query handler that throws `NotFoundException` when an entity doesn't exist. Queries return data or null — they don't enforce business rules. Return `null` and let the API layer decide whether that's a 404.

7. **Shared DTOs between commands and queries.** `OrderDto` used as both a command result and a query response. Commands and queries evolve independently. Keep their types separate.

8. **Forgetting to index the read side.** The whole point of separating reads is performance. If your query handler runs flat SQL but the table has no indexes for those queries, you've gained nothing. Design indexes for your read access patterns.
