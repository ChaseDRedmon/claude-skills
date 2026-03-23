---
name: csharp-aesthetic-architecture
description: >
  Enforce and apply modern, aesthetic C# architecture principles across entire .NET codebases.
  Use this skill whenever writing new C# code, reviewing existing C# code, refactoring .NET projects,
  scaffolding new features or slices, or when the user mentions any of: Vertical Slice Architecture,
  CQRS, pipeline behaviors, clean code, code smells, C# idioms, sealed classes, records, immutability,
  EF Core patterns, minimal APIs, guard clauses, code review, architecture review, .NET best practices,
  aesthetic code, or modern C# style. Also trigger when the user asks to "clean up", "refactor",
  "review", or "modernize" any C# or .NET codebase. This skill applies to both greenfield development
  and brownfield refactoring of C# projects.
---

# C# Aesthetic Architecture Skill

A comprehensive skill for building, reviewing, and refactoring C# / .NET codebases using modern
aesthetic architecture principles. Combines Vertical Slice Architecture, CQRS, functional design,
modern C# idioms, and clean code practices into a unified development philosophy.

## Core Philosophy

Everything derives from one principle: **Optimize for human comprehension, not theoretical purity.**

This means: fewer abstractions, fewer layers, clear intent, predictable structure. A developer
should open any file and understand it in seconds.

Three golden characteristics of beautiful code:
1. **Predictable structure** — consistent patterns across the entire codebase
2. **Minimal abstraction** — no layers that exist only to satisfy a pattern
3. **Clear intent** — code reads like English prose describing business behavior

### The Golden Rules

- Prefer **clarity** over cleverness
- Prefer **duplication** over coupling (WET > DRY when DRY creates cross-slice dependencies)
- Prefer **composition** over inheritance
- Prefer **immutability** by default
- Prefer **feature-based** organization over layer-based
- Prefer **explicit** registration over magic/scanning
- Prefer **simple** over abstract

## When to Use This Skill

### For New Code Generation
Read the relevant reference files before writing code:
- **Architecture decisions** → Read `references/architecture.md`
- **C# language patterns** → Read `references/csharp-idioms.md`
- **Naming and formatting** → Read `references/code-quality.md`

### For Code Review / Refactoring
Read the review checklist and apply systematically:
- **Full review** → Read `references/review-checklist.md`
- **Architecture audit** → Read `references/architecture.md` then `references/review-checklist.md`
- **Code smell detection** → Read `references/code-quality.md` then `references/review-checklist.md`

---

## Quick Reference — Architecture (details in references/architecture.md)

### Feature-First Organization (Vertical Slice Architecture)

```
src/
  Features/
    Orders/
      CreateOrder/
        Endpoint.cs
        Command.cs
        Handler.cs
        Validator.cs
      GetOrder/
        Endpoint.cs
        Query.cs
        Handler.cs
    Users/
      CreateUser/
        ...
  Infrastructure/
    Database/
    Integrations/
  Shared/
    Contracts/
    Behaviors/
  Program.cs
```

**NEVER** organize by layer:
```
Controllers/    ← NO
Services/       ← NO
Repositories/   ← NO
DTOs/           ← NO
```

### Request Flow

```
HTTP Request
  → Minimal API Endpoint
    → Pipeline Behaviors (Validation, Auth, Logging, Metrics)
      → Command/Query Handler
        → EF Core DbContext
          → Database
```

### CQRS — Separate Reads and Writes

- **Commands** → mutate state (`CreateOrderCommand` / `CreateOrderHandler`)
- **Queries** → read data (`GetOrderQuery` / `GetOrderHandler`)

### Anti-Patterns to Reject

- ❌ Repository pattern over EF Core (EF already implements Repository + Unit of Work)
- ❌ Service-layer pass-throughs that only delegate to another layer
- ❌ Controller-based APIs (use Minimal APIs)
- ❌ Premature abstractions (`IUserService` / `UserService` with one implementation)
- ❌ Shared service layers that couple features together
- ❌ Implicit operators for DTO/entity conversion (use explicit `ToEntity()` / `ToResponse()` methods)

---

## Quick Reference — C# Idioms (details in references/csharp-idioms.md)

