# Closed set of variants → tagged union + exhaustiveness

**Smell:** a "kind" string plus a bag of sometimes-present fields; `if kind == ...` ladders;
impossible field combinations representable.

```python
# after
@dataclass(frozen=True)
class Header:
    version: int
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
        case Header(version, size): ...
        case Payload(data): ...
        case Trailer(data, checksum): ...
        case _: assert_never(packet)   # the checker errors here if a variant is unhandled
```

**When:** a closed, known set of shapes that are processed together. Use `assert_never` (not
`assert False`, which `python -O` strips) for compile-time exhaustiveness. The union is
declared *outside* the member classes, so they stay decoupled and one class can belong to
several unions. Great for config versioning (`Config = ConfigV1 | ConfigV2`) and
(de)serialization.

**Unless:** the set of kinds is open-ended or user-extensible; then a `Protocol` or ABC that
each variant implements fits better than a union you can never close.
