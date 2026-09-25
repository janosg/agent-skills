# Generous-in / strict-out boundary

The pattern that ties the others together. The widely-called function accepts a forgiving
union and returns a strict type; it parses the input once, then hands a clean type to strict
internals.

```python
# entry point: generous in, strict out
def save(
    data: bytes,
    compression: Compressor | type[Compressor] | str,
) -> SaveReport:                     # a proper result type, never dict[str, Any]
    compressor = _parse_compressor(compression)    # parse: normalize + validate, once
    return _save(data, compressor)                 # internal: strict in, strict out

def _save(data: bytes, compression: Compressor) -> SaveReport: ...
```

**When:** every entry point users touch, and any function with many call sites. Generosity
scales with reach: a single-caller helper demands the exact type; a function imported all over
the codebase meets its callers where they are. Keep the convenience (and the backwards
compatibility, see `../migration-strategy.md`) at the edge; keep the core free of
`if`-laddering over input shapes. The parse site is also where runtime validation lives,
so a bad input fails early with a clear message.

**Unless:** the output really is heterogeneous. Then name the heterogeneity with a
[tagged union](tagged-union.md) rather than returning `dict[str, Any]`.
