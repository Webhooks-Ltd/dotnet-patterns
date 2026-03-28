# Hexagonal Architecture (Ports & Adapters)

> **Ref:** `STR006` | **Category:** Structural

Domain at the centre, with ports (interfaces) defining how the domain interacts with the outside world and adapters providing concrete implementations.

## When to Use

- The **domain model is the most valuable part** of the system — complex business rules, rich invariants, and evolving logic that represents a competitive advantage
- Multiple entry points: the same domain logic is accessed via REST API, gRPC, message queue consumers, CLI tools, or scheduled jobs
- You need to swap or add infrastructure without touching business logic — changing from SQL Server to PostgreSQL, adding a Redis cache, switching payment providers
- The system will live for years and the domain must be insulated from infrastructure churn
- **3–10+ developers** with a dedicated domain expert or a team that invests in understanding the business deeply

## When NOT to Use

- CRUD applications with thin business logic — you'll create ports and adapters for operations that are just pass-through to the database
- You don't actually have multiple entry points or any plan to swap infrastructure — the abstraction cost isn't justified
- Small, short-lived applications where the overhead of defining ports and adapters exceeds the value of the separation
- If you'd use the same pattern as Full Clean Architecture ([STR003](STR003%20-%20full-clean-architecture.md)) — hexagonal and clean architecture overlap significantly. The difference is emphasis: hexagonal focuses on the domain's relationship with the outside world through explicit ports; clean architecture focuses on dependency direction between layers. Choose one mental model, not both.

## Solution Structure

```
MyApp/
├── MyApp.sln
│
├── src/
│   ├── MyApp.Domain/                          ← THE HEXAGON
│   │   ├── MyApp.Domain.csproj                 (zero external references)
│   │   ├── Model/
│   │   │   ├── Order.cs
│   │   │   ├── OrderItem.cs
│   │   │   ├── Product.cs
│   │   │   └── Money.cs
│   │   ├── Ports/
│   │   │   ├── Driving/
│   │   │   │   ├── IPlaceOrderUseCase.cs
│   │   │   │   ├── ICancelOrderUseCase.cs
│   │   │   │   └── IGetOrderQuery.cs
│   │   │   └── Driven/
│   │   │       ├── IOrderRepository.cs
│   │   │       ├── IProductRepository.cs
│   │   │       ├── IPaymentGateway.cs
│   │   │       └── IEventPublisher.cs
│   │   ├── Services/
│   │   │   ├── PlaceOrderService.cs
│   │   │   ├── CancelOrderService.cs
│   │   │   └── GetOrderQueryService.cs
│   │   └── Exceptions/
│   │       ├── DomainException.cs
│   │       └── InsufficientStockException.cs
│   │
│   ├── MyApp.Adapters.Web/                    ← DRIVING ADAPTER
│   │   ├── MyApp.Adapters.Web.csproj           (references Domain)
│   │   ├── Controllers/
│   │   │   └── OrdersController.cs
│   │   └── DTOs/
│   │       ├── CreateOrderRequest.cs
│   │       └── OrderResponse.cs
│   │
│   ├── MyApp.Adapters.Messaging/              ← DRIVING ADAPTER
│   │   ├── MyApp.Adapters.Messaging.csproj     (references Domain)
│   │   └── Consumers/
│   │       └── PlaceOrderMessageConsumer.cs
│   │
│   ├── MyApp.Adapters.Persistence/            ← DRIVEN ADAPTER
│   │   ├── MyApp.Adapters.Persistence.csproj   (references Domain)
│   │   ├── AppDbContext.cs
│   │   ├── Configurations/
│   │   │   ├── OrderConfiguration.cs
│   │   │   └── ProductConfiguration.cs
│   │   └── Repositories/
│   │       ├── OrderRepository.cs
│   │       └── ProductRepository.cs
│   │
│   ├── MyApp.Adapters.Payment/               ← DRIVEN ADAPTER
│   │   ├── MyApp.Adapters.Payment.csproj       (references Domain)
│   │   └── StripePaymentGateway.cs
│   │
│   └── MyApp.Host/
│       ├── MyApp.Host.csproj                   (references all adapters)
│       └── Program.cs
│
└── tests/
    ├── MyApp.Domain.Tests/
    ├── MyApp.Adapters.Persistence.Tests/
    └── MyApp.Host.Tests/
```

