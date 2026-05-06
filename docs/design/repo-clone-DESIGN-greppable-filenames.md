# Greppable, fuzzy-searchable filenames

Filenames in this repo follow the pattern:

    <tool>-<KIND>-<topic>.md

Examples:

* `repo-clone-DESIGN-fork-flag.md`
* `repo-clone-MAINTAINERS-tone-guide.md`
* `repo-clone-TROUBLESHOOTING.md`

## Why

* **Grep / fuzzy-search friendly.** `sk -f DESIGN | head` lists every design doc
  in one shot. `fzf -q MAINTAINERS` jumps straight to maintainer notes.
* **Tool-prefixed.** When a directory aggregates docs from multiple tools (which
  shibuido superrepos do via symlinks), the prefix disambiguates.
* **KIND in caps.** Visually scannable; case-sensitive matches ignore prose.

## Usage examples

```bash
# Find every design doc related to fork
sk -f DESIGN-fork

# Find every troubleshooting page across the workstation
find ~/github/shibuido -name '*-TROUBLESHOOTING.md'

# Open the spec for one tool
fzf -q "repo-clone-DESIGN" </dev/null
```

## Anti-patterns

* `design.md`, `notes.md`, `README.md` inside a `docs/` subdirectory — fine for a
  single-tool repo, breaks down once a superrepo aggregates many tools.
* Lowercase KIND (`repo-clone-design-...`) — defeats case-sensitive scanning.
* Sentence-spaced filenames — hostile to shell tab-completion.

## Reusing this convention

Other shibuido tools should adopt the same pattern. Copy this file (rename the
prefix), or link to it from your own docs.
