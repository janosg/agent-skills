# Pluggable behavior (`str | Callable` + options dict) → configured strategy object

**Smell:** a pluggable strategy chosen by a string or a raw callable, *plus* a separate options
dict to configure it. The selector is stringly-typed, the options are brittle and don't
autocomplete, and the two are coupled yet passed separately.

```python
# before
def save(..., compression: str | Callable = "gzip", options: dict | None = None): ...
save(..., compression="gzip", options={"level": 9})
```

```python
# after: the strategy IS a frozen dataclass; config as fields, behavior as a method
from typing import Protocol

class Compressor(Protocol):
    def __call__(self, data: bytes) -> bytes: ...

@dataclass(frozen=True)
class Gzip:
    level: int = 6
    def __post_init__(self):
        if not 0 <= self.level <= 9:
            raise ValueError(f"level must be in [0, 9], got {self.level}")
    def __call__(self, data: bytes) -> bytes: ...   # the behavior lives here

def save(..., compression: Compressor): ...
save(..., compression=Gzip(level=9))      # one validated, autocompleting object
```

**When:** any strategy or plugin point where you'd otherwise pair a `str`/`Callable` selector
with an options dict: compression backends, retry policies, serializers, samplers,
optimization algorithms. Bundling configuration (frozen fields, validated in `__post_init__`,
discoverable via autocomplete) with behavior (the method) into one immutable object replaces
both the selector and the dict. Fix the method name with a `Protocol` or ABC so all strategies
are interchangeable. A widely-used entry point can still accept the bare `str`/`Callable` for
convenience and normalize to the object at the [boundary](boundary.md).

**Prefer this over an enum of methods.** An Enum can name *which* strategy but can't run it or
hold its configuration; you'd still need a dispatch table plus a separate options dict.

**Unless:** the behavior has no configuration and no variants. Then a plain function is
enough.