### Sealed by Default
```csharp
public sealed class CreateUserHandler  // ✅ Always sealed unless designing for extension
```

### Records for Immutable Data
```csharp
public sealed record CreateUserCommand(string Email, string Password);
```

### Init Properties for Entities
```csharp
public sealed class User
{
    public Guid Id { get; init; }
    public string Email { get; init; } = string.Empty;
}
```

### File-Scoped Namespaces
```csharp
namespace MyApp.Features.Users.CreateUser;  // ✅ Always file-scoped

public sealed class Handler { }
```

### Pattern Matching and Switch Expressions
```csharp
if (user is not null) { }

return status switch
{
    OrderStatus.Pending => "Awaiting processing",
    OrderStatus.Complete => "Delivered",
    _ => throw new InvalidOperationException()
};
```

### Value Objects Over Primitives
```csharp
// ❌ Primitive obsession
public string Email { get; init; }

// ✅ Value object
public EmailAddress Email { get; init; }
```

---

## Quick Reference — Code Quality (details in references/code-quality.md)

### Naming — Code Should Read Like English
```csharp
// ❌ Bad
var data = Process();

// ✅ Good
var pendingOrders = GetPendingOrdersForCurrentMonth();
```

### No Comments — If You Need Them, The Code Is Unclear
```csharp
// ❌
// Calculate discount
var d = price * .15;

// ✅
var discount = CalculateSeasonalDiscount(price);
```

### XML Docs Only on Public APIs
```csharp
/// <summary>
/// Creates a new user account and returns the generated identifier.
/// </summary>
public Task<Guid> Handle(CreateUserCommand command, CancellationToken ct)
```

### No Magic Strings or Numbers
```csharp
// ❌
if (role == "admin") { }

// ✅
internal static class Roles
{
    public const string Admin = "admin";
}
```

### Maximum 3 Levels of Indentation
```csharp
// ❌ Deep nesting
if (order != null)
{
    if (order.Items.Any())
    {
        if (order.Status == OrderStatus.Pending)
        {
            if (order.Total > 0) { }
        }
    }
}

// ✅ Guard clauses
if (order is null) return;
if (!order.Items.Any()) return;
if (order.Status != OrderStatus.Pending) return;
if (order.Total <= 0) return;
```

### One Method = One Responsibility
```csharp
// ❌ Method does multiple things
public async Task ProcessOrder(Order order) { /* validate, tax, save, email */ }

// ✅ Decomposed
public async Task HandleOrder(Order order, CancellationToken ct)
{
    ValidateOrder(order);
    var tax = CalculateTax(order);
    await SaveOrder(order, tax, ct);
    await SendReceipt(order, ct);
}
```

---

## Quick Reference — C# 12 Features (details in references/csharp-idioms.md)

**Prerequisite:** Verify the project targets `net8.0` or has `<LangVersion>12</LangVersion>`
before using any C# 12 syntax. Check `.csproj` and `Directory.Build.props`.

### Collection Expressions
```csharp
int[] numbers = [1, 2, 3];           // replaces new[] { 1, 2, 3 }
List<string> names = ["a", "b"];     // replaces new List<string> { "a", "b" }
int[] empty = [];                    // replaces Array.Empty<int>()
int[] merged = [.. first, .. second]; // spread operator
```

### Alias Any Type
```csharp
using Coordinate = (double Lat, double Lng);  // alias tuples for domain clarity
using JsonOpts = System.Text.Json.JsonSerializerOptions;
```

### Default Lambda Parameters
```csharp
var greet = (string name, string greeting = "Hello") => $"{greeting}, {name}!";
```

---

## Quick Reference — C# 13 Features (details in references/csharp-idioms.md)

**Prerequisite:** Verify the project targets `net9.0` or has `<LangVersion>13</LangVersion>`
before using any C# 13 syntax. Check `.csproj` and `Directory.Build.props`.

### `params` Collections — Zero-Allocation Variadic Methods
```csharp
// params now works with Span, ReadOnlySpan, and collection interfaces
public void Log(params ReadOnlySpan<string> messages) { /* stack-allocated */ }
```

### `System.Threading.Lock` — Modern Lock Type
```csharp
private readonly Lock _lock = new();  // replaces object _lock = new()
lock (_lock) { /* uses optimized Lock.EnterScope() */ }
```

