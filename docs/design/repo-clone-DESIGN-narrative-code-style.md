# Narrative-flow code style

A Knuth-inspired convention for embedding enough prose in source files that a
reader scrolling top-to-bottom builds a working mental model — without ever
leaving the file. Described here for `repo-clone`, but the standard applies to
any shibuido tool.

---

## What the style is

Knuth's literate programming inverts the usual priority: prose drives, code
fills in the blanks. WEB tangles source from a narrative document. We don't go
that far — no preprocessor, no tangle/weave step, source files run directly.

What we borrow is the principle: the file should read. A thoughtful author is
walking you through. The reader anticipates a function before seeing it; by the
time they reach the definition, their question is already answered. The
alternative — blocks of code interspersed with comments that restate the obvious
— forces the reader to reconstruct intent from first principles. That's
archaeology, not reading.

## Why this matters in shibuido

Code is read far more than written. Many shibuido tools are single-file scripts
that grow to 500–2 000 lines. Future maintainers, AI agents reviewing the file,
and curious users skimming before running all need the same thing: a continuous
reading experience, not a scavenger hunt.

Inline narrative pays disproportionate dividends in single-file tools because
there is nowhere else to put the middle layer of explanation. A library project
can afford architecture diagrams and internal wikis. A 1 200-line script in
`~/.local/bin/` needs to carry its own context.

Consistency across tools multiplies the benefit: a contributor onboarding to a
new shibuido tool already knows where to look.

## Core principles

**1. Motivation precedes implementation.** The banner above a subsystem
explains the problem it solves, so the function signature that follows is the
obvious next step.

**2. Architectural map at the top.** Immediately after `from __future__ import
annotations`, list subsystems in file order and call out the 2-3 most
consequential design decisions with cross-links. No reader should have to
reverse-engineer the layout.

**3. Connect at boundaries.** Section transitions reference prior context and
motivate what comes next: "Now that we have X, we need Y because Z." No abrupt
context switches.

**4. Stream-of-consciousness at non-obvious branches.** When code makes a
surprising choice, explain the thinking inline: alternatives considered, the
constraint that ruled them out. Two to four lines beside the surprising line.
Not every branch — only the ones where a reader would otherwise pause.

**5. Surface external realities.** If the code is shaped by `gh`'s behavior,
GitHub's propagation lag, or a platform quirk, name it at the point it shapes
the code.

**6. Introduce domain terms on first use.** One sentence, narratively. Don't
assume domain fluency.

**7. Forward-reference signals.** If a section uses something defined later, say
so: "we'll use `ForkOptions` here — defined further down around line 720; treat
it as a bag of fork settings."

**8. Voice is present and consistent.** "We" voice sparingly but unmistakably.
The author is in the room. Don't overuse it; alternate with neutral statements.
Consistent register throughout — no formal-docstring → chatty-inline → formal
switching.

**9. Signposts in long orchestrators.** One-line markers at major decision
junctions in `main()`: `# --- dispatch: fork vs non-fork ---`. Navigation, not
prose.

**10. Inline is "enough to follow"; design docs hold the deeper truth.**
Cross-link, don't duplicate. When a why-note grows past ~6 lines, move it to a
design doc.

**11. Comments that age well.** No inline references to authors, dates, or issue
numbers. Those belong in commit messages and the decisions log — code outlives
its context.

**12. Balance.** A five-line helper doesn't need a ten-line banner. The right
amount of narrative is the amount that makes the surrounding code legible on
first read.

---

## Practical patterns

### Architecture summary at top of file

The module docstring gives the user-facing summary. Immediately after
`from __future__ import annotations`, an architecture note gives the
maintainer-facing one.

