# Mutable config → immutable + copy constructors

**Smell:** option dicts or objects mutated in place; looping over variants accidentally shares
state.

```python
# after
import dataclasses
from dataclasses import dataclass

@dataclass(frozen=True)
class Config:
    max_iter: int = 1000
    tolerance: float = 0.1

    def with_tolerance(self, tolerance: float) -> "Config":
        return dataclasses.replace(self, tolerance=tolerance)

base = Config(max_iter=1000)
variants = [base.with_tolerance(t) for t in (0.1, 0.2, 0.5)]   # base untouched
```

**When:** configuration and value objects. `frozen=True` gives safe modified copies;
invariants across fields go in `__post_init__`, which `dataclasses.replace` runs again.

A copy constructor is only as discoverable as its signature:

- **Few fields:** typed copy methods. One per field (`with_tolerance(tolerance: float)`), or
  one per group of related fields with typed keyword arguments
  (`with_stopping(max_iter: int | None = None, max_time: float | None = None)`), which also
  recreates a namespace ergonomically.
- **Many fields, or many classes sharing the method** (e.g. hundreds of configured
  algorithms), where typed methods would mean unacceptable repetition: a generic
  `with_option(**changes: Any)` over `dataclasses.replace`. It gives up autocomplete and
  static checking of field names, so document the trade-off. Generating typed methods at
  runtime (`functools.wraps`, dynamic signatures) does not win autocomplete back: most
  editors can't see such signatures.

**Unless:** the object is a genuine accumulator, buffer, cache, or builder, or it is rebuilt
in a hot loop where copying is too costly. Then document that it mutates.
