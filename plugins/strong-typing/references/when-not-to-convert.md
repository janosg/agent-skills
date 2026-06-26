# Strong Typing — When NOT to Convert (Guardrails)

Strong typing is about choosing the *right* type, not the *most* type. Over-application is
as much a failure as stringly-typed code — and it's the failure an automated agent is most
likely to commit. Before applying any pattern from `patterns.md`, check it against this list.

## Keep the dict when the keys are dynamic

A dict is the **correct** type — do not convert to a dataclass — when:

- **Keys are unknown or user-defined.** e.g. a function returns a scalar result plus an
  open-ended bag of extra info to be logged: `{"value": 1.2, "p_mean": ..., "p_std": ...}`,
  where the extra keys are whatever the user chose. Model the known part as a field and keep
  the unknown part as a dict: `FunctionValue(value=..., info=dict[str, Any])`.
- **Keys vary at runtime / are data, not schema.** e.g. a collection of benchmark problems
  keyed by problem name, where which names exist depends on what the user requested. A
  fixed-field class can't represent "merge two benchmark sets" or "filter by name". Use
  `dict[str, BenchmarkProblem]` — the *value* type is strict, the keys stay dynamic.
- **Pass-through configuration** handed to some other library you don't control, e.g.
  `scatter_kws` forwarded to a plotting backend. You can't enumerate the keys; a dict (or
  `Mapping[str, Any]`) is honest.
- **The mapping IS the abstraction** — registries, caches, sparse lookups, counts by label.

Rule of thumb: **strict on the value type, loose on the keys.** `dict[str, Problem]` is
strongly typed; `dict[str, Any]` with secretly-fixed keys is not.

## Prefer `TypedDict` over a dataclass when a dict is structurally required

If the shape is fixed and known, but the value must remain a real `dict` (JSON
(de)serialization, `**kwargs`, an existing wire/API contract), use `TypedDict`. It gives
known keys and checker support without forcing callers onto a class.

## `Any` is sometimes the honest type

Don't launder a genuinely-dynamic value through a fake-precise type. Legitimate `Any`
(or `object`, which is safer):
- truly arbitrary user payloads (additional criterion args, opaque context objects);
- the boundary of (de)serialization before validation;
- deliberately-generic plumbing that forwards values it never inspects.
Prefer `object` over `Any` when you only store/forward the value (it forces a checked
narrowing before use). Reserve `Any` for where even that is too strict.

## Don't over-nest types

If describing one parameter takes five nested generics, **stop**. A type like
`dict[str, list[tuple[int, Callable[..., dict[str, Any]]]]]` is a design smell. Either:
- introduce named intermediate types (often a dataclass or two) that make the structure
  readable, or
- accept a deliberately looser type at that spot.
A parameter that can legitimately be "a number, a tuple of strings, or a dict of dicts" is
telling you to refactor the function, not to write a bigger `Union`.

## Don't add ceremony that prevents no real error

- **`NewType` only where confusion actually happens.** Wrapping every `int` in a distinct
  type is noise. Use it for IDs/units that genuinely get swapped.
- **An Enum/dataclass with a single member or single use** rarely earns its keep — a
  `Literal` or a plain value may be clearer.
- Each new type should buy a prevented bug, better autocomplete, or clearer code. If it buys
  none, don't introduce it.

## Mutability is sometimes right

Default to `frozen=True`, but don't contort code to avoid all mutation:
- genuine accumulators, buffers, caches, and builders are naturally mutable;
- large objects rebuilt frequently in a hot loop may justify in-place mutation for
  performance.
Prefer "immutable + copy constructor" first; fall back to documented mutability when copying
is wrong or too costly. Never use a mutable default (`field(default_factory=...)`, not
`= []`).

## Don't break public APIs for purity (libraries)

Type-driven cleanups must not silently break users. For anything public:
- keep the old interface working through a **deprecation cycle** (accept the old type, warn,
  convert);
- the "generous inputs" principle exists partly for this — keep accepting `str`/dict at the
  boundary even after introducing the strict type.
See `design-proposal` (it asks up front whether breaking changes are allowed) and
`tooling.md`.

## Don't delete runtime validation

Static types are invisible to users who don't run a checker. Removing input validation
because "the type guarantees it" turns a friendly error into a deep crash. Keep boundary
checks and `__post_init__` invariants.

## Prefer composition over (most) subclassing

When reducing duplication among related types, reach for composition first. Subclass only
for genuine subtyping (Liskov-substitutable), and prefer shallow hierarchies with abstract
base classes / `Protocol`s over deep concrete inheritance. Inheriting to share a few helpers
often couples things that should stay independent.

## The over-conversion checklist (for reviews)

A change is **over-converted** if any of these is true:
- a dict whose keys are dynamic/user-defined was frozen into a dataclass/`TypedDict`;
- a type acquired deep nesting instead of a named intermediate or a looser type;
- a `NewType`/Enum/dataclass was added where no real misuse was possible;
- `frozen=True` was forced onto something that genuinely needs mutation, adding awkward copies;
- a public signature got *stricter* (narrower input) without a deprecation path;
- runtime validation was dropped in favor of "the types cover it";
- inheritance was introduced where composition was simpler and looser coupling was lost.
