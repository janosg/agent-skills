# Hand-written validation → pydantic dataclass + constrained types

**Only if pydantic is an acceptable dependency.** If it already is one, use it. If not, ask:
for an application it is usually fine, but for a library it is a heavy compiled runtime
dependency that every downstream user inherits.

**Smell:** a `__post_init__` or parse function that checks and converts field by field
(`if self.tolerance <= 0: raise`, `object.__setattr__(self, "path", Path(self.path))`).

```python
# after
from pathlib import Path
from pydantic import PositiveFloat, PositiveInt
from pydantic.dataclasses import dataclass

@dataclass(frozen=True)
class Config:
    path: Path                      # "a.txt" -> Path("a.txt")
    bounds: tuple[float, ...] = ()  # [1, 2] -> (1.0, 2.0)
    max_iter: PositiveInt = 1000
    tolerance: PositiveFloat = 0.1  # -1.0 -> ValidationError naming the field
```

**When:** records and configs that users construct directly. The class becomes its own
boundary: generous in at construction, strict out on every attribute, with no parse code to
keep in sync with the fields. `dataclasses.replace` re-validates, so the `with_*` copy
constructors of [immutable config](immutable-config.md) stay safe for free. Cross-field
invariants still go in `__post_init__`, which runs after the fields are converted.

## Constrained types

`PositiveFloat`, `NonNegativeInt`, `Annotated[float, Field(ge=0, le=1)]` and friends state a
constraint in the signature, where users and docs see it, and pydantic enforces it wherever it
validates: a pydantic dataclass, `validate_call`, a `TypeAdapter`. Use them for real domain
constraints (a tolerance, a probability, a count), not ranges invented to look precise.

Statically they are plain `float` and `int`: checkers catch no violation. In a plain dataclass
or an unvalidated function they are documentation only, which is still better than a comment,
but don't count them as validation there.

## Conversion and the type checker

Checkers type `__init__` with the declared field types, so `Config(path="a.txt")` is a
checker error even though it works at runtime. That is the right trade: the attribute type is
the strict out-type, and the conversion serves the users who never run a checker. Don't widen
the field annotation to the input union, and don't `cast` at call sites. Where typed callers
need the generous form too, give them a [boundary](boundary.md) function or a `from_*`
classmethod whose signature says what it accepts.

Default (lax) mode also coerces `"0.5"` to `0.5` and `"1"` to `True`. Fine when values come
from config files or the command line; where silent coercion would hide a bug, set
`config=ConfigDict(strict=True)` or mark single fields `Strict()`, and keep the conversions
you want explicit.

**Unless:** the type is internal and only built from already-strict values (validation there
is cost without benefit), or it sits in a hot loop where per-instance validation is too slow.
