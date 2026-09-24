# Migration strategy

How to take a whole library or application from stringly typed to strongly typed without
breaking users or producing unreviewable diffs. For a single module with no external callers
most of this collapses to: convert it, remove its checker exemption, keep the tests green.

## Repo type sets the treatment

- **Library / public API.** Packaged and published, an `__all__` or documented surface,
  imported by other projects. Every public change needs a before/after and a compatibility
  decision.
- **Internal / application code.** You control all call sites. Breaking changes are usually
  fine; skip the deprecation machinery and focus on the type design.

Ask when it is ambiguous.

## The compatibility decision

For each affected public interface, or as a global default, one of:

- **Drop.** A clean breaking change; the old interface goes now.
- **Deprecate.** The old interface keeps working and emits a `DeprecationWarning` (not an
  error) that names the replacement and the removal target, for a release or two.
- **Keep.** Both interfaces supported indefinitely, no warning.

The answer depends on context outside the code: user count, stability promises, release
cadence. Ask the maintainer and record the answer; it shapes every public-facing section of a
proposal and every step of the implementation. Interfaces marked deprecate or keep stay
working through the generous-in boundary: accept the old form, convert, and for deprecate,
warn. Internal-only interfaces get no shims.

## Design the types before adding hints

Hints retrofitted onto a stringly-typed design only document the status quo. Audit first:
choices as raw strings, dicts with fixed keys, tuples of 3+ values, "kind" fields with
optional attributes, many-optional `__init__`s, mutable config, `flag` + `flag_options` pairs,
public returns of dicts or loose unions, god-objects with ordered-call rulebooks, stray `Any`,
type information living only in docstrings. For each: where, current type, proposed type,
public or not. Mark explicitly the dicts, `Any`s, and mutable state that stay as they are, so
nobody "fixes" them later.

A written proposal follows the repo's own enhancement-proposal conventions where it has any.
Per area it covers: the current situation with a real excerpt, what to keep (losing a useful
capability to gain type purity is a regression), the problems, before → after with the chosen
types, which payoff each type buys, and for public symbols the deprecation strategy. A library
proposal ends with flat lists of breaking changes and deprecations; an internal-code proposal
drops the compatibility sections and stays short.

## The order that keeps every step mergeable

Each step leaves the codebase importable, checker-clean to the extent already ratcheted, and
green. Prefer this order:

1. **Tooling first.** Checker config, pre-commit, CI, with not-yet-converted modules exempted
   so the baseline is green (`type-checker.md`).
2. **Introduce new types additively.** New dataclasses, enums, and unions alongside the old
   interfaces, unused or internal-only. Pure addition, trivial to review.
3. **Migrate internals.** Switch internal call sites to the strict types behind an unchanged
   public surface; the boundary converts old → new.
4. **Public surface accepts the new types** alongside the old. Still non-breaking.
5. **Apply the compatibility decision.** Deprecated paths start warning and the docs feature
   the new way; dropped paths are removed here.
6. **Remove deprecated paths** in the targeted release, usually far later.

Start at the leaves of the import graph (utilities, data types) and work up toward entry
points: leaf types become the vocabulary everything above reuses. Remove a module from the
checker exemption list in the same change that converts it; the list only shrinks.

## Splitting changes that resist being small

- **Behavior-preserving refactor first, then types.** If a function must be restructured
  before it can be typed cleanly, land the pure refactor separately so it reviews against
  unchanged tests.
- **Isolate wide mechanical changes** (a key renamed across many files, one dict swapped for
  a dataclass everywhere) into a change that does only that, so the diff is boring.
- **An unavoidably large step** gets a documented sub-structure and is flagged as large.
- **Internal interfaces** with no external users change in one breaking step.
