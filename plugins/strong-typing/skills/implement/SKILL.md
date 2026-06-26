---
name: implement
description: Execute ONE step of a strong-typing implementation plan — applying the dataclass/enum/union/NewType conversions for that step only, keeping the codebase green, without running ahead to other steps. Use when the user wants to carry out a planned type refactor, implement the next PR of the typing migration, or convert a specific module/component to strong typing. Applies the shared design philosophy so it isn't restated in the plan. Triggers: "implement step N", "do the next step of the typing plan", "carry out the strong typing migration", "convert this module to strongly typed".
---

# Strong Typing — Implement

Carry out **one step** of a strong-typing implementation plan and stop. This skill holds the
design philosophy and patterns so the plan doesn't have to restate them. Quality bar: a
small, correct, reviewable change that leaves the repo green.

## 0. Load the canon (this is your rulebook)

Read before editing:

- `${CLAUDE_PLUGIN_ROOT}/references/philosophy.md`
- `${CLAUDE_PLUGIN_ROOT}/references/patterns.md`
- `${CLAUDE_PLUGIN_ROOT}/references/when-not-to-convert.md`
- `${CLAUDE_PLUGIN_ROOT}/references/tooling.md`

These define *how* to type things. The plan defines *what* this step changes. Where they
seem to conflict, the plan wins on scope; the canon wins on technique — if they conflict on
substance, stop and ask.

## 1. Identify exactly one step

- Find the plan (ask the user for its path if needed) and the **single step** to implement.
  If the user didn't name one, take the next unfinished step and confirm it with them before
  editing.
- Read that step's **Scope: in**, **Scope: out**, **Depends on**, **Deprecation shims**, and
  **Verify** sections. Confirm its dependencies are done.
- Restate, in one or two sentences, what you will and will not touch. This is your contract.

## 2. Implement that step — and only that step

Apply the relevant patterns from `patterns.md`:

- frozen dataclasses for known-field structures; `Enum`/`Literal` for closed string choices;
  tagged unions + `assert_never` for variant sets; `NewType` for confusable scalars;
  construction functions over many-optional `__init__`; copy constructors for config;
  `bool | OptionsObject` for flag+options pairs; generous-in/strict-out at public boundaries.
- Respect `when-not-to-convert.md`: leave dynamic-key dicts, honest `Any`, and
  genuinely-mutable state alone. Do not "improve" them.
- For libraries under a deprecation requirement: keep old inputs working, add the
  `DeprecationWarning`, convert old→new at the boundary.
- Keep runtime validation; add `__post_init__` checks for invariants.
- Remove the mypy exemption for any module this step finishes (the ratchet).

## 3. Verify

Run exactly what the step's **Verify** section specifies, at minimum:

- the type checker (`mypy`/`pyright`) on the package — clean (no new `type: ignore`/`Any`
  smuggled in to silence it);
- the relevant tests — passing;
- imports still work.

Report the actual results. If something fails, fix within scope or stop and report — do not
expand scope to make it pass.

## 4. Stop. Do not run ahead. (Hardest rule.)

This is the most important instruction in this skill:

- **Implement only the current step.** When it's done and verified, **STOP.** Do not start
  the next step.
- **Do not expand scope.** No drive-by refactors, no "while I'm here" cleanups, no converting
  things the step didn't list, no touching files outside **Scope: in**. Tempting adjacent
  improvements are *out of scope by definition*.
- **Do not do everything at once.** A plan exists precisely to keep each PR small; collapsing
  steps defeats it and produces an unreviewable diff.
- **If the step is mis-scoped** (too big, missing a dependency, the type design doesn't fit
  reality, or you found a needed change the plan didn't anticipate): **stop and report it to
  the user.** Propose a plan amendment. Do not silently absorb the extra work.
- If you notice unrelated problems or further conversion opportunities, **list them for the
  user** as follow-ups — do not fix them now.

## 5. Hand off

Summarize: what changed (files/symbols), how it maps to the step, verification results, any
exemptions removed, and any follow-ups you noticed but deliberately left out of scope. Note
that this PR/step is ready for review (the `review-*` skills can run on this diff) and name
the next step without starting it.
