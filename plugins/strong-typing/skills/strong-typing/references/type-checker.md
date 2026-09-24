# Type checker, ratchet, CI

Types pay off only if a checker enforces them, and a stringly-typed codebase cannot go strict
in one step.

## Checker

Pick one and make it the source of truth. **mypy** is the common CI default; **pyright** (the
engine behind Pylance) matches what VS Code users see. Target configuration for mypy in
`pyproject.toml`, introduced gradually:

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

## The ratchet

Flipping strict mode on a stringly-typed codebase produces hundreds of errors. Ratchet
instead:

1. **Add the checker green and loose.** A lax config so the baseline passes in CI.
2. **Tighten globally, exempt per module.** Strict flags on globally, relaxed for
   not-yet-converted modules:

   ```toml
   [[tool.mypy.overrides]]
   module = ["pkg.legacy.*", "pkg.not_yet_typed"]
   disallow_untyped_defs = false
   ```

3. **Remove each module from the override list in the change that converts it.** The list
   only shrinks.
4. **A new file is born typed.** Nothing is ever added back to the list.

## CI and pre-commit

- Run the checker in CI on every PR; make it blocking once the baseline is green.
- Run it in pre-commit on the whole package, not only changed files: type errors are
  non-local.

```yaml
# .pre-commit-config.yaml (illustrative)
-   repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.x.x
    hooks:
    -   id: mypy
        additional_dependencies: [<your typed deps and stubs>]
```

## Runtime checking in tests

Optionally enforce hints at runtime in the test suite with `beartype` or `jaxtyping`'s pytest
hook, catching violations across dynamic boundaries the static checker cannot see. A
nice-to-have, not a priority. Input validation for end users lives at the boundary and in
`__post_init__` regardless.