```python
from __future__ import annotations

# =============================================================================
# Architecture note for code readers
# =============================================================================
# Literate order: small primitives first, composing helpers next, orchestrators
# last, main() at the end. A reader scrolling top-to-bottom builds the model.
#
# Subsystems in file order:
#  1. Logging primitives   — log_info / log_warn / log_error / log_debug
#  2. Structured output    — emit(), set_output_mode(), human vs jsonl
#  3. gh preflight         — verify gh present and authed before fork work
#  ...
#
# Stdlib only: no third-party runtime deps (pyyaml is optional).
# See docs/design/repo-clone-DESIGN-stdlib-portability.md.
# =============================================================================
```

The note names subsystems in order, explains the ordering rationale, and calls
out the 2-3 most consequential design decisions with cross-links. Not exhaustive
— the design docs are. Enough to orient.

### Banner sections

A thin banner names. A connected banner contextualizes.

**Thin (names only — not enough):**
```python
# ---- Output / structured logging ----
```

**Connected (what problem, what solution, where to go):**
```python
# ---- Output / structured logging ----
# Now that we have basic logging primitives, we need a single output channel
# that supports both human-mode prefixed lines AND programmatic JSONL events.
# emit() is the preferred path. Two modes: "human" (INFO:/WARN:/ERROR: on stderr)
# and "jsonl" (one JSON object per line on stdout).
# See docs/design/repo-clone-DESIGN-jsonl-output.md.
```

### Continuity sentences at section transitions

The banner itself is the continuity sentence — it references what came before
and motivates what comes next. From `repo-clone`:

```python
# ---- Fork-side clone & upstream wiring ----
# After fork creation, two more things must happen: clone the fork side (with a
# small backoff for GitHub's occasional propagation lag) and wire the `upstream`
# remote pointing back at the source. The wiring is intentionally non-overwriting
# — if `upstream` already exists and points elsewhere, we WARN and leave it alone
# (the user may have set it deliberately). See docs/design/repo-clone-DESIGN-fork-flag.md.
```

### Inline why-notes at decision points

`clone_fork_with_retry` retries only on `CLONE_FAILED`, not every non-zero exit
code. Without a note, a reader wonders: bug or deliberate?

```python
# GitHub occasionally lags 1-3 s between fork creation and the new repo
# being clonable. We retry with exponential backoff for that specific case.
# Auth/permission/network errors are NOT propagation lag — they're terminal
# — so we surface them immediately rather than spin for 10 s with a
# misleading 'retrying' log.
if rc != ExitCode.CLONE_FAILED:
    return rc
```

The external reality (GitHub propagation lag), the constraint (other errors are
terminal), and the conclusion (return immediately) in four lines.

### Forward-reference signals

When a function uses a type defined 400 lines later, say so:

```python
# ForkOptions is defined further down around the dataclasses block (~line 720);
# treat it here as a bag of fork settings (target org, fork name, upstream URL).
def repo_clone_fork(url: str, options: CloneOptions, fork_opts: ForkOptions) -> int:
```

### Signposts in long functions

`main()` in `repo-clone` runs ~200 lines. One-line markers at decision
junctions keep it navigable:

```python
def main() -> int:
    # --- argument parsing ---
    # --- load config; merge env and CLI ---
    # --- apply output mode before any emit() calls ---
    # --- dispatch: fork path vs non-fork path ---
```

Navigation markers, not prose. Keep them terse.

---

## Voice and style notes

* **"We" voice** — sparingly, intentionally. One per few paragraphs is signal;
  one per sentence is noise. Alternate with neutral statements.
* **Active voice, present tense.** "We choose X" not "X was chosen."
* **Second person occasionally OK.** "You'll see the retry logic below" lands;
  don't lean on it.
* **Personable but professional.** No emoji except ⚠️ for genuine danger and
  🤖 for agent-facing pointers.
* **No hedging.** "This retries up to 10 s" beats "this may retry depending on
  the error." Specificity is a feature.
* **Don't anthropomorphize.** "The function decides..." obscures who is
  responsible. "We choose X here because..." is clearer.

---

## What goes where

Each layer has a job. They are not interchangeable.

