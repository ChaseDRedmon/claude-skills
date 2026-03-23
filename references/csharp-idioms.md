# C# Language Idioms & Performance Reference

Modern C# constructs, functional programming patterns, and performance best practices.

## Table of Contents
1. [Immutability by Default](#immutability-by-default)
2. [Sealed Classes](#sealed-classes)
3. [File-Scoped Namespaces](#file-scoped-namespaces)
4. [Pattern Matching](#pattern-matching)
5. [Primary Constructors](#primary-constructors)
6. [Value Objects](#value-objects)
7. [Result Types Over Exceptions](#result-types-over-exceptions)
8. [Functional Patterns](#functional-patterns)
9. [Async Best Practices](#async-best-practices)
10. [Performance Patterns](#performance-patterns)
11. [C# 12 Features (.NET 8)](#c-12-features-net-8)
12. [C# 13 Features (.NET 9)](#c-13-features-net-9)
13. [C# 14 Features (.NET 10)](#c-14-features-net-10)
14. [EF Core Query Patterns](#ef-core-query-patterns)

---

## Immutability by Default

All data transfer objects, commands, queries, and events should be immutable.

### Records for DTOs and Messages
```csharp
// ✅ Immutable record — positional syntax
public sealed record CreateUserCommand(string Email, string Password);

// ✅ Immutable record — property syntax (for more complex types)
public sealed record OrderResponse
{
    public required Guid Id { get; init; }
    public required string CustomerName { get; init; }
    public required decimal Total { get; init; }
    public required IReadOnlyList<OrderLineItem> Items { get; init; }
}

// ❌ Mutable DTO with setters
public class CreateUserDto
{
    public string Email { get; set; }
    public string Password { get; set; }
}
```

### Init Properties for Entities
```csharp
public sealed class User
{
    public Guid Id { get; init; }
    public string Email { get; init; } = string.Empty;
    public DateTime CreatedAt { get; init; }
    
    // Only properties that genuinely change after creation use set
    public string DisplayName { get; set; } = string.Empty;
    public DateTime? LastLoginAt { get; set; }
}
```

### Immutable Collections
```csharp
// ✅ Prefer read-only collection interfaces in public APIs
public IReadOnlyList<OrderItem> Items { get; init; } = [];

// ❌ Mutable collection exposed publicly
public List<OrderItem> Items { get; set; } = new();
```

---

## Sealed Classes

**Seal all classes by default.** Only unseal when explicitly designing for inheritance.

```csharp
// ✅ Sealed — prevents unintended inheritance, enables JIT optimizations
public sealed class CreateUserHandler { }
public sealed class UserValidator : AbstractValidator<CreateUserCommand> { }
public sealed class OrderPricingService { }

// ✅ Sealed record — same principle
public sealed record CreateUserCommand(string Email, string Password);

// ❌ Unsealed with no reason — invites misuse
public class UserService { }
```

### When to Unseal
- Designing a base class for a known hierarchy (e.g., `DomainEvent`)
- Framework requirements (some libraries need non-sealed types)
- Abstract base classes with defined extension points

### Performance Benefit
The JIT compiler can devirtualize method calls on sealed types, resulting in direct
calls instead of virtual dispatch. This is measurable on hot paths.

---

## File-Scoped Namespaces

**Always** use file-scoped namespaces. They reduce indentation by one level across the
entire file.

```csharp
// ✅ File-scoped
namespace MyApp.Features.Users.CreateUser;

public sealed class Handler(AppDbContext db)
{
    public async Task<Guid> Handle(Command command, CancellationToken ct)
    {
        // Code at 1 level of indentation from the class
    }
}
```

```csharp
// ❌ Block-scoped — wastes horizontal space
namespace MyApp.Features.Users.CreateUser
{
    public sealed class Handler
    {
        // Code at 2 levels of indentation from the namespace
    }
}
```

---

## Pattern Matching

Use modern pattern matching for clearer, more expressive conditionals.

### Null Checks
```csharp
// ✅ Pattern matching
if (user is null) return NotFound();
if (user is not null) { }

// ❌ Old style
if (user == null) return NotFound();
if (user != null) { }
```

### Type Checks
```csharp
// ✅ Pattern with variable binding
if (result is ErrorResult { Message: var message })
    return BadRequest(message);

// ✅ Switch expression
return notification switch
{
    EmailNotification email => SendEmail(email),
    SmsNotification sms => SendSms(sms),
    PushNotification push => SendPush(push),
    _ => throw new NotSupportedException($"Unknown notification type: {notification.GetType()}")
};
```

### Property Patterns
```csharp
// ✅ Property pattern
if (order is { Status: OrderStatus.Pending, Total: > 0 })
{
    await ProcessOrder(order, ct);
}
```

### Switch Expressions Over Switch Statements
```csharp
// ✅ Expression — returns a value, exhaustive
var statusText = order.Status switch
{
    OrderStatus.Pending => "Awaiting processing",
    OrderStatus.Processing => "In progress",
    OrderStatus.Shipped => "On its way",
    OrderStatus.Delivered => "Complete",
    OrderStatus.Cancelled => "Cancelled",
    _ => throw new InvalidOperationException($"Unknown status: {order.Status}")
};

// ❌ Statement — verbose, easy to forget break
switch (order.Status)
{
    case OrderStatus.Pending:
        statusText = "Awaiting processing";
        break;
    // ... 20 more lines
}
```

---

## Primary Constructors

Use primary constructors for dependency injection in C# 12+.

```csharp
// ✅ Primary constructor — clean, minimal
public sealed class CreateUserHandler(AppDbContext db, ILogger<CreateUserHandler> logger)
{
    public async Task<Guid> Handle(Command command, CancellationToken ct)
    {
        logger.LogInformation("Creating user {Email}", command.Email);
        
        var user = new User { Id = Guid.NewGuid(), Email = command.Email };
        db.Users.Add(user);
        await db.SaveChangesAsync(ct);
        
        return user.Id;
    }
}

// ❌ Verbose constructor with field assignments
public sealed class CreateUserHandler
{
    private readonly AppDbContext _db;
    private readonly ILogger<CreateUserHandler> _logger;

    public CreateUserHandler(AppDbContext db, ILogger<CreateUserHandler> logger)
    {
        _db = db;
        _logger = logger;
    }
}
```

---

## Value Objects

Avoid primitive obsession. Wrap domain concepts in value objects.

```csharp
// ✅ Value object with validation
public sealed record EmailAddress
{
    public string Value { get; }

    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email address", nameof(value));

        Value = value.Trim().ToLowerInvariant();
    }

    public static implicit operator string(EmailAddress email) => email.Value;
    public override string ToString() => Value;
}

// ✅ Value object for money
public sealed record Money(decimal Amount, string Currency)
{
    public static Money USD(decimal amount) => new(amount, "USD");
    public static Money Zero(string currency) => new(0m, currency);
}

// ❌ Primitive types lose meaning
public string Email { get; set; }       // Any string? No validation?
public decimal Price { get; set; }      // In what currency? Tax included?
public int Quantity { get; set; }       // Can it be negative?
```

**Note on implicit operators in value objects:** The `implicit operator string` on
`EmailAddress` above is acceptable — it unwraps a value object to its underlying
primitive for interop. This is **not** the same as DTO-to-entity conversion (see below).

### Implicit Operators for DTO/Entity Conversion — Anti-Pattern

**Never** use implicit (or explicit) conversion operators to convert between DTOs, domain
entities, commands, responses, or other cross-layer types. This is a well-known anti-pattern
that creates hidden coupling and surprises.

```csharp
// ❌ ANTI-PATTERN — implicit operator on DTO converting to domain entity
public sealed record CreateUserDto
{
    public string Name { get; init; } = string.Empty;
    public string Email { get; init; } = string.Empty;

    public static implicit operator User(CreateUserDto dto)
    {
        return new User { Name = dto.Name, Email = dto.Email };
    }
}

// This compiles silently — no way to know a conversion happened
User user = dto;  // Surprise! Where did this mapping come from?
```

**Why this is harmful:**
- **Hidden behavior** — assignments silently invoke conversion logic. Developers reading
  `User user = dto` have no indication a mapping is occurring.
- **Tight coupling** — the DTO knows about the domain entity's structure, violating
  separation of concerns. Changes to `User` silently break the DTO.
- **Untestable mapping** — conversion logic buried inside an operator is harder to
  unit test and impossible to mock or swap.
- **Debugging difficulty** — stack traces through implicit operators are confusing.
  Breakpoints on assignment don't reveal the conversion.

```csharp
// ✅ Explicit mapping method — intent is visible, testable, discoverable
public sealed record CreateUserDto
{
    public string Name { get; init; } = string.Empty;
    public string Email { get; init; } = string.Empty;

    public User ToEntity() => new User { Name = Name, Email = Email };
}

// Usage — the mapping is visible and intentional
User user = dto.ToEntity();
```

```csharp
// ✅ Manual mapping in the handler — keeps DTOs completely decoupled
public async Task<Guid> Handle(CreateUserCommand command, CancellationToken ct)
{
    var user = new User
    {
        Id = Guid.NewGuid(),
        Name = command.Name,
        Email = command.Email
    };
    db.Users.Add(user);
    await db.SaveChangesAsync(ct);
    return user.Id;
}
```

```csharp
// ✅ Extension method — mapping is explicit and discoverable
public static class UserMappings
{
    public static User ToEntity(this CreateUserCommand command)
        => new User { Name = command.Name, Email = command.Email };

    public static UserResponse ToResponse(this User user)
        => new UserResponse(user.Id, user.Name, user.Email);
}
```

**Acceptable uses of implicit operators:**
- Value objects unwrapping to their underlying primitive (`EmailAddress` → `string`)
- Strongly-typed IDs unwrapping (`UserId` → `Guid`)
- Numeric widening within the same domain concept (`Percentage` → `decimal`)

**Never use implicit operators for:**
- DTO → Entity or Entity → DTO
- Command/Query → Entity
- Entity → Response/ViewModel
- Any cross-layer or cross-boundary type conversion

---

## Result Types Over Exceptions

Do not use exceptions for control flow. Use discriminated union / result types.

```csharp
// ✅ Result pattern
public sealed record Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess => Error is null;

    private Result(T value) => Value = value;
    private Result(string error) => Error = error;

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Fail(string error) => new(error);
}

// Usage in handler
public async Task<Result<Guid>> Handle(Command command, CancellationToken ct)
{
    var existingUser = await db.Users
        .AnyAsync(u => u.Email == command.Email, ct);

    if (existingUser)
        return Result<Guid>.Fail("Email already registered");

    var user = new User { Id = Guid.NewGuid(), Email = command.Email };
    db.Users.Add(user);
    await db.SaveChangesAsync(ct);

    return Result<Guid>.Success(user.Id);
}
```

Consider libraries: `OneOf`, `ErrorOr`, `FluentResults` for production use.

---

## Functional Patterns

### Pure Functions
```csharp
// ✅ Pure — deterministic, no side effects
public static decimal CalculateTax(decimal subtotal, decimal taxRate)
    => subtotal * taxRate;

// ❌ Impure — hidden side effect + external dependency
public decimal CalculateTax(Order order)
{
    var rate = _taxService.GetRate(order.State);  // external call
    order.Tax = order.Subtotal * rate;            // mutation
    _logger.Log("Tax calculated");                // side effect
    return order.Tax;
}
```

### Expressions Over Statements
```csharp
// ✅ Expression-based — reads as a data pipeline
var totalRevenue = orders
    .Where(o => o.Status == OrderStatus.Delivered)
    .Where(o => o.DeliveredAt >= startOfMonth)
    .Select(o => o.Total)
    .Sum();

// ❌ Statement-based — imperative, harder to follow
decimal totalRevenue = 0;
foreach (var order in orders)
{
    if (order.Status == OrderStatus.Delivered)
    {
        if (order.DeliveredAt >= startOfMonth)
        {
            totalRevenue += order.Total;
        }
    }
}
```

### Immutable Data Flow
Data should flow through handlers without mutation across layers:
```
Immutable Command → Handler (creates/modifies entities) → Immutable Response
```

---

## Async Best Practices

### Always Async
```csharp
// ✅ Properly awaited
var user = await db.Users.FindAsync(userId, ct);

// ❌ NEVER block on async
var user = db.Users.FindAsync(userId).Result;  // DEADLOCK RISK
db.SaveChangesAsync().Wait();                   // DEADLOCK RISK
```

### Pass CancellationToken Everywhere
```csharp
public async Task<Guid> Handle(Command command, CancellationToken ct)
{
    var user = await db.Users.FindAsync(new object[] { command.UserId }, ct);
    await db.SaveChangesAsync(ct);
    await emailService.SendWelcomeEmail(user.Email, ct);
    return user.Id;
}
```

### ConfigureAwait in Library Code
```csharp
// In library/shared code (not in ASP.NET Core handlers)
await SomeAsyncMethod().ConfigureAwait(false);
```

---

## Performance Patterns

### Avoid Allocations on Hot Paths
```csharp
// ✅ Stack-allocated value type
public readonly record struct OrderSummary(Guid Id, decimal Total);

// ✅ Span for string processing
ReadOnlySpan<char> trimmed = input.AsSpan().Trim();

// ✅ ValueTask for frequently synchronous completion
public ValueTask<Order?> GetCachedOrder(Guid id)
{
    if (_cache.TryGetValue(id, out var order))
        return ValueTask.FromResult<Order?>(order);

    return new ValueTask<Order?>(LoadFromDatabaseAsync(id));
}
```

### Avoid LINQ in Tight Loops
```csharp
// ✅ Direct iteration — no iterator allocation
foreach (var item in items)
{
    if (!item.IsActive) continue;
    Process(item);
}

// ❌ LINQ allocates iterator objects
foreach (var item in items.Where(x => x.IsActive))
{
    Process(item);
}
```

### String Handling
```csharp
// ✅ String interpolation (compiler-optimized)
var message = $"User {userId} created at {createdAt:O}";

// ✅ StringBuilder for loops
var sb = new StringBuilder();
foreach (var item in items)
    sb.AppendLine(item.Name);

// ❌ String concatenation in loops
var result = "";
foreach (var item in items)
    result += item.Name + "\n";
```

---

## C# 12 Features (.NET 8)

Adopt these features where they improve clarity and reduce boilerplate.
Primary constructors and collection expressions are already covered in earlier sections —
this section adds the remaining C# 12 features.

### Prerequisites — Validate Before Applying

C# 12 features require the **.NET 8 SDK** and a project targeting `net8.0` or explicitly
setting `<LangVersion>12</LangVersion>`.

```xml
<!-- ✅ Targeting .NET 8 — C# 12 is the default language version -->
<PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
</PropertyGroup>
```

**Before applying any C# 12 feature:**
1. Check `<TargetFramework>` in `.csproj` — must be `net8.0` or higher
2. OR check `<LangVersion>` — must be `12` or `preview` or `latest` (when SDK is 8+)
3. Check `Directory.Build.props` if the project uses centralized build settings
4. If neither condition is met, **do not use C# 12 features** — suggest upgrading first

### Collection Expressions — Unified Collection Syntax

Collection expressions replace `new[]`, `new List<T>`, `Array.Empty<T>()`, and
`Enumerable.Empty<T>()` with a single `[]` syntax. The spread operator `..` flattens
collections inline.

```csharp
// ✅ C# 12 — collection expressions
int[] numbers = [1, 2, 3, 4, 5];
List<string> names = ["Alice", "Bob", "Charlie"];
Span<byte> buffer = [0x00, 0xFF, 0xAB];
ImmutableArray<int> immutable = [10, 20, 30];

// ✅ Empty collections — replaces Array.Empty<T>() and Enumerable.Empty<T>()
int[] empty = [];
List<string> noNames = [];
IReadOnlyList<int> noItems = [];

// ✅ Spread operator — flatten collections inline
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [.. first, .. second];       // [1, 2, 3, 4, 5, 6]
int[] withExtra = [0, .. first, .. second, 7]; // [0, 1, 2, 3, 4, 5, 6, 7]
```

```csharp
// ❌ Before C# 12 — verbose and inconsistent
int[] numbers = new[] { 1, 2, 3, 4, 5 };
List<string> names = new List<string> { "Alice", "Bob", "Charlie" };
int[] empty = Array.Empty<int>();
int[] combined = first.Concat(second).ToArray();
```

**When to use:** All new collection initialization. Prefer `[]` over `new[]`, `new List<T>{}`,
`Array.Empty<T>()`, and `Enumerable.Empty<T>()`.

### Primary Constructors on Classes and Structs

Primary constructors now work on all `class` and `struct` types, not just `record` types.
Parameters are in scope for the entire class body. Unlike records, the compiler does **not**
generate public properties for the parameters — they remain captured parameters.

```csharp
// ✅ C# 12 — primary constructor for DI (already covered in Primary Constructors section)
public sealed class CreateUserHandler(AppDbContext db, ILogger<CreateUserHandler> logger)
{
    public async Task<Guid> Handle(Command command, CancellationToken ct)
    {
        logger.LogInformation("Creating user {Email}", command.Email);
        db.Users.Add(new User { Email = command.Email });
        await db.SaveChangesAsync(ct);
        return Guid.NewGuid();
    }
}

// ✅ Primary constructor on struct
public readonly struct Point(double x, double y)
{
    public double X { get; } = x;
    public double Y { get; } = y;
    public double Distance => Math.Sqrt(X * X + Y * Y);
}
```

**Important:** Primary constructor parameters on non-record types are **not** auto-properties.
If you need them exposed publicly, declare explicit properties initialized from the parameters.

### Alias Any Type — `using` for Tuples, Arrays, and More

The `using` alias directive now works with any type, not just named types. This enables
semantic aliases for tuples, arrays, pointer types, and generic specializations.

```csharp
// ✅ C# 12 — alias tuple types for domain clarity
using Coordinate = (double Latitude, double Longitude);
using UserInfo = (string Name, string Email, int Age);

public sealed class LocationService
{
    public Coordinate GetUserLocation(Guid userId)
        => (47.6062, -122.3321);
}
```

```csharp
// ✅ Alias generic types for readability
using OrderLookup = System.Collections.Generic.Dictionary<System.Guid, Order>;
using JsonOptions = System.Text.Json.JsonSerializerOptions;

// ✅ Alias array types
using Matrix = double[][];
```

```csharp
// ❌ Before C# 12 — could only alias named types
using JsonOptions = System.Text.Json.JsonSerializerOptions;  // this worked
// using Pair = (int, int);  // this was a compile error
```

**When to use:** Complex tuple types that appear in multiple method signatures. Generic
specializations with long namespaces. Gives tuples a domain-meaningful name without
creating a full record/struct.

### Default Lambda Parameters

Lambda expressions can now have default parameter values, matching the behavior of
methods and local functions.

```csharp
// ✅ C# 12 — default values in lambdas
var greet = (string name, string greeting = "Hello") => $"{greeting}, {name}!";
Console.WriteLine(greet("Alice"));           // "Hello, Alice!"
Console.WriteLine(greet("Bob", "Hey"));      // "Hey, Bob!"

// ✅ Useful for configuring pipeline behaviors
Func<int, int, int> add = (a, b = 0) => a + b;
```

```csharp
// ❌ Before C# 12 — lambdas couldn't have defaults
// var greet = (string name, string greeting = "Hello") => ...;  // compile error
```

**When to use:** Lambdas passed to higher-order functions where a sensible default
reduces caller verbosity. Avoid overusing — if the lambda is complex enough to need
defaults, consider a named method instead.

### `ref readonly` Parameters

`ref readonly` provides clearer API semantics than `in` for parameters that require a
variable (not a value) but won't modify it.

```csharp
// ✅ C# 12 — ref readonly is explicit about intent
public static double DistanceBetween(ref readonly Point a, ref readonly Point b)
    => Math.Sqrt(Math.Pow(b.X - a.X, 2) + Math.Pow(b.Y - a.Y, 2));

// Caller must pass a variable (not a temporary), making intent clear
var p1 = new Point(0, 0);
var p2 = new Point(3, 4);
var dist = DistanceBetween(ref p1, ref p2);
```

**When to use:** Interop scenarios and low-level APIs where `in` is too permissive
(accepts temporaries) and `ref` implies mutation. Most application code should prefer
`in` for readonly pass-by-reference.

### Experimental Attribute

Mark types, methods, or assemblies as experimental to warn consumers. The compiler
issues a diagnostic when experimental APIs are used.

```csharp
// ✅ C# 12 — mark preview APIs
[Experimental("MYLIB001")]
public static class PreviewFeatures
{
    public static void NewAlgorithm() { /* ... */ }
}

// Consumer gets: warning MYLIB001: 'PreviewFeatures' is for evaluation purposes only
PreviewFeatures.NewAlgorithm();
```

**When to use:** Library development when shipping preview/unstable APIs alongside
stable ones. Consumers can suppress the specific diagnostic ID to opt in.

---

## C# 13 Features (.NET 9)

Adopt these features where they improve clarity, performance, and expressiveness.

### Prerequisites — Validate Before Applying

C# 13 features require the **.NET 9 SDK** and a project targeting `net9.0` or explicitly
setting `<LangVersion>13</LangVersion>`.

```xml
<!-- ✅ Targeting .NET 9 — C# 13 is the default language version -->
<PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
</PropertyGroup>
```

**Before applying any C# 13 feature:**
1. Check `<TargetFramework>` in `.csproj` — must be `net9.0` or higher
2. OR check `<LangVersion>` — must be `13` or `preview` or `latest` (when SDK is 9+)
3. Check `Directory.Build.props` if the project uses centralized build settings
4. If neither condition is met, **do not use C# 13 features** — suggest upgrading first

### `params` Collections — Not Just Arrays

`params` now works with any collection type: `Span<T>`, `ReadOnlySpan<T>`, `IEnumerable<T>`,
`IReadOnlyList<T>`, and any type with an `Add` method. Prefer `params ReadOnlySpan<T>`
for zero-allocation variadic methods.

```csharp
// ✅ C# 13 — params with span (stack-allocated, no array allocation)
public void Log(params ReadOnlySpan<string> messages)
{
    foreach (var message in messages)
        Console.WriteLine(message);
}

// ✅ params with any collection interface
public void AddTags(params IReadOnlyList<string> tags) { /* ... */ }

// Usage — no explicit array creation needed
Log("Starting", "Processing", "Done");
```

```csharp
// ❌ Before C# 13 — params only worked with arrays (heap allocation)
public void Log(params string[] messages) { /* ... */ }
```

**When to use:** Prefer `params ReadOnlySpan<T>` on hot paths to eliminate array
allocations. Use `params` with collection interfaces when flexibility matters more
than allocation.

### New `Lock` Type — Modern Thread Synchronization

The `System.Threading.Lock` type replaces `object` for `lock` statements with better
performance and clearer intent. The compiler recognizes `Lock` and uses its optimized
`EnterScope()` API instead of `Monitor`.

```csharp
// ✅ C# 13 — dedicated Lock type
public sealed class OrderCache
{
    private readonly Lock _lock = new();
    private readonly Dictionary<Guid, Order> _cache = [];

    public void Add(Order order)
    {
        lock (_lock)  // compiler uses Lock.EnterScope(), not Monitor.Enter()
        {
            _cache[order.Id] = order;
        }
    }
}
```

```csharp
// ❌ Before C# 13 — locking on object
private readonly object _lock = new();
lock (_lock)  // uses Monitor.Enter/Exit
{
    // ...
}
```

**When to use:** All new `lock` usage should use `System.Threading.Lock`. When modernizing,
replace `private readonly object _lock = new()` with `private readonly Lock _lock = new()`.

### `\e` Escape Sequence

Use `\e` for the ESCAPE character (`U+001B`) instead of `\u001b` or the error-prone `\x1b`.

```csharp
// ✅ C# 13 — clear and safe
var reset = "\e[0m";
var bold = "\e[1m";

// ❌ Before — \x1b is dangerous if followed by hex digits
var reset = "\x1b[0m";   // works, but fragile
var reset = "\u001b[0m";  // works, but verbose
```

**When to use:** Any ANSI escape sequence output (terminal colors, cursor control).
Always prefer `\e` over `\x1b` or `\u001b`.

### Implicit Index in Object Initializers

The `^` (from-the-end) index operator now works in object initializer expressions.

```csharp
// ✅ C# 13 — index from the end in initializers
var countdown = new TimerState
{
    Buffer =
    {
        [^1] = 0,
        [^2] = 1,
        [^3] = 2,
    }
};
```

**When to use:** When populating arrays/collections relative to the end in object
initializers. Improves clarity for reverse-indexed initialization.

### `ref` Locals in Async and Iterator Methods

`async` methods and iterator methods (`yield return`) can now declare `ref` locals and
use `ref struct` types like `Span<T>` — as long as they don't cross `await` or `yield`
boundaries.

```csharp
// ✅ C# 13 — Span<T> in async method (before the await)
public async Task ProcessAsync(byte[] data, CancellationToken ct)
{
    // Span usage is fine before an await
    Span<byte> header = data.AsSpan(0, 4);
    int version = BitConverter.ToInt32(header);

    // After processing the span, await as normal
    await SaveVersionAsync(version, ct);
}
```

```csharp
// ❌ Still illegal — span cannot live across an await
public async Task ProcessAsync(byte[] data, CancellationToken ct)
{
    Span<byte> header = data.AsSpan(0, 4);
    await Task.Delay(1, ct);  // COMPILE ERROR: header spans the await
    int version = BitConverter.ToInt32(header);
}
```

**When to use:** When you need zero-allocation span processing in methods that also
happen to be async or iterators. Keep span usage and `await`/`yield` in separate scopes.

### `allows ref struct` — Generic Anti-Constraint

Generic type parameters can now accept `ref struct` types with the `allows ref struct`
anti-constraint. This unlocks `Span<T>` and `ReadOnlySpan<T>` in generic algorithms.

```csharp
// ✅ C# 13 — generic method that accepts Span<T>
public static T FirstOrDefault<T>(ReadOnlySpan<T> span, T fallback)
    where T : allows ref struct
{
    return span.Length > 0 ? span[0] : fallback;
}

// ✅ Generic type that works with ref structs
public ref struct SpanWrapper<T> where T : allows ref struct
{
    public T Value;
}
```

**When to use:** Library code and utility methods that need to work with both regular
types and `ref struct` types. Essential for writing generic span-based algorithms.

### `ref struct` Interface Implementation

`ref struct` types can now implement interfaces, though they cannot be boxed to the
interface type. Access interface members through constrained generic parameters.

```csharp
// ✅ C# 13 — ref struct implementing an interface
public interface IBufferWriter
{
    void Write(ReadOnlySpan<byte> data);
}

public ref struct StackBufferWriter : IBufferWriter
{
    private Span<byte> _buffer;
    private int _position;

    public void Write(ReadOnlySpan<byte> data)
    {
        data.CopyTo(_buffer[_position..]);
        _position += data.Length;
    }
}

// ✅ Access through constrained generic — no boxing
public static void WriteAll<T>(ref T writer, ReadOnlySpan<byte> data)
    where T : IBufferWriter, allows ref struct
{
    writer.Write(data);
}
```

**When to use:** High-performance scenarios where `ref struct` types need to participate
in polymorphic APIs without boxing.

### Partial Properties and Indexers

Properties and indexers can now be `partial`, following the same pattern as partial methods.
One declaration defines the signature, another provides the implementation.

```csharp
// ✅ C# 13 — partial property (useful with source generators)
public partial class UserEntity
{
    // Declaring declaration
    public partial string DisplayName { get; set; }
}

public partial class UserEntity
{
    // Implementing declaration
    private string _displayName = string.Empty;
    public partial string DisplayName
    {
        get => _displayName;
        set => _displayName = value.Trim();
    }
}
```

**When to use:** Source generator scenarios where generated code declares the shape
and hand-written code provides the implementation. Also useful for splitting large
partial classes across files.

### Overload Resolution Priority

Library authors can use `[OverloadResolutionPriority]` to guide the compiler to prefer
a newer, more performant overload without breaking existing callers.

```csharp
// ✅ C# 13 — prefer the span overload when both match
public static class StringHelper
{
    // Legacy overload — still available for binary compatibility
    public static bool Contains(string[] items, string value)
        => items.Contains(value);

    // New overload — preferred by compiler
    [OverloadResolutionPriority(1)]
    public static bool Contains(ReadOnlySpan<string> items, string value)
        => items.Contains(value);
}
```

**When to use:** Library development when adding higher-performance overloads to existing
APIs. Application code should rarely need this attribute.

---

## C# 14 Features (.NET 10)

Adopt these features where they improve clarity and reduce boilerplate.

### Prerequisites — Validate Before Applying

C# 14 features require the **.NET 10 SDK** and a project targeting `net10.0` or explicitly
setting `<LangVersion>14</LangVersion>`. **Always verify before using C# 14 syntax.**

```xml
<!-- ✅ Targeting .NET 10 — C# 14 is the default language version -->
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
</PropertyGroup>

<!-- ✅ Explicit LangVersion for multi-targeting or older TFMs -->
<PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <LangVersion>14</LangVersion>
</PropertyGroup>
```

**Before applying any C# 14 feature:**
1. Check `<TargetFramework>` in `.csproj` — must be `net10.0` or higher
2. OR check `<LangVersion>` — must be `14` or `preview` or `latest` (when SDK is 10+)
3. Check `Directory.Build.props` if the project uses centralized build settings
4. If neither condition is met, **do not use C# 14 features** — suggest upgrading first

### Extension Members — Properties, Statics, and Operators

C# 14 replaces the old `static class` + `this` extension method syntax with structured `extension` blocks.
Extension blocks support properties, static members, and user-defined operators — not just methods.

```csharp
// ✅ C# 14 — extension block with instance members
public static class EnumerableExtensions
{
    extension<TSource>(IEnumerable<TSource> source)
    {
        // Extension property — no more .Count() == 0 vs .Any() debates
        public bool IsEmpty => !source.Any();

        // Extension method
        public IEnumerable<TSource> WhereNotNull()
            => source.Where(item => item is not null);
    }
}

// Usage reads naturally:
if (orders.IsEmpty) return NoContent();
var validItems = items.WhereNotNull();
```

```csharp
// ✅ Static extensions and operators
public static class SpanExtensions
{
    extension<T>(Span<T>)
    {
        // Static extension method — called as Span<int>.CreateEmpty()
        public static Span<T> CreateEmpty() => Span<T>.Empty;
    }
}

public static class MoneyExtensions
{
    extension(Money)
    {
        // User-defined operator — enables Money + Money syntax
        public static Money operator +(Money left, Money right)
        {
            if (left.Currency != right.Currency)
                throw new InvalidOperationException("Cannot add different currencies");
            return new Money(left.Amount + right.Amount, left.Currency);
        }
    }
}
```

```csharp
// ❌ Old-style extension method — still works, but prefer extension blocks for new code
public static class EnumerableExtensions
{
    public static bool IsEmpty<T>(this IEnumerable<T> source) => !source.Any();
}
```

**When to use:** Prefer extension blocks for all new extension code. Extension properties
eliminate awkward `GetX()` / `HasX()` method naming for simple derived values.

### `field` Backed Properties

The `field` keyword in property accessors eliminates manual backing fields for properties
that need custom logic in one accessor.

```csharp
// ✅ C# 14 — field keyword, no manual backing field
public sealed class User
{
    public string Email
    {
        get;
        set => field = value?.Trim().ToLowerInvariant()
            ?? throw new ArgumentNullException(nameof(value));
    }

    public DateTime? LastLoginAt
    {
        get;
        set => field = value > DateTime.UtcNow
            ? throw new ArgumentException("Cannot set future login time")
            : value;
    }
}
```

```csharp
// ❌ Before C# 14 — verbose manual backing field
public sealed class User
{
    private string _email = string.Empty;
    public string Email
    {
        get => _email;
        set => _email = value?.Trim().ToLowerInvariant()
            ?? throw new ArgumentNullException(nameof(value));
    }
}
```

**When to use:** Any property where one accessor needs custom logic but the other is
a simple get/set. Eliminates boilerplate backing fields.

**Disambiguation:** If you have a symbol named `field` in the same scope, use `@field`
or `this.field` to reference the identifier instead of the keyword.

### Null-Conditional Assignment

The `?.` and `?[]` operators now work on the left side of assignments. The right side
is only evaluated when the left side is not null.

```csharp
// ✅ C# 14 — concise null-safe assignment
customer?.Order = GetCurrentOrder();
customer?.Address?.City = "Seattle";
config?.Settings?["timeout"] = "30";

// Also works with compound assignment
customer?.LoyaltyPoints += 100;
order?.Items?.Count -= returnedCount;
```

```csharp
// ❌ Before C# 14 — verbose null checks
if (customer is not null)
{
    customer.Order = GetCurrentOrder();
}

if (customer?.Address is not null)
{
    customer.Address.City = "Seattle";
}
```

**When to use:** Whenever you null-check before assigning a property. Keeps code flat.
Note: `++` and `--` are **not** supported with null-conditional assignment.

### Simple Lambda Parameters with Modifiers

Lambda parameters can now have modifiers (`ref`, `out`, `in`, `scoped`, `ref readonly`)
without specifying the full type.

```csharp
// ✅ C# 14 — modifiers without explicit types
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);

Span<int> numbers = [3, 1, 4, 1, 5];
numbers.Sort((ref x, ref y) => x.CompareTo(y));
```

```csharp
// ❌ Before C# 14 — required full type declarations with modifiers
TryParse<int> parse = (string text, out int result) => int.TryParse(text, out result);
```

**When to use:** Whenever lambda parameter modifiers are needed. Reduces noise while
preserving the intent conveyed by the modifier.

### Implicit Span Conversions

C# 14 adds first-class implicit conversions between `T[]`, `Span<T>`, and `ReadOnlySpan<T>`.
This makes span-based APIs natural to call without explicit casts.

```csharp
// ✅ C# 14 — arrays implicitly convert to spans
void ProcessItems(ReadOnlySpan<int> items) { /* ... */ }

int[] numbers = [1, 2, 3, 4, 5];
ProcessItems(numbers);  // implicit conversion, no .AsSpan() needed

// Span<T> implicitly converts to ReadOnlySpan<T>
void ReadItems(ReadOnlySpan<byte> data) { /* ... */ }
Span<byte> buffer = stackalloc byte[256];
ReadItems(buffer);  // implicit conversion
```

```csharp
// ❌ Before C# 14 — explicit conversion required
ProcessItems(numbers.AsSpan());
ReadItems((ReadOnlySpan<byte>)buffer);
```

**When to use:** Design APIs accepting `ReadOnlySpan<T>` for read-only access and
`Span<T>` for mutation. Callers with arrays can now pass them directly. This pairs
well with the performance patterns section — prefer span-based APIs on hot paths.

### `nameof` with Unbound Generic Types

`nameof` now accepts unbound generic types, which is useful for logging, diagnostics,
and error messages referencing generic type names.

```csharp
// ✅ C# 14 — unbound generic type
var name = nameof(List<>);       // "List"
var name2 = nameof(Dictionary<,>); // "Dictionary"

// Useful in exception messages and logging
throw new InvalidOperationException(
    $"No handler registered for {nameof(IRequestHandler<,>)}");
```

```csharp
// ❌ Before C# 14 — required a closed generic
var name = nameof(List<int>);  // "List" — had to pick an arbitrary type argument
```

### Partial Constructors and Events

`partial` now extends to instance constructors and events, complementing the existing
partial methods and properties from earlier versions.

```csharp
// ✅ Partial constructor — defining declaration
public partial class OrderEntity
{
    public partial OrderEntity(string orderId);
}

// ✅ Partial constructor — implementing declaration (separate file)
public partial class OrderEntity
{
    public partial OrderEntity(string orderId)
    {
        OrderId = orderId ?? throw new ArgumentNullException(nameof(orderId));
        CreatedAt = DateTime.UtcNow;
    }
}
```

**When to use:** Source generators and code-gen scenarios where generated code declares
the shape and hand-written code provides the implementation. Only the implementing
declaration can include `this()` or `base()` constructor initializers.

### User-Defined Compound Assignment Operators

Types can now define compound assignment operators (`+=`, `-=`, etc.) directly, instead
of relying on the compiler to synthesize them from the corresponding binary operator.

```csharp
// ✅ C# 14 — direct compound assignment for mutable types
public struct Counter
{
    public int Value;

    public static Counter operator +(Counter a, Counter b)
        => new() { Value = a.Value + b.Value };

    // Compound assignment can mutate in place — avoids copy
    public static Counter operator +=(Counter a, Counter b)
    {
        a.Value += b.Value;
        return a;
    }
}
```

**When to use:** Performance-sensitive mutable value types where the default `a = a + b`
copy is wasteful. For immutable records, the compiler-synthesized compound assignment
from the binary operator is sufficient.

---

## EF Core Query Patterns

### Always Project Queries
```csharp
// ✅ Project — only loads needed columns
var users = await db.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Select(u => new UserListItem(u.Id, u.Email, u.DisplayName))
    .ToListAsync(ct);

// ❌ Load entire entity then map
var users = await db.Users.ToListAsync(ct);
var dtos = users.Select(u => new UserListItem(u.Id, u.Email, u.DisplayName)).ToList();
```

### Avoid N+1 Queries
```csharp
// ✅ Eager load related data
var orders = await db.Orders
    .Include(o => o.Items)
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(ct);

// ✅ Even better — project to avoid loading navigation properties
var orders = await db.Orders
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderResponse(
        o.Id,
        o.Items.Select(i => new OrderItemResponse(i.ProductName, i.Quantity)).ToList()
    ))
    .ToListAsync(ct);
```

### Read vs Write Contexts
```csharp
// Queries — always AsNoTracking
var user = await db.Users.AsNoTracking().SingleAsync(u => u.Id == id, ct);

// Commands — tracking enabled (default) for change detection
var user = await db.Users.SingleAsync(u => u.Id == id, ct);
user.DisplayName = command.NewDisplayName;
await db.SaveChangesAsync(ct);
```
