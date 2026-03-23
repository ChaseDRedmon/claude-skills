# Architecture Principles Reference

Detailed guidance for macro-level architecture decisions in modern .NET codebases.

## Table of Contents
1. [Vertical Slice Architecture](#vertical-slice-architecture)
2. [CQRS Pattern](#cqrs-pattern)
3. [Pipeline Behaviors](#pipeline-behaviors)
4. [Minimal APIs](#minimal-apis)
5. [Composition Over Inheritance](#composition-over-inheritance)
6. [Dependency Injection Rules](#dependency-injection-rules)
7. [Project Structure](#project-structure)
8. [Migration from Traditional Architecture](#migration-from-traditional-architecture)

---

## Vertical Slice Architecture

### Core Principles
- Organize around **use cases**, not technical layers
- Each slice owns its entire flow: endpoint → handler → persistence
- Minimal cross-slice dependencies
- Slices are independently deployable units of behavior

### Correct Structure
```
Features/
  Orders/
    CreateOrder/
      Endpoint.cs       ← Minimal API route definition
      Command.cs         ← Immutable request record
      Handler.cs         ← Business logic
      Validator.cs       ← FluentValidation rules
      Response.cs        ← Response DTO (if different from domain)
    GetOrder/
      Endpoint.cs
      Query.cs
      Handler.cs
    GetOrdersByCustomer/
      Endpoint.cs
      Query.cs
      Handler.cs
```

### Incorrect Structure — Layer-Based
```
Controllers/
  OrdersController.cs    ← GOD controller with 20 actions
Services/
  IOrderService.cs       ← Unnecessary abstraction
  OrderService.cs        ← Pass-through to repository
Repositories/
  IOrderRepository.cs    ← Wrapping EF Core for no reason
  OrderRepository.cs
DTOs/
  OrderDto.cs            ← Disconnected from feature context
```

### Slice Independence Rules
- A slice MUST NOT reference another slice's internal types
- Shared contracts go in `Shared/Contracts/`
- Shared behaviors go in `Shared/Behaviors/`
- If two slices need the same logic, **duplicate it** (WET principle)
- Only extract to Shared when 3+ slices genuinely share the same stable contract

### When Duplication Is Acceptable
```csharp
// Feature A has its own mapping
namespace Features.Orders.CreateOrder;
public sealed record Command(string ProductId, int Quantity);

// Feature B duplicates similar shape — this is OK
namespace Features.Orders.UpdateOrder;
public sealed record Command(Guid OrderId, string ProductId, int Quantity);
```

Duplication between slices is cheaper than coupling between slices. The cost of changing
a shared DTO propagates to every consumer. The cost of changing a duplicated record is local.

---

## CQRS Pattern

### Separation of Commands and Queries

**Commands** mutate state and return minimal data (usually an ID or success indicator):
```csharp
public sealed record CreateOrderCommand(
    Guid CustomerId,
    IReadOnlyList<OrderLineItem> Items
);

public sealed class CreateOrderHandler(AppDbContext db)
{
    public async Task<Guid> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        var order = Order.Create(command.CustomerId, command.Items);
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        return order.Id;
    }
}
```

**Queries** read state and return rich projections:
```csharp
public sealed record GetOrderQuery(Guid OrderId);

public sealed class GetOrderHandler(AppDbContext db)
{
    public async Task<OrderResponse?> Handle(GetOrderQuery query, CancellationToken ct)
    {
        return await db.Orders
            .AsNoTracking()
            .Where(o => o.Id == query.OrderId)
            .Select(o => new OrderResponse(
                o.Id,
                o.CustomerName,
                o.Total,
                o.Status.ToString()
            ))
            .SingleOrDefaultAsync(ct);
    }
}
```

### Benefits
- Handlers are small and focused on a single use case
- Reads and writes can be optimized independently
- Caching is trivial on the query side
- Testing is straightforward — each handler has clear inputs and outputs

### Rules
- Commands SHOULD validate input before persisting
- Queries MUST use `AsNoTracking()` and projection (`Select`)
- Handlers SHOULD accept `CancellationToken` as the last parameter
- Handlers SHOULD NOT call other handlers — compose at the endpoint level if needed

---

## Pipeline Behaviors

Behaviors are middleware for your CQRS pipeline. They wrap handler execution with
cross-cutting concerns, following a chain-of-responsibility pattern.

### Pipeline Order
```
Request arrives
  → ValidationBehavior      (reject invalid requests early)
    → AuthorizationBehavior  (check permissions)
      → LoggingBehavior      (structured request/response logging)
        → MetricsBehavior    (latency, throughput counters)
          → TransactionBehavior (wrap in DB transaction if needed)
            → Handler        (actual business logic)
```

### Example: Validation Behavior
```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);

        var failures = (await Task.WhenAll(
                validators.Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next();
    }
}
```

### Benefits of Pipeline Behaviors
- Handlers stay clean — zero cross-cutting logic in business code
- Behaviors are reusable across all handlers
- Easy to add/remove concerns without touching handler code
- Testable in isolation

---

## Minimal APIs

### Why Not Controllers
Controllers add unnecessary ceremony:
- Slower startup (MVC middleware pipeline)
- More allocations (controller instantiation per request)
- Verbose routing attributes
- Encourage "god controller" anti-pattern

### Minimal API Pattern
```csharp
// Endpoint.cs — clean, focused, one route
namespace MyApp.Features.Users.CreateUser;

public static class Endpoint
{
    public static void Map(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/users", async (
            Command command,
            Handler handler,
            CancellationToken ct) =>
        {
            var userId = await handler.Handle(command, ct);
            return Results.Created($"/api/users/{userId}", new { Id = userId });
        })
        .WithName("CreateUser")
        .WithTags("Users")
        .Produces<object>(StatusCodes.Status201Created)
        .ProducesValidationProblem();
    }
}
```

### Endpoint Registration in Program.cs
```csharp
// Organized registration
Features.Users.CreateUser.Endpoint.Map(app);
Features.Users.GetUser.Endpoint.Map(app);
Features.Orders.CreateOrder.Endpoint.Map(app);

// Or use a convention-based mapper that scans for IEndpoint implementations
```

---

## Composition Over Inheritance

### Prefer Composable Services
```csharp
// ✅ Composition — small, focused, composable
public sealed class OrderPricingService(
    TaxCalculator taxCalculator,
    DiscountCalculator discountCalculator)
{
    public decimal CalculateTotal(Order order)
    {
        var subtotal = order.Items.Sum(i => i.Price * i.Quantity);
        var discount = discountCalculator.Calculate(order);
        var tax = taxCalculator.Calculate(subtotal - discount);
        return subtotal - discount + tax;
    }
}
```

```csharp
// ❌ Inheritance — rigid, coupled, hard to test
public abstract class BaseOrderService { /* shared state, virtual methods */ }
public class StandardOrderService : BaseOrderService { }
public class PremiumOrderService : BaseOrderService { }
```

### When Inheritance Is Acceptable
- Implementing a framework-required base class (e.g., `BackgroundService`)
- True "is-a" relationships in domain modeling (rare)
- Never for code reuse alone — use composition

---

## Dependency Injection Rules

### Explicit Registration
```csharp
// ✅ Explicit — clear, predictable, debuggable
services.AddScoped<CreateUserHandler>();
services.AddScoped<GetUserHandler>();
services.AddSingleton<IClock, SystemClock>();

// ❌ Magic scanning — opaque, surprising, hard to debug
services.Scan(scan => scan.FromAssemblyOf<Program>()...);
```

### Abstractions Only When Needed
Create interfaces only for:
- **External dependencies** that need to be faked in tests: `IPaymentGateway`, `IEmailSender`
- **Time**: `IClock` (for deterministic testing)
- **Polymorphic behavior**: when you genuinely have multiple implementations

Do NOT create interfaces for:
- Internal handlers (`ICreateUserHandler` → just use `CreateUserHandler`)
- Services with a single implementation
- DbContext (it's already mockable via in-memory provider)

---

## Project Structure

### Single-Project API (Small/Medium)
```
src/MyApp/
  Features/
    Users/
    Orders/
  Infrastructure/
    Database/
      AppDbContext.cs
      Configurations/
    Integrations/
      PaymentGateway.cs
  Shared/
    Contracts/
    Behaviors/
    Extensions/
  Program.cs
  appsettings.json
```

### Multi-Project (Large)
```
src/
  MyApp.Api/              ← Endpoints + Program.cs
  MyApp.Application/      ← Features (Commands, Queries, Handlers)
  MyApp.Domain/           ← Domain entities, value objects
  MyApp.Infrastructure/   ← EF Core, external integrations
```

Even in multi-project, features remain organized by slice within each project.

---

## Migration from Traditional Architecture

### Step-by-Step
1. **Start with one feature** — pick a simple CRUD operation
2. **Create the slice folder** under `Features/`
3. **Move the endpoint** from the controller to a Minimal API static class
4. **Inline the service** — move logic from `XService` into the handler
5. **Remove the repository** — use `DbContext` directly in the handler
6. **Add pipeline behaviors** for validation, logging (centralized)
7. **Delete the empty abstractions** (interfaces with single implementations)
8. **Repeat** for each feature, working outward from the simplest

### What to Delete
- `IRepository<T>` / `Repository<T>` — EF Core IS the repository
- `IUnitOfWork` — `DbContext.SaveChangesAsync()` IS the unit of work
- `IXService` / `XService` where the service only delegates to a repo
- `BaseController` — each endpoint is independent
- `MappingProfile` (AutoMapper) — use manual mapping in the handler or `Select` projections
