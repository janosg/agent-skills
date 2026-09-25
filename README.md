# agent-skills

[Claude Code](https://code.claude.com/docs/en/skills) skills. Each skill is one folder
holding a `SKILL.md`, the reference files it points to, and a `README.md` that shows how
to use it.

```
agent-skills/
├── type-driven-design/
│   ├── README.md
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
