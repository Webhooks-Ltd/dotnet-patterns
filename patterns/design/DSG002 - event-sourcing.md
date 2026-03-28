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

Most event store libraries support snapshots — either automatically or via explicit configuration.

## Event Store

The event store is the persistence layer. It must support:

- **Append-only writes** — events are immutable once stored
- **Stream-based reads** — load all events for a given aggregate by stream ID
- **Optimistic concurrency** — reject appends if the stream version has changed since the aggregate was loaded
- **Ordered delivery** — events within a stream are returned in append order

Your event store library handles serialisation, stream management, and concurrency. The handlers work through an abstraction:

```csharp
public sealed class PlaceOrderHandler(IEventStore eventStore)
{
    public async Task HandleAsync(PlaceOrder command, CancellationToken ct)
    {
        var order = new Order();
        var events = order.Place(command.CustomerId, command.Items);

        await eventStore.AppendAsync($"Order-{order.Id}", events, ct);
    }
}
```

Loading and modifying an aggregate:

```csharp
public sealed class CancelOrderHandler(IEventStore eventStore)
{
    public async Task HandleAsync(CancelOrder command, CancellationToken ct)
    {
        var streamId = $"Order-{command.OrderId}";

        var (order, version) = await eventStore.LoadAsync<Order>(streamId, ct)
            ?? throw new NotFoundException(nameof(Order), command.OrderId);

        var events = order.Cancel(command.Reason);

        await eventStore.AppendAsync(streamId, events, expectedVersion: version, ct);
    }
}
```

The `IEventStore` abstraction is provided by your chosen event store library. Two main options exist in .NET: PostgreSQL-based stores that use JSONB columns (simpler operations, leverages existing infrastructure) and purpose-built event databases (optimised for high-throughput event workloads, built-in subscriptions).

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
2. **Upcast** — register a transformation that converts old event shapes to new ones during deserialisation. Most event store libraries support upcaster registration.
3. **New event type** — create `ItemAddedV2` for genuinely different semantics. The `Apply` method handles both versions.

Never rename or remove fields from a persisted event type.

## Projections (Read Models)

Event sourcing requires projections to answer queries efficiently. Without them, every query would replay the full event stream. This is where CQRS ([DSG001](DSG001%20-%20cqrs.md)) becomes essential.

### Inline Projections (Synchronous)

Updated within the same transaction as the event append. Consistent but slower writes.

