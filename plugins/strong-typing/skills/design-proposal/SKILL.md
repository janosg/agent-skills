---
name: design-proposal
description: Write a strong-typing design proposal (an enhancement-proposal-style doc) for migrating a Python codebase from stringly-typed to strongly-typed. Use when the user wants to plan a type refactor, design the types before touching code, decide what becomes a dataclass/enum/union, or write an EP/RFC for adopting static typing. Produces a design document, NOT code. For libraries it covers public-API breaking changes and deprecation strategy. Triggers: "plan the strong typing refactor", "design the types", "write a typing enhancement proposal", "from stringly typed to strongly typed".
---

# Strong Typing — Design Proposal

Produce a written design proposal for converting a codebase (or one component of it) to be
strongly typed. The output is a **document that decides the target type design** — it is the
input to `implementation-plan` and then `implement`. **Do not write implementation code in
this skill** beyond short before/after illustrations inside the proposal.

## 0. Load the canon first

Read these before doing anything else:

- `${CLAUDE_PLUGIN_ROOT}/references/philosophy.md`
- `${CLAUDE_PLUGIN_ROOT}/references/patterns.md`
- `${CLAUDE_PLUGIN_ROOT}/references/when-not-to-convert.md`
- `${CLAUDE_PLUGIN_ROOT}/references/tooling.md`

## 1. Establish scope and two decisions

First determine **repo type**, because it changes how much of the proposal you write:

- **Library / has a public API** — signs: it's packaged and published, has an `__all__` or a
  documented public surface, is imported by other projects, versions matter. Then the
  proposal must treat public-facing changes carefully (before/after per public symbol +
  deprecation strategy).
- **Internal / application code** — no external consumers; you control all call sites. Then
  the proposal is lighter: no deprecation machinery, breaking changes are usually fine, focus
  on the type design itself.

If it's ambiguous, ask the user.

Then ask the user the **backwards-compatibility question up front** (skip for clearly-internal
code where it's obviously fine to break). For each affected public interface — or as a global
default — which applies:

> - **Drop** it — a clean breaking change; the old interface is removed now.
> - **Deprecate** it — the old interface keeps working but emits a `DeprecationWarning` for a
>   release or two, then is removed.
> - **Keep** it indefinitely — both the old and new interfaces are supported for the
>   foreseeable future.

This depends on context outside the code (how many users the project has, its stability
promises, release cadence) — so **ask, don't assume.** Record the answer; it shapes every
public-facing section and the summaries at the end. For interfaces marked *deprecate* or
*keep*, every public change must keep the old path working (lean on "generous inputs").

## 2. Audit the stringly-typed surface

Survey the code and inventory the conversion opportunities. Look for:

- choices passed/compared as raw **strings** (→ Enum/Literal)
- **dicts with known, fixed keys** (→ dataclass) — and separately note dicts whose keys are
  **dynamic** (those stay dicts; call them out so nobody converts them)
- **tuples** of 3+ unnamed values (→ dataclass/NamedTuple)
- "**kind**" fields + optionally-present attributes (→ tagged union + `assert_never`)
- same-typed **IDs/units** that could be swapped (→ `NewType`)
- `__init__`s with many mutually-exclusive optional params (→ construction functions)
- **mutable config** / option dicts (→ frozen dataclass + copy constructors)
- **`flag: bool` + `flag_options: dict`** pairs (→ `bool | OptionsObject`)
- public functions returning `dict`/loose unions (→ strict result types)
- mutable **god-objects** with ordered-call rulebooks (→ typestate / context managers)
- missing/weak hints, stray `Any`, type info living only in docstrings

For each item note: where it is, current type, proposed type, and whether it's public.
**Apply `when-not-to-convert.md` while auditing** — explicitly mark the dicts/`Any`/mutable
spots that should be *left alone*, so they don't get "fixed" later.

## 3. Write the proposal

Write to a markdown file in the repo (e.g. `docs/strong-typing-proposal.md`, or follow the
repo's existing proposal/EP location if one exists). Structure, per the optimagic EP model:

- **Title + status table** (Author, Status: Draft, Created date, repo type, and the
  compatibility decision — drop / deprecate / keep, per interface or as a default).
- **Abstract** — the benefits being pursued: IDE discoverability/autocomplete, readability,
  robustness via static + stricter internal types. State that this needs more than adding
  hints.
- **Design philosophy** — a short restatement of the principles *as they apply to this repo*
  (don't paste the canon; reference it and specialize it). Include the repo's
  generous-in/strict-out boundary.
- **Per-area sections** — one per audited area. Each follows this shape:
  - *Current situation* (with a short real code excerpt)
  - *Things we want to keep* (don't lose existing flexibility/ergonomics)
  - *Problems*
  - *Proposal* with **before → after** code and the chosen types
  - *Benefits* (which of: prevented bug class / autocomplete / readability)
  - For public symbols when deprecation is required: the *deprecation strategy* (what keeps
    working, what warns, removal target).
- **Tooling section** — the checker, strict config, incremental ratchet plan, docs changes
  (summarize from `tooling.md`, specialized to this repo).
- **Summary of breaking changes** (libraries) — a flat list.
- **Summary of deprecations** (libraries) — a flat list with the release they start/end.

## 3a. Scale to the repo

- **Library:** full treatment above, heavy on public-API before/after and deprecations.
- **Internal/app:** drop the breaking-changes/deprecation sections; keep abstract, philosophy,
  per-area before/after, and tooling. Keep it short.

## 4. Hard rules

- **Design types before code.** The proposal's job is to decide the types; resist jumping to
  implementation.
- **Do not over-convert.** Every proposed type must earn its place (prevented bug /
  autocomplete / clarity). Explicitly list the things you are *not* converting and why
  (dynamic dicts, honest `Any`, intentionally mutable state).
- **Preserve existing flexibility.** Each section's "things we want to keep" is not optional —
  losing a useful capability to gain type purity is a regression.
- **Generous in, strict out** at every public boundary; strict in and out internally.
- End by telling the user the proposal is ready for review and that `implementation-plan` is
  the next step. Do not start implementing.
