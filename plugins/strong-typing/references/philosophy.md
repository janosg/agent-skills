# Strong Typing — Design Philosophy

This is the shared canon for the `strong-typing` skills. "Strongly typed" means **more
than Python with type hints added on top.** The goal is *soundness*: designing interfaces
that are hard or impossible to misuse, so that whole classes of bugs become unrepresentable
rather than merely tested-for. Type hints are necessary but not sufficient — the real work
is choosing the right data types.

> Mental model: in Rust, an incorrectly used interface fails to compile. In Python we can't
> get that for free, but with strict types + a type checker (mypy/pyright) we get most of
> the same fast feedback, and with the right data design we make many illegal states
> impossible to express at all.

## The core principles

1. **Make illegal states unrepresentable.** Prefer designs where a wrong call simply
   cannot be written, over designs that detect wrongness at runtime. This is the north star;
   everything below serves it.

2. **Type hints everywhere they carry information** — function signatures and class
   attributes above all. But don't be an absolutist: if a single parameter needs five
   nested type hints, that's a *code smell*, not a typing goal. Simplify the code or accept
   a looser type. A signature that can be a number, a tuple of strings, or a dict-of-dicts
   is telling you the design is wrong.

3. **Fixed-field types only when all fields are known.** When the set of attributes is known
   and stable, use a (frozen) dataclass — never a bare dict or tuple. When the keys are
   genuinely dynamic, unknown, or user-defined, **keep the dict.** Getting this distinction
   right is the single most important judgment call. See `when-not-to-convert.md`.

4. **Generous inputs, strict outputs.**
   - *Public* functions are generous about input types (e.g. accept a `str`, a class, or a
     config object), then immediately convert to a strict internal type and call internal
     code.
   - *Public* functions are strict about output types: not just "the type is known" but a
     *proper* type that enables static analysis of its attributes (a result dataclass, not a
     `dict[str, Any]`).
   - *Internal* functions are strict about **both** input and output. The boundary does the
     converting and validating; the core stays clean.

5. **Immutable by default.** Use `frozen=True` dataclasses unless something genuinely needs
   to mutate. When users need a modified version, give them copy constructors
   (`with_option(...)`, `with_x(...)`) rather than mutation. Mutable shared config is a
   recurring source of bugs (e.g. tweaking one option in a loop silently mutates the rest).

6. **Functions as the main entry point; objects for configuration and results.** This takes
   the best of both worlds: the safety and autocomplete that (in Python) only objects give,
   with the beginner-friendliness and composability of functions. A top-level `do_thing()`
   function that *accepts* config objects beats a `.run()` method on a config object —
   it's far easier to extend (e.g. run several configs in parallel).

7. **Construction functions over overloaded `__init__`.** Python has no constructor
   overloading. When an object can be built in several ways, prefer explicitly named
   classmethods (`Rectangle.from_corners(...)`, `Rectangle.from_size(...)`) over one
   `__init__` with many mutually-exclusive, individually-optional parameters.

8. **Keep runtime validation.** Static types help developers and IDE users; most *end
   users* of a library never run a type checker. Do not delete input validation just because
   the types now "say" it can't happen. Validate at the public boundary and in
   `__post_init__`; give early, friendly errors.

9. **Backwards compatibility is a first-class constraint (for libraries).** It is fine to
   keep supporting non-ideal-but-popular interfaces (selecting things by string, accepting
   option dicts) for compatibility. Make the *strict, autocomplete-friendly* way the one the
   docs show prominently; keep the legacy way working but de-emphasized, and remove it only
   through a real deprecation cycle.

10. **Optimize for the reader and the IDE, not for type-hint maximalism.** Every type
    decision should pay off in one of: a prevented bug, better autocomplete/discoverability,
    or clearer code. If it does none of those and only adds ceremony, don't do it.

## How the pieces fit together

- `patterns.md` — the before/after catalog: the concrete transformations that implement
  these principles (string→enum, dict→dataclass, tuple→dataclass, tagged unions, `NewType`,
  construction functions, typestate, copy constructors, generous-in/strict-out boundaries).
- `when-not-to-convert.md` — the guardrails: where a dict / `Any` / mutability / a loose type
  is the *correct* choice, and how to avoid over-engineering.
- `tooling.md` — type checker config, pre-commit/CI, incremental adoption, docs, runtime
  checking.

The skills that build on this canon:
`design-proposal` → `implementation-plan` → `implement`, plus the reviews
`review-types`, `review-ergonomics`, `review-safety`.
