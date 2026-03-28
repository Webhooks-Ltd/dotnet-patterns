# Event Sourcing (with CQRS)

> **Ref:** `DSG002` | **Category:** Design

Store state as an append-only sequence of domain events instead of current-state snapshots. The event stream is the source of truth. Read models are projections rebuilt from events. Almost always combined with CQRS ([DSG001](DSG001%20-%20cqrs.md)) because querying an event stream directly is impractical.

## When to Use

- **Audit trail is a business requirement** — finance, healthcare, legal, compliance. Not "we might want audit someday" but "regulators require a complete, tamper-evident history of every state change."
- **Temporal queries** — "what was the order status at 3pm yesterday?", "what was the portfolio valuation on March 1st?". If the business regularly asks "what was the state at time T?", event sourcing answers this natively.
- **Complex domains where new read models emerge over time** — you can replay events to build projections that didn't exist when the events were recorded. No schema migration, no backfill script.
- **Event-driven architectures** — systems where events are already the primary communication mechanism between bounded contexts. If you're publishing events anyway, storing them as the source of truth is a natural step.
- **Undo/replay** — systems that need to reverse operations, replay scenarios, or perform what-if analysis by forking an event stream.

## When NOT to Use

**Most systems should NOT use Event Sourcing.** It is the most over-adopted pattern in DDD. Be honest about whether you need it.

- **CRUD where current state is all that matters.** If nobody ever asks "how did this record get here?", you don't need event sourcing. A normal database row is simpler.
- **When a simple audit table or soft deletes solve the audit requirement.** A `Changes` table with `EntityId`, `Field`, `OldValue`, `NewValue`, `ChangedBy`, `ChangedAt` covers 90% of audit needs at 10% of the complexity. Try this first.
- **When Change Data Capture (CDC) solves the requirement.** SQL Server CDC, PostgreSQL logical replication, or Debezium can capture changes without modifying application code. Consider this before adopting event sourcing.
- **Teams unfamiliar with eventual consistency.** Event sourcing forces eventual consistency between the write model (event stream) and all read models (projections). If the team can't invest in understanding this, the system will have subtle bugs.
- **Simple domains.** A CRUD entity with 3 fields and no business rules doesn't benefit from events. The overhead of aggregate rehydration, event serialisation, and projection management is unjustifiable.
- **As a replacement for a message bus.** The event store persists domain history. It is not a transport mechanism for inter-service communication. Use a message broker for that. You may publish events to a broker *from* the event store, but they serve different purposes.

**Start with [DSG001](DSG001%20-%20cqrs.md) Level 1.** If Level 1 CQRS solves your problem, stop there. If you need Level 2 projections, try them with a current-state write DB first. Only add event sourcing when you have concrete evidence that the event stream itself — not just event-driven projections — provides business value.

## Core Concepts

### Events as the Source of Truth

In a traditional system, the database stores current state. In event sourcing, the database stores an ordered sequence of events. Current state is derived by replaying events.

```
Traditional:  Order row → { Id: 1, Status: "Shipped", Total: 59.98 }

Event Sourced:  Stream "Order-1" →
  1. OrderCreated { OrderId: 1, CustomerId: 42 }
  2. ItemAdded { ProductId: 7, Quantity: 2, UnitPrice: 29.99 }
  3. OrderSubmitted { Total: 59.98 }
  4. PaymentReceived { TransactionId: "tx-abc" }
  5. OrderShipped { TrackingNumber: "1Z999" }
```

There is no `Orders` table with an `UPDATE` statement. The event stream *is* the order. To know the current status, replay events 1–5.

### Event Stream Per Aggregate

Each aggregate instance has its own event stream, identified by a stream ID (typically `{AggregateType}-{AggregateId}`). Streams are independent — loading `Order-1` never reads events from `Order-2`.

### Aggregate Rehydration

To load an aggregate, read its event stream from the store and apply each event in order:

```csharp
var events = await eventStore.ReadStreamAsync($"Order-{orderId}");
var order = new Order();
foreach (var @event in events)
{
    order.Apply(@event);
}
```

### Snapshots

For aggregates with many events, replaying from the beginning becomes slow. A snapshot captures the aggregate's state at a point in time. Rehydration loads the snapshot, then replays only events after it:

```
Snapshot at event 500: { Status: "Active", Balance: 1234.56, ... }
Replay events 501–523 on top of snapshot
```

Marten handles snapshots automatically. With EventStoreDB, you manage them manually or use projections.

## Event Store Options in .NET

### Marten (PostgreSQL)

Marten uses PostgreSQL as the event store. Events are stored in a `mt_events` table as JSONB. It provides aggregate rehydration, inline and async projections, and snapshot support out of the box.

```csharp
// Program.cs
builder.Services.AddMarten(options =>
{
    options.Connection(builder.Configuration.GetConnectionString("Default"));
    options.Events.StreamIdentity = StreamIdentity.AsString;
})
.UseLightweightSessions()
.AddAsyncDaemon(DaemonMode.HotCold);
```

Writing events:

```csharp
public sealed class PlaceOrderHandler(IDocumentSession session)
{
    public async Task HandleAsync(PlaceOrder command, CancellationToken ct)
    {
        var order = new Order();
        var events = order.Place(command.CustomerId, command.Items);

        session.Events.StartStream<Order>($"Order-{order.Id}", events);
        await session.SaveChangesAsync(ct);
    }
}
```

Loading an aggregate:

```csharp
public sealed class CancelOrderHandler(IDocumentSession session)
{
    public async Task HandleAsync(CancelOrder command, CancellationToken ct)
    {
        var order = await session.Events
            .AggregateStreamAsync<Order>($"Order-{command.OrderId}", token: ct)
            ?? throw new NotFoundException(nameof(Order), command.OrderId);

        var events = order.Cancel(command.Reason);

        session.Events.Append($"Order-{command.OrderId}", events);
        await session.SaveChangesAsync(ct);
    }
}
```

### EventStoreDB

EventStoreDB is a purpose-built event store. It's more operationally complex than Marten (separate infrastructure) but offers built-in projections, subscriptions, and is optimised for event-heavy workloads.

```csharp
// Writing events
var eventData = events.Select(e => new EventData(
    Uuid.NewUuid(),
    e.GetType().Name,
    JsonSerializer.SerializeToUtf8Bytes(e)
)).ToArray();

await client.AppendToStreamAsync(
    $"Order-{orderId}",
    StreamState.Any,
    eventData,
    cancellationToken: ct);

// Reading and rehydrating
var result = client.ReadStreamAsync(
    Direction.Forwards,
    $"Order-{orderId}",
    StreamPosition.Start,
    cancellationToken: ct);

var order = new Order();
await foreach (var resolved in result)
{
    var @event = Deserialise(resolved);
    order.Apply(@event);
}
```

**Choose Marten** if you already use PostgreSQL and want a simpler operational model. **Choose EventStoreDB** if event sourcing is central to your system and you want a purpose-built store with built-in subscriptions.

## Aggregate Design

An event-sourced aggregate has two types of methods:

