---
name: review-safety
description: Review a strong-typing change for soundness and safety — the generous-in/strict-out boundary, backwards-compatibility and deprecation correctness, a clean type checker (no smuggled Any/type-ignore), retained runtime validation, and unrepresentable illegal states (assert_never exhaustiveness, __post_init__ invariants, no mutable god-objects). Use to review a strong-typing diff/PR before merge for correctness and non-regression. Reviews, does not rewrite. Triggers: "is this typing change safe", "did I break the public API", "review for deprecation correctness", "check the type boundary and validation", "does mypy actually pass cleanly".
---

# Strong Typing Review — Safety & Soundness

Check that a strong-typing change is **sound** and **doesn't regress users**: the input/output
boundary is right, compatibility/deprecation promises are kept, the checker genuinely passes,
runtime validation survives, and illegal states are actually unrepresentable.

Reviews and recommends; **does not rewrite.**

## 0. Load the canon

- `${CLAUDE_PLUGIN_ROOT}/references/philosophy.md`
- `${CLAUDE_PLUGIN_ROOT}/references/when-not-to-convert.md`
- `${CLAUDE_PLUGIN_ROOT}/references/tooling.md`

If a design proposal exists, read its **breaking-changes decision and deprecation list** —
that's the contract this review enforces.

## 1. Set the scope

Default to the **current diff / PR**. Whole-repo on request. State the scope used.

## 2. Generous-in / strict-out boundary

- [ ] **public** functions are **generous in inputs** (still accept the old/convenient types
      where the proposal requires) and convert at the boundary
- [ ] **public** functions are **strict in outputs** — a proper result type, not `dict`/loose
      `Union`
- [ ] **internal** functions are strict in **and** out; the messy `if`-over-input-shape logic
      lives only at the boundary, not scattered through the core
- [ ] conversion happens **once**, at the edge, not repeatedly deep in the call tree

## 3. Backwards compatibility & deprecation (libraries)

- [ ] no public interface was **narrowed/broken** unless the proposal allows breaking changes
- [ ] where deprecation is required: old inputs **still work**, emit a `DeprecationWarning`
      (not a hard error), and convert to the new type
- [ ] deprecation warnings name the **replacement** and a **removal target**
- [ ] new required arguments have safe defaults / aren't forced on existing callers
- [ ] for internal-only interfaces, confirm shims **weren't** added needlessly (don't
      over-engineer compatibility no one needs)

## 4. Type checker actually clean

- [ ] the checker (`mypy`/`pyright`) **passes** on the package — run it if you can
- [ ] no new **`# type: ignore`** added just to silence an error (each remaining one is
      justified and specific)
- [ ] no **`Any`/`cast(...)`** smuggled in to fake a green check across a real type mismatch
- [ ] the mypy **exemption list shrank** (or held) — a converted module was removed from it;
      nothing new was added to it
- [ ] strict flags from `tooling.md` are in effect for the converted code

## 5. Runtime validation retained

- [ ] input validation at the public boundary **was not deleted** on the assumption "the types
      cover it" — end users may not run a checker
- [ ] dataclass **`__post_init__`** invariants exist where values have constraints (ranges,
      normalized vs not, non-empty, etc.)
- [ ] errors are **early and specific** (e.g. `Enum("bad")` → clear `ValueError`), not deep
      crashes

## 6. Illegal states unrepresentable

- [ ] every tagged-union consumer is **exhaustive** with **`assert_never`** (not `assert
      False`, which `python -O` strips) — and mypy would flag a missing variant
- [ ] impossible field combinations are unconstructible (right types / construction functions),
      not merely asserted at runtime
- [ ] ordered-lifecycle objects use **typestate / context managers** rather than a flag-guarded
      god-object, where the proposal calls for it
- [ ] mutation can't break invariants after construction (frozen, or guarded)

## 7. Report

For each finding:

- **file:line**, one-line description
- **category**: boundary / compat / checker / validation / illegal-state
- **severity**: high (user-facing break, false-green checker, dropped validation, missed
  exhaustiveness) / medium / low
- **before → after** or the concrete fix
- **why** (which guarantee or contract it violates)

Run the type checker and tests if the environment allows and **report the real output**. Lead
with anything that breaks users or fakes a green check. If clean, say so and note the
guarantees you verified. Do not edit files unless asked to apply a finding.
