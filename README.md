# agent-skills

[Claude Code](https://code.claude.com/docs/en/skills) skills. Each skill is one folder
holding a `SKILL.md` plus the reference files it points to.

```
agent-skills/
├── type-driven-design/
│   ├── SKILL.md
│   └── references/
└── notes/                      # source material the skills were distilled from
```

## Installing a skill

Copy or symlink a skill folder into `~/.claude/skills/` (personal, every project) or
`<repo>/.claude/skills/` (one project):

```bash
ln -s "$(pwd)/type-driven-design" ~/.claude/skills/type-driven-design
```

## Skills

### `type-driven-design`

Take a Python codebase from *stringly typed* to *strongly typed*, which means more than
adding type hints: choose data types so that illegal states cannot be expressed. Prefer
immutable dataclasses (and enums, tagged unions) when fields are known, keep dicts when keys
are dynamic, be generous at the boundary and strict beneath it.

User-invoked. The request sets the form of the work; the skill's canon sets the substance:

```text
/type-driven-design write an enhancement proposal for moving this library to strong typing
/type-driven-design refactor the config module
/type-driven-design review this diff for over- and under-typing
```

`SKILL.md` carries the ten principles and a pattern index; the detail lives in
`type-driven-design/references/`:

| File | Holds |
| --- | --- |
| `philosophy.md` | The full rationale behind the principles. |
| `patterns/` | One file per before/after pattern: `enum`, `dataclass-record`, `tagged-union`, `construction-functions`, `typestate`, `immutable-config`, `options-argument`, `strategy-object`, `boundary`. |
| `when-not-to-convert.md` | Guardrails against over-conversion: dynamic dicts, honest `Any`, legitimate mutability, composition over subclassing. |
| `migration-strategy.md` | Library vs. internal code, the drop / deprecate / keep decision, what a proposal covers, the order that keeps every step mergeable. |
| `type-checker.md` | Strict mypy/pyright config, the per-module ratchet, CI and pre-commit. |
| `documentation.md` | Types out of docstrings, per-attribute docstrings, featuring the strict path, what defeats autocomplete. |
