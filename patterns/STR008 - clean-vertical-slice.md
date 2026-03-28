# Clean Vertical Slice

> **Ref:** `STR008` | **Category:** Structural

Single-project architecture combining Clean Architecture's dependency inversion and domain protection with Vertical Slice's feature-per-file organisation — layered boundaries enforced by convention, features organised by use case.

## When to Use

- **2–6 developers** building a feature-rich application with real business rules
- You want feature-based organisation ([STR004](STR004%20-%20vertical-slice.md)) but your domain model has invariants worth protecting ([STR002](STR002%20-%20clean-architecture-lite.md))
- Features are mostly independent, but shared domain logic exists and needs a home
- You're tired of the layered ceremony (five files across five folders to add one endpoint) but still need clear boundaries between domain, application, and infrastructure
- API-heavy applications with 15–60+ endpoints where pure vertical slices would accumulate duplicated domain logic

This pattern gives you the developer experience of vertical slices (one place to look for each feature) with the architectural safety net of Clean Architecture (domain logic is isolated and testable).

## When NOT to Use

- Pure CRUD with no domain logic — use [STR001](STR001%20-%20n-tier.md), the domain layer would be empty ceremony
- The domain is so complex it needs compiler-enforced boundaries across separate projects — use [STR003](STR003%20-%20full-clean-architecture.md)
- Features share almost no domain logic — pure [STR004](STR004%20-%20vertical-slice.md) is simpler and you won't benefit from the shared domain layer
- Large teams (8+) where convention-based boundaries won't hold — use multi-project [STR003](STR003%20-%20full-clean-architecture.md) or [STR005](STR005%20-%20modular-monolith.md)

## Solution Structure

```
MyApp/
├── MyApp.sln
└── src/
    └── MyApp/
        ├── MyApp.csproj
        ├── Program.cs
        ├── appsettings.json
        │
        ├── Domain/
        │   ├── Entities/
        │   │   ├── Order.cs
        │   │   ├── OrderItem.cs
        │   │   └── Product.cs
        │   ├── ValueObjects/
        │   │   ├── Money.cs
        │   │   └── Address.cs
        │   ├── Enums/
        │   │   └── OrderStatus.cs
        │   ├── Exceptions/
        │   │   └── InsufficientStockException.cs
        │   └── Interfaces/
        │       ├── IOrderRepository.cs
        │       └── IProductRepository.cs
        │
        ├── Features/
        │   ├── Orders/
        │   │   ├── CreateOrder.cs
        │   │   ├── GetOrderById.cs
        │   │   ├── ListOrders.cs
        │   │   ├── CancelOrder.cs
        │   │   └── UpdateOrderStatus.cs
        │   │
        │   └── Products/
        │       ├── CreateProduct.cs
        │       ├── GetProductById.cs
        │       └── ListProducts.cs
        │
        ├── Infrastructure/
        │   ├── Data/
        │   │   ├── AppDbContext.cs
        │   │   └── Configurations/
        │   │       ├── OrderConfiguration.cs
        │   │       └── ProductConfiguration.cs
        │   ├── Repositories/
        │   │   ├── OrderRepository.cs
        │   │   └── ProductRepository.cs
        │   └── DependencyInjection.cs
        │
        └── Shared/
            ├── Behaviours/
            │   ├── ValidationBehaviour.cs
            │   └── LoggingBehaviour.cs
            └── Middleware/
                └── ExceptionHandlingMiddleware.cs
```

