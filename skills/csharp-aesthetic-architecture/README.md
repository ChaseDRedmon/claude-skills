# C# Aesthetic Architecture Skill

A Claude Code skill for building, reviewing, and refactoring C# / .NET codebases using modern aesthetic architecture principles. Combines Vertical Slice Architecture, CQRS, functional design, modern C# idioms (C# 12-14), and clean code practices into a unified development philosophy.

## Philosophy

Everything derives from one principle: **Optimize for human comprehension, not theoretical purity.**

Three golden characteristics of beautiful code:
1. **Predictable structure** - consistent patterns across the entire codebase
2. **Minimal abstraction** - no layers that exist only to satisfy a pattern
3. **Clear intent** - code reads like English prose describing business behavior

## What This Skill Covers

| Reference File | Topics |
|---|---|
| `references/architecture.md` | Vertical Slice Architecture, CQRS, Minimal APIs, pipeline behaviors, composition over inheritance, DI rules, project structure |
| `references/csharp-idioms.md` | Sealed classes, records, pattern matching, primary constructors, value objects, functional patterns, C# 12/13/14 features |
| `references/code-quality.md` | Naming, formatting, complexity control, premature optimization, anti-patterns, clean code practices |
| `references/review-checklist.md` | Systematic code review checklist with severity levels (100+ items) |
| `references/efcore-patterns.md` | EF Core query patterns, configuration, migrations |
| `references/cross-cutting.md` | Validation, logging, metrics, authorization, error handling pipelines |

## Key Principles

- Prefer **clarity** over cleverness
- Prefer **duplication** over coupling (WET > DRY when DRY creates cross-slice dependencies)
- Prefer **composition** over inheritance
- Prefer **immutability** by default
- Prefer **feature-based** organization over layer-based
- Prefer **explicit** registration over magic/scanning
- Prefer **simple** over abstract
- Prefer **measuring** over guessing (no premature optimization)
- Prefer **iteration** over recursion (C# has no guaranteed TCO)

## C# Version Coverage

The skill includes detailed guidance and prerequisite validation for:

- **C# 12 (.NET 8)** - Collection expressions, primary constructors, alias any type, default lambda parameters, `ref readonly` parameters, `[Experimental]` attribute
- **C# 13 (.NET 9)** - `params` collections, `System.Threading.Lock`, `\e` escape, `allows ref struct`, `ref struct` interfaces, partial properties, `[OverloadResolutionPriority]`
- **C# 14 (.NET 10)** - Extension members, `field` keyword, null-conditional assignment, implicit span conversions, simple lambda parameter modifiers, `nameof` with unbound generics, partial constructors

Each version section includes prerequisite checks to validate `TargetFramework` and `LangVersion` before applying syntax.

## Anti-Patterns Detected

The skill actively flags and provides fixes for:

### Architecture
- Repository pattern over EF Core
- Implicit operators for DTO/entity conversion
- Anemic domain models
- God controllers and service pass-throughs
- Premature abstractions (`IXService` with one implementation)
- Singleton abuse with mutable state

### Async & Exceptions
- Forgotten `await` / `async void`
- `throw ex` (stack trace destruction)
- Swallowed exceptions (empty catch blocks)
- Throwing `NullReferenceException` manually
- Sync-over-async (`.Result` / `.Wait()`)

### Naming & Style
- Hungarian notation (`strName`, `iCount`)
- Abbreviated variables
- `Utils` / `Helper` / `Base` / `Abstract` class names
- Magic strings and numbers

### Resource Management
- `new HttpClient()` (socket exhaustion)
- Recursive methods for tree/graph traversal (use `Stack<T>` / `Queue<T>`)

### Functional Patterns
- Impure core logic (side effects mixed with business rules)
- Mutable state where immutability is possible

## Code Quality Guidelines

The skill incorporates guidance from 7 areas of C# code quality:

1. **Abstraction tradeoffs** - coupling is the cost of abstraction; composition over inheritance
2. **Naming** - never abbreviate, no Hungarian notation, use types for units, don't name things `Utils`
3. **Nesting** - max 3 levels; use guard clauses (inversion) and extraction to flatten
4. **Comments** - code should speak for itself; comments only for *why* (perf, algorithms, workarounds)
5. **Premature optimization** - measure before optimizing; macro moves before micro-tuning
6. **Dependency injection** - pass it in, don't reach for it; factories for runtime decisions
7. **Functional thinking** - impure shell/pure core; LINQ pipelines; iteration over recursion

## Installation

Clone the repo and symlink this skill into your Claude Code skills directory:

```bash
git clone git@github.com:ChaseDRedmon/claude-skills.git
ln -s "$(pwd)/claude-skills/csharp-aesthetic-architecture" ~/.claude/skills/csharp-aesthetic-architecture
```

The skill activates automatically when writing, reviewing, or refactoring C# code.

### Manual Trigger

The skill triggers when you mention: Vertical Slice Architecture, CQRS, clean code, code review, architecture review, sealed classes, records, immutability, EF Core patterns, minimal APIs, or when you ask to "clean up", "refactor", "review", or "modernize" any C# codebase.

## License

[MIT](../LICENSE.md)
