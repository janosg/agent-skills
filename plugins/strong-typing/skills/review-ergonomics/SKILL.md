---
name: review-ergonomics
description: Review code for the user-facing payoff of strong typing — does it actually produce good IDE autocomplete/discoverability, and does the documentation leverage the types? Catches Any/**kwargs/dynamic signatures that kill autocomplete, loose public return types that block attribute completion, types in docstrings that should be removed, and docs that feature the stringly-typed path over the autocomplete-friendly one. Use to review a strong-typing diff/PR or public API for ergonomics. Reviews, does not rewrite. Triggers: "does this give good autocomplete", "review the typing ergonomics", "do the docs leverage the types", "review the IDE experience of this API".
---

# Strong Typing Review — Ergonomics (Autocomplete & Docs)

Strong typing is only worth it if it pays off for the human at the keyboard: **discoverable
options via autocomplete** and **documentation that leans on the types.** This skill checks
that the type design actually delivers that, and flags the things that quietly defeat it.

Reviews and recommends; **does not rewrite.**

## 0. Load the canon

- `${CLAUDE_PLUGIN_ROOT}/references/philosophy.md`
- `${CLAUDE_PLUGIN_ROOT}/references/patterns.md`
- `${CLAUDE_PLUGIN_ROOT}/references/tooling.md` (the Documentation section)

## 1. Set the scope

Default to the **current diff / the public surface it touches**. Whole-repo or whole-API on
request. Focus on **public, user-facing** symbols — ergonomics is about the caller's
experience. State the scope used.

## 2. Autocomplete & discoverability checklist

Reason about what an editor (Pylance/pyright, PyCharm) would show a caller:

- [ ] **`Any` in public signatures / return types** — kills attribute autocomplete on the
      result. A public function returning `Any`/`dict[str, Any]` gives the caller nothing to
      complete. → strict return type.
- [ ] **`**kwargs` as the real option surface** — options hidden behind `**kwargs` don't
      autocomplete. If a fixed set of options exists, they should be explicit
      parameters/fields (or a config object whose fields complete).
- [ ] **dynamic signatures** (`functools.wraps`, signature built at runtime, decorators that
      erase params) — most editors won't complete these. Flag where autocomplete is silently
      lost, and note the trade-off.
- [ ] **config via a dict** where an object would complete — `options={"timeout": 30}`
      offers no completion; a configured object/`with_*` constructor does.
- [ ] **discoverability of choices** — when there are many valid options (algorithms, modes),
      can the user *find* them by typing `lib.<TAB>`? Consider whether a typed namespace /
      selection object would surface them (and, with many options, a filtered/categorized
      selection). Note where the only way to discover a value is reading the docs.
- [ ] **strict-out enables completion** — confirm public returns are concrete types whose
      attributes complete, not unions the user must `isinstance`-narrow first.
- [ ] **overly-nested types** that, while precise, make signatures unreadable in tooltips →
      named intermediates read better.

## 3. Documentation-leverages-types checklist

- [ ] **type info duplicated in docstrings** (`param x: int — ...`) that now lives in the
      signature → should be removed; docstrings describe *meaning*, not types.
- [ ] docstrings/examples that **feature the stringly-typed / dict path** instead of the
      strict, autocomplete-friendly one → the discoverable way should be shown first; legacy
      shown in tabs/expandable blocks, not removed (for libraries).
- [ ] public types/fields **missing docstrings** where the meaning isn't obvious from the name.
- [ ] examples that **don't import/use the new types**, so readers never learn the good path.
- [ ] generated/stub considerations: if codegen produces the public types, confirm the result
      is what an editor sees (the user shouldn't be able to tell it was generated).

## 4. Report

For each finding:

- **file:line**, one-line description
- **category**: autocomplete / discoverability / docs
- **severity**: high (public ergonomics broken — e.g. `Any` return on a key API) / medium /
  low
- **before → after** sketch, and **what the editor would show before vs after**
- **why** it matters to the caller

Call out wins too (places where autocomplete/docs are already good). Where a dynamic signature
is a deliberate, documented trade-off, note it rather than flagging it as a pure defect. Do
not edit files unless asked to apply a finding.