| Layer | What lives here | Aimed at |
|---|---|---|
| **Inline** (code file) | Enough to read the file without leaving it: motivation, context, why-notes, cross-links | First-time reader of the script |
| **Design docs** (`docs/design/`) | Architectural decisions, schemas, contracts — the "why this shape" above any one function | Maintainers, contributors, agents |
| **Decisions log** (`maintainers/<tool>-MAINTAINERS-decisions.md`) | Dated record: choice, alternatives, rationale | Future maintainers asking "why not differently?" |
| **README / `<tool>.README.md`** | User-facing usage, features, options | Users |
| **DEV_NOTES** | Gotchas, edge cases, manual test recipes | Contributors actively changing code |

When an inline why-note grows past ~6 lines it is probably a design doc
fragment wearing an inline comment's clothing. Move it.

See `repo-clone-DESIGN-docs-architecture.md` for the full layer breakdown.

---

## Anti-patterns

* **Stating the obvious.** `# Increment counter` above `counter += 1` adds
  reading work for zero information gain.
* **Restating the docstring.** If the banner repeats the function's docstring
  word for word, one of them is wrong.
* **Walls of text before tiny functions.** A three-line helper doesn't need
  a ten-line biography.
* **Voice-switching mid-file.** Formal docstring → chatty inline → formal
  again reads like multiple authors arguing. Pick a register and hold it.
* **Comments that rot.** "Added by Alice in 2023 for issue #45" belongs in
  the commit message and the decisions log, not inline. Code outlives its
  authors and its issue numbers.
* **"Because reasons" without the reason.** `# Special case` is not an
  explanation. "Special case: `gh` returns exit 0 even when the fork exists,
  so we probe first" is.
* **Repeating yourself across layers.** If the design doc already covers
  the two-clone decision in depth, the inline note should be one sentence
  and a link — not a second full explanation.

---

## Before / after examples

### Example 1: Section banner

**Before** — reader knows they're in an output section. No idea why it exists
separately from the `log_*` primitives above, what "two modes" means, or where
to learn more.

```python
# ---- Output / structured logging ----

import json as _json
```

**After** — four lines; reader knows what came before, what the split is, and
where to go.

```python
# ---- Output / structured logging ----
# Now that we have basic logging primitives, we need a single output channel
# that serves both human-mode prefixed lines AND programmatic JSONL events.
# emit() is the preferred path. The legacy log_* helpers stay for older call sites.
# See docs/design/repo-clone-DESIGN-jsonl-output.md.

import json as _json
```

### Example 2: Inline why-note at a non-obvious guard

**Before** — why is `CLONE_FAILED` the only retriable code? Is this a bug?

```python
        if rc != ExitCode.CLONE_FAILED:
            return rc
```

**After** — the external reality, the constraint, and the conclusion.

```python
        # GitHub occasionally lags 1-3 s between fork creation and the new repo
        # being clonable. We retry with exponential backoff for that specific case.
        # Auth/permission/network errors are NOT propagation lag — they're terminal
        # — so we surface them immediately rather than spin for 10 s with a
        # misleading 'retrying' log.
        if rc != ExitCode.CLONE_FAILED:
            return rc
```

---

## Reusing this convention

Other shibuido tools should adopt this style. The standard is the same; only
the filename prefix changes.

1. Copy this doc into `docs/design/` and rename the prefix:
   `your-tool-DESIGN-narrative-code-style.md`.
2. Add an architecture note block after `from __future__ import annotations`
   (or after the opening shebang in non-Python scripts).
3. When writing banners, use the connected form: prior context, problem, link.
4. Retrofit older sections incrementally. A PR that adds banners and why-notes
   to one subsystem is valuable even if others are still thin.

For filename conventions, see `repo-clone-DESIGN-greppable-filenames.md`.
For the documentation layer architecture, see
`repo-clone-DESIGN-docs-architecture.md`.
