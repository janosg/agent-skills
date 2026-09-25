# Documentation and autocomplete

## Docstrings

- **Types out of docstrings.** They live in the signature; duplicated in prose they rot.
  Docstrings describe meaning, ranges, and interactions.
- **One docstring per dataclass field**, placed directly beneath it, instead of one
  `Attributes:` block. The class docstring covers purpose, behavior, and cross-field rules;
  the per-field docstring is what the editor shows on hover for that field, and it doesn't
  rot into a stale wall of text.

```python
@dataclass(frozen=True)
class ExponentialBackoff:
    """Retry policy whose wait grows geometrically between attempts."""

    base_delay: float = 0.1
    """Delay before the first retry, in seconds. Must be >= 0."""
```

## User docs

- **Feature the strict, autocomplete-friendly way first.** When several input styles are
  supported (object, string, dict), show the discoverable one; legacy styles go in tabs or
  expandable boxes, not deleted. Examples import and use the new types.

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
