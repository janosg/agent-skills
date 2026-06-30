# Strong Typing — Pattern Catalog

The concrete before→after transformations that implement the principles in
`philosophy.md`. Each pattern lists the smell, the fix, and *when* to apply it. Always
cross-check against `when-not-to-convert.md` before applying — several of these have a
"correct" counterpart where the loose version should stay.

Assume `from __future__ import annotations` and Python 3.10+ syntax (`X | Y`, `list[T]`).

---

## 1. Stringly-typed value → `Enum`

**Smell:** a fixed set of choices passed and compared as raw strings. Typos detonate at
runtime; `if mode == "rais":` is a silent bug; no autocomplete.

```python
# before
def run(error_handling: str = "raise"): ...
if error_handling == "raise": ...          # typo-prone on both sides
```

```python
# after
from enum import Enum

class ErrorHandling(Enum):
    RAISE = "raise"
    CONTINUE = "continue"

# public boundary stays generous: accept the string, normalize to the enum
def run(error_handling: ErrorHandling | str = ErrorHandling.RAISE):
    error_handling = ErrorHandling(error_handling)   # ValueError on a bad string
    if error_handling is ErrorHandling.RAISE: ...
```

**When:** the set of valid values is closed and known. The `Enum(value)` call turns a
user typo into an immediate, clear `ValueError` while still letting beginners pass a string.
Compare with `is`, not `==`. (`Literal["raise", "continue"]` is a lighter alternative when
you don't need a real object — good for simple flags, but an Enum gives you a namespace,
autocomplete, and a place to hang helpers.)

---

## 2. Dict with known keys → frozen dataclass

**Smell:** a `dict[str, Any]` (or `dict[str, ...]`) whose keys are fixed and known at
design time. No autocomplete, renames are search-and-replace, the type says nothing about
shape, and callers must read the producer's body to learn the keys.

```python
# before
def find_person(...) -> dict[str, Any]:
    return {"name": ..., "city": ..., "age": ...}
```

```python
# after
from dataclasses import dataclass

@dataclass(frozen=True)
class Person:
    name: str
    city: City
    age: int

def find_person(...) -> Person: ...
```

**When:** all fields are known and stable. Renames become an editor refactor; the type
checker finds every caller when the shape changes; IDEs autocomplete attributes. **Do not
apply** when keys are dynamic/unknown/user-supplied — see `when-not-to-convert.md`. If the
shape is fixed but a real `dict` is required (JSON interop, `**kwargs`), reach for
`TypedDict` instead of a dataclass.

---

## 3. Tuple return → dataclass

**Smell:** returning `tuple[str, str, int]` — positional, unnamed, easy to unpack in the
wrong order.

```python
# before
def find_person(...) -> tuple[str, str, int]: ...     # name? surname? age? ssn?
```

```python
# after  -> Person dataclass as in pattern 2
def find_person(...) -> Person: ...
```

**When:** the tuple has more than ~2 elements or its elements aren't obviously
interchangeable/ordered. `NamedTuple` is the middle ground if you truly need tuple behavior
(iteration, unpacking, backwards-compatible positional access).

---

## 4. Open set of variants → tagged union (ADT) + exhaustiveness

**Smell:** a "kind" string plus a bag of sometimes-present fields; `if kind == ...`
ladders; impossible field combinations representable.

```python
# after
@dataclass(frozen=True)
class Header:
    protocol: Protocol
    size: int

@dataclass(frozen=True)
class Payload:
    data: bytes

@dataclass(frozen=True)
class Trailer:
    data: bytes
    checksum: int

Packet = Header | Payload | Trailer    # a name for the closed set
```

Handle every variant, and let the type checker prove you didn't miss one:

```python
from typing import assert_never

def handle(packet: Packet) -> None:
    match packet:
        case Header(protocol, size): ...
        case Payload(data): ...
        case Trailer(data, checksum): ...
        case _: assert_never(packet)   # mypy errors here if a variant is unhandled
```

**When:** a closed, known set of shapes that are processed together. Use `assert_never`
(not `assert False`, which `python -O` strips) for compile-time exhaustiveness. The union is
declared *outside* the member classes, so they stay decoupled and one class can belong to
several unions. Great for config versioning (`Config = ConfigV1 | ConfigV2`) and
(de)serialization.

---

## 5. Overloaded `__init__` → named construction functions

**Smell:** one `__init__` with many optional, mutually-exclusive parameters; invalid
combinations (`y1` with `width`) are constructible.

```python
# after
@dataclass(frozen=True)
class Rectangle:
    x: float; y: float; width: float; height: float

    @classmethod
    def from_corners(cls, x1: float, y1: float, x2: float, y2: float) -> "Rectangle": ...

    @classmethod
    def from_origin_size(cls, x: float, y: float, w: float, h: float) -> "Rectangle": ...
```

**When:** an object has several distinct construction recipes. Each named constructor makes
the intent obvious and makes invalid field mixes unconstructible.

---

## 6. Mutable god-object across phases → typestate

**Smell:** one class with a documented "call these in this order, never call X after Y"
rulebook, enforced (if at all) by runtime asserts and boolean flags.

```python
# before: one Client with connect/authenticate/send/close + a page of rules
```

```python
# after: each state is its own type; you can only reach the next via the previous
def connect(address: str) -> ConnectedClient | None: ...

class ConnectedClient:
    def authenticate(self, password: str) -> AuthenticatedClient | None: ...

class AuthenticatedClient:
    def send(self, msg: str) -> None: ...
```

