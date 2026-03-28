# Clean Architecture Lite

> **Ref:** `STR002` | **Category:** Structural

Single-project Clean Architecture using folders, namespaces, and interfaces to enforce layer boundaries without the ceremony of multiple projects.

## When to Use

- **2–5 developers** working on a codebase with non-trivial business logic
- The domain has real rules beyond CRUD — but not enough complexity to justify four separate projects
- You've outgrown N-Tier ([STR001](STR001%20-%20n-tier.md)): service classes are getting large, business logic is leaking into controllers or repositories
- You want the discipline of Clean Architecture without the overhead of managing multiple .csproj files
- APIs with 10–50 endpoints and a domain model worth protecting

This is the sweet spot for most line-of-business applications that aren't pure CRUD but aren't enterprise-scale domain-driven systems either.

## When NOT to Use

- Pure CRUD with no business rules — use N-Tier ([STR001](STR001%20-%20n-tier.md)), it's simpler and you won't benefit from the extra structure
- Large teams or complex domains where you need **compile-time** enforcement of layer boundaries — use Full Clean Architecture ([STR003](STR003%20-%20full-clean-architecture.md))
- Multiple deployment targets (API + worker + CLI) sharing domain logic — multiple projects ([STR003](STR003%20-%20full-clean-architecture.md)) make this cleaner
- If your `Domain/` folder only contains entities with public getters/setters and no methods, you don't need this — you have an anaemic model dressed up as Clean Architecture

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
        ├── Application/
        │   ├── Orders/
        │   │   ├── CreateOrder.cs
        │   │   ├── GetOrderById.cs
        │   │   ├── ListOrders.cs
        │   │   └── UpdateOrderStatus.cs
        │   ├── Products/
        │   │   └── GetProductById.cs
        │   ├── Common/
        │   │   ├── IRequestHandler.cs
        │   │   └── IAppDbContext.cs
        │   └── Exceptions/
        │       ├── NotFoundException.cs
        │       └── ValidationException.cs
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
        └── Api/
            ├── Controllers/
            │   ├── OrdersController.cs
            │   └── ProductsController.cs
            ├── DTOs/
            │   ├── CreateOrderRequest.cs
            │   └── OrderResponse.cs
            ├── Middleware/
            │   └── ExceptionHandlingMiddleware.cs
            └── DependencyInjection.cs
```

**Domain/** — Entities with behaviour, value objects, domain exceptions, and repository interfaces. This is the core — it defines *what* the system does without knowing *how*.

**Application/** — Use cases grouped by feature. Each use case is a single class with a `Handle` method. Depends only on `Domain/`. Defines application-level interfaces (e.g., `IAppDbContext`) that Infrastructure implements.

**Infrastructure/** — EF Core DbContext, repository implementations, external service clients. Implements interfaces defined in `Domain/` and `Application/`.

**Api/** — Controllers, DTOs, middleware. The HTTP shell. Translates HTTP into application use case calls.

## Dependency Rules

```
Api/  →  Application/  →  Domain/
 │            │
 └──→ Infrastructure/ ──→ Domain/
```

- **Domain/** references nothing. No `using MyApp.Infrastructure;`, no `using MyApp.Api;`. Ever.
- **Application/** references only **Domain/**
- **Infrastructure/** references **Domain/** (to implement its interfaces) and **Application/** (to implement `IAppDbContext` etc.)
- **Api/** references **Application/** (to call use cases) and **Infrastructure/** (only in `Program.cs` for DI wiring)

**Namespace enforcement:** Since there are no project boundaries to prevent violations, use namespace discipline. If you see `using MyApp.Infrastructure;` inside a file under `Domain/` or `Application/`, that's a violation. Use an `.editorconfig` rule or an ArchUnit test to catch this:

```csharp
// Using NetArchTest.Rules
[Fact]
public void Domain_ShouldNotReference_Infrastructure()
{
    var result = Types.InNamespace("MyApp.Domain")
        .ShouldNot()
        .HaveDependencyOn("MyApp.Infrastructure")
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}
```

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Entity | singular noun, behaviour-rich | `Order` |
| Value Object | singular noun, immutable | `Money`, `Address` |
| Repository interface | `I{Entity}Repository` in Domain | `IOrderRepository` |
| Repository implementation | `{Entity}Repository` in Infrastructure | `OrderRepository` |
| Use case | `{Verb}{Entity}` | `CreateOrder`, `GetOrderById` |
| Use case request | nested `record` inside use case | `CreateOrder.Command` |
| Use case response | nested `record` inside use case | `GetOrderById.Response` |
| Request DTO | `{Verb}{Entity}Request` in Api | `CreateOrderRequest` |
| Response DTO | `{Entity}Response` in Api | `OrderResponse` |
| Domain exception | `{Noun}Exception` | `InsufficientStockException` |

Use case classes contain their own request/response types as nested records. This keeps each use case self-contained:

```csharp
// Application/Orders/CreateOrder.cs
public static class CreateOrder
{
    public sealed record Command(Guid ProductId, int Quantity, string ShippingAddress);
    public sealed record Result(Guid OrderId, decimal Total);