### `allows ref struct` — Spans in Generics
```csharp
public static T First<T>(ReadOnlySpan<T> span) where T : allows ref struct
    => span[0];
```

### Partial Properties
```csharp
public partial class Entity { public partial string Name { get; set; } }        // declaring
public partial class Entity { public partial string Name { get => _n; set => _n = value; } } // implementing
```

---

## Quick Reference — C# 14 Features (details in references/csharp-idioms.md)

**Prerequisite:** Verify the project targets `net10.0` or has `<LangVersion>14</LangVersion>`
before using any C# 14 syntax. Check `.csproj` and `Directory.Build.props`.

### Extension Members — Properties, Statics, Operators
```csharp
public static class EnumerableExtensions
{
    extension<TSource>(IEnumerable<TSource> source)
    {
        public bool IsEmpty => !source.Any();
    }
}
```

### `field` Backed Properties
```csharp
public string Email
{
    get;
    set => field = value?.Trim().ToLowerInvariant()
        ?? throw new ArgumentNullException(nameof(value));
}
```

### Null-Conditional Assignment
```csharp
customer?.Order = GetCurrentOrder();  // RHS only evaluated if customer is not null
customer?.LoyaltyPoints += 100;      // compound assignment works too
```

### Simple Lambda Parameter Modifiers
```csharp
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

### Implicit Span Conversions
```csharp
void Process(ReadOnlySpan<int> items) { }
int[] data = [1, 2, 3];
Process(data);  // implicit conversion — no .AsSpan() needed
```

---

## Quick Reference — Performance (details in references/csharp-idioms.md)

- Use `async/await` everywhere — never `.Result` or `.Wait()`
- Use `AsNoTracking()` for read-only EF Core queries
- Use projection queries (`Select`) instead of loading full entities
- Prefer `ValueTask<T>` on hot paths
- Avoid allocations: prefer `readonly struct`, `record struct`, `Span<T>`
- Avoid LINQ in tight loops — use `foreach` with `if`/`continue`
- Avoid reflection in hot paths — use compiled expressions

---

## Canonical Handler Example

This is the gold standard for what a feature handler should look like:

```csharp
namespace MyApp.Features.Users.CreateUser;

public sealed record Command(string Email, string Password);

public sealed class Handler(AppDbContext db)
{
    public async Task<Guid> Handle(Command command, CancellationToken ct)
    {
        var user = new User
        {
            Id = Guid.NewGuid(),
            Email = command.Email
        };

        db.Users.Add(user);
        await db.SaveChangesAsync(ct);

        return user.Id;
    }
}
```

Properties: small, readable, feature-focused, minimal abstractions, single responsibility,
immutable input, sealed classes, file-scoped namespace, primary constructor DI.

---

## Reference Files

Read these for detailed guidance on specific areas:

| File | When to Read |
|------|-------------|
| `references/architecture.md` | Scaffolding new projects, adding features, architecture decisions |
| `references/csharp-idioms.md` | Writing C# code, choosing language constructs, performance tuning |
| `references/code-quality.md` | Naming, formatting, complexity control, clean code practices |
| `references/review-checklist.md` | Code review, refactoring, architecture audits, smell detection |
| `references/efcore-patterns.md` | Database access, queries, EF Core configuration |
| `references/cross-cutting.md` | Validation, logging, metrics, authorization pipelines |

---

## Applying This Skill

### When Generating New Code
1. Read relevant reference files for the task
2. Follow the canonical handler pattern as the baseline
3. Organize into vertical slices
4. Apply all C# idioms (sealed, records, file-scoped namespaces, etc.)
5. Verify against the review checklist before presenting

### When Reviewing Existing Code
1. Read `references/review-checklist.md`
2. Walk through each checklist category systematically
3. Report findings grouped by severity (critical → minor)
4. Provide concrete refactoring suggestions with code examples
5. Reference specific rules from this skill when explaining issues

### When Refactoring
1. Identify the target architecture (VSA + CQRS + Minimal APIs)
2. Read `references/architecture.md` for the migration path
3. Refactor one feature slice at a time
4. Verify each slice against the review checklist
5. Apply C# idioms progressively
