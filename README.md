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
adding type hints: prefer immutable dataclasses (and enums, tagged unions, `NewType`) when
fields are known, keep dicts when keys are dynamic, and design types so illegal states can't
be expressed. All skills share one canon in `plugins/strong-typing/references/`
(`philosophy.md`, `patterns.md`, `when-not-to-convert.md`, `tooling.md`).

A four-stage pipeline plus three grouped reviews:

| Skill | Stage | Does |
| --- | --- | --- |
| `design-proposal` | plan | Audits the stringly-typed surface and writes an enhancement-proposal-style design doc. Asks up front whether breaking changes are allowed or need a deprecation period; scales between libraries (full public-API + deprecation treatment) and internal/app code. |
| `implementation-plan` | plan | Turns the proposal into small, independently-reviewable PR steps that each keep the repo green; sequences with deprecation shims and marks independent vs. sequential steps. |
| `implement` | build | Executes **one** step at a time, applying the canon, with a hard rule to stay in scope and not run ahead. |
| `review-types` | review | Were the right types chosen? Catches under-typing *and* over-typing (dynamic dicts wrongly frozen, needless `NewType`s, harmful subclassing). |
| `review-ergonomics` | review | Does it produce good IDE autocomplete, and do the docs leverage the types? |
| `review-safety` | review | Generous-in/strict-out boundary, deprecation correctness, a genuinely-clean type checker, retained runtime validation, unrepresentable illegal states. |

Typical flow: `design-proposal` → `implementation-plan` → (`implement` → `review-*`) per step.
Reviews default to the current diff so they slot in after each `implement` step, but also run
whole-repo on request.

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