**MyApp.Domain** — the hexagon. Contains the domain model, business rules, **driving ports** (what the outside world can ask the domain to do), **driven ports** (what the domain needs from the outside world), and domain services that implement driving ports. Zero NuGet packages. Note: placing use-case orchestration services inside the Domain project follows Alistair Cockburn's original hexagonal definition. Many .NET teams instead add a separate `MyApp.Application` project for orchestration (converging with [STR003](STR003%20-%20full-clean-architecture.md)) — that's a valid adaptation, but this document describes the pure hexagonal approach where all logic inside the hexagon is one project.

**Driving adapters** (Web, Messaging) — translate external input into calls on driving ports. A REST controller receives an HTTP request and calls `IPlaceOrderUseCase`. A message consumer receives a queue message and calls the same port. The domain doesn't know which adapter triggered it.

**Driven adapters** (Persistence, Payment) — implement driven ports. The domain calls `IOrderRepository.SaveAsync(order)` and the persistence adapter translates that into EF Core operations. The domain doesn't know it's using a database.

**MyApp.Host** — wires adapters to ports via DI. Contains no logic.

## Dependency Rules

```
         DRIVING ADAPTERS              DRIVEN ADAPTERS
    ┌──────────┐  ┌──────────┐    ┌──────────────┐  ┌─────────────┐
    │   Web    │  │Messaging │    │ Persistence  │  │  Payment    │
    │ Adapter  │  │ Adapter  │    │   Adapter    │  │  Adapter    │
    └────┬─────┘  └────┬─────┘    └──────┬───────┘  └──────┬──────┘
         │             │                 │                  │
         │ calls       │ calls           │ implements       │ implements
         ▼             ▼                 ▼                  ▼
    ┌─────────────────────────────────────────────────────────────┐
    │                      MyApp.Domain                           │
    │                                                             │
    │  Driving Ports ◄─── Services ───► Driven Ports              │
    │  (interfaces)        (impl)       (interfaces)              │
    └─────────────────────────────────────────────────────────────┘
```

**The fundamental rule:** All arrows point inward. The domain references nothing. Everything else references the domain.

- **Driving adapters** reference the Domain to call driving ports
- **Driven adapters** reference the Domain to implement driven ports
- **Adapters never reference other adapters**
- The domain defines **both** types of ports as interfaces:
  - **Driving ports** = what the domain offers (use cases). Domain services implement these.
  - **Driven ports** = what the domain needs (infrastructure). Adapters implement these.

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Driving port | `I{Verb}{Entity}UseCase` or `I{Verb}{Entity}Query` | `IPlaceOrderUseCase`, `IGetOrderQuery` |
| Driven port | `I{Entity}Repository`, `I{Noun}Gateway`, `I{Noun}Publisher` | `IOrderRepository`, `IPaymentGateway` |
| Domain service | `{Verb}{Entity}Service` | `PlaceOrderService` |
| Domain entity | singular noun | `Order`, `Product` |
| Value object | singular noun | `Money`, `Address` |
| Adapter project | `MyApp.Adapters.{Concern}` | `MyApp.Adapters.Persistence` |
| Adapter implementation | `{Technology}{Port}` | `SqlOrderRepository`, `StripePaymentGateway` |
| Driving adapter handler | `{Entity}sController`, `{Action}MessageConsumer` | `OrdersController`, `PlaceOrderMessageConsumer` |

## Key Abstractions

Driving port (what the world can ask the domain):

```csharp
// Domain/Ports/Driving/IPlaceOrderUseCase.cs
public interface IPlaceOrderUseCase
{
    Task<OrderResult> ExecuteAsync(PlaceOrderCommand command);
}

public sealed record PlaceOrderCommand(
    Guid CustomerId,
    Address ShippingAddress,
    IReadOnlyList<OrderLineItem> Items);

public sealed record OrderLineItem(Guid ProductId, int Quantity);

public sealed record OrderResult(Guid OrderId, Money Total);
```

Driven port (what the domain needs from the outside):

```csharp
// Domain/Ports/Driven/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id);
    Task SaveAsync(Order order);
}

// Domain/Ports/Driven/IPaymentGateway.cs
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(Guid customerId, Money amount);
}
```

Domain service implementing a driving port:

