# C# Aesthetic Architecture

A [Claude Code](https://claude.com/claude-code) plugin for building, reviewing, and refactoring C# / .NET codebases using modern aesthetic architecture principles. Combines Vertical Slice Architecture, CQRS, functional design, modern C# idioms (C# 12-14), and clean code practices into a unified development philosophy.

## Installation

```bash
git clone git@github.com:ChaseDRedmon/csharp-aesthetic-architecture.git ~/.claude/skills/csharp-aesthetic-architecture
```

## Philosophy

Everything derives from one principle: **Optimize for human comprehension, not theoretical purity.**

- Prefer **clarity** over cleverness
- Prefer **duplication** over coupling (WET > DRY when DRY creates cross-slice dependencies)
- Prefer **composition** over inheritance
- Prefer **immutability** by default
- Prefer **feature-based** organization over layer-based
- Prefer **measuring** over guessing (no premature optimization)
- Prefer **iteration** over recursion (C# has no guaranteed TCO)

## What's Included

| File | Topics |
|---|---|
| `references/architecture.md` | Vertical Slice Architecture, CQRS, Minimal APIs, pipeline behaviors, composition over inheritance, DI rules |
| `references/csharp-idioms.md` | Sealed classes, records, pattern matching, value objects, functional patterns, C# 12/13/14 features |
| `references/code-quality.md` | Naming, formatting, complexity control, premature optimization, anti-patterns |
| `references/review-checklist.md` | 100+ item code review checklist with severity levels |
| `references/efcore-patterns.md` | EF Core query patterns, configuration, migrations |
| `references/cross-cutting.md` | Validation, logging, metrics, authorization, error handling pipelines |

## C# Version Coverage

Each version includes prerequisite validation (`TargetFramework` / `LangVersion` checks):

- **C# 12 (.NET 8)** - Collection expressions, primary constructors, alias any type, default lambda parameters
- **C# 13 (.NET 9)** - `params` collections, `System.Threading.Lock`, `allows ref struct`, partial properties
- **C# 14 (.NET 10)** - Extension members, `field` keyword, null-conditional assignment, implicit span conversions

## Anti-Patterns Detected

**Architecture** - Repository over EF Core, implicit operator DTO conversion, anemic domain models, god controllers, premature abstractions, singleton abuse

**Async & Exceptions** - Forgotten `await`, `async void`, `throw ex`, swallowed exceptions, sync-over-async

**Naming & Style** - Hungarian notation, abbreviations, `Utils`/`Helper`/`Base` classes, magic strings

**Resources** - `new HttpClient()` socket exhaustion, recursive traversal (prefer `Stack<T>`/`Queue<T>`)

## Code Quality Guidelines

1. **Abstraction tradeoffs** - coupling is the cost of abstraction; composition over inheritance
2. **Naming** - never abbreviate, no Hungarian notation, use types for units
3. **Nesting** - max 3 levels; guard clauses and extraction to flatten
4. **Comments** - code speaks for itself; comments only for *why* (perf, algorithms, workarounds)
5. **Premature optimization** - measure before optimizing; macro moves before micro-tuning
6. **Dependency injection** - pass it in; factories for runtime decisions
7. **Functional thinking** - impure shell/pure core; LINQ pipelines; iteration over recursion

## License

[MIT](LICENSE.md)
