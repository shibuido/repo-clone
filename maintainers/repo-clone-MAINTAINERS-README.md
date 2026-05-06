# Maintainer's orientation

Welcome. This file points you at everything you need to make a clean
contribution to `repo-clone`.

## The repo in 30 seconds

* One Python3 stdlib script (`repo-clone`), no extension, executable.
* Optional `pyyaml` for richer config; not required.
* Shells out to `git`, `gh` (for `--fork`), and clipboard tools (for `--watch`).
* All docs are markdown with greppable filenames. See
  `docs/design/repo-clone-DESIGN-greppable-filenames.md`.

## Read these before touching code

In order:

1. `repo-clone.README.md` — what the tool does (user perspective).
2. `repo-clone.DEV_NOTES.md` — implementation rationale, edge cases.
3. `docs/design/` — every design rationale, scoped one-topic-per-file.
4. `maintainers/repo-clone-MAINTAINERS-tone-guide.md` — voice for every
   surface you might edit.
5. `maintainers/repo-clone-MAINTAINERS-decisions.md` — why we chose what
   we chose.

## Code organization

The script is structured in literate ordering:

```
1. Imports, constants, exit codes, dataclasses
2. Small helpers (logging, URL parsing, etc.)
3. Composing functions (clone_repo, update_repo, watch_clipboard)
4. Fork-related sections (banner-comment delimited):
     # ---- Config file loading ----
     # ---- Output / structured logging ----
     # ---- gh preflight ----
     # ---- Fork identity resolution ----
     # ---- Fork creation (idempotent) ----
     # ---- Fork-side clone & upstream wiring ----
     # ---- Agent help ----
5. main()  — entry point at the bottom
```

When adding code, follow this order: define helpers first, the function that
composes them later. Reading the file top-to-bottom should explain the
*why* before the *how*.

## Adding a new feature

1. **Open an issue** with the proposed change (label `byAI` if AI-authored).
2. **Brainstorm** in the issue or a `ramblings/` doc.
3. **Write a design doc** in `docs/design/repo-clone-DESIGN-<topic>.md`
   if the change has shape (more than a one-liner fix).
4. **Implement in small commits.** See `ai_docs/small_descriptive_commits.md`
   in the parent shibuido repo.
5. **Update the user manual** (`repo-clone.README.md`) and (if applicable)
   `--help` and `--help-for-agents`.
6. **Add a manual test case** to `repo-clone.DEV_NOTES.md`.
7. **Append a decision entry** to
   `maintainers/repo-clone-MAINTAINERS-decisions.md` if there's a non-obvious
   choice worth recording.

## Adding a new design doc

1. Filename: `docs/design/repo-clone-DESIGN-<topic>.md`. Lowercase topic,
   kebab-case.
2. Length target: ≤ 3 screens. If longer, split.
3. Cross-link up (where this doc fits) and down (related docs).
4. Add it to the cross-link list in any doc it relates to.

## Running manual tests

See `repo-clone.DEV_NOTES.md` § Manual Test Cases. Run before opening a PR.

## Commit conventions

Defer to `ai_docs/small_descriptive_commits.md` and
`ai_docs/meta_commit_motivation.md` in the parent shibuido repo. Quick
summary: small, prefixed (`feat:`, `fix:`, `docs:`, `META:`, etc.), one
logical change each, AI co-author footer where relevant.

## When you're stuck

* Issue tracker: https://github.com/shibuido/repo-clone/issues
* Other shibuido tools may have solved a similar problem; check
  `~/github/shibuido/`.
