---
name: implementation-plan
description: Turn a strong-typing design proposal (EP/RFC) into a step-by-step implementation plan where each step is a small, independently reviewable PR that keeps the codebase working and tests green. Use after a design proposal exists and the user wants to sequence the work, break a big type refactor into mergeable chunks, or order changes with deprecation shims. Produces a plan document, NOT code. Triggers: "turn this proposal into a plan", "break the type refactor into PRs", "sequence the implementation", "implementation plan for the typing migration".
---

# Strong Typing — Implementation Plan

Convert an existing design proposal into an ordered sequence of **small, independently
reviewable PRs**. The output is a **plan document**, not code. Each step must leave the
codebase importable, type-checked (to the extent already ratcheted), and with tests passing.

## 0. Load context

Read:

- `${CLAUDE_PLUGIN_ROOT}/references/philosophy.md`
- `${CLAUDE_PLUGIN_ROOT}/references/patterns.md`
- `${CLAUDE_PLUGIN_ROOT}/references/when-not-to-convert.md`
- `${CLAUDE_PLUGIN_ROOT}/references/tooling.md`
- **The design proposal** (ask the user for its path if you can't find it). The proposal is
  the source of truth for *what* changes; this skill decides *in what order* and *in what
  size*.

## 1. Decompose into units of change

List every concrete change the proposal implies (a new type, a converted function, a
deprecation shim, a config tightening). For each, capture: the symbol(s) touched, whether
it's public, what it depends on, and what depends on it. Build the dependency order — leaf
types and low-level modules first (per the ratchet in `tooling.md`), public entry points
last.

## 2. Sequence into reviewable steps

Order the units so that **every step is independently mergeable and behavior-preserving
unless the proposal says otherwise.** Prefer this canonical ordering, which keeps each PR
small and non-breaking for as long as possible:

1. **Tooling first** — add/clean the type checker config, pre-commit, CI; exempt
   not-yet-converted modules so the baseline is green.
2. **Introduce new types additively** — add the new dataclasses/enums/unions *alongside* the
   old interfaces, unused or used only internally. Pure addition, trivial to review.
3. **Migrate internals to the new types** — switch internal call sites to the strict types,
   behind the unchanged public surface. The generous-in/strict-out boundary converts old →
   new so nothing public changes yet.
4. **Make the public surface accept the new types** — public functions now accept *both* old
   and new (generous inputs). Still non-breaking.
5. **Deprecate the old paths** (libraries) — old inputs keep working but emit
   `DeprecationWarning`; docs switch to featuring the new way.
6. **Remove deprecated paths** — only in the targeted future release; usually its own much
   later PR, often out of scope for the current effort.

Remove a module from the mypy exemption list **in the same PR** that converts it (the ratchet
only shrinks).

## 3. Handle the "splitting is hard" cases

Some changes resist being small. Don't pretend otherwise — make them reviewable anyway:

- **Behavior-preserving refactor first, then types.** If a function must be restructured
  before it can be typed cleanly, split into (a) a pure refactor PR with no type changes
  (easy to review against tests) and (b) a typing PR on top.
- **Wide mechanical changes** (rename a key across many files, swap a dict for a dataclass
  everywhere) → isolate into one PR that does *only* that mechanical change, nothing else, so
  the diff is boring and skimmable.
- **An unavoidably large step** → still document its sub-parts and a self-review checklist so
  the reviewer has a map. Flag it explicitly as large.
- **Internal interfaces with no external users** can change in one breaking PR (no
  deprecation needed) — note this so they aren't over-engineered with shims.

## 4. Write the plan

Write to a markdown file (e.g. `docs/strong-typing-plan.md`). For **each step**:

- **Step N — title** and one-line goal.
- **Scope: in** — exactly what changes.
- **Scope: out** — what is deliberately deferred (so `implement` doesn't overreach).
- **Depends on** — earlier steps / nothing.
- **Independent?** — can it land in any order relative to its siblings, or is it strictly
  sequential? Mark parallelizable steps so they can be batched.
- **Deprecation shims** — what old behavior must keep working (libraries).
- **Verify** — the exact check: which tests, `mypy`/`pyright` clean, which exemptions
  removed, manual checks if any.
- **Rough size** — S / M / L; aim for S–M. Re-split any L that isn't justified in §3.

Add a short **ordering overview** at the top (a numbered list or simple dependency sketch)
and call out which steps are independent.

## 5. Hard rules

- **Each step keeps the repo green.** No step may leave imports broken or tests failing.
- **Smallest reviewable PRs that still make sense.** Bias to more, smaller steps.
- **Respect the proposal's breaking-changes decision.** If deprecation was required, no step
  may break a public interface; if breaking is allowed, don't add needless shims.
- **Don't re-design here.** If the proposal is ambiguous or you'd change the type design,
  stop and raise it with the user rather than deciding silently — that belongs in
  `design-proposal`.
- End by telling the user the plan is ready and that `implement` executes it **one step at a
  time**. Do not start implementing.
