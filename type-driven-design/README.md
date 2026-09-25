# type-driven-design

Take Python from *stringly typed* to *strongly typed*: choose data types that make illegal
states unrepresentable, not just add hints.

The skill is user-invoked only (`disable-model-invocation: true`), so it never fires on its
own. Everything after the slash command becomes the request. The request decides the form of
the work; the principles and patterns in `SKILL.md` decide the substance.

```text
/type-driven-design <what you want, and where>
```

## Before you start

Expect a few questions up front. The skill asks instead of guessing when the answer depends
on things outside the code:

- **Library or internal code?** A published library needs a before/after and a compatibility
  decision for every public change; internal code skips the deprecation machinery.
- **Drop, deprecate, or keep?** For every public interface that changes, or as one global
  default. Answering in the request saves a round trip.
- **Which type checker?** Only asked if the repo has none configured. Otherwise the skill
  uses the one it finds in the checker config, CI, pre-commit, or dependency files.
- **Is pydantic an acceptable dependency?** It decides whether validation is written by hand
  in `__post_init__` or done declaratively.

## Requests

### Enhancement proposal

A written design for moving a whole library, or a large part of it, to strong typing.

```text
/type-driven-design write an enhancement proposal for moving this library to strong typing.
Public API, deprecate by default, one minor release of warnings.
```

The skill audits the codebase before proposing any hints: raw-string choices, dicts with
fixed keys, "kind" fields with optional attributes, `flag` + `flag_options` pairs, loose
public returns, mutable config, types that live only in docstrings. Per area the proposal
shows:

- the current situation, with a real excerpt;
- what to keep, so no useful capability is lost for type purity;
- the problems;
- before → after, and what each new type buys;
- for public symbols, the deprecation strategy.

A library proposal ends with flat lists of breaking changes and deprecations. It also says
which dicts, `Any`s, and mutable state deliberately stay as they are. If the repo has its own
enhancement-proposal format, the skill follows it.

### Migration plan

The order of work: a sequence of changes where each one can be merged on its own.

```text
/type-driven-design plan the migration of src/mypkg to strong typing as a sequence of PRs
```

The plan follows the order in `references/migration-strategy.md`:

1. tooling first, with unconverted modules exempted;
2. new types added alongside the old ones;
3. internals migrated behind an unchanged public surface;
4. the public surface accepts the new types too;
5. the compatibility decision applied;
6. deprecated paths removed.

Work starts at the leaves of the import graph and moves up toward the entry points. Pure
refactors and wide mechanical renames get their own changes, and any step that cannot be
kept small is flagged as large.

### Refactor of one module

The conversion itself, limited to the scope you name.

```text
/type-driven-design refactor the config module
/type-driven-design convert optimizers/options.py; internal code, breaking changes are fine
```

The skill picks patterns from the table in `SKILL.md` (enums, frozen dataclasses, tagged
unions, strategy objects, boundaries, ...) and reads each pattern's file, including the cases
where the loose version is correct, before applying it. It converts only what you named.
Anything else it notices goes into a follow-up list for you, not into the diff.

It counts as done when:

- the named scope is converted;
- its checker exemption, if it had one, is removed;
- the type checker and the tests pass on the touched package, and the real output is
  reported;
- no `# type: ignore`, `cast`, or `Any` was added to silence the checker. Any the skill
  finds are reported as findings, not presented as fixes.

### Review of a diff

A judgment on a change someone else, or an agent, wrote.

```text
/type-driven-design review this diff for over- and under-typing
/type-driven-design review PR #412
```

Over-conversion counts as much as under-typing. Examples of over-conversion: a single-use
Enum, a dataclass with no bug to prevent, a fake-precise type on a value that really is
arbitrary. Every touched signature and public return gets one verdict: over-typed,
under-typed, or correctly loose. The correctly loose ones are named explicitly so nobody
"fixes" them later.

### Type checker setup

```text
/type-driven-design set up strict type checking with a per-module ratchet in CI
```

Covered by `references/type-checker.md`. The checker starts green and loose. Strict settings
are then turned on globally, with per-module exemptions for code not yet converted. The
exemption list only shrinks, and new files are born typed. The checker runs on the whole
package in CI with a pinned version.

### Docstrings and user docs

```text
/type-driven-design move types out of the docstrings in mypkg/results.py
/type-driven-design update the user docs to feature the new config objects
```

Covered by `references/documentation.md`. Types move out of prose and into signatures, and
each dataclass field gets its own docstring directly beneath it, not an `Attributes:` block.
User docs show the strict, autocomplete-friendly way first, with legacy styles kept in tabs
or collapsible sections. The skill also flags what defeats autocomplete on public symbols:
`Any` returns, `**kwargs` used as the real option surface, signatures built at runtime.

## What is in the folder

| File | Holds |
| --- | --- |
| `SKILL.md` | The ten principles, the pattern index, working rules. Loaded on invocation. |
| `references/patterns/` | One file per pattern: `enum`, `dataclass-record`, `tagged-union`, `typestate`, `immutable-config`, `options-argument`, `strategy-object`, `pydantic-dataclass`, `boundary`. |
| `references/migration-strategy.md` | Library vs. internal code, the compatibility decision, what a proposal covers, the mergeable order. |
| `references/type-checker.md` | Strict config, the ratchet, CI and pre-commit. |
| `references/documentation.md` | Docstrings, user docs, what defeats autocomplete. |
