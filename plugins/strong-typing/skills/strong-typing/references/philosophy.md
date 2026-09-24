# Design philosophy

The full rationale behind the principles in `SKILL.md`. "Strongly typed" means **more than
Python with type hints added on top.** The goal is *soundness*: designing interfaces that are
hard or impossible to misuse, so that whole classes of bugs become unrepresentable rather
than merely tested-for. Type hints are necessary but not sufficient; the real work is choosing
the right data types.

> Mental model: in Rust, an incorrectly used interface fails to compile. In Python we can't
> get that for free, but with strict types + a type checker (mypy/pyright) we get most of the
> same fast feedback, and with the right data design we make many illegal states impossible
> to express at all.

1. **Make illegal states unrepresentable.** Prefer designs where a wrong call simply cannot
   be written, over designs that detect wrongness at runtime. This is the north star;
   everything below serves it.

2. **Type hints everywhere they carry information**, function signatures and class attributes
   above all. But don't be an absolutist: if a single parameter needs five nested type hints,
   that's a *code smell*, not a typing goal. Simplify the code or accept a looser type. A
   signature that can be a number, a tuple of strings, or a dict-of-dicts is telling you the
   design is wrong.

3. **Fixed-field types only when all fields are known.** When the set of attributes is known
   and stable, use a (frozen) dataclass, never a bare dict or tuple. When the keys are
   genuinely dynamic, unknown, or user-defined, **keep the dict.** Getting this distinction
   right is the single most important judgment call. See `when-not-to-convert.md`.

4. **Generous inputs scale with reach; outputs stay strict.** The more widely a function is
   called, the more generous it should be about its input types: accept the convenient unions
   (a `str`, a class, a config object) and normalize them to one strict type at the top. A
   function with a single caller can demand the exact type; one imported all over the codebase
   (and above all the entry points users touch) should meet its callers where they are. This
   is *not* a public/private split; plenty of widely-imported functions aren't user-facing.
   What matters is how many call sites you'd otherwise force to adapt.
   - **Outputs stay strict regardless of reach:** return a *proper* type that enables static
     analysis of its attributes (a result dataclass, never a `dict[str, Any]` or a loose
     union).
   - Convert and validate **once**, in the outermost generous layer; everything beneath it
     then enjoys strict-in / strict-out and stays free of input-shape branching.

5. **Immutable by default.** Use `frozen=True` dataclasses unless something genuinely needs
   to mutate. When users need a modified version, give them copy constructors
   (`with_option(...)`, `with_x(...)`) rather than mutation. Mutable shared config is a
   recurring source of bugs (e.g. tweaking one option in a loop silently mutates the rest).

6. **Functions as the main entry point; objects for configuration, strategies, and results.**
   This takes the best of both worlds: the safety and autocomplete that (in Python) only
   objects give, with the beginner-friendliness and composability of functions. A top-level
   `do_thing()` function that *accepts* config objects beats a `.run()` method on a config
   object: it's far easier to extend (e.g. run several configs in parallel).
   - For *pluggable behavior* (the strategy pattern), libraries commonly accept
     `str | Callable` plus a separate options dict. Prefer a **configured strategy object**: a
     frozen dataclass whose fields are the configuration and which carries the behavior as a
     method, often `__call__`, or a named method fixed by a `Protocol`/ABC. One immutable,
     validated, autocomplete-friendly object then replaces *both* the string selector and the
     loose options dict (e.g. `compression=Gzip(level=9)` instead of
     `compression="gzip", options={...}`). Per point 4, a widely-used entry point can still
     also accept the bare `str`/`Callable` for convenience and normalize to the object. See
     `patterns/strategy-object.md`.

7. **Construction functions over overloaded `__init__`.** Python has no constructor
   overloading. When an object can be built in several ways, prefer explicitly named
   classmethods (`Rectangle.from_corners(...)`, `Rectangle.from_size(...)`) over one
   `__init__` with many mutually-exclusive, individually-optional parameters.

8. **Keep runtime validation.** Static types help developers and IDE users; most *end users*
   of a library never run a type checker. Do not delete input validation just because the
   types now "say" it can't happen. Validate at the public boundary and in `__post_init__`;
   give early, friendly errors.

9. **The backwards-compatibility strategy is a decision, not a default: ask.** Whether an old
   interface should be *dropped* (clean breaking change), *deprecated* (keeps working with a
   warning for a release or two, then removed), or *kept indefinitely* (both ways supported
   for the foreseeable future) depends on context that isn't in the code: how many users the
   project has, its stability promises, its release cadence. Don't assume; ask the maintainer
   and record the answer. Independently of which is chosen, it's fine to keep supporting
   non-ideal-but-popular interfaces (selecting by string, accepting option dicts) for
   convenience, and the docs should feature the *strict, autocomplete-friendly* way most
   prominently while any legacy way is de-emphasized. See `migration-strategy.md`.

10. **Optimize for the reader and the IDE, not for type-hint maximalism.** Every type
    decision should pay off in one of: a prevented bug, better autocomplete/discoverability,
    or clearer code. If it does none of those and only adds ceremony, don't do it.
