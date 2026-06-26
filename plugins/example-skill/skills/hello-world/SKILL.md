---
name: hello-world
description: A minimal example skill. Replace this with a real skill — keep the description specific so Claude knows exactly when to use it (mention the task and trigger words a user would say).
---

# Hello World

This is a template skill. A skill is a folder containing this `SKILL.md` file plus
any supporting files (scripts, references, templates) the instructions point to.

## How Claude uses a skill

1. Claude reads only the `name` + `description` from the frontmatter until a task looks relevant.
2. When relevant, it loads this full file and follows the instructions below.
3. Supporting files are read on demand, so keep this file focused and link out to detail.

## Writing your own

- Make `description` concrete: state *what* the skill does and *when* to trigger it.
- Put step-by-step instructions here. Reference helper files by relative path, e.g.
  `scripts/run.sh` or `references/checklist.md`.
- Optional frontmatter you can add:
  - `allowed-tools:` — restrict which tools the skill may use (e.g. `Read`, `Bash`).
  - `disable-model-invocation: true` — make it user-invoked only (`/example-skill:hello-world`).

Replace this file with your actual instructions.
