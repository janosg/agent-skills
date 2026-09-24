# Overloaded `__init__` → named construction functions

**Smell:** one `__init__` with many optional, mutually exclusive parameters; invalid
combinations (`y1` together with `width`) are constructible.

```python
# after
@dataclass(frozen=True)
class Rectangle:
    x: float
    y: float
    width: float
    height: float

    @classmethod
    def from_corners(cls, x1: float, y1: float, x2: float, y2: float) -> "Rectangle": ...

    @classmethod
    def from_origin_size(cls, x: float, y: float, w: float, h: float) -> "Rectangle": ...
```

**When:** an object has several distinct construction recipes. Python has no constructor
overloading; each named constructor makes the intent obvious and makes invalid field mixes
unconstructible.

**Unless:** there is one obvious construction path. Then the plain `__init__` (or dataclass
default) is enough.