For "cannot use after close", drop `close()` and expose a context manager so the resource is
released at block exit and can't be used afterward:

```python
with connect(addr) as client:      # ConnectedClient/AuthenticatedClient
    client.send("hi")
# closed here; no way to call a closed client
```

**When:** an object moves through mutually-exclusive lifecycle states. Splitting them makes
illegal sequences unrepresentable instead of runtime-checked.

---

## 7. Mutable config → immutable + copy constructors

**Smell:** option dicts/objects mutated in place; looping over variants accidentally shares
state.

```python
# after
@dataclass(frozen=True)
class Config:
    max_iter: int = 1000
    tolerance: float = 0.1

    def with_option(self, **changes) -> "Config":
        return dataclasses.replace(self, **changes)

base = Config(max_iter=1000)
variants = [base.with_option(tolerance=t) for t in (0.1, 0.2, 0.5)]   # base untouched
```

**When:** configuration objects. `frozen=True` + `dataclasses.replace` (or a `with_*`
method) gives safe "modified copies". You can add `with_<namespace>(...)` helpers that
validate and strip a common prefix to recreate namespaces ergonomically.

---

## 8. `flag: bool` + `flag_options: dict` pair → one generous argument

**Smell:** two coupled arguments — one to switch a feature on, one dict to configure it.
Setting the options without the flag silently does nothing; the dict is brittle.

```python
# before
fetch(..., cache=True, cache_options={"backend": "redis", "ttl": 60})
```

```python
# after
@dataclass(frozen=True)
class CacheOptions:
    backend: str = "memory"
    ttl: float = 60.0

# bool keeps the easy on/off; the object replaces the dict and enables autocomplete
def fetch(..., cache: bool | CacheOptions = False): ...
```

**When:** any "enable + configure" pair. Collapse to `bool | OptionsObject`: `False`/`True`
keep the zero-import easy path, the object gives discoverable, validated, immutable config.
Same idea for `logging: bool | str | Path | Logger`, etc. Deprecate the old `_options`
argument rather than removing it outright (for libraries).

---

## 9. Pluggable behavior (`str | Callable` + options dict) → configured strategy object

**Smell:** a pluggable strategy chosen by a string or a raw callable, *plus* a separate
options dict to configure it. The selector is stringly-typed, the options are brittle and
don't autocomplete, and the two are coupled yet passed separately.

```python
# before
def save(..., compression: str | Callable = "gzip", options: dict | None = None): ...
save(..., compression="gzip", options={"level": 9})
```

```python
# after: the strategy IS a frozen dataclass — config as fields, behavior as a method
from typing import Protocol

class Compressor(Protocol):
    def __call__(self, data: bytes) -> bytes: ...

@dataclass(frozen=True)
class Gzip:
    level: int = 6
    def __post_init__(self): assert 0 <= self.level <= 9
    def __call__(self, data: bytes) -> bytes: ...   # the behavior lives here

def save(..., compression: Compressor): ...
save(..., compression=Gzip(level=9))      # one validated, autocompleting object
```

**When:** any strategy/plugin point where you'd otherwise pair a `str`/`Callable` selector
with an options dict. Bundling configuration (frozen fields, validated in `__post_init__`,
discoverable via autocomplete) *with* behavior (the method) into one immutable object replaces
both the selector and the dict. Fix the method name with a `Protocol` or ABC so all strategies
are interchangeable. By the boundary pattern below (and principle 4), a widely-used entry point
can *also* accept the bare `str`/`Callable` for convenience and normalize to the object —
generous in, strict core. This fits compression backends, retry policies, serializers,
samplers, and similar pluggable components. Don't apply it where the behavior has no
configuration or variants — a plain function is enough.

---

## 10. Generous-in / strict-out boundary

The pattern that ties the others together. The *public* function accepts a forgiving union
and returns a strict type; it converts once, then hands a clean type to strict internals.

```python
# public: generous in, strict out
def save(
    data: bytes,
    compression: Compressor | type[Compressor] | str,
) -> SaveReport:                     # a proper result type, never dict[str, Any]
    compressor = _coerce_compressor(compression)   # normalize + validate at the boundary
    return _save(data, compressor)                 # internal: strict in, strict out

def _save(data: bytes, compression: Compressor) -> SaveReport: ...
```

**When:** every public entry point. Keep the convenience (and backwards compatibility) at
the edge; keep the core free of `if`-laddering over input shapes. The conversion site is
also where runtime validation lives.

---

## Quick reference

| You see… | Reach for… | Unless… |
|---|---|---|
| choice passed as a `str` | `Enum` (or `Literal`) | the set is open/user-extensible |
| `dict` with known fixed keys | frozen `dataclass` | keys are dynamic → keep dict / `TypedDict` |
| `tuple` of 3+ unnamed values | `dataclass` | you truly need tuple behavior → `NamedTuple` |
| "kind" + optional fields | tagged union + `assert_never` | the set of kinds is open-ended |
| many-optional `__init__` | `@classmethod` constructors | one obvious construction path |
| ordered-lifecycle object + rules | typestate / context manager | the states aren't mutually exclusive |
| mutated config | `frozen=True` + `with_*` | it genuinely must mutate in place |
| `flag` + `flag_options` dict | `bool \| OptionsObject` | — |
| pluggable `str`/`Callable` + options dict | configured strategy object (frozen dataclass + `__call__`, fixed by a `Protocol`) | the behavior has no config/variants |
| loose union in a public return | a strict result type | the output really is heterogeneous |
