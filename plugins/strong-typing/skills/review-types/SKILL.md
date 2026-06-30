---
name: review-types
description: Review code for whether the RIGHT types and data abstractions were chosen in a strong-typing context — catching both under-typing (dicts/tuples/strings that should be dataclasses/enums/unions) AND over-typing (dynamic dicts wrongly frozen into dataclasses, forced immutability, harmful subclassing). Use to review a strong-typing diff/PR or a module for correct type and abstraction choices. Reviews, does not rewrite. Triggers: "review the types", "did I pick the right types", "check this for over/under-typing", "review this strong-typing PR for type choices".
---

# Strong Typing Review — Type & Abstraction Choices

Review whether the **data design** is right: are values modeled with the type that best
prevents bugs without adding empty ceremony? This catches the two opposite failures equally —
**under-typing** (loose where it should be strict) and **over-typing** (strict where loose is
correct). Over-typing is the more common mistake in automated conversions, so weight it
accordingly.

This skill **reviews and recommends; it does not rewrite.** Produce findings the user (or
`implement`) can act on.

## 0. Load the canon

- `${CLAUDE_PLUGIN_ROOT}/references/philosophy.md`
- `${CLAUDE_PLUGIN_ROOT}/references/patterns.md`
- `${CLAUDE_PLUGIN_ROOT}/references/when-not-to-convert.md`

## 1. Set the scope

Default to the **current diff** (`git diff`, or the PR/branch under discussion) so this runs
naturally after an `implement` step. If the user asks for a whole-module or whole-repo pass,
do that instead. State which scope you used.

## 2. Under-typing checklist (loose → should be strict)

- [ ] choice values passed/compared as raw **strings** that are a closed set → `Enum`/`Literal`
- [ ] **dict with fixed, known keys** used as a record → frozen dataclass (or `TypedDict` if a
      dict is structurally required)
- [ ] **tuple** of 3+ unnamed values → dataclass / `NamedTuple`
- [ ] "**kind**"/type field + conditionally-present attributes → tagged union + `assert_never`
- [ ] public function returning `dict[str, Any]` or a loose `Union` → a strict result type
- [ ] `__init__` with many mutually-exclusive optional params → construction functions
- [ ] a pluggable strategy as `str`/`Callable` **+ a separate options dict** → a configured
      strategy object (frozen dataclass + `__call__`, fixed by a `Protocol`)
- [ ] stray `Any`/missing hints where a precise type is known and cheap

## 3. Over-typing checklist (strict → should be loose) — weight heavily

- [ ] a dataclass/`TypedDict` whose keys are actually **dynamic/user-defined** → should be a
      dict (`dict[str, ValueType]`, strict on value, loose on keys)
- [ ] a type that grew **deeply nested generics** instead of a named intermediate or a
      deliberately looser type
- [ ] an **Enum/dataclass added where no real misuse was possible** (ceremony with no
      prevented bug, autocomplete, or clarity gain)
- [ ] **`frozen=True` forced** onto something that genuinely mutates, producing awkward copies
- [ ] **`Any` laundered** into a fake-precise type across a genuinely dynamic boundary (prefer
      `object` or an honest `Any`)
- [ ] **inheritance** introduced where composition was simpler / coupling was lost

## 4. Correctness-of-modeling checklist

- [ ] dataclasses that should be `frozen=True` but aren't (config/value objects)
- [ ] **mutable defaults** (`= []`/`= {}` instead of `field(default_factory=...)`)
- [ ] illegal states still representable (impossible field combinations, a god-object that
      should be split via typestate)
- [ ] does each new type **earn its place**? (prevented bug / autocomplete / clarity)

## 5. Report

For each finding give:

- **file:line** and a one-line description
- **category**: under-typing / over-typing / modeling
- **severity**: high (real bug or clearly-wrong abstraction) / medium / low (style)
- **before → after** sketch of the recommended type
- **why** (which payoff it buys, or which canon rule it violates)

Lead with high-severity and with over-typing findings. If the diff is clean, say so
explicitly and note anything you verified was *correctly* left loose (so it isn't "fixed"
later). Do not edit files unless the user explicitly asks you to apply a finding.
