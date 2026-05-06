# Documentation architecture

Why this repo has the documentation shape it has, and where each kind of
content belongs. Reusable principle for other shibuido tools.

## The four roles

| Doc | Role | Audience | Length |
|---|---|---|---|
| `README.md` (repo root) | GitHub landing page; hook in 30s; route outward | First-time visitors | ≤ one screen |
| `<tool>.README.md` | Comprehensive man-page-quality user manual | Users actually using the tool | Tiered, can be long |
| `docs/design/<tool>-DESIGN-<topic>.md` | Scoped design rationale; one topic per file | Maintainers, future contributors, agents | ≤ 3 screens |
| `docs/<tool>-TROUBLESHOOTING.md` | Symptom-keyed flat list of fixes | Stuck users + agents | Long but scannable |
| `maintainers/<tool>-MAINTAINERS-<topic>.md` | Internal-facing notes | Contributors | As needed |

## Cognitive learning curve

Every doc starts with the simplest working example a reader can copy. Each
subsequent section asks the reader to know one more thing — never two. Every
section ends with an explicit "next-step" link pointing one rung up.

```
README.md  →  <tool>.README.md (Quickstart)
            →  <tool>.README.md (Configuration tier)
            →  docs/design/<tool>-DESIGN-config-file.md
            →  docs/design/<tool>-DESIGN-multi-account-config-layout.md
```

## Cross-linking convention

Every doc links **up** (where you came from / parent context) and **down**
(where to go next). No orphan docs.

## Why two READMEs (not one)

* `README.md` (root): GitHub renders this on the repo landing page. Visitors
  decide in 30 seconds whether the tool fits. Conciseness is the whole
  point. Long content here is a feature-discovery failure.
* `<tool>.README.md`: where users actually live once they've decided to use
  the tool. Tiered structure: zeroconf → basic → advanced → reference.
  Each tier self-contained — readers can stop at any level.

## Why scoped design docs (not one big DESIGN.md)

* **Greppable** (see `repo-clone-DESIGN-greppable-filenames.md`).
* **Reusable** — other tools can copy or link to single docs.
* **Independently editable** — a config-file change doesn't touch the
  fork-flag doc; reviews stay focused.
* **Bounded reading time** — a contributor onboarding can skim
  `docs/design/` in one sitting.

## Why a separate `maintainers/` directory

Internal-facing notes (tone guide, decisions log, contributor orientation)
shouldn't clutter user-facing docs. Users don't need to read about how the
project is run. Maintainers do.

## Reusing this architecture

Other shibuido tools should adopt the same shape. Copy this doc (rename the
prefix), then:

1. Repo-root `README.md` → landing page.
2. `<tool>.README.md` → user manual.
3. `docs/design/<tool>-DESIGN-*.md` → small scoped design docs.
4. `docs/<tool>-TROUBLESHOOTING.md` → symptom-keyed fixes.
5. `maintainers/<tool>-MAINTAINERS-*.md` → internal notes.