```csharp
// Inline projection
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

Register inline projections with your event store library so they run within the same transaction as the event append.

### Async Projections

Updated in the background by a projection daemon/worker. Eventually consistent but don't slow down writes. Use for read models that can tolerate lag. Most event store libraries support both inline and async projection modes via configuration.

### Cross-Stream Projections

Aggregate data across multiple streams — e.g., "all orders for a customer":

```csharp
public sealed class CustomerOrderHistoryProjection
{
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

Cross-stream projections require the event store to route events from multiple streams to the same projection. Your event store library handles this — you define which event types map to which projection key.

### Rebuilding Projections

One of event sourcing's superpowers: you can rebuild any projection from scratch by replaying all events. This enables:

- Adding new read models without data migration
- Fixing bugs in projection logic and rebuilding with corrected code
- Changing the read store technology entirely

Your event store library should provide a rebuild mechanism that replays the full event history through a projection. This is a batch operation — expect it to take time for large event stores.

**Design every projection to be rebuildable.** If rebuilding a projection would take days because it depends on external API calls or side effects, the projection is doing too much.

## Handling Out-of-Order and Duplicate Events

Within a single aggregate stream, events are always ordered — the event store guarantees this. The aggregate's `Apply` method always receives events in sequence. **This section is about projections and inter-service consumers**, where a message broker sits between the event store and the consumer.

### The Reality of Message Brokers

Most systems use a cloud message broker (Azure Service Bus, AWS SQS, Kafka, RabbitMQ) to deliver events to projections and downstream services. These brokers give you:

- **At-least-once delivery** — duplicates are guaranteed. The same event will be delivered more than once.
- **No global ordering** — events from different partitions/queues arrive in unpredictable order.
- **Concurrent consumers** — multiple instances processing from the same subscription.

These are facts of distributed messaging, not problems you can solve with clever SQL. Any projection strategy must account for all three.

### The Practical Default: Single Consumer, Sequential Processing

**One consumer per projection, processing events sequentially, with a checkpoint.**

```sql
-- PostgreSQL
CREATE TABLE projection_checkpoints (
    projection_name TEXT   PRIMARY KEY,
    last_position   BIGINT NOT NULL DEFAULT -1
);
```

The consumer reads events in order from the broker (or directly from the event store via a catch-up subscription), applies each one, and advances the checkpoint in the same transaction:

```sql
UPDATE projection_checkpoints
   SET last_position = @newPos
 WHERE projection_name = @name
   AND last_position < @newPos;
```

If zero rows are affected, the event was already processed (duplicate delivery) — skip it.

This approach is **correct by construction**: one consumer means no concurrency races, sequential processing means no ordering problems, and the checkpoint handles duplicates. It limits throughput to one event at a time per projection, but for most systems this is fast enough — projections are simple writes and a single consumer can process thousands of events per second.

### Scaling Beyond a Single Consumer

If a single consumer genuinely can't keep up, **partition by aggregate/stream ID**. Events for the same aggregate always go to the same consumer instance. This preserves per-aggregate ordering while allowing parallel processing across aggregates.

Most brokers support this natively: Kafka partitions by key, Azure Service Bus sessions, RabbitMQ consistent hash exchange. Configure the partition key to be the stream/aggregate ID.

Each consumer instance tracks its own checkpoint. Since no two instances process events for the same aggregate, there are no concurrency races on projection state.

### Idempotency Is Non-Negotiable

Regardless of strategy, every consumer must handle duplicate delivery. Two approaches:

**1. Idempotent operations.** Design writes so that applying the same event twice produces the same result. `SET status = 'Shipped'` is idempotent. `SET total = 59.98` is idempotent. `SET count = count + 1` is **not** idempotent — a duplicate delivery increments twice.

**2. Processed-event tracking.** Store the event ID in a deduplication table within the same transaction as the projection update:

```sql
-- PostgreSQL
INSERT INTO processed_events (event_id, projection_name)
VALUES (@eventId, @name)
ON CONFLICT DO NOTHING;
```

If the insert returns zero rows, the event was already processed — skip it. This works for any operation, including non-idempotent numeric updates, but adds a write per event.


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

**Projection tests:** Seed events, run the projection, assert the read model state. Use your event store library's test helpers or build projections manually in tests.

**Integration tests:** Use a test container library with your event store's database. Test the full write path: send command → verify events appended → verify projections updated.

## Common Mistakes

1. **Using Event Sourcing when an audit table would suffice.** A `ChangeLog` table with entity ID, field name, old value, new value, timestamp, and user covers most audit requirements. It's simpler to implement, simpler to query, and doesn't force eventual consistency on your entire read path. Try this first.

2. **Not designing for projection rebuilds.** Projections will need rebuilding — bugs, new requirements, schema changes. If a projection rebuild takes hours because it calls external APIs or triggers side effects, it's doing too much. Projections must be pure transformations of events.

3. **Treating the event store as a message bus.** The event store persists domain history. Inter-service communication uses a message broker (RabbitMQ, Azure Service Bus). You may publish events from the store to a broker, but they are separate infrastructure with different guarantees.

4. **Over-granular events.** `OrderFieldUpdated { Field: "Status", Value: "Shipped" }` is a disguised CRUD operation, not a domain event. Events should capture business intent: `OrderShipped { TrackingNumber: "1Z999" }`. If your events look like database change logs, you're not doing event sourcing.

5. **Breaking event schemas.** Renaming a field, removing a property, or changing a type in a persisted event breaks rehydration for every aggregate that has that event in its history. Events are immutable contracts. Add fields (with defaults), upcast old versions, or create new event types.

6. **Forgetting snapshots.** An aggregate with 10,000 events takes noticeable time to rehydrate. Implement snapshots from the start for aggregates that will accumulate many events. Configure snapshot intervals in your event store library.

7. **Putting business logic in projections.** A projection that calculates discounts based on order history. Projections are read-only transformations. If the calculation matters for business decisions, it belongs in the aggregate's command method.

8. **No concurrency control.** Two commands load the same aggregate simultaneously, both at version 5, both append events. Without optimistic concurrency (expected version checks), one will silently overwrite the other. Event store libraries support expected version — use it.

9. **Event Sourcing without CQRS.** Querying the event store directly for read operations means replaying events on every query. This is prohibitively slow for any non-trivial system. Always combine with CQRS ([DSG001](DSG001%20-%20cqrs.md)) — use projections for reads.

10. **Skipping [DSG001](DSG001%20-%20cqrs.md) Level 1.** Teams jump straight to Event Sourcing when Level 1 CQRS (same database, separate read/write code paths) would solve their problem at a fraction of the complexity. Prove you need event history as a first-class concept before adopting this pattern.

## Related Packages

- **Event store:** [Marten](https://github.com/JasperFx/marten) (PostgreSQL) · [EventStoreDB](https://github.com/EventStore/EventStore) · [Equinox](https://github.com/jet/equinox)
- **Projections:** [Marten](https://github.com/JasperFx/marten) async daemon · [EventStoreDB](https://github.com/EventStore/EventStore) projections
- **Messaging (for publishing events to other services):** [MassTransit](https://github.com/MassTransit/MassTransit) · [NServiceBus](https://github.com/Particular/NServiceBus) · [Wolverine](https://github.com/JasperFx/wolverine)
- **Serialisation:** [System.Text.Json](https://www.nuget.org/packages/System.Text.Json) · Newtonsoft.Json (for legacy upcasting)
- **Testing:** [xUnit](https://github.com/xunit/xunit), [NUnit](https://github.com/nunit/nunit) · [FluentAssertions](https://github.com/fluentassertions/fluentassertions) · [Testcontainers](https://github.com/testcontainers/testcontainers-dotnet)
