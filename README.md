# Claude Skills

A collection of [Claude Code](https://claude.com/claude-code) skills for software development.

## Skills

| Skill | Description |
|---|---|
| [`csharp-aesthetic-architecture`](csharp-aesthetic-architecture/) | Modern C# architecture with Vertical Slice Architecture, CQRS, C# 12-14 features, anti-pattern detection, and a 100+ item review checklist |

## Installation

Clone the repo and symlink individual skills into your Claude Code skills directory:

```bash
git clone git@github.com:ChaseDRedmon/claude-skills.git

# Symlink a specific skill
ln -s "$(pwd)/claude-skills/csharp-aesthetic-architecture" ~/.claude/skills/csharp-aesthetic-architecture
```

Or clone the entire repo directly:

```bash
git clone git@github.com:ChaseDRedmon/claude-skills.git ~/.claude/skills/claude-skills
```

Each skill folder contains a `SKILL.md` that Claude Code discovers automatically.

## License

[MIT](LICENSE.md)
