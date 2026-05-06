# Stdlib-only + shell-out to CLI tools

`repo-clone` is a single-file Python3 script that runs anywhere Python 3.9+ runs.
It uses **only the standard library** for hard dependencies. Optional features
may use third-party packages, but core paths must work without them.

## Why

* **Drop-in install.** `chmod +x repo-clone && mv repo-clone ~/.local/bin/` is
  the entire install procedure. No `pip install`, no `uv`, no venv.
* **Auditable.** A single file (~1000 lines) is reviewable in one sitting. No
  hidden transitive dependencies.
* **Boring, durable.** Stdlib doesn't break across Python releases the way
  pinned third-party packages do.

## What this means in practice

* **URL parsing:** `urllib.parse` + `re`, never `requests`.
* **YAML config:** the loader prefers `pyyaml` if installed (richer syntax),
  falls back to a tiny stdlib YAML-*subset* reader for the keys/types this tool
  uses (top-level keys, one level of nested mappings, strings, booleans, lists
  of strings).
* **HTTP:** if needed for a host-API check, use `urllib.request`. Otherwise
  shell out to `gh` (which already handles GitHub API auth, 2FA, SSO).
* **Process orchestration:** `subprocess.run` exclusively. No threads, no
  asyncio, no third-party process libs.

## When to shell out vs implement in Python

Shell out when an existing CLI does the work better and is widely available:

* `git` for all repo operations
* `gh` for GitHub API calls (Q1 decision in spec)
* `xclip` / `wl-paste` / `pbpaste` for clipboard reads (watch mode)
* `git-lfs` for Hugging Face

Stay in Python when the work is structural:

* URL parsing
* Config merging and precedence resolution
* Output formatting (human vs JSONL)
* Argument parsing

## YAML-subset reader: what's covered, what isn't

Covered (matches the schema in `repo-clone-DESIGN-config-file.md`):

```yaml
key: value
key: true
key: false
nested:
  key: value
  key: another
list_key:
  - item1
  - item2
```

Not covered (require pyyaml; reader errors with a clear pointer):

* Multi-line strings, anchors, aliases, tagged scalars
* Numbers (we don't need them; everything in our schema is string/bool/list)
* Mappings nested deeper than one level
* Inline `{...}` / `[...]` flow style

If your config triggers an unsupported construct, the reader prints:
`ERROR: config syntax beyond the stdlib subset; install pyyaml or simplify`.

## Reusing this principle

When extending `repo-clone` (or starting another shibuido tool):

1. Default to stdlib.
2. Shell out before adding a Python dependency.
3. Make richer dependencies (like `pyyaml`) **optional upgrades**, never
   required for core paths.
