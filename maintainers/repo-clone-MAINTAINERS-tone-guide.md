# Tone guide for repo-clone

Three audiences, three voices. Use the right one for the surface you're editing.

## Audience 1: README + `--help` (humans)

**Voice:** down-to-earth builder. CLI-native. Concise. Gentle humor allowed.
Show, don't tell.

**Reference:** the existing pitch in `repo-clone.README.md` ("GitHub
window-shopping...") is the calibration target.

**Do:**

* Open with the user's pain, not the tool's features.
* One-line punchy claims, then code blocks that show them.
* Light humor where it lands ("window-shopping mode") — not every line.
* Respect newcomers without condescending.

**Don't:**

* Marketing fluff ("blazingly fast", "the best way to...").
* Multi-clause adjectives. Cut them.
* Over-explanation of obvious things.

**Before / after:**

> ❌ "repo-clone provides a sophisticated, ergonomic interface for the
>    intelligent organization of cloned repositories across multiple hosting
>    platforms..."
>
> ✅ "Paste a URL, get an organized clone. That's it."

## Audience 2: `--help-for-agents`

**Voice:** information density over politeness. Pseudocode, tables, equations,
fragments. No prose padding.

**See:** `docs/design/repo-clone-DESIGN-help-for-agents.md`.

**Do:**

* Tables for any list of more than 3 items.
* EBNF-style grammar for invocation.
* Crisp constraints ("idempotent ⇔ re-run is safe").
* Cite the issue tracker and troubleshooting URL.

**Don't:**

* Reassuring transitions ("Now let's look at...").
* Hedging ("may sometimes", "in certain cases").
* Restate `--help` content. Cross-link instead.

**Before / after:**

> ❌ "When you use the fork flag, the tool will first check whether you
>    have the gh CLI installed and properly authenticated, which is necessary
>    because we use it to..."
>
> ✅ "fork-path requires: gh ∈ PATH ∧ `gh auth status` = ok. Failure ⇒ exit 8."

## Audience 3: error messages

**Voice:** state what failed, where, and the next action. No apologies.

**Do:**

* Name the failed operation.
* Name the file/path/URL involved.
* State the next step the user can take (or a doc to read).
* Cite an exit code if it helps.

**Don't:**

* "Sorry, but...".
* Vague "something went wrong".
* Dump raw stack traces to the user (they go to `-vvv` debug output).

**Before / after:**

> ❌ "Error: failed."
>
> ✅ "ERROR: fork creation failed for acme/widget → VariousForks/widget-by-acme-fork
>     gh exited 1: HTTP 422 Unprocessable Entity (name already exists at target_org)
>     Fix: pick a different --fork-name, or check existing repo at:
>          https://github.com/VariousForks/widget-by-acme-fork
>     Exit 10."

## Audience 4: commit messages

Defer to `ai_docs/small_descriptive_commits.md` and
`ai_docs/meta_commit_motivation.md` in the parent shibuido repo. Do not
duplicate that guidance here.

Quick reminders:

* Prefixes: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `META:`, `chore:`,
  `WIP:` (for failing-test commits).
* Small commits, one logical change each.
* Subject ≤ 72 chars; body wraps at ~72.
* Co-author footer for AI-assisted commits.

## When in doubt

Ask: "what is the reader trying to do, in this exact moment?" Write the
sentence that helps them most.
