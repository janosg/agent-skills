# Type checker, ratchet, CI

## Checker

One checker is the source of truth. Contributors' editors may run a different one, but CI
decides. The target is the checker's strict mode. Where it is assembled from individual
settings, make sure untyped functions are errors: otherwise the checker skips their bodies,
exactly the code that needs it most.

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

- Make the checker blocking in CI once the baseline is green, and run it on the whole
  package, not only changed files: changing a signature breaks callers in files the diff
  never touched.
- Pin the checker version so an upgrade is a deliberate, separate change rather than a
  surprise on an unrelated PR.
