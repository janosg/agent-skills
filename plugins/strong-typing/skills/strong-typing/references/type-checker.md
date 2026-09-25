# Type checker, ratchet, CI

Types pay off only if a checker enforces them, and a stringly-typed codebase cannot go strict
in one step.

## Checker

One checker is the source of truth. Contributors' editors may run a different one, but CI
decides. Which one is the user's choice, not the skill's: infer it from the repo (checker
config, CI workflows, pre-commit hooks, dependency files), and if nothing is set up, ask
before configuring anything. Every mainstream checker has a strict mode; that is the target.
Whatever the checker calls them, the settings that matter are:

- **Untyped and partially typed functions are errors.** Otherwise the checker silently skips
  the bodies of exactly the functions that need it most.
- **Bare generics are errors.** `list` and `dict` without element types hide the same
  information a missing annotation does.
- **Implicit Optional is an error.** A `= None` default does not make the parameter
  `T | None`; the annotation has to say so.
- **Any leaking out of a typed function is an error.** A function that returns `Any` is a
  hole through which untyped values flow into typed code.
- **Redundant casts and unused suppressions are errors.** A suppression that is no longer
  needed should disappear, not outlive the problem it was added for.

## The ratchet

Flipping strict mode on a stringly-typed codebase produces hundreds of errors. Ratchet
instead:

1. **Add the checker green and loose.** A lax config so the baseline passes in CI.
2. **Tighten globally, exempt per module.** Strict settings on globally, relaxed for
   not-yet-converted modules through the checker's per-module override mechanism.
3. **Remove each module from the exemption list in the change that converts it.** The list
   only shrinks.
4. **A new file is born typed.** Nothing is ever added back to the list.

## CI and pre-commit

- Run the checker in CI on every PR; make it blocking once the baseline is green.
- Run it on the whole package, not only changed files: type errors are non-local. Changing
  a signature breaks callers in files the diff never touched.
- Pin the checker version so an upgrade is a deliberate, separate change rather than a
  surprise on an unrelated PR.

## Runtime checking in tests

Optionally enforce hints at runtime in the test suite with a runtime type-checking library,
catching violations across dynamic boundaries the static checker cannot see. A nice-to-have,
not a priority. Input validation for end users lives at the boundary and in `__post_init__`
regardless.
