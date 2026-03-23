# Cross-Cutting Concerns Reference

Pipeline behaviors, validation, logging, metrics, authorization, and error handling patterns.

## Table of Contents
1. [Pipeline Architecture](#pipeline-architecture)
2. [Validation](#validation)
3. [Logging](#logging)
4. [Metrics & Observability](#metrics--observability)
5. [Authorization](#authorization)
6. [Error Handling](#error-handling)
7. [Background Services](#background-services)
8. [Health Checks](#health-checks)

---

## Pipeline Architecture

Cross-cutting concerns are handled via pipeline behaviors that wrap handler execution.
This keeps handlers clean and focused on business logic only.

### Pipeline Execution Order
```
1. ValidationBehavior     → reject invalid requests early (cheapest check first)
2. AuthorizationBehavior  → verify permissions before doing work
3. LoggingBehavior        → structured request/response logging
4. MetricsBehavior        → latency and throughput measurement
5. TransactionBehavior    → wrap in database transaction (if needed)
6. Handler                → actual business logic
```

### Generic Behavior Interface
```csharp
public interface IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct);
}
```

### Registration Order Matters
```csharp
// Program.cs — register in execution order
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(AuthorizationBehavior<,>));
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(MetricsBehavior<,>));
```

---

## Validation

Use FluentValidation with a pipeline behavior. Validators live next to the feature they validate.

### Validator Per Command
```csharp
// Features/Users/CreateUser/Validator.cs
namespace MyApp.Features.Users.CreateUser;

public sealed class Validator : AbstractValidator<Command>
{
    public Validator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format")
            .MaximumLength(256);

        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Password is required")
            .MinimumLength(8).WithMessage("Password must be at least 8 characters")
            .Matches("[A-Z]").WithMessage("Password must contain an uppercase letter")
            .Matches("[0-9]").WithMessage("Password must contain a digit");
    }
}
```

### Async Validation (Database Checks)
```csharp
public sealed class Validator : AbstractValidator<Command>
{
    public Validator(AppDbContext db)
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .MustAsync(async (email, ct) =>
                !await db.Users.AnyAsync(u => u.Email == email, ct))
            .WithMessage("Email is already registered");
    }
}
```

### Validation Behavior
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

### Rules
- One validator per command/query (co-located in the feature folder)
- Validation behavior runs first in the pipeline
- Handlers assume input is already validated — no defensive checks
- Use `MustAsync` for database-dependent validation (uniqueness checks)

---

## Logging

Use structured logging exclusively. Never use string interpolation in log calls.

### Structured Logging Pattern
```csharp
// ✅ Structured — properties captured as queryable fields
logger.LogInformation("User created {UserId} with email {Email}", user.Id, user.Email);
logger.LogWarning("Order {OrderId} payment failed: {Reason}", orderId, failureReason);
logger.LogError(exception, "Failed to process order {OrderId}", orderId);

// ❌ String interpolation — properties lost, not queryable
logger.LogInformation($"User created {user.Id} with email {user.Email}");
```

### Logging Behavior
```csharp
public sealed class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {RequestName}: {@Request}", requestName, request);

        var stopwatch = Stopwatch.StartNew();
        var response = await next();
        stopwatch.Stop();

        logger.LogInformation(
            "Handled {RequestName} in {ElapsedMs}ms",
            requestName,
            stopwatch.ElapsedMilliseconds);

        return response;
    }
}
```

### Log Level Guidelines
| Level | Use For |
|-------|---------|
| `Trace` | Detailed diagnostic data (disabled in production) |
| `Debug` | Development troubleshooting info |
| `Information` | Business events: user created, order placed, payment processed |
| `Warning` | Recoverable issues: retry succeeded, cache miss, rate limit approaching |
| `Error` | Operation failures: payment failed, external service unavailable |
| `Critical` | System-level failures: database unreachable, out of memory |

---

## Metrics & Observability

### OpenTelemetry Integration
```csharp
// Program.cs
services.AddOpenTelemetry()
    .WithTracing(builder => builder
        .AddAspNetCoreInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(builder => builder
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddOtlpExporter());
```

### Metrics Behavior
```csharp
public sealed class MetricsBehavior<TRequest, TResponse>(
    IMeterFactory meterFactory)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private static readonly Meter Meter = meterFactory.Create("MyApp.Handlers");
    private static readonly Histogram<double> Duration =
        Meter.CreateHistogram<double>("handler.duration", "ms");
    private static readonly Counter<long> Requests =
        Meter.CreateCounter<long>("handler.requests");

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        Requests.Add(1, new KeyValuePair<string, object?>("handler", requestName));

        var stopwatch = Stopwatch.StartNew();
        try
        {
            return await next();
        }
        finally
        {
            Duration.Record(
                stopwatch.Elapsed.TotalMilliseconds,
                new KeyValuePair<string, object?>("handler", requestName));
        }
    }
}
```

### Key Metrics to Track
- **Latency** — handler execution time (p50, p95, p99)
- **Throughput** — requests per second per handler
- **Error rate** — failures per handler
- **Database query time** — EF Core query duration

---

## Authorization

### Authorization Behavior
```csharp
// Marker interface for commands requiring authorization
public interface IAuthorizedRequest
{
    Guid UserId { get; }
    string RequiredRole { get; }
}

public sealed class AuthorizationBehavior<TRequest, TResponse>(
    IHttpContextAccessor httpContextAccessor)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (request is not IAuthorizedRequest authorized)
            return await next();

        var user = httpContextAccessor.HttpContext?.User;
        if (user?.Identity?.IsAuthenticated != true)
            throw new UnauthorizedAccessException();

        if (!user.IsInRole(authorized.RequiredRole))
            throw new ForbiddenAccessException();

        return await next();
    }
}
```

### Rules
- Authorization checks belong in the pipeline, not in handlers
- Handlers assume the caller is authorized — pipeline enforces it
- Use marker interfaces to declaratively tag which requests need authorization

---

## Error Handling

### Result Types for Expected Failures
```csharp
// Use a result type for business logic failures
public async Task<Result<Guid>> Handle(CreateOrderCommand command, CancellationToken ct)
{
    var customer = await db.Customers.FindAsync(command.CustomerId, ct);
    if (customer is null)
        return Result<Guid>.Fail("Customer not found");

    if (!customer.IsActive)
        return Result<Guid>.Fail("Customer account is inactive");

    // ... create order
    return Result<Guid>.Success(order.Id);
}
```

### Global Exception Handler for Unexpected Failures
```csharp
// Middleware for unhandled exceptions
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();

        logger.LogError(exception, "Unhandled exception");

        context.Response.StatusCode = exception switch
        {
            ValidationException => StatusCodes.Status400BadRequest,
            UnauthorizedAccessException => StatusCodes.Status401Unauthorized,
            ForbiddenAccessException => StatusCodes.Status403Forbidden,
            NotFoundException => StatusCodes.Status404NotFound,
            _ => StatusCodes.Status500InternalServerError
        };

        await context.Response.WriteAsJsonAsync(new
        {
            Error = exception?.Message ?? "An unexpected error occurred"
        });
    });
});
```

### Rules
- Use **Result types** for expected failures (validation, business rules)
- Use **exceptions** for unexpected failures (infrastructure errors)
- Use **global exception handler** for mapping exceptions to HTTP status codes
- **Never** use exceptions for control flow in business logic

---

## Background Services

### Hosted Service Pattern
```csharp
public sealed class OrderExpirationService(
    IServiceScopeFactory scopeFactory,
    ILogger<OrderExpirationService> logger) : BackgroundService
{
    private static readonly TimeSpan Interval = TimeSpan.FromMinutes(5);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

                var expiredCount = await db.Orders
                    .Where(o => o.Status == OrderStatus.Pending)
                    .Where(o => o.CreatedAt < DateTime.UtcNow.AddHours(-24))
                    .ExecuteUpdateAsync(
                        o => o.SetProperty(x => x.Status, OrderStatus.Expired),
                        stoppingToken);

                if (expiredCount > 0)
                    logger.LogInformation("Expired {Count} pending orders", expiredCount);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Error expiring orders");
            }

            await Task.Delay(Interval, stoppingToken);
        }
    }
}
```

### Rules
- Always use `CancellationToken` (the `stoppingToken` parameter)
- Create a new DI scope for each iteration (`IServiceScopeFactory`)
- Catch exceptions inside the loop — don't let the service crash
- Use `Channels` for producer/consumer queue patterns

---

## Health Checks

```csharp
// Program.cs
services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("database")
    .AddCheck("external-api", () =>
    {
        // Check external dependencies
        return HealthCheckResult.Healthy();
    });

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

Every production service should expose health check endpoints for:
- Database connectivity
- External service availability
- Cache availability
- Message queue connectivity
