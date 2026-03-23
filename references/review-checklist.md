# Code Review & Architecture Audit Checklist

Systematic checklist for reviewing C# code against aesthetic architecture principles.
Walk through each section in order. Report findings grouped by severity.

## Severity Levels
- **CRITICAL** — Architectural violation, security risk, or correctness issue. Must fix.
- **MAJOR** — Significant code smell, performance risk, or maintainability concern. Should fix.
- **MINOR** — Style inconsistency, naming improvement, or minor optimization. Nice to fix.

---

## 1. Architecture & Structure

### Project Organization
- [ ] Features organized by use case, not by layer (no `Controllers/`, `Services/`, `Repositories/` folders)
- [ ] Each feature has its own folder under `Features/`
- [ ] Feature slices contain: Endpoint, Command/Query, Handler, Validator (as needed)
- [ ] No feature directly references another feature's internal types
- [ ] Shared code lives in `Shared/` with clear contracts
- [ ] Infrastructure concerns isolated in `Infrastructure/`

### CQRS Compliance
- [ ] Commands (writes) separated from Queries (reads)
- [ ] Commands return minimal data (ID, success indicator, or Result type)
- [ ] Queries use `AsNoTracking()` and projection (`Select`)
- [ ] Handlers are focused on a single use case
- [ ] No handler calls another handler directly

### Anti-Pattern Detection
- [ ] **No Repository pattern over EF Core** — DbContext used directly in handlers
- [ ] **No service pass-throughs** — no service that only delegates to another layer
- [ ] **No premature abstractions** — no `IXService` with a single `XService` implementation
- [ ] **No god controllers** — no controller with more than 2-3 actions
- [ ] **No shared DTOs** coupling multiple features together
- [ ] **No AutoMapper** — use manual mapping or `Select` projections
- [ ] **No MediatR abuse** — if using MediatR, handlers are in feature folders, not a separate layer
- [ ] **No implicit operators for DTO/entity conversion** — use explicit `ToEntity()` / `ToResponse()` methods or manual mapping in handlers. Implicit operators are only acceptable for value object primitive unwrapping (`EmailAddress` → `string`)

### Dependency Injection
- [ ] Services registered explicitly, not via assembly scanning
- [ ] Interfaces created only for external dependencies needing test fakes
- [ ] No interface for DbContext
- [ ] Correct lifetimes: Scoped for handlers/DbContext, Singleton for stateless services

---

## 2. C# Language Idioms

### Sealing
- [ ] All classes sealed unless explicitly designed for inheritance
- [ ] All records sealed
- [ ] Unsealed classes have a documented reason (comment or base class design)

### Immutability
- [ ] Commands and queries are `sealed record` types
- [ ] Response DTOs use `init` properties or positional records
- [ ] Entity properties use `init` where the value shouldn't change after creation
- [ ] Collection properties expose `IReadOnlyList<T>` or `IReadOnlyCollection<T>` publicly
- [ ] No mutable DTOs with public setters