1. **Command methods** — validate business rules and return new events (but don't mutate state directly)
2. **Apply methods** — mutate state from events (no validation, no side effects)

```csharp
public sealed class Order
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; } = Money.Zero;
    private readonly List<OrderItem> _items = [];
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    public IReadOnlyList<object> Place(Guid customerId, IReadOnlyList<OrderLineItem> items)
    {
        if (items.Count == 0)
            throw new DomainException("Order must have at least one item.");

        var events = new List<object>();

        events.Add(new OrderCreated(Guid.NewGuid(), customerId));

        foreach (var item in items)
        {
            events.Add(new ItemAdded(item.ProductId, item.Quantity, item.UnitPrice));
        }

        var total = items.Sum(i => i.Quantity * i.UnitPrice);
        events.Add(new OrderSubmitted(total));

        foreach (var e in events) Apply(e);
        return events;
    }

    public IReadOnlyList<object> Cancel(string reason)
    {
        if (Status == OrderStatus.Shipped)
            throw new DomainException("Cannot cancel a shipped order.");

        var events = new List<object> { new OrderCancelled(reason) };
        foreach (var e in events) Apply(e);
        return events;
    }

    public void Apply(object @event)
    {
        switch (@event)
        {
            case OrderCreated e:
                Id = e.OrderId;
                CustomerId = e.CustomerId;
                Status = OrderStatus.Draft;
                break;
            case ItemAdded e:
                _items.Add(new OrderItem(e.ProductId, e.Quantity, e.UnitPrice));
                break;
            case OrderSubmitted e:
                Total = Money.From(e.Total);
                Status = OrderStatus.Submitted;
                break;
            case OrderCancelled:
                Status = OrderStatus.Cancelled;
                break;
            case PaymentReceived:
                Status = OrderStatus.Paid;
                break;
            case OrderShipped:
                Status = OrderStatus.Shipped;
                break;
        }
    }
}
```

**Key rules:**
- Command methods validate, then return events. They also apply those events to keep the aggregate's in-memory state consistent for subsequent commands in the same operation.
- `Apply` methods are pure state mutations. No validation, no exceptions, no I/O. They must handle every event the aggregate has ever produced, including historical event versions.
- Events are **past tense** facts: `OrderCreated`, `ItemAdded`, `OrderShipped`. They record what happened, not what should happen.

## Event Definitions

Events are immutable records. They carry only the data needed to describe what happened:

```csharp
public sealed record OrderCreated(Guid OrderId, Guid CustomerId);
public sealed record ItemAdded(Guid ProductId, int Quantity, decimal UnitPrice);
public sealed record OrderSubmitted(decimal Total);
public sealed record OrderCancelled(string Reason);
public sealed record PaymentReceived(Guid TransactionId);
public sealed record OrderShipped(string TrackingNumber);
```

**Event versioning:** Events are a permanent contract. Once persisted, their schema cannot change. When you need to evolve an event:

1. **Add fields** — add new optional fields with defaults. Old events deserialise with the default. This is the safest approach.
2. **Upcast** — register a transformation that converts old event shapes to new ones during deserialisation. Marten supports this via `IEventUpcaster`.
3. **New event type** — create `ItemAddedV2` for genuinely different semantics. The `Apply` method handles both versions.

Never rename or remove fields from a persisted event type.

## Projections (Read Models)

Event sourcing requires projections to answer queries efficiently. Without them, every query would replay the full event stream. This is where CQRS ([DSG001](DSG001%20-%20cqrs.md)) becomes essential.

### Inline Projections (Synchronous)

Updated within the same transaction as the event append. Consistent but slower writes.

```csharp
// Marten inline projection
public sealed class OrderSummaryProjection : SingleStreamProjection<OrderSummary>
{
    public void Apply(OrderCreated @event, OrderSummary view)
    {
        view.Id = @event.OrderId;
        view.CustomerId = @event.CustomerId;
        view.Status = "Draft";
        view.CreatedAt = DateTimeOffset.UtcNow;
    }

    public void Apply(OrderSubmitted @event, OrderSummary view)
    {
        view.Total = @event.Total;
        view.Status = "Submitted";
    }

    public void Apply(OrderShipped @event, OrderSummary view)
    {
        view.Status = "Shipped";
        view.TrackingNumber = @event.TrackingNumber;
    }
}

public sealed class OrderSummary
{
    public Guid Id { get; set; }
    public Guid CustomerId { get; set; }
    public string Status { get; set; } = "";
    public decimal Total { get; set; }
    public string? TrackingNumber { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
}
```

Register in Marten:

```csharp
options.Projections.Add<OrderSummaryProjection>(ProjectionLifecycle.Inline);
```

### Async Projections

Updated in the background by a daemon. Eventually consistent but don't slow down writes. Use for read models that can tolerate lag.

```csharp
options.Projections.Add<OrderSummaryProjection>(ProjectionLifecycle.Async);
```

Marten's async daemon (`AddAsyncDaemon(DaemonMode.HotCold)`) processes these automatically.

### Cross-Stream Projections

Aggregate data across multiple streams — e.g., "all orders for a customer":

```csharp
public sealed class CustomerOrderHistoryProjection : MultiStreamProjection<CustomerOrderHistory, Guid>
{
    public CustomerOrderHistoryProjection()
    {
        Identity<OrderCreated>(e => e.CustomerId);
        Identity<OrderSubmitted>(e => e.CustomerId);
    }

    public void Apply(OrderCreated @event, CustomerOrderHistory view)
    {
        view.CustomerId = @event.CustomerId;
        view.OrderIds.Add(@event.OrderId);
        view.TotalOrders++;
    }

    public void Apply(OrderSubmitted @event, CustomerOrderHistory view)
    {
        view.TotalSpent += @event.Total;
    }
}
```

### Rebuilding Projections

One of event sourcing's superpowers: you can rebuild any projection from scratch by replaying all events. This enables:

- Adding new read models without data migration
- Fixing bugs in projection logic and rebuilding with corrected code
- Changing the read store technology entirely

With Marten:

```csharp
await using var daemon = await store.BuildProjectionDaemonAsync();
await daemon.RebuildProjectionAsync<OrderSummaryProjection>(CancellationToken.None);
```

**Design every projection to be rebuildable.** If rebuilding a projection would take days because it depends on external API calls or side effects, the projection is doing too much.

## Handling Out-of-Order Events

Within a single aggregate stream, events are always ordered — the event store guarantees this. Out-of-order processing is a concern for **cross-stream projections** and **inter-service event consumption** where events from different streams or services arrive in unpredictable order.

### Strategies

**1. Idempotent projections with version tracking.** Store the last processed event sequence number per stream. Skip events with a sequence ≤ the stored version:

```csharp
public void Apply(OrderSubmitted @event, CustomerOrderHistory view, IEvent metadata)
{
    if (view.ProcessedVersions.ContainsKey(metadata.StreamKey!))
        return;

    view.TotalSpent += @event.Total;
    view.ProcessedVersions[metadata.StreamKey!] = metadata.Version;
}
```

**2. Eventually consistent read models.** Accept that the projection may be temporarily inconsistent. Design the read model so that applying events in any order converges to the correct state. This works when events are commutative (e.g., incrementing a counter — order doesn't matter).

**3. Buffer and reorder.** Hold events in a buffer and process them in sequence number order. This adds latency but guarantees ordering. Most useful for inter-service consumers:

```csharp
public sealed class OrderedEventBuffer
{
    private readonly SortedDictionary<long, object> _buffer = new();
    private long _nextExpected = 0;

    public IEnumerable<object> Add(long position, object @event)
    {
        _buffer[position] = @event;

        while (_buffer.TryGetValue(_nextExpected, out var next))
        {
            _buffer.Remove(_nextExpected);
            _nextExpected++;
            yield return next;
        }
    }
}
```

**4. Compensation.** Process events as they arrive, then correct if a later event reveals the ordering was wrong. This is the most complex approach but works when you can't afford to wait.

**Within a single stream, none of this matters.** Marten and EventStoreDB both guarantee per-stream ordering. The aggregate's `Apply` method always receives events in order.

## Data Flow

**Command (write) flow:**

```
POST /api/orders
    │
    ▼
PlaceOrderHandler receives command
    │
    ▼
Load aggregate: read event stream from store, replay via Apply()
    │  (or load from snapshot + replay remaining events)
    ▼
Call command method: order.Place(customerId, items)
    │  validates business rules
    │  returns new events: [OrderCreated, ItemAdded, OrderSubmitted]
    ▼
Append new events to stream in event store
    │  (optimistic concurrency via expected stream version)
    ▼
Inline projections update synchronously
Async projections update in background
    │
    ▼
Return result (e.g., new order ID)
```

**Query (read) flow:**

```
GET /api/orders/{id}
    │
    ▼
Query handler reads from projection (OrderSummary)
    │  NO event replay — reads a pre-built view
    ▼
Return OrderSummary DTO → 200 OK
```

The read path never touches the event store. It reads from projections — denormalised views optimised for the query. This is why CQRS ([DSG001](DSG001%20-%20cqrs.md)) is nearly always combined with event sourcing.

## Where Business Logic Lives

**In the aggregate.** Specifically:

- **Command methods** (`Place`, `Cancel`, `Ship`) validate business rules and decide which events to raise. This is where invariants are enforced.
- **Apply methods** mutate state. No business logic — they are mechanical state transitions.
- **The event store is infrastructure.** It persists events but contains no logic.
- **Projections are read-only transformations.** They do not enforce rules. If you find business logic in a projection, move it to the aggregate.

## Testing Strategy

Event sourcing enables a powerful **Given/When/Then** testing style:

```csharp
[Fact]
public void Place_ValidItems_RaisesCreatedAndSubmitted()
{
    // Given
    var order = new Order();

    // When
    var events = order.Place(
        customerId: Guid.NewGuid(),
        items: [new OrderLineItem(Guid.NewGuid(), 2, 29.99m)]);

    // Then
    events.Should().HaveCount(3);
    events[0].Should().BeOfType<OrderCreated>();
    events[1].Should().BeOfType<ItemAdded>();
    events[2].Should().BeOfType<OrderSubmitted>()
        .Which.Total.Should().Be(59.98m);
}

[Fact]
public void Cancel_ShippedOrder_ThrowsDomainException()
{
    // Given: an order that has been shipped
    var order = new Order();
    order.Apply(new OrderCreated(Guid.NewGuid(), Guid.NewGuid()));
    order.Apply(new OrderSubmitted(59.98m));
    order.Apply(new PaymentReceived(Guid.NewGuid()));
    order.Apply(new OrderShipped("1Z999"));

    // When/Then
    var act = () => order.Cancel("Changed my mind");
    act.Should().Throw<DomainException>()
        .WithMessage("Cannot cancel a shipped order.");
}
```

**Why this is powerful:**
- Tests are pure — no database, no mocks, no DI
- Given is expressed as past events (the actual storage format)
- When is the command method
- Then is the resulting events or exceptions
- Tests read like business specifications

**Projection tests:** Seed events, run the projection, assert the read model state. Use Marten's test helpers or build projections manually in tests.

**Integration tests:** Use Testcontainers with PostgreSQL (for Marten) or EventStoreDB's Docker image. Test the full write path: send command → verify events appended → verify projections updated.

## Common Mistakes

1. **Using Event Sourcing when an audit table would suffice.** A `ChangeLog` table with entity ID, field name, old value, new value, timestamp, and user covers most audit requirements. It's simpler to implement, simpler to query, and doesn't force eventual consistency on your entire read path. Try this first.

2. **Not designing for projection rebuilds.** Projections will need rebuilding — bugs, new requirements, schema changes. If a projection rebuild takes hours because it calls external APIs or triggers side effects, it's doing too much. Projections must be pure transformations of events.

3. **Treating the event store as a message bus.** The event store persists domain history. Inter-service communication uses a message broker (RabbitMQ, Azure Service Bus). You may publish events from the store to a broker, but they are separate infrastructure with different guarantees.

4. **Over-granular events.** `OrderFieldUpdated { Field: "Status", Value: "Shipped" }` is a disguised CRUD operation, not a domain event. Events should capture business intent: `OrderShipped { TrackingNumber: "1Z999" }`. If your events look like database change logs, you're not doing event sourcing.

5. **Breaking event schemas.** Renaming a field, removing a property, or changing a type in a persisted event breaks rehydration for every aggregate that has that event in its history. Events are immutable contracts. Add fields (with defaults), upcast old versions, or create new event types.

6. **Forgetting snapshots.** An aggregate with 10,000 events takes noticeable time to rehydrate. Implement snapshots from the start for aggregates that will accumulate many events. With Marten, configure `options.Events.UseIdentityMapForInlineAggregates = true` and snapshot intervals.

7. **Putting business logic in projections.** A projection that calculates discounts based on order history. Projections are read-only transformations. If the calculation matters for business decisions, it belongs in the aggregate's command method.

8. **No concurrency control.** Two commands load the same aggregate simultaneously, both at version 5, both append events. Without optimistic concurrency (expected version checks), one will silently overwrite the other. Both Marten and EventStoreDB support expected version — use it.

9. **Event Sourcing without CQRS.** Querying the event store directly for read operations means replaying events on every query. This is prohibitively slow for any non-trivial system. Always combine with CQRS ([DSG001](DSG001%20-%20cqrs.md)) — use projections for reads.

10. **Skipping [DSG001](DSG001%20-%20cqrs.md) Level 1.** Teams jump straight to Event Sourcing when Level 1 CQRS (same database, separate read/write code paths) would solve their problem at a fraction of the complexity. Prove you need event history as a first-class concept before adopting this pattern.
