# Dict with known keys, or unnamed tuple → frozen dataclass

**When:** all fields are known and stable at design time. Convert a tuple once it has more
than ~2 elements or its elements aren't obviously interchangeable.

**Unless the keys are data, not schema.** Then keep the dict, strict on the value type:
`dict[str, Dataset]` is strongly typed; `dict[str, Any]` with secretly fixed keys is not.

- **User-defined keys**, e.g. a result plus whatever extra metrics the user chose to log:
  model the known part as fields and keep the open part as a dict,
  `Result(value=..., info=dict[str, Any])`.
- **Keys that vary at runtime**, e.g. datasets keyed by name where the names depend on what
  was loaded. A fixed-field class can't express "merge two collections" or "filter by name".
- **Pass-through configuration** forwarded to a library you don't control, e.g.
  `scatter_kws`: `Mapping[str, Any]` is honest.
- **The mapping is the abstraction**: registries, caches, sparse lookups, counts by label.

A fixed shape can still call for a different type:

- a real `dict` is structurally required (JSON interop, `**kwargs`, a wire contract):
  `TypedDict`;
- tuple behavior is needed (iteration, unpacking, backwards-compatible positional access):
  `NamedTuple`.
