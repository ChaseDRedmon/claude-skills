# Code Quality & Clean Code Reference

Naming conventions, formatting rules, complexity control, and aesthetic practices.

## Table of Contents
1. [Naming Principles](#naming-principles)
2. [Comment Policy](#comment-policy)
3. [XML Documentation](#xml-documentation)
4. [Magic String & Number Elimination](#magic-string--number-elimination)
5. [Complexity Control](#complexity-control)
6. [Method Design](#method-design)
7. [File Organization](#file-organization)
8. [Static Analysis & Enforcement](#static-analysis--enforcement)
9. [EditorConfig](#editorconfig)
10. [Common C# Anti-Patterns](#common-c-anti-patterns)
11. [Team Culture Rules](#team-culture-rules)

---

## Naming Principles

Code should read like English. If a name requires a comment to explain it, rename it.

### Method Names — Verb + Noun (Describe the Action)
```csharp
// ✅ Clear intent
CalculateMonthlyRevenue()
ValidateUserRegistration()
SendPasswordResetEmail()
GetPendingOrdersForCustomer()

// ❌ Vague, meaningless
Process()
DoThing()
HandleStuff()
Run()
Execute()
Helper()
```

### Variable Names — Describe the Content
```csharp
// ✅ Descriptive
var pendingOrders = GetPendingOrders();
var monthlyRevenue = CalculateMonthlyRevenue(startDate, endDate);
var activeUserCount = await db.Users.CountAsync(u => u.IsActive, ct);
var hasExistingSubscription = await db.Subscriptions.AnyAsync(s => s.UserId == userId, ct);

// ❌ Cryptic, abbreviated, generic
var data = GetData();
var result = Process();
var d = Calculate();
var x = db.Users.Count();
var flag = Check();
```

### Boolean Names — Reads as a Question
```csharp
// ✅ Reads naturally in conditionals
bool isActive
bool hasPermission
bool canEdit
bool shouldRetry

// ❌ Ambiguous
bool active      // active what?
bool permission  // has it? needs it?
bool edit        // can edit? was edited?
```

### Class Names — Noun Describing the Responsibility
```csharp
// ✅ Clear responsibility
CreateUserHandler
OrderPricingService
EmailNotificationSender
PaymentGatewayClient

// ❌ Vague containers
UserHelper         // "helper" means the design is unclear
OrderUtils         // "utils" is a junk drawer
DataManager        // "manager" manages what exactly?
ProcessorService   // redundant, vague
```

### Banned Suffixes
Avoid these unless the class genuinely represents the concept:
- `Helper` → indicates a missing domain concept
- `Utils` / `Utilities` → indicates a junk drawer class
- `Manager` → vague, almost always decomposable
- `Processor` → vague without domain context
- `Info` / `Data` → use the actual domain term

---

## Comment Policy

**If you need a comment, your code is unclear. Refactor the code instead.**

### Remove Explanatory Comments
```csharp
// ❌ Comment masks unclear code
// Calculate discount based on seasonal rates
var d = price * .15;

// ✅ The code itself is clear
var seasonalDiscount = CalculateSeasonalDiscount(price);
```

### Remove Section Comments
```csharp
// ❌ Comments as section headers indicate the method does too much
public async Task ProcessOrder(Order order)
{
    // Validate the order
    if (order.Items.Count == 0) throw new Exception("No items");

    // Calculate totals
    var subtotal = order.Items.Sum(i => i.Price);
    var tax = subtotal * 0.08m;

    // Save to database
    await db.SaveChangesAsync();

    // Send notification
    await emailService.Send(order.CustomerEmail, "Order placed");
}

// ✅ Decompose into methods — no comments needed
public async Task HandleOrder(Order order, CancellationToken ct)
{
    ValidateOrder(order);
    var totals = CalculateOrderTotals(order);
    await PersistOrder(order, totals, ct);
    await NotifyCustomer(order, ct);
}
```

### Acceptable Comments
- **XML documentation** on public APIs (see below)
- **`// TODO:`** for tracked work items (with ticket reference)
- **Regulatory/legal** requirements
- **Non-obvious "why"** when the code intentionally looks wrong:
  ```csharp
  // Intentionally using Thread.Sleep here because the external API
  // rate-limits and does not provide a retry-after header.
  Thread.Sleep(500);
  ```

---

## XML Documentation

Use XML documentation **only** on public APIs. Not on private/internal code.

### Required On
```csharp
/// <summary>
/// Creates a new user account with the specified credentials.
/// </summary>
/// <param name="command">The user creation request containing email and password.</param>
/// <param name="ct">Cancellation token for the async operation.</param>
/// <returns>The unique identifier of the newly created user.</returns>
/// <exception cref="ValidationException">Thrown when the email is already registered.</exception>
public Task<Guid> Handle(CreateUserCommand command, CancellationToken ct)
```

### Not Required On
```csharp
// Private methods — naming should be sufficient
private static decimal CalculateSeasonalDiscount(decimal price) => price * 0.15m;

// Internal types — context is local
internal sealed class OrderTaxCalculator { }
```

---

## Magic String & Number Elimination

Every string literal and numeric literal used for comparison, configuration, or business
logic must be extracted to a named constant.

### Constants for String Comparisons
```csharp
// ❌ Magic string
if (user.Role == "admin") { }
if (order.Status == "pending") { }

// ✅ Named constants
internal static class Roles
{
    public const string Admin = "admin";
    public const string User = "user";
    public const string Manager = "manager";
}

internal static class OrderStatuses
{
    public const string Pending = "pending";
    public const string Processing = "processing";
    public const string Complete = "complete";
}

if (user.Role == Roles.Admin) { }
if (order.Status == OrderStatuses.Pending) { }
```

### Constants for Numeric Values
```csharp
// ❌ Magic numbers
if (retryCount > 3) { }
var timeout = TimeSpan.FromSeconds(30);
var pageSize = 25;

// ✅ Named constants
internal static class RetryPolicy
{
    public const int MaxRetries = 3;
    public static readonly TimeSpan DefaultTimeout = TimeSpan.FromSeconds(30);
}

internal static class Pagination
{
    public const int DefaultPageSize = 25;
    public const int MaxPageSize = 100;
}
```

### Organization
- Group related constants in `internal static class` containers
- Place near the feature that uses them, or in `Shared/Constants/` if cross-cutting
- Use `const` for compile-time constants (strings, ints)
- Use `static readonly` for runtime constants (`TimeSpan`, `DateTime`, complex objects)

---

## Complexity Control

### Maximum 3 Levels of Indentation Per Method

```csharp
// ❌ Deep nesting — hard to read, hard to test
public void ProcessOrders(List<Order> orders)
{
    foreach (var order in orders)                    // Level 1
    {
        if (order.Status == OrderStatus.Pending)     // Level 2
        {
            foreach (var item in order.Items)        // Level 3
            {
                if (item.IsAvailable)                // Level 4 ← VIOLATION
                {
                    // ...
                }
            }
        }
    }
}

// ✅ Flatten with guard clauses and extracted methods
public void ProcessOrders(List<Order> orders)
{
    foreach (var order in orders)
    {
        if (order.Status != OrderStatus.Pending) continue;
        ProcessPendingOrder(order);
    }
}

private void ProcessPendingOrder(Order order)
{
    foreach (var item in order.Items)
    {
        if (!item.IsAvailable) continue;
        FulfillItem(item);
    }
}
```

### Guard Clauses — Fail Fast at the Top
```csharp
// ✅ Guards first, then the happy path
public async Task<OrderResponse> Handle(GetOrderQuery query, CancellationToken ct)
{
    if (query.OrderId == Guid.Empty)
        throw new ArgumentException("Order ID is required", nameof(query));

    var order = await db.Orders
        .AsNoTracking()
        .SingleOrDefaultAsync(o => o.Id == query.OrderId, ct);

    if (order is null)
        throw new NotFoundException($"Order {query.OrderId} not found");

    return MapToResponse(order);
}
```

### Early Returns Over Nested Else
```csharp
// ✅ Early return
if (user is null) return NotFound();
if (!user.IsActive) return Forbid();
return Ok(user);

// ❌ Nested if/else
if (user is not null)
{
    if (user.IsActive)
    {
        return Ok(user);
    }
    else
    {
        return Forbid();
    }
}
else
{
    return NotFound();
}
```

---

## Method Design

### One Method = One Responsibility
A method should do exactly one thing. If you can describe it with "and", split it.

### Ideal Method Length: 5–20 Lines
- Under 5: might be extracting too aggressively (unless it's a simple expression)
- Over 20: look for extraction opportunities
- Over 40: almost certainly doing too much

### Method Signature Rules
- Maximum 3–4 parameters (use a record if you need more)
- `CancellationToken` is always the last parameter
- Return types should be specific, not `object` or `dynamic`

```csharp
// ❌ Too many parameters
public Task<Order> CreateOrder(
    Guid customerId, string productId, int quantity,
    string shippingAddress, string billingAddress,
    bool isGift, string giftMessage, decimal discount)

// ✅ Use a command record
public sealed record CreateOrderCommand(
    Guid CustomerId,
    string ProductId,
    int Quantity,
    Address ShippingAddress,
    Address BillingAddress,
    GiftOptions? Gift = null,
    decimal Discount = 0m
);

public Task<Guid> Handle(CreateOrderCommand command, CancellationToken ct)
```

---

## File Organization

### One Type Per File
Every public class, record, enum, or interface gets its own file.

```
CreateUserCommand.cs    ← one record
CreateUserHandler.cs    ← one class
CreateUserValidator.cs  ← one class
CreateUserEndpoint.cs   ← one static class
```

Exception: small supporting types (private nested classes, simple enums used by one file)
can stay in the same file.

### Ideal File Length: 50–200 Lines
- Under 50: acceptable for simple types (records, validators)
- Over 200: likely has too many responsibilities
- Over 400: definitely needs decomposition

### Using Directive Order
```csharp
// System namespaces first, then third-party, then project
using System.Collections.Generic;
using Microsoft.EntityFrameworkCore;
using FluentValidation;
using MyApp.Shared.Contracts;
```

---

## Static Analysis & Enforcement

### Required Analyzers
- **Roslynator** — comprehensive code quality rules
- **StyleCop.Analyzers** — naming and formatting consistency
- **SonarAnalyzer** — code smells and complexity metrics

### Architecture Tests
Use **NetArchTest** or **ArchUnitNET** to enforce boundaries:
```csharp
// Features cannot reference other features
var result = Types.InAssembly(typeof(Program).Assembly)
    .That().ResideInNamespace("Features.Orders")
    .ShouldNot().HaveDependencyOn("Features.Users")
    .GetResult();

Assert.True(result.IsSuccessful);
```

---

## EditorConfig

Every project should include an `.editorconfig` enforcing consistent style:

```ini
[*.cs]
indent_size = 4
indent_style = space
max_line_length = 120
dotnet_sort_system_directives_first = true
csharp_style_namespace_declarations = file_scoped:error
csharp_prefer_simple_using_statement = true:suggestion
csharp_style_var_for_built_in_types = false:suggestion
csharp_style_prefer_switch_expression = true:suggestion
csharp_style_prefer_pattern_matching = true:suggestion
dotnet_style_prefer_is_null_check_over_reference_equality_method = true:warning
```

---

## Common C# Anti-Patterns

Patterns that compile but cause deadlocks, resource leaks, hidden bugs, or unmaintainable code.

### Async Anti-Patterns

#### Forgotten `await` — Silent Fire-and-Forget
```csharp
// ❌ Missing await — exception goes unobserved, method runs unsupervised
public async Task Handle(Command command, CancellationToken ct)
{
    SaveAuditLog(command);  // WARNING: this returns Task but is never awaited
    await db.SaveChangesAsync(ct);
}

// ✅ Always await async calls
public async Task Handle(Command command, CancellationToken ct)
{
    await SaveAuditLog(command, ct);
    await db.SaveChangesAsync(ct);
}
```

**Why:** Unobserved tasks swallow exceptions silently. The compiler emits CS4014 — treat
this warning as an error in `.editorconfig` or `Directory.Build.props`:
```xml
<PropertyGroup>
    <WarningsAsErrors>CS4014</WarningsAsErrors>
</PropertyGroup>
```

#### `async void` — Unhandled Crash Risk
```csharp
// ❌ async void — exceptions crash the process, caller cannot await
async void SaveUser(User user) { await db.SaveChangesAsync(); }

// ✅ async Task — exceptions propagate, caller can await
async Task SaveUserAsync(User user, CancellationToken ct)
{
    await db.SaveChangesAsync(ct);
}
```

**Only exception:** Event handlers in UI frameworks (`async void OnClick`).

### Exception Handling Anti-Patterns

#### `throw ex` Destroys the Stack Trace
```csharp
// ❌ throw ex — resets the stack trace, hiding the original failure location
try { await ProcessOrder(order, ct); }
catch (Exception ex)
{
    logger.LogError(ex, "Order processing failed");
    throw ex;  // WRONG: stack trace now points HERE, not the original failure
}

// ✅ throw; — preserves the original stack trace
try { await ProcessOrder(order, ct); }
catch (Exception ex)
{
    logger.LogError(ex, "Order processing failed");
    throw;  // stack trace intact
}

// ✅ Wrap with inner exception when adding context
catch (Exception ex)
{
    throw new OrderProcessingException($"Failed to process order {order.Id}", ex);
}
```

#### Swallowing Exceptions — Empty Catch Blocks
```csharp
// ❌ CRITICAL — silently hides failures, system enters unknown state
try { await SendEmail(user.Email, ct); }
catch (Exception) { }  // What happened? Nobody knows.

// ❌ Also bad — catch-log-swallow without re-throwing
try { await SendEmail(user.Email, ct); }
catch (Exception ex) { logger.LogError(ex, "Email failed"); }
// Caller thinks everything succeeded

// ✅ Log and rethrow, or let it propagate
try { await SendEmail(user.Email, ct); }
catch (Exception ex)
{
    logger.LogError(ex, "Failed to send email to {Email}", user.Email);
    throw;
}

// ✅ If the failure is truly non-critical, be explicit about why
try { await SendEmail(user.Email, ct); }
catch (Exception ex)
{
    // Email failure is non-critical — order still succeeds.
    // Alert is monitored via structured log query.
    logger.LogWarning(ex, "Non-critical: welcome email failed for {UserId}", user.Id);
}
```

#### Throwing Wrong Exception Types
```csharp
// ❌ Never throw NullReferenceException — it signals a bug, not a validation failure
if (user is null) throw new NullReferenceException("User was null");

// ❌ Never throw the base Exception type
throw new Exception("Something went wrong");

// ✅ Use specific, appropriate exception types
if (user is null) throw new ArgumentNullException(nameof(user));
if (orderId == Guid.Empty) throw new ArgumentException("Order ID is required", nameof(orderId));
throw new InvalidOperationException($"Cannot cancel order in {order.Status} status");
```

**Reserved for the runtime:** Never manually throw `NullReferenceException`,
`StackOverflowException`, `IndexOutOfRangeException`, or `AccessViolationException`.

### Resource Management Anti-Patterns

#### HttpClient Misuse — Socket Exhaustion
```csharp
// ❌ Creating HttpClient per request — causes socket exhaustion under load
public async Task<string> GetData(string url, CancellationToken ct)
{
    using var client = new HttpClient();  // WRONG: new sockets every call
    return await client.GetStringAsync(url, ct);
}

// ✅ Use IHttpClientFactory (registered via DI)
public sealed class ExternalApiClient(HttpClient httpClient)
{
    public async Task<string> GetData(string url, CancellationToken ct)
        => await httpClient.GetStringAsync(url, ct);
}

// Registration:
builder.Services.AddHttpClient<ExternalApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.Timeout = TimeSpan.FromSeconds(10);
});
```

**Rule:** Never `new HttpClient()` in application code. Always use `IHttpClientFactory`
or typed/named HTTP clients via DI.

### Architectural Anti-Patterns

#### Anemic Domain Model — Logic-Free Entities
```csharp
// ❌ Anemic — entity is just a data bag, all logic in external service
public class Order
{
    public Guid Id { get; set; }
    public OrderStatus Status { get; set; }
    public decimal Total { get; set; }
}

public class OrderService
{
    public void Cancel(Order order)
    {
        if (order.Status == OrderStatus.Shipped)
            throw new InvalidOperationException("Cannot cancel shipped order");
        order.Status = OrderStatus.Cancelled;
    }
}

// ✅ Rich domain — entity owns its behavior and enforces invariants
public sealed class Order
{
    public Guid Id { get; init; }
    public OrderStatus Status { get; private set; }
    public decimal Total { get; init; }

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
            throw new InvalidOperationException("Cannot cancel a shipped order");
        Status = OrderStatus.Cancelled;
    }
}
```

**Note:** In Vertical Slice Architecture, some entities will be simple data holders
(especially read models). The anti-pattern is when entities that *should* have behavior
are stripped of it and forced into a service class.

#### Singleton Abuse — Hidden Global Mutable State
```csharp
// ❌ Singleton with mutable state — hidden dependency, untestable, thread-unsafe
public sealed class AppState
{
    public static AppState Instance { get; } = new();
    public User? CurrentUser { get; set; }        // mutable shared state
    public List<string> Errors { get; } = [];      // grows unbounded
}

// Usage hides the dependency
var user = AppState.Instance.CurrentUser;  // where did this come from?

// ✅ Use DI with explicit scoping instead
builder.Services.AddScoped<IUserContext, HttpUserContext>();
// or
builder.Services.AddSingleton<IClock, SystemClock>();  // only for truly stateless services
```

**Rule:** Singleton lifetime via DI is fine for stateless services (clocks, configuration,
factories). Manual singleton patterns with mutable state are always wrong in modern .NET.

---

## Team Culture Rules

### The Boy Scout Rule
Leave code cleaner than you found it. Every PR should improve at least one thing
beyond the immediate task.

### Prefer WET Over DRY When DRY Creates Coupling
In Vertical Slice Architecture, small duplication between features is acceptable and
preferred over shared abstractions that couple features together.

**Rule of thumb:** Extract to shared only when 3+ features use the exact same stable contract.
If you're modifying the shared code every time a single feature changes, it was premature extraction.

### Code Review Focus Areas
Reviews should check (in priority order):
1. **Correctness** — does it work?
2. **Readability** — can you understand it in 30 seconds?
3. **Architecture boundaries** — does it respect slice isolation?
4. **Naming clarity** — do names communicate intent?
5. **Performance risks** — N+1 queries, missing async, allocations
6. **Unnecessary abstractions** — can any layer be removed?
