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

    def with_option(self, **changes) -> "Config":
        return dataclasses.replace(self, **changes)

base = Config(max_iter=1000)
variants = [base.with_option(tolerance=t) for t in (0.1, 0.2, 0.5)]   # base untouched
```

**When:** configuration and value objects. `frozen=True` + `dataclasses.replace` (or a
`with_*` method) gives safe "modified copies". `with_<namespace>(...)` helpers that validate
and strip a common prefix recreate namespaces ergonomically. Invariants across fields go in
`__post_init__`.

**Unless:** the object is a genuine accumulator, buffer, cache, or builder, or it is rebuilt
in a hot loop where copying is too costly. See `../when-not-to-convert.md`. Never a mutable
default either way: `field(default_factory=list)`, not `= []`.
