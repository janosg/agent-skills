# Strong Typing — Tooling, Adoption, Docs

Types only pay off if a checker enforces them and the adoption path is incremental. This
file covers the type checker, CI/pre-commit, the order of adoption, runtime checking, and
documentation.

## Type checker

Pick one and run it in CI. **mypy** and **pyright** are both fine; pyright (the engine
behind Pylance/VS Code) gives the closest match to what users see in the editor, mypy is the
common CI default. Stick to one as the source of truth.

Target strict configuration (mypy `pyproject.toml`), introduced gradually:

```toml
[tool.mypy]
check_untyped_defs = true
disallow_any_generics = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_return_any = true
# strict = true   # turns on the above plus more once the codebase is ready
```

pyright equivalent (`pyrightconfig.json` or `[tool.pyright]`): `typeCheckingMode = "strict"`,
or `"basic"` while ramping up.

## Incremental adoption (the ratchet)

You cannot flip strict mode on a stringly-typed codebase without hundreds of errors. Ratchet
instead:

1. **Add the checker green, loose.** Get it running in CI with a lax config so the baseline
   passes.
2. **Tighten globally, exempt per-module.** Turn on strict flags globally, then relax them
   for not-yet-converted modules with per-module overrides:

   ```toml
   [[tool.mypy.overrides]]
   module = ["pkg.legacy.*", "pkg.not_yet_typed"]
   disallow_untyped_defs = false
   ```
3. **Convert low-level files first.** Start at the leaves of the import graph (utilities,
   data types) and work up toward public entry points. Leaf types become the vocabulary
   everything above reuses. Remove each module from the override list as you finish it — the
   list only ever shrinks (the ratchet).
4. **Never add back to the exemption list.** A new file must be born typed.

Design the types **before** adding hints (use `design-proposal`). Hints retrofitted without
a type design just document the stringly-typed status quo.

## CI and pre-commit

- Run the checker in CI on every PR; make it blocking once the baseline is green.
- Add it to pre-commit so errors surface before push. Run it on the whole package (not just
  changed files) so cross-file breakages are caught — type errors are non-local.

```yaml
# .pre-commit-config.yaml (illustrative)
-   repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.x.x
    hooks:
    -   id: mypy
        additional_dependencies: [<your typed deps and stubs>]
```

## Runtime type checking

Static checks don't reach users who never run a checker — so keep validating real inputs:
- validate at the public boundary (the generous-in/strict-out conversion site) and in
  dataclass `__post_init__`;
- give early, specific errors (e.g. `Enum("bad")` → a clear `ValueError`).

Optionally add runtime enforcement **in tests** with `beartype` or `jaxtyping`'s pytest hook
to catch hint violations the static checker can't see (e.g. across dynamic boundaries). This
is a nice-to-have, not a priority.

## Documentation

Once types carry the information, the docs should lean on them:

- **Remove type information from docstrings.** Types live in the signature; duplicating them
  in prose just rots. Keep docstrings for *meaning*, not types.
- **Document dataclass fields with a docstring *per attribute*, not one giant class
  docstring.** The class docstring covers the type as a whole — its purpose, behavior, and any
  cross-field rules; each field then carries its own docstring placed directly beneath it.
  Per-attribute docstrings are what an editor surfaces on hover for that specific field, and
  they don't rot into a stale wall of text the way a monolithic `Attributes:` block does.
  Describe meaning, ranges, and interactions — not the type, which is already in the annotation.
- **Feature the strict, autocomplete-friendly way most prominently.** When multiple input
  styles are supported (e.g. object vs string vs dict), show the discoverable one first;
  hide legacy/compat styles in tabs or expandable boxes rather than removing them.
- **Let autocomplete be the docs.** A configured-object API means options are discoverable in
  the editor; make sure the public types actually surface them (see `review-ergonomics`).

Per-attribute docstrings in practice — the class docstring explains the whole; each field
documents itself:

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

## Checklist

- [ ] One checker chosen and run in CI (blocking once green).
- [ ] Strict flags on globally; per-module overrides only for un-converted code.
- [ ] Override list shrinks over time; new files are born typed.
- [ ] Pre-commit runs the checker on the whole package.
- [ ] Low-level/leaf modules converted first.
- [ ] Runtime validation retained at boundaries and in `__post_init__`.
- [ ] Type info removed from docstrings; strict path featured in docs.
- [ ] Dataclass fields documented with per-attribute docstrings, not one class-level block.
