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
| `references/architecture.md` | Vertical Slice Architecture, CQRS, Minimal APIs, pipeline behaviors, project structure |
| `references/csharp-idioms.md` | Sealed classes, records, pattern matching, primary constructors, value objects, C# 12/13/14 features |
| `references/code-quality.md` | Naming, formatting, complexity control, anti-patterns, clean code practices |
| `references/review-checklist.md` | Systematic code review checklist with severity levels |
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

## C# Version Coverage

The skill includes detailed guidance and prerequisite validation for:

- **C# 12 (.NET 8)** - Collection expressions, primary constructors, alias any type, default lambda parameters
- **C# 13 (.NET 9)** - `params` collections, `System.Threading.Lock`, `allows ref struct`, partial properties
- **C# 14 (.NET 10)** - Extension members, `field` keyword, null-conditional assignment, implicit span conversions

Each version section includes prerequisite checks to validate `TargetFramework` and `LangVersion` before applying syntax.

## Anti-Patterns Detected

The skill actively flags and provides fixes for:
- Repository pattern over EF Core
- Implicit operators for DTO/entity conversion
- `throw ex` (stack trace destruction)
- Swallowed exceptions
- `new HttpClient()` (socket exhaustion)
- Forgotten `await` / `async void`
- Anemic domain models
- Singleton abuse with mutable state
- God controllers and service pass-throughs

## Installation

### As a Claude Code Skill

Clone this repository into your Claude Code skills directory:

```bash
git clone git@github.com:ChaseDRedmon/claude-skills.git ~/.claude/skills/csharp-aesthetic-architecture
```

The skill activates automatically when writing, reviewing, or refactoring C# code.

### Manual Trigger

The skill triggers when you mention: Vertical Slice Architecture, CQRS, clean code, code review, architecture review, sealed classes, records, immutability, EF Core patterns, minimal APIs, or when you ask to "clean up", "refactor", "review", or "modernize" any C# codebase.

## License

[MIT](LICENSE.md)
