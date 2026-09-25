# `flag: bool` + `flag_options: dict` → one generous argument

**Smell:** two coupled arguments, one to switch a feature on and one dict to configure it.
Setting the options without the flag silently does nothing; the dict is brittle and doesn't
autocomplete.

```python
# before
fetch(..., cache=True, cache_options={"backend": "redis", "ttl": 60})
```

```python
# after
@dataclass(frozen=True)
class CacheOptions:
    backend: str = "memory"
    ttl: float = 60.0

# bool keeps the easy on/off; the object replaces the dict and enables autocomplete
def fetch(..., cache: bool | CacheOptions = False): ...
```

**When:** any "enable + configure" pair. Collapse to `bool | OptionsObject`: `False`/`True`
keep the zero-import easy path, the object gives discoverable, validated, immutable config.
The same idea covers `logging: bool | str | Path | Logger` and similar. Parse into the
object once, at the [boundary](boundary.md).
