# Documentation and autocomplete

Once types carry the information, the docs lean on them and the editor becomes part of the
documentation.

## Docstrings

- **Types out of docstrings.** They live in the signature; duplicated in prose they rot.
  Docstrings describe meaning, ranges, and interactions.
- **One docstring per dataclass field**, placed directly beneath it, instead of one
  `Attributes:` block. The class docstring covers purpose, behavior, and cross-field rules;
  the per-field docstring is what the editor shows on hover for that field, and it doesn't
  rot into a stale wall of text.

```python
from dataclasses import dataclass
from typing import ClassVar


@dataclass(frozen=True)
class ExponentialBackoff:
    """Retry policy whose wait grows geometrically between attempts.

    Each failed attempt waits `factor` times longer than the previous one,
    starting from `base_delay`. Retrying stops once `max_attempts` is reached;
    if `max_attempts` is None, the number of attempts is unbounded.
    """

    name: ClassVar[str] = "exponential"

    base_delay: float = 0.1
    """Delay before the first retry, in seconds. Must be >= 0."""

    factor: float = 2.0
    """Multiplier applied to the delay after each failed attempt. Must be > 1."""

    max_attempts: int | None = None
    """Hard cap on the number of attempts.

    If None, the number of attempts is unbounded.
    """
```

## User docs

- **Feature the strict, autocomplete-friendly way first.** When several input styles are
  supported (object, string, dict), show the discoverable one; legacy styles go in tabs or
  expandable boxes, not deleted.
- **Examples import and use the new types**, so readers learn the good path.

## What defeats autocomplete

The type design only pays off if the editor can show it. On public symbols these quietly
cancel the payoff:

- `Any` or `dict[str, Any]` in a return type: nothing completes on the result.
- `**kwargs` as the real option surface: options hidden there never complete. Use explicit
  parameters or a config object whose fields complete.
- Signatures built at runtime (decorators that erase parameters, dynamically constructed
  functions): most editors show nothing. Where this is a deliberate trade-off, document it.
- Many valid choices discoverable only by reading the docs: a typed namespace or selection
  object lets the user find them with `lib.<TAB>`.