    public sealed class Handler(IOrderRepository orders, IProductRepository products)
    {
        public async Task<Result> HandleAsync(Command command)
        {
            // ...
        }
    }
}
```

## Key Abstractions

Repository interfaces live in **Domain/**:

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id);
    Task<IReadOnlyList<Order>> ListAsync();
    Task AddAsync(Order order);
    Task SaveChangesAsync();
}
```

Application-level abstractions live in **Application/Common/**:

```csharp
public interface IAppDbContext
{
    DbSet<Order> Orders { get; }
    DbSet<Product> Products { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

DI wiring is split per layer. Each layer has a `DependencyInjection.cs`:

```csharp
// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("Default")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IAppDbContext>(sp => sp.GetRequiredService<AppDbContext>());

        return services;
    }
}
```

## Data Flow

A `POST /api/orders` request:

```
HTTP Request
    │
    ▼
OrdersController.Create(CreateOrderRequest dto)
    │  maps API DTO → CreateOrder.Command
    ▼
CreateOrder.Handler.HandleAsync(Command)
    │  validates command
    │  calls IProductRepository.GetByIdAsync()
    │  creates Order entity (domain logic in entity)
    │  calls IOrderRepository.AddAsync(order)
    │  calls IOrderRepository.SaveChangesAsync()
    ▼
OrderRepository (Infrastructure) executes via AppDbContext
    │
    ▼
Database INSERT
    │
    ▼
Handler maps Order → CreateOrder.Result
    │
    ▼
Controller maps Result → OrderResponse (API DTO)
    │
    ▼
HTTP 201 Created
```

The key difference from N-Tier ([STR001](STR001%20-%20n-tier.md)): the `Order` entity itself contains business logic (e.g., `order.AddItem(product, quantity)` validates stock), rather than the service doing everything.

## Where Business Logic Lives

**In the Domain entities and value objects.** This is the fundamental shift from N-Tier.

- **Domain entities** enforce their own invariants. `Order.AddItem()` checks stock. `Order.Cancel()` validates the order is in a cancellable state. Entities are never in an invalid state.
- **Application use cases** orchestrate: load entities, call entity methods, persist changes. Use cases contain *application workflow* (fetch, act, save) but not *business rules*.
- **If you're writing `if` statements about business rules in a use case handler**, that logic belongs in the entity. The handler should call `order.AddItem(product, quantity)` and let the entity throw if the rules are violated.

The test: can you describe what a use case handler does as "load X, tell X to do Y, save X"? If yes, you've got the split right.

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
    │   └── Application/
    │       ├── CreateOrderTests.cs
    │       └── GetOrderByIdTests.cs
    └── MyApp.IntegrationTests/
        ├── MyApp.IntegrationTests.csproj
        ├── CustomWebApplicationFactory.cs
        └── Endpoints/
            ├── OrdersEndpointTests.cs
            └── ProductsEndpointTests.cs
```

**Domain unit tests** — test entity behaviour with plain C#. No mocks, no DI, no database. These are the most valuable tests in the system.

```csharp
[Fact]
public void AddItem_InsufficientStock_ThrowsInsufficientStockException()
{
    var product = new Product("Widget", stockQuantity: 2, price: 9.99m);
    var order = new Order("123 Main St");

    Assert.Throws<InsufficientStockException>(
        () => order.AddItem(product, quantity: 5));
}
```

**Application unit tests** — test use case handlers with mocked repositories. Verify orchestration logic.

**Integration tests** — test the full HTTP pipeline with `WebApplicationFactory<Program>` and Testcontainers.

## Common Mistakes

1. **Anaemic domain entities.** Entities with only properties and no methods are DTOs, not domain objects. If `Order` has public setters and no behaviour methods, you're doing N-Tier with extra folders. Either add real behaviour or drop back to [STR001](STR001%20-%20n-tier.md).

2. **Business logic in use case handlers.** The handler calculates discounts, validates stock, applies tax rules. All of this belongs in domain entities or domain services. Handlers orchestrate; entities decide.

3. **Infrastructure types in Domain/.** A repository interface that returns `IQueryable<T>` has leaked EF Core into the domain. Return `Task<IReadOnlyList<T>>` or `Task<T?>`. The domain should not know it's backed by a database.

4. **Skipping the interface for DbContext.** Without `IAppDbContext`, the Application layer references EF Core directly. Define the interface in Application, implement it in Infrastructure.

5. **API DTOs reused as use case commands.** `CreateOrderRequest` (API) and `CreateOrder.Command` (Application) look similar but serve different purposes. The API DTO handles serialisation; the command represents intent. Map between them in the controller.

6. **No namespace discipline.** Without separate projects, there's nothing stopping `Domain/Entities/Order.cs` from adding `using MyApp.Infrastructure.Data;`. Use architecture tests (NetArchTest, ArchUnitNET) to enforce rules in CI.

7. **Putting interfaces in the wrong layer.** `IOrderRepository` goes in **Domain/** (it's a domain concept — persistence of aggregates). `IEmailSender` goes in **Application/** (it's an application concern). `IOrderRepository` does NOT go in Infrastructure.

8. **Over-engineering for a CRUD app.** If your entities have no behaviour methods and your use cases are all just "get from repo, map, return," you don't need this pattern. Use [STR001](STR001%20-%20n-tier.md) and save yourself the ceremony.