**Domain/** — Entities with behaviour, value objects, enums, domain exceptions, and repository interfaces. This is the Clean Architecture core — it defines business rules and has zero dependencies on anything outside itself. No `using MyApp.Infrastructure;`, no `using MyApp.Features;`.

**Features/** — One file per use case, organised by resource. Each file is a self-contained vertical slice containing its request/response types, validator, handler, and endpoint mapping. Features depend on Domain/ (for entities and repository interfaces) and are wired to Infrastructure/ through DI.

**Infrastructure/** — DbContext, repository implementations, external service clients. Implements interfaces defined in Domain/. This is the only place that knows about EF Core, HTTP clients, or any external dependency.

**Shared/** — MediatR pipeline behaviours and middleware. Infrastructure plumbing only — if business logic appears here, move it to Domain/ or the relevant feature.

## Dependency Rules

```
Features/  ──→  Domain/
    │
    │ (via DI)
    ▼
Infrastructure/  ──→  Domain/

Features/ ✗──→ Infrastructure/    ← NEVER directly
Domain/   ✗──→ anything           ← NEVER
```

- **Domain/** references nothing. It contains pure C# with no NuGet packages, no infrastructure types, no framework dependencies.
- **Features/** reference **Domain/** only. Each handler depends on repository interfaces from Domain/, injected via DI. Features never reference Infrastructure/ directly — they don't know if they're backed by SQL Server, PostgreSQL, or a file system.
- **Infrastructure/** references **Domain/** to implement its interfaces.
- **Features never reference other features.** `CreateOrder` does not call `GetProductById`. If a feature needs product data, it injects `IProductRepository` and queries directly.
- **Shared/** is referenced by the MediatR pipeline — features flow through it automatically.

Enforce with architecture tests:

```csharp
[Fact]
public void Domain_ShouldNotReference_Features_Or_Infrastructure()
{
    var domainTypes = typeof(Order).Assembly.GetTypes()
        .Where(t => t.Namespace?.StartsWith("MyApp.Domain") == true);

    foreach (var type in domainTypes)
    {
        var referencedNamespaces = type.GetFields(BindingFlags.NonPublic | BindingFlags.Instance)
            .Select(f => f.FieldType.Namespace);
        referencedNamespaces.Should().NotContain(ns =>
            ns != null && (ns.StartsWith("MyApp.Features") || ns.StartsWith("MyApp.Infrastructure")));
    }
}
```

## Naming Conventions

| Element | Convention | Location | Example |
|---------|-----------|----------|---------|
| Entity | singular noun, behaviour-rich | Domain/Entities | `Order` |
| Value Object | singular noun, immutable | Domain/ValueObjects | `Money` |
| Repository interface | `I{Entity}Repository` | Domain/Interfaces | `IOrderRepository` |
| Repository implementation | `{Entity}Repository` | Infrastructure/Repositories | `OrderRepository` |
| Feature folder | plural resource noun | Features/ | `Orders/`, `Products/` |
| Feature file | `{Verb}{Entity}` | Features/{Resource}/ | `CreateOrder.cs` |
| Request type | nested `Command` or `Query` | inside feature class | `CreateOrder.Command` |
| Response type | nested `Result` | inside feature class | `CreateOrder.Result` |
| Validator | nested `Validator` | inside feature class | `CreateOrder.Validator` |
| Handler | nested `Handler` | inside feature class | `CreateOrder.Handler` |
| Domain exception | `{Noun}Exception` | Domain/Exceptions | `InsufficientStockException` |

## Key Abstractions

Domain entity with behaviour (same as [STR002](STR002%20-%20clean-architecture-lite.md)):

```csharp
public class Order
{
    private readonly List<OrderItem> _items = [];

    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public Address ShippingAddress { get; private set; }
    public Money Total => _items.Aggregate(Money.Zero, (sum, i) => sum + i.LineTotal);
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    public Order(Address shippingAddress)
    {
        Id = Guid.NewGuid();
        Status = OrderStatus.Draft;
        ShippingAddress = shippingAddress;
    }

    public void AddItem(Product product, int quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify a submitted order.");
        if (!product.HasSufficientStock(quantity))
            throw new InsufficientStockException(product.Id, quantity);

        _items.Add(new OrderItem(product, quantity));
    }

    public void Submit()
    {
        if (_items.Count == 0)
            throw new DomainException("Cannot submit an empty order.");
        Status = OrderStatus.Submitted;
    }
}
```

Feature slice (same shape as [STR004](STR004%20-%20vertical-slice.md), but handler calls domain entity methods instead of doing logic inline):

```csharp
// Features/Orders/CreateOrder.cs
public static class CreateOrder
{
    public sealed record Command(
        string Street, string City, string PostCode,
        List<OrderLineItem> Items) : IRequest<Result>;

    public sealed record OrderLineItem(Guid ProductId, int Quantity);
    public sealed record Result(Guid OrderId, decimal Total);

    public sealed class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(x => x.Items).NotEmpty();
            RuleFor(x => x.Street).NotEmpty();
            RuleFor(x => x.City).NotEmpty();
        }
    }

    public sealed class Handler(
        IOrderRepository orders,
        IProductRepository products) : IRequestHandler<Command, Result>
    {
        public async Task<Result> Handle(Command request, CancellationToken ct)
        {
            var address = new Address(request.Street, request.City, request.PostCode);
            var order = new Order(address);

            foreach (var item in request.Items)
            {
                var product = await products.GetByIdAsync(item.ProductId)
                    ?? throw new NotFoundException(nameof(Product), item.ProductId);
                order.AddItem(product, item.Quantity);
            }

            order.Submit();
            await orders.AddAsync(order);
            await orders.SaveChangesAsync(ct);

            return new Result(order.Id, order.Total.Amount);
        }
    }

    public static void MapEndpoints(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/orders", async (Command command, IMediator mediator) =>
        {
            var result = await mediator.Send(command);
            return Results.Created($"/api/orders/{result.OrderId}", result);
        });
    }
}
```

The critical difference from pure [STR004](STR004%20-%20vertical-slice.md): the handler doesn't check stock or validate order state — it calls `order.AddItem()` and `order.Submit()`, which enforce those rules internally. Business logic lives in the entity, not the handler.

DI wiring in `Program.cs`:

```csharp
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<Program>());
builder.Services.AddValidatorsFromAssemblyContaining<Program>();
builder.Services.AddInfrastructure(builder.Configuration);

var app = builder.Build();

CreateOrder.MapEndpoints(app);
GetOrderById.MapEndpoints(app);
ListOrders.MapEndpoints(app);
// ... or use assembly scanning
```

## Data Flow

A `POST /api/orders` request:

```
HTTP POST /api/orders
    │
    ▼
Minimal API endpoint (defined in CreateOrder.MapEndpoints)
    │  deserialises body → CreateOrder.Command
    ▼
MediatR.Send(Command)
    │
    ▼
ValidationBehaviour (Shared/)
    │  runs CreateOrder.Validator
    ▼
CreateOrder.Handler.Handle()
    │  calls IProductRepository (Domain interface → Infrastructure impl)
    │  creates Order entity, calls order.AddItem() — DOMAIN validates stock
    │  calls order.Submit() — DOMAIN validates non-empty
    │  calls IOrderRepository.AddAsync() + SaveChangesAsync()
    ▼
OrderRepository (Infrastructure) → AppDbContext → Database INSERT
    │
    ▼
CreateOrder.Result returned → HTTP 201 Created
```

Compare with the two parent patterns:
- **vs [STR002](STR002%20-%20clean-architecture-lite.md):** Same domain model, but use cases are self-contained files in Features/ instead of separate command/handler/validator files in Application/.
- **vs [STR004](STR004%20-%20vertical-slice.md):** Same file-per-feature structure, but business logic is in domain entities instead of inline in handlers.

## Where Business Logic Lives

**In Domain/ entities and value objects — orchestrated by feature handlers.**

This is the hybrid rule:

- **Domain entities** enforce invariants. `Order.AddItem()` checks stock. `Order.Submit()` validates state. An entity is never in an invalid state. This comes from Clean Architecture.
- **Feature handlers** orchestrate: load entities via repository interfaces, call entity methods, save. This comes from Vertical Slice — the handler is the use case, and it's self-contained in one file.
- **If a handler contains `if` statements about business rules**, move them into the entity. The handler should read as "load → tell entity to do something → save."
- **If domain logic is specific to one feature and will never be shared**, it's acceptable to keep it in the handler. But default to the entity.

## Testing Strategy

```
MyApp/
├── src/
│   └── MyApp/
└── tests/
    ├── MyApp.UnitTests/
    │   ├── MyApp.UnitTests.csproj
    │   ├── Domain/
    │   │   ├── OrderTests.cs
    │   │   └── MoneyTests.cs
    │   └── Features/
    │       ├── CreateOrderTests.cs
    │       └── CancelOrderTests.cs
    └── MyApp.IntegrationTests/
        ├── MyApp.IntegrationTests.csproj
        ├── CustomWebApplicationFactory.cs
        └── Features/
            ├── Orders/
            │   ├── CreateOrderTests.cs
            │   └── ListOrdersTests.cs
            └── Products/
                └── ListProductsTests.cs
```

**Domain unit tests** — the highest-value tests. Pure C#, no mocks, no DI. Test entity behaviour directly:

```csharp
[Fact]
public void AddItem_InsufficientStock_ThrowsInsufficientStockException()
{
    var product = new Product("Widget", stockQuantity: 2, price: 9.99m);
    var order = new Order(new Address("123 Main St", "London", "SW1A 1AA"));

    Assert.Throws<InsufficientStockException>(
        () => order.AddItem(product, quantity: 5));
}
```

**Feature handler unit tests** — test orchestration with mocked repositories. Verify the handler calls entity methods correctly and handles not-found/error cases.

**Integration tests** — test features end-to-end via HTTP using `WebApplicationFactory` + Testcontainers. Mirror the feature folder structure in tests.

## Common Mistakes

1. **Business logic in handlers instead of entities.** The handler checks `if (product.StockQuantity < quantity)` instead of calling `order.AddItem(product, quantity)`. If the handler is making business decisions, move them to the entity. The handler orchestrates; the entity decides.

2. **Features referencing Infrastructure directly.** A handler injects `AppDbContext` instead of `IOrderRepository`. This couples the feature to EF Core. Always depend on the repository interface from Domain/.

3. **Empty Domain layer.** If your entities have no methods and the handlers do all the work, you've built [STR004](STR004%20-%20vertical-slice.md) with an empty Domain folder. Either add real entity behaviour or drop the domain layer and use pure vertical slices.

4. **Features calling other features.** `CreateOrder.Handler` sends `GetProductById.Query` via MediatR to fetch a product. Don't do this — inject `IProductRepository` and query directly. Features are independent slices.

5. **Shared abstractions that create coupling.** `IFeatureHandler<TRequest, TResult>`, `FeatureBase<T>` — these shared base classes undermine the independence of slices. Use MediatR's `IRequest`/`IRequestHandler` directly.

6. **Domain/ referencing Features/ or Infrastructure/.** If an entity has `using MyApp.Features;` or `using MyApp.Infrastructure;`, the dependency direction is wrong. Domain/ is the core — it references nothing.

7. **Over-engineering simple CRUD features.** If `GetProductById` is just "fetch from DB, map to DTO, return" — that's fine. Not every feature needs rich domain interaction. Let simple features be simple; reserve domain entities for features with real business rules.

8. **No architecture tests.** Without separate projects, nothing prevents a developer from adding `using MyApp.Infrastructure;` in a domain entity. Add NetArchTest or ArchUnitNET tests to CI to enforce the dependency rules.
