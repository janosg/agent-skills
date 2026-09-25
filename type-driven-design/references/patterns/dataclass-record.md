# Dict with known keys, or unnamed tuple → frozen dataclass

**Smell:** a `dict[str, Any]` whose keys are fixed and known at design time, or a return of
`tuple[str, str, int]`. No autocomplete, renames are search-and-replace, the type says nothing
about shape, callers must read the producer's body to learn the keys, and tuples unpack
silently in the wrong order.

```python
# before
def find_person(...) -> dict[str, Any]:
    return {"name": ..., "city": ..., "age": ...}

def find_person(...) -> tuple[str, str, int]: ...     # name? surname? age? ssn?
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

**When:** all fields are known and stable. Renames become an editor refactor, the type
checker finds every caller when the shape changes, and IDEs autocomplete attributes. For a
tuple, convert once it has more than ~2 elements or its elements aren't obviously
interchangeable.

**Unless:**

- keys are dynamic, unknown, or user-supplied: keep the dict, strict on the value type. See
  `../when-not-to-convert.md`;
- the shape is fixed but a real `dict` is structurally required (JSON interop, `**kwargs`, a
  wire contract): `TypedDict`;
- you truly need tuple behavior (iteration, unpacking, backwards-compatible positional
  access): `NamedTuple`.
