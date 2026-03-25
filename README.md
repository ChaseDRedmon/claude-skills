# Claude Skills

A collection of [Claude Code](https://claude.com/claude-code) skills packaged as plugins.

## Skills

| Skill | Description |
|---|---|
| [`csharp-aesthetic-architecture`](skills/csharp-aesthetic-architecture/) | Modern C# architecture with Vertical Slice Architecture, CQRS, C# 12-14 features, anti-pattern detection, and a 100+ item review checklist |

## Installation

Install as a Claude Code plugin:

```bash
claude plugins add git@github.com:ChaseDRedmon/claude-skills.git
```

Or clone and symlink manually:

```bash
git clone git@github.com:ChaseDRedmon/claude-skills.git ~/.claude/skills/claude-skills
```

## Structure

This repo follows the Claude Code plugin convention:

```
claude-skills/
├── plugin.json                         # plugin manifest
├── skills/
│   └── csharp-aesthetic-architecture/
│       ├── SKILL.md                    # skill entry point
│       ├── README.md                   # skill documentation
│       └── references/                 # detailed reference material
└── LICENSE.md
```

## License

[MIT](LICENSE.md)
