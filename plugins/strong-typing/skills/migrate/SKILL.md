---
name: migrate
description: Refactor Python code from stringly typed to strongly typed. Use it for an enhancement proposal, a migration plan, a module refactor, or a review of a diff.
disable-model-invocation: true
---

# Strong typing

*Strongly typed* means more than type hints. The goal is **soundness**: interfaces that are
hard or impossible to misuse, so that whole classes of bugs become unrepresentable rather
than tested-for. Hints are necessary but not sufficient; the real work is choosing the right
data types.

The request sets the form of the work: an enhancement proposal, a migration plan, the
refactor of one module, a review of a diff. The canon below sets the substance in every
form. `references/philosophy.md` holds the full rationale for when a judgment call is hard.

## Principles

1. **Make illegal states unrepresentable.** A wrong call that cannot be written beats one
   that is detected at runtime. Everything below serves this.
2. **Hint wherever the hint carries information**, signatures and attributes above all. Five
   nested generics on one parameter is a design smell: simplify the code or loosen the type.
3. **Known fields → frozen dataclass. Dynamic keys → keep the dict.** The single most
   important judgment call. Strict on the value type, loose on the keys.
4. **Generous in, strict out.** Input generosity scales with reach: the more call sites a
   function has, the more convenient forms it accepts and normalizes, once, at its boundary.
   Outputs are proper types regardless of reach, never `dict[str, Any]` or a loose union.
   Beneath the boundary everything is strict in and strict out.
5. **Immutable by default.** `frozen=True`, with copy constructors (`with_x(...)`) where a
   modified version is needed.
6. **Functions as entry points; objects for configuration, strategies, and results.**
   `do_thing(config)` beats `config.run()`. Pluggable behavior is a configured strategy
   object, not `str | Callable` plus an options dict.
7. **Named construction functions over an overloaded `__init__`.**
8. **Keep runtime validation.** Most end users never run a type checker. Validate at the
   boundary and in `__post_init__`.
9. **Backwards compatibility is a decision, not a default: ask.** Whether an old interface
   is dropped, deprecated, or kept depends on users, stability promises, and release cadence.
   Record the answer before designing public changes.
10. **Every type must earn its place** with a prevented bug, better autocomplete, or clearer
    code. Ceremony that buys none of these is **over-conversion**, the failure an automated
    refactor commits most readily.

## Patterns

Read a pattern's file before applying it; each holds the smell, the before/after, and the
cases where the loose version is correct. Check every conversion against
`references/when-not-to-convert.md`.

| You see… | Reach for… | Unless… |
|---|---|---|
| a choice passed as a `str` | [`Enum` or `Literal`](references/patterns/enum.md) | the set is open or user-extensible |
| a `dict` with known fixed keys, or a `tuple` of 3+ values | [frozen dataclass](references/patterns/dataclass-record.md) | keys are dynamic → keep the dict; a real dict is required → `TypedDict` |
| a "kind" field plus optional fields | [tagged union + `assert_never`](references/patterns/tagged-union.md) | the set of kinds is open-ended |
| an `__init__` with many mutually exclusive optionals | [named constructors](references/patterns/construction-functions.md) | one obvious construction path |
| an object with an ordered-call rulebook | [typestate or context manager](references/patterns/typestate.md) | the states are not mutually exclusive |
| config mutated in place | [`frozen=True` + `with_*`](references/patterns/immutable-config.md) | it genuinely must mutate |
| `flag: bool` + `flag_options: dict` | [`bool` or options object](references/patterns/options-argument.md) | — |
| pluggable `str`/`Callable` + options dict | [configured strategy object](references/patterns/strategy-object.md) | the behavior has no config or variants |
| a loose union or dict in a public return | [generous-in / strict-out boundary](references/patterns/boundary.md) | the output really is heterogeneous |

## Beyond one module

- A public API, or work spanning several modules → `references/migration-strategy.md`:
  the compatibility decision (drop / deprecate / keep), what a proposal covers, the order
  that keeps every step mergeable, and how to split changes that resist being small.
- Setting up or tightening the type checker → `references/type-checker.md`: strict config,
  the per-module ratchet, CI and pre-commit.
- Docstrings and user docs → `references/documentation.md`: types out of docstrings,
  per-attribute docstrings, feature the strict path, what defeats autocomplete.

## Working rules

- Convert what the request names. Further opportunities you notice go into a follow-up list
  for the user, not into the diff.
- Run the type checker and the tests on the touched package and report the real output. A
  `# type: ignore`, `cast`, or `Any` added to silence the checker is a finding, not a fix.
- When reviewing, weight over-conversion at least as heavily as under-typing, and name the
  places that were correctly left loose so nobody "fixes" them later.

## Request

$ARGUMENTS
