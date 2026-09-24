# agent-skills

A [Claude Code](https://code.claude.com/docs/en/plugins.html) plugin marketplace
of skills. A *marketplace* is a git repo with a manifest that lets others install
your skills/plugins with a single command instead of copying files by hand.

## Layout

```
agent-skills/
├── .claude-plugin/
│   └── marketplace.json        # the marketplace manifest (lists all plugins)
└── plugins/
    └── example-skill/          # one plugin
        ├── .claude-plugin/
        │   └── plugin.json     # the plugin manifest
        └── skills/
            └── hello-world/
                └── SKILL.md    # one skill
```

- **Marketplace** (`marketplace.json`) — the catalog. Add an entry to its `plugins`
  array for each plugin in this repo. `metadata.pluginRoot` is `./plugins`, so a
  plugin's `source` can be just its folder name (e.g. `"example-skill"`).
- **Plugin** (`plugin.json`) — a bundle. By default Claude Code auto-discovers
  skills in the plugin's `skills/` directory; it can also carry slash commands,
  subagents, hooks, and MCP servers.
- **Skill** (`SKILL.md`) — instructions for one task, loaded on demand via
  progressive disclosure. `description` is required and should say *what* the
  skill does and *when* to use it.

## Plugins

### `strong-typing`

Migrate a Python codebase from *stringly-typed* to *strongly-typed* — which means more than
adding type hints: prefer immutable dataclasses (and enums, tagged unions) when fields are
known, keep dicts when keys are dynamic, and design types so illegal states can't be
expressed.

One user-invoked skill, `migrate`. The request sets the form of the work; the skill's canon
sets the substance:

```text
/strong-typing:migrate write an enhancement proposal for moving this library to strong typing
/strong-typing:migrate refactor the config module to strong typing
/strong-typing:migrate review this diff for over- and under-typing
```

`SKILL.md` carries the ten principles and a pattern index; the detail lives beside it in
`skills/migrate/references/`:

| File | Holds |
| --- | --- |
| `philosophy.md` | The full rationale behind the principles. |
| `patterns/` | One file per before/after pattern: `enum`, `dataclass-record`, `tagged-union`, `construction-functions`, `typestate`, `immutable-config`, `options-argument`, `strategy-object`, `boundary`. |
| `when-not-to-convert.md` | Guardrails against over-conversion: dynamic dicts, honest `Any`, legitimate mutability, composition over subclassing. |
| `migration-strategy.md` | Library vs. internal code, the drop / deprecate / keep decision, what a proposal covers, the order that keeps every step mergeable. |
| `type-checker.md` | Strict mypy/pyright config, the per-module ratchet, CI and pre-commit. |
| `documentation.md` | Types out of docstrings, per-attribute docstrings, featuring the strict path, what defeats autocomplete. |

## Using this marketplace

```bash
# Register the marketplace
/plugin marketplace add janosg/agent-skills

# Install a plugin from it
/plugin install example-skill@agent-skills

# Manage
/plugin marketplace list
/plugin marketplace update agent-skills
```

For local testing before pushing, you can point at the working copy:

```bash
/plugin marketplace add ./
```

## Adding a new skill

1. Create `plugins/<plugin-name>/.claude-plugin/plugin.json`.
2. Add skills under `plugins/<plugin-name>/skills/<skill-name>/SKILL.md`.
3. Register the plugin in `.claude-plugin/marketplace.json`.
4. Commit and push; users run `/plugin marketplace update agent-skills` to pick it up.

The `example-skill` plugin is a working template — copy it, then delete it once
you have your own.