```csharp
// Domain/Services/PlaceOrderService.cs
public sealed class PlaceOrderService(
    IOrderRepository orders,
    IProductRepository products,
    IPaymentGateway payments) : IPlaceOrderUseCase
{
    public async Task<OrderResult> ExecuteAsync(PlaceOrderCommand command)
    {
        var order = new Order(command.CustomerId, command.ShippingAddress);

        foreach (var item in command.Items)
        {
            var product = await products.GetByIdAsync(item.ProductId)
                ?? throw new DomainException($"Product {item.ProductId} not found");
            order.AddItem(product, item.Quantity);
        }

        var payment = await payments.ChargeAsync(command.CustomerId, order.Total);
        order.ConfirmPayment(payment.TransactionId);

        await orders.SaveAsync(order);

        return new OrderResult(order.Id, order.Total);
    }
}
```

Driving adapter calling the port:

```csharp
// Adapters.Web/Controllers/OrdersController.cs
[ApiController]
[Route("api/orders")]
public sealed class OrdersController(IPlaceOrderUseCase placeOrder) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest request)
    {
        var command = request.ToCommand();  // map API DTO → domain command
        var result = await placeOrder.ExecuteAsync(command);
        return CreatedAtAction(nameof(GetById), new { id = result.OrderId }, result);
    }
}
```

DI wiring in Host:

```csharp
// Host/Program.cs
builder.Services.AddScoped<IPlaceOrderUseCase, PlaceOrderService>();
builder.Services.AddScoped<ICancelOrderUseCase, CancelOrderService>();
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();
```

## Data Flow

**API request — placing an order:**

```
HTTP POST /api/orders
    │
    ▼
OrdersController (DRIVING ADAPTER)
    │  maps CreateOrderRequest → PlaceOrderCommand
    │  calls IPlaceOrderUseCase.ExecuteAsync()
    ▼
PlaceOrderService (DOMAIN SERVICE)
    │  creates Order entity
    │  calls order.AddItem() — domain logic validates stock
    │  calls IPaymentGateway.ChargeAsync() — DRIVEN PORT
    │  calls order.ConfirmPayment()
    │  calls IOrderRepository.SaveAsync() — DRIVEN PORT
    ▼
StripePaymentGateway (DRIVEN ADAPTER)
    │  translates domain Money → Stripe API call
    │  returns PaymentResult
    ▼
SqlOrderRepository (DRIVEN ADAPTER)
    │  translates Order entity → EF Core operations
    │  calls DbContext.SaveChangesAsync()
    ▼
Result flows back up: SqlOrderRepository → PlaceOrderService → OrdersController → HTTP 201
```

**Message queue entry point — same domain logic, different adapter:**

```
RabbitMQ message received
    │
    ▼
PlaceOrderMessageConsumer (DRIVING ADAPTER)
    │  deserialises message → PlaceOrderCommand
    │  calls IPlaceOrderUseCase.ExecuteAsync()
    ▼
PlaceOrderService (SAME DOMAIN SERVICE)
    │  exact same business logic executes
    │  domain doesn't know it was triggered by a message instead of HTTP
    ▼
...same driven adapter flow...
```

This is the power of hexagonal: the domain is entry-point agnostic.

## Where Business Logic Lives

**Inside the hexagon. Every rule, every calculation, every invariant.**

- **Domain entities** enforce their own invariants: `order.AddItem()` checks stock, `order.Cancel()` validates state transitions. An entity is never in an invalid state.
- **Domain services** implement driving ports and orchestrate multi-entity operations: `PlaceOrderService` coordinates Order creation, stock checking, and payment in one workflow.
- **Value objects** encapsulate value-level rules: `Money` prevents negative amounts, `Address` validates format.
- **Adapters are pure translation.** They convert between external formats (HTTP requests, database rows, message payloads) and domain types. If you see an `if` statement about a business rule in an adapter, move it into the domain.

The test: **can you describe an adapter as "receive input, translate to domain type, call port, translate result back"?** If the adapter is making decisions, those decisions belong in the domain.

## Testing Strategy