### Modern Syntax
- [ ] File-scoped namespaces used everywhere (never block-scoped)
- [ ] Primary constructors for DI where possible (C# 12+)
- [ ] Pattern matching for null checks (`is null` / `is not null`)
- [ ] Switch expressions over switch statements where returning a value
- [ ] Collection expressions (`[]`) where appropriate (C# 12+)

### C# 12 Features (requires `net8.0` or `<LangVersion>12</LangVersion>`)
- [ ] `TargetFramework` or `LangVersion` verified before using C# 12 syntax
- [ ] Collection expressions (`[]`) used instead of `new[]`, `new List<T>{}`, `Array.Empty<T>()`
- [ ] Spread operator (`..`) used to flatten collections instead of `.Concat().ToArray()`
- [ ] Primary constructors used for DI on handler/service classes
- [ ] `using` alias directives used for complex tuple types and long generic specializations
- [ ] Default lambda parameters used instead of overloads or null-coalescing in lambdas
- [ ] `[Experimental]` attribute applied to preview/unstable public APIs in libraries

### C# 13 Features (requires `net9.0` or `<LangVersion>13</LangVersion>`)
- [ ] `TargetFramework` or `LangVersion` verified before using C# 13 syntax
- [ ] `params ReadOnlySpan<T>` used instead of `params T[]` on hot-path variadic methods
- [ ] `System.Threading.Lock` used instead of `object` for new `lock` fields
- [ ] `\e` used instead of `\u001b` or `\x1b` for ESCAPE character literals
- [ ] `allows ref struct` constraint used for generic methods accepting `Span<T>` / `ReadOnlySpan<T>`
- [ ] Partial properties used for source-generator scenarios (declaring + implementing split)
- [ ] `ref` locals in async methods don't cross `await` boundaries

### C# 14 Features (requires `net10.0` or `<LangVersion>14</LangVersion>`)
- [ ] `TargetFramework` or `LangVersion` verified before using C# 14 syntax
- [ ] Extension blocks used instead of old-style `this` extension methods for new code
- [ ] Extension properties used instead of `GetX()` methods for simple derived values
- [ ] `field` keyword used to eliminate manual backing fields in property accessors
- [ ] Null-conditional assignment (`?.` on left side) used instead of `if (x is not null)` before assign
- [ ] Lambda parameter modifiers (`out`, `ref`, `in`) used without redundant type declarations
- [ ] Implicit span conversions leveraged — no unnecessary `.AsSpan()` calls
- [ ] `nameof(List<>)` used instead of `nameof(List<int>)` for unbound generic references

### Type Design
- [ ] Value objects used instead of primitive types for domain concepts
  (e.g., `EmailAddress` instead of `string`, `Money` instead of `decimal`)
- [ ] Enums used for finite sets of values
- [ ] No `string` comparisons for status/role/type — use enums or constants

---

## 3. Naming & Readability

### Method Names
- [ ] All methods use Verb + Noun pattern describing the action
- [ ] No methods named `Process()`, `Handle()` (without context), `DoWork()`, `Run()`
- [ ] Async methods end with `Async` suffix (or follow team convention consistently)

### Variable Names
- [ ] All variables describe their content, not their type
- [ ] Boolean variables read as questions: `isActive`, `hasPermission`, `canEdit`
- [ ] No single-letter variables outside of lambdas and loop indices
- [ ] No abbreviations unless universally understood (`id`, `db`, `ct` for CancellationToken)

### Class Names
- [ ] No `Helper`, `Utils`, `Manager`, `Processor` suffixes without strong justification
- [ ] Class name matches the file name
- [ ] Handler classes named `Handler` or `{Action}Handler`

### Comments
- [ ] No explanatory comments — code is self-documenting
- [ ] No commented-out code
- [ ] No section-header comments (indicates method does too much)
- [ ] XML documentation present on public API methods
- [ ] `// TODO:` items reference a ticket or work item

---

## 4. Complexity Control

### Indentation Depth
- [ ] No method exceeds 3 levels of indentation
- [ ] Guard clauses used to flatten nested conditionals
- [ ] Deeply nested logic extracted to separate methods

### Method Size
- [ ] Methods are 5–20 lines (with exceptions for complex domain logic)
- [ ] No method over 40 lines
- [ ] Methods with 3+ sequential responsibilities are decomposed

### Method Parameters
- [ ] No method has more than 4 parameters
- [ ] Methods with many parameters use a command/request record instead
- [ ] `CancellationToken` is always the last parameter

### File Size
- [ ] Files are 50–200 lines
- [ ] No file over 400 lines
- [ ] One public type per file

---

## 5. Magic String / Number Elimination

- [ ] No string literals used in comparisons (extracted to `const` in named class)
- [ ] No numeric literals used in business logic (extracted to named constants)
- [ ] Constants organized in `internal static class` containers
- [ ] Constants placed near usage or in `Shared/Constants/` if cross-cutting
- [ ] `const` used for compile-time constants; `static readonly` for runtime constants

---

## 6. EF Core Patterns

### Query Patterns
- [ ] Read queries use `AsNoTracking()`
- [ ] Queries use projection (`Select`) to load only needed columns
- [ ] No N+1 queries — `Include()` or projected sub-selects used
- [ ] No `ToList()` followed by LINQ — filtering done in SQL via `Where()`

### Command Patterns
- [ ] Write operations use tracked entities (no `AsNoTracking` on writes)
- [ ] `SaveChangesAsync()` called with `CancellationToken`
- [ ] No manual transaction management unless explicitly needed

### Anti-Patterns
- [ ] No `IRepository<T>` wrapping DbContext
- [ ] No `IUnitOfWork` wrapping `SaveChangesAsync`
- [ ] No raw SQL unless performance-justified and documented

---

## 7. Async & Performance

### Async Discipline
- [ ] No `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` on async calls
- [ ] `CancellationToken` passed through all async call chains
- [ ] `async void` not used (except event handlers)
- [ ] No forgotten `await` — all `Task`/`ValueTask` returns are awaited (CS4014 treated as error)

### Exception Handling Discipline
- [ ] `throw;` used instead of `throw ex;` to preserve stack traces
- [ ] No empty catch blocks (swallowed exceptions)
- [ ] No manually thrown `NullReferenceException`, `StackOverflowException`, or `IndexOutOfRangeException`
- [ ] No `throw new Exception(...)` — use specific exception types (`InvalidOperationException`, `ArgumentException`, etc.)

### Resource Management
- [ ] No `new HttpClient()` — `IHttpClientFactory` or typed clients via DI used instead
- [ ] No manual singleton patterns with mutable state — use DI lifetime scoping

### Allocation Awareness
- [ ] No string concatenation in loops (use `StringBuilder`)
- [ ] No LINQ in performance-critical tight loops (use `foreach` + `if`)
- [ ] `ValueTask<T>` considered for hot paths with frequent synchronous completion
- [ ] `readonly struct` or `record struct` used for small, stack-allocated types

### Logging
- [ ] Structured logging used: `logger.LogInformation("User created {UserId}", id)`
- [ ] No string interpolation in log calls: `logger.Log($"User {id}")` ← WRONG
- [ ] Log levels appropriate: Information for business events, Warning for recoverable issues, Error for failures

---

## 8. Cross-Cutting Concerns

### Validation
- [ ] Validation handled via pipeline behavior or FluentValidation
- [ ] Validation logic not duplicated in handlers
- [ ] Validators co-located with the feature they validate

### Error Handling
- [ ] Exceptions not used for control flow
- [ ] Result/OneOf types used for expected failure cases
- [ ] Global exception handler configured for unexpected exceptions
- [ ] Domain exceptions are specific (`OrderNotFoundException`, not `Exception`)

### Authorization
- [ ] Authorization checks in pipeline behavior or endpoint middleware
- [ ] No authorization logic inside handlers

---

## 9. Testing (if reviewing tests)

- [ ] Tests verify behavior, not implementation details
- [ ] Test names describe the scenario: `CreateUser_WithDuplicateEmail_ReturnsError`
- [ ] No brittle mocks — prefer integration tests with real database (TestContainers)
- [ ] Each test is independent — no shared mutable state
- [ ] Tests organized to mirror feature structure

---

## 10. Static Analysis & Tooling

- [ ] `.editorconfig` present with file-scoped namespace enforcement
- [ ] Roslyn analyzers configured (Roslynator, StyleCop, or equivalent)
- [ ] Architecture tests enforce feature boundary isolation
- [ ] CI pipeline enforces all warnings-as-errors

---

## Reporting Format

When reporting review findings, use this structure:

```
## Review Summary
- Critical: X findings
- Major: Y findings  
- Minor: Z findings

## Critical Findings
### [C1] Repository pattern wrapping EF Core
**File:** Services/UserRepository.cs
**Issue:** UserRepository wraps DbContext methods without adding value.
**Fix:** Remove UserRepository. Inject AppDbContext directly into the handler.

## Major Findings
### [M1] Deep nesting in OrderProcessor.Process()
**File:** Features/Orders/OrderProcessor.cs, lines 45-89
**Issue:** 5 levels of indentation. Method handles validation, calculation, persistence, and notification.
**Fix:** Decompose into ValidateOrder(), CalculateTotal(), PersistOrder(), NotifyCustomer().

## Minor Findings
### [m1] Missing sealed modifier on UserValidator
**File:** Features/Users/CreateUser/Validator.cs
**Issue:** Class is not sealed.
**Fix:** Add `sealed` modifier.
```
