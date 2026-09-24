# Stringly-typed value → `Enum`

**Smell:** a fixed set of choices passed and compared as raw strings. Typos detonate at
runtime, `if mode == "rais":` is a silent bug, and nothing autocompletes.

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

**When:** the set of valid values is closed and known. `Enum(value)` turns a user typo into an
immediate, clear `ValueError` while beginners can still pass a string. Compare with `is`, not
`==`. `Literal["raise", "continue"]` is the lighter alternative when you need no real object:
good for simple flags, but an Enum gives a namespace, autocomplete, and a place to hang
helpers.

**Not for selecting behavior.** An Enum names a closed set of *values*; it cannot carry
behavior or per-choice configuration. When the string selects *which code runs* (an algorithm,
backend, or strategy), use a [configured strategy object](strategy-object.md).