```
tests/
├── MyApp.Domain.Tests/
│   ├── MyApp.Domain.Tests.csproj           ← references Domain only
│   ├── Model/
│   │   ├── OrderTests.cs
│   │   ├── MoneyTests.cs
│   │   └── AddressTests.cs
│   └── Services/
│       ├── PlaceOrderServiceTests.cs
│       └── CancelOrderServiceTests.cs
│
├── MyApp.Adapters.Persistence.Tests/
│   ├── MyApp.Adapters.Persistence.Tests.csproj
│   └── Repositories/
│       └── OrderRepositoryTests.cs
│
└── MyApp.Host.Tests/
    ├── MyApp.Host.Tests.csproj
    ├── CustomWebApplicationFactory.cs
    └── Endpoints/
        └── OrdersEndpointTests.cs
```

**Domain tests** — the most important tests. Pure unit tests with driven ports mocked. Test that `PlaceOrderService` calls entity methods in the right order, validates correctly, and calls driven ports appropriately. No database, no HTTP, no external services.

```csharp
public class PlaceOrderServiceTests
{
    private readonly IOrderRepository _orders = Substitute.For<IOrderRepository>();
    private readonly IProductRepository _products = Substitute.For<IProductRepository>();
    private readonly IPaymentGateway _payments = Substitute.For<IPaymentGateway>();
    private readonly PlaceOrderService _sut;

    public PlaceOrderServiceTests()
    {
        _sut = new PlaceOrderService(_orders, _products, _payments);
    }

    [Fact]
    public async Task InsufficientStock_ThrowsDomainException()
    {
        _products.GetByIdAsync(Arg.Any<Guid>())
            .Returns(new Product("Widget", stockQuantity: 0, price: Money.From(10)));

        var command = new PlaceOrderCommand(
            Guid.NewGuid(),
            new Address("123 Main St", "London", "SW1A 1AA"),
            [new OrderLineItem(Guid.NewGuid(), Quantity: 5)]);

        await Assert.ThrowsAsync<InsufficientStockException>(
            () => _sut.ExecuteAsync(command));
    }
}
```

**Adapter tests** — integration tests. Test driven adapters against real infrastructure (Testcontainers for databases, WireMock for HTTP APIs). Verify that the adapter correctly translates between domain types and external formats.

**Host tests** — end-to-end API tests with `WebApplicationFactory`. Test the full wiring from HTTP to domain to database and back.

The hexagonal shape makes testing natural: swap real driven adapters for test doubles. The domain doesn't know the difference.

## Common Mistakes

1. **Domain referencing infrastructure types.** An entity has a `[Column]` attribute, a port returns `IQueryable<T>`, or a domain service logs with `ILogger<T>`. The domain must have zero infrastructure dependencies. If you need logging, define a domain port `IAuditLog` and implement it with an adapter.

2. **Business logic in adapters.** The controller calculates a discount. The repository applies a filter based on business rules. The message consumer validates state transitions. All of this belongs in the domain. Adapters translate; they don't decide.

3. **Confusing driving and driven ports.** A driving port is what the outside world asks the domain to do — the domain **provides** the implementation. A driven port is what the domain needs — the adapter **provides** the implementation. If you mix these up, the dependency arrows point the wrong way.

4. **Creating ports for everything.** `IDateTimePort`, `IGuidPort`, `IStringFormatterPort`. Only create ports where you need genuine substitutability. For `DateTime.UtcNow`, use `TimeProvider` (built into .NET 8). Not everything needs a port.

5. **Adapters that know about each other.** The web adapter directly uses the persistence adapter's `AppDbContext`. Adapters communicate only through the domain — they implement or call ports, never each other.

6. **One giant domain service.** `OrderService` with 30 methods. Each driving port should represent a single use case. `IPlaceOrderUseCase`, `ICancelOrderUseCase`, `IGetOrderQuery` — small, focused interfaces.

7. **Treating hexagonal and Clean Architecture as different patterns to combine.** They're different emphases on the same idea. Hexagonal says "domain at the centre with explicit ports." Clean Architecture says "dependencies point inward." Don't create a project structure that tries to be both hexagonal AND clean with separate layers and separate port/adapter projects. Pick the mental model that resonates with your team.

8. **Leaky port interfaces.** `IOrderRepository` has a method `Task<List<Order>> GetByCustomerIdWithItemsAndProductsIncluded(Guid customerId)`. Ports should express domain intent, not infrastructure mechanics. Use `GetByCustomerIdAsync(Guid customerId)` and let the adapter decide how to load the data.
