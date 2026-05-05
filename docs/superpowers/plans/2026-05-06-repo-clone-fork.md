# repo-clone `--fork` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `--fork` (with `--fork-all`, `--fork-default`, `--fork-org`, `--fork-name`) plus structured output (`--jsonl`), `--help-for-agents`, a config-file extension, and a documentation/maintainer scaffold to the existing `repo-clone` script.

**Architecture:** The existing single-file `repo-clone` Python3 stdlib script grows in place — new code added in literate-ordered banner-comment sections (small helpers → composing functions → wiring in `main`). Fork creation shells out to `gh` (preflight-checked). Source and fork land in two separate directories under the standard `$base/$host/$org/$repo` layout; fork side gets an `upstream` remote auto-wired. Config file at `~/.config/shibuido/repo-clone/repo-clone-config.yaml` gains new `fork:` and `output:` sections; loaded with pyyaml when available, falls back to a tiny stdlib YAML-subset reader.

**Tech Stack:** Python 3.9+ stdlib only (pyyaml optional); shells out to `git`, `gh`, and (already) clipboard tools. No new third-party dependencies.

**Source spec:** [`docs/superpowers/specs/2026-05-05-repo-clone-fork-design.md`](../specs/2026-05-05-repo-clone-fork-design.md) (commit `f19240f`).

**Testing approach:** Pytest is FUTURE_WORK per spec §13. This plan uses two verification styles:
* For pure functions: inline `python3 -c "..."` smoke tests that import the module and assert behavior.
* For subprocess flows: `--dry-run` invocations and shell-level inspection of stderr/stdout. Manual test scripts land in `repo-clone.DEV_NOTES.md` per existing convention.

The TDD rhythm (write check → see fail → implement → see pass → commit) is preserved using these mechanisms.

---

## Discrepancies discovered during planning (Task 0 fixes the spec)

| Spec said | Reality | Resolution |
|---|---|---|
| New exit codes 7/8/9/10 | Existing code 7 = `NO_CLIPBOARD_TOOL` | Renumber new codes to **8/9/10/11** |
| Config at `config.yaml` | Existing config is `repo-clone-config.yaml` | Keep existing name; matches greppable convention |
| `WARNING:` prefix | Existing code uses `WARN:` | Standardize on **`WARN:`** (existing); spec updated |

These adjustments preserve backward compatibility for existing users and existing exit-code consumers.

---

## File structure (created or modified)

**Created (new files):**

```
repo-clone/
├── README.md                                                   (NEW landing page)
├── docs/design/
│   ├── repo-clone-DESIGN-fork-flag.md
│   ├── repo-clone-DESIGN-config-file.md
│   ├── repo-clone-DESIGN-help-for-agents.md
│   ├── repo-clone-DESIGN-jsonl-output.md
│   ├── repo-clone-DESIGN-output-prefixes.md
│   ├── repo-clone-DESIGN-stdlib-portability.md
│   ├── repo-clone-DESIGN-greppable-filenames.md
│   ├── repo-clone-DESIGN-docs-architecture.md
│   └── repo-clone-DESIGN-multi-account-config-layout.md
├── docs/repo-clone-TROUBLESHOOTING.md
└── maintainers/
    ├── repo-clone-MAINTAINERS-README.md
    ├── repo-clone-MAINTAINERS-tone-guide.md
    └── repo-clone-MAINTAINERS-decisions.md
```

**Modified:**

```
repo-clone                                  (the script — main growth target)
repo-clone.README.md                        (rewritten into tiered structure)
repo-clone.DEV_NOTES.md                     (lightly updated; new test cases added)
repo-clone.FUTURE_WORK.md                   (items moved out as they land)
docs/superpowers/specs/2026-05-05-repo-clone-fork-design.md   (Task 0 corrections)
```

---

## Task ordering rationale

1. **Task 0** corrects spec discrepancies first (so all downstream tasks reference correct values).
2. **Tasks 1–14** scaffold all documentation BEFORE code lands — this lets later code commits link to design docs that already exist, and gives the engineer a complete mental model of the system before touching the script.
3. **Tasks 15–19** add the config-and-output plumbing — pure-function-heavy, easy to verify in isolation.
4. **Tasks 20–25** add the fork identity and creation logic — gh preflight, identity resolution, idempotent fork creation.
5. **Tasks 26–28** wire the fork-side clone path and upstream remote.
6. **Tasks 29–31** add `--help-for-agents`.
7. **Tasks 32–35** wire everything into argparse / `main()` and run end-to-end manual tests.
8. **Tasks 36–38** finalize: rewrite `repo-clone.README.md`, write the new repo-root `README.md`, trim `FUTURE_WORK.md`, update `DEV_NOTES.md`.

---

## Task 0: Correct spec discrepancies

**Files:**
- Modify: `docs/superpowers/specs/2026-05-05-repo-clone-fork-design.md`

- [ ] **Step 1: Open the spec and locate the three problem areas**

Targets: §3 CLI surface (no change needed), §5 config (filename), §6 output prefixes (`WARNING:` → `WARN:`), §8 error matrix and exit-codes paragraph (renumber 7→8, 8→9, 9→10, 10→11).

- [ ] **Step 2: Apply the three corrections**

In §5 ("Location: ..."), keep filename as `repo-clone-config.yaml`:

```
**Location:** `~/.config/shibuido/repo-clone/repo-clone-config.yaml` (respects `$XDG_CONFIG_HOME`).
```

In §6 ("Human mode:") replace `WARNING:` with `WARN:` (two occurrences) and adjust the paragraph that says "no `INFO:` strings; instead each event gets `\"level\": \"info\"|\"warning\"|\"error\"`" to use `"warn"` to match. Note: human prefix is `WARN:`; JSONL `level` value is `"warn"`.

In §8 error matrix, renumber the four new codes:

```
| `gh` missing or unauthed                                | ERROR with actionable hint                       | 8 (new)  |
| Non-GitHub URL with `--fork`                            | ERROR pointing at FUTURE_WORK                    | 9 (new)  |
| Fork already exists on GH but `parent` mismatches source| ERROR (name collision with unrelated repo)       | 11 (new) |
| Fork creation API call fails                            | ERROR; source clone is left as-is (no rollback)  | 10 (new) |
```

And below the table:

```
**Newly minted codes:** 8 (`gh` missing/unauth), 9 (host not yet supported for fork),
10 (fork creation API failure), 11 (fork name collision with unrelated repo).
Existing codes 1–7 retain their current meanings (7 = NO_CLIPBOARD_TOOL).
```

- [ ] **Step 3: Commit**

```bash
cd /home/gw-t490/github/shibuido/repo-clone
git add docs/superpowers/specs/2026-05-05-repo-clone-fork-design.md
git commit -m "docs(spec): fix three discrepancies discovered during planning

* Exit codes 7/8/9/10 → 8/9/10/11 (existing code 7 = NO_CLIPBOARD_TOOL)
* Config filename stays repo-clone-config.yaml (matches existing convention,
  preserves backward compat for existing users)
* Output prefix WARNING: → WARN: (existing log_warn already prints WARN:)

Discovered while reading the existing script during plan authoring.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 1: Create directory scaffold

**Files:**
- Create: `docs/design/` (directory)
- Create: `maintainers/` (directory)

- [ ] **Step 1: Create directories**

```bash
cd /home/gw-t490/github/shibuido/repo-clone
mkdir -p docs/design maintainers
```

- [ ] **Step 2: Add `.gitkeep` files so empty dirs commit cleanly**

```bash
touch docs/design/.gitkeep maintainers/.gitkeep
```

- [ ] **Step 3: Commit**

```bash
git add docs/design/.gitkeep maintainers/.gitkeep
git commit -m "chore: scaffold docs/design and maintainers directories

Empty placeholders; populated by subsequent commits.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Write `docs/design/repo-clone-DESIGN-greppable-filenames.md`

**Files:**
- Create: `docs/design/repo-clone-DESIGN-greppable-filenames.md`
- Delete: `docs/design/.gitkeep` (no longer needed once dir has content)

This doc lands first because all other design docs reference its naming convention.

- [ ] **Step 1: Write the doc**

```markdown
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
```

- [ ] **Step 2: Verify it renders**

```bash
head -30 docs/design/repo-clone-DESIGN-greppable-filenames.md
```

Expected: clean markdown output, no obvious typos.

- [ ] **Step 3: Commit**

```bash
git rm docs/design/.gitkeep
git add docs/design/repo-clone-DESIGN-greppable-filenames.md
git commit -m "docs(design): add greppable filenames convention

Reusable across shibuido tools. Establishes the
<tool>-<KIND>-<topic>.md pattern used by every doc in this repo.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Write `docs/design/repo-clone-DESIGN-stdlib-portability.md`

**Files:**
- Create: `docs/design/repo-clone-DESIGN-stdlib-portability.md`

- [ ] **Step 1: Write the doc**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add docs/design/repo-clone-DESIGN-stdlib-portability.md
git commit -m "docs(design): add stdlib portability principle

Codifies stdlib-only + shell-out-to-CLI as the project guideline,
including the YAML-subset reader scope.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Write `docs/design/repo-clone-DESIGN-config-file.md`

**Files:**
- Create: `docs/design/repo-clone-DESIGN-config-file.md`

- [ ] **Step 1: Write the doc**

```markdown
# Config file

**Path:** `~/.config/shibuido/repo-clone/repo-clone-config.yaml`
(respects `$XDG_CONFIG_HOME`; if set, replaces `~/.config`).

Loaded if present, ignored if absent. Forward-compatible layout (see
`repo-clone-DESIGN-multi-account-config-layout.md`).

## Schema

```yaml
# Existing keys (watch mode):
extra_hosts:                       # Additional hosts beyond BUILTIN_HOSTS
  - gitlab.mycompany.com
watch_interval: 0.5                # Polling interval (seconds)
watch_selection: clipboard         # clipboard | primary | both
clipboard_command: null            # Override auto-detected clipboard tool
base_dir: ~/repos                  # Override $HOME for clone destinations

# New: fork section
fork:
  target_org: VariousForks         # Default: gh-authed user
  name_template: "{repo}-by-{org}-fork"   # Default: "{repo}"
  all_branches: false              # Default: false (matches GitHub webui)
  upstream_remote_name: upstream   # Default: "upstream"

# New: output section
output:
  mode: human                      # human | jsonl
  show_info: false                 # If true, INFO: visible at -v0
```

## Precedence (highest wins)

```
CLI flag  >  env var  >  config file  >  built-in default
```

## Env vars

| Var | Maps to |
|---|---|
| `REPO_CLONE_BASE_DIR` | `base_dir` |
| `REPO_CLONE_VERBOSE` | verbosity (0..3) |
| `REPO_CLONE_FORK_ORG` | `fork.target_org` |

## Loader behavior

1. If `~/.config/shibuido/repo-clone/repo-clone-config.yaml` exists, parse it.
2. Else, if `~/.config/shibuido/repo-clone/profiles/default.yaml` exists,
   parse it (forward-compat path; see multi-account layout doc).
3. Else, use built-in defaults.
4. Try `pyyaml` first; on `ImportError`, fall back to the stdlib YAML-subset
   reader (`repo-clone-DESIGN-stdlib-portability.md`).
5. On parse failure, print `ERROR:` and exit nonzero (do not silently use
   defaults — config errors should be loud).

## Why one file, not multiple profiles (yet)

v1 keeps it simple: one user, one config. Power users with multi-account
needs use `--config PATH` to point at an alternate file. Profile support is
forward-compatible (see multi-account layout doc) but **not implemented**.

## Next step

* Loader implementation: see `repo-clone` source, section
  `# ---- Config file loading ----`.
* Multi-account layout: `repo-clone-DESIGN-multi-account-config-layout.md`.
```

- [ ] **Step 2: Commit**

```bash
git add docs/design/repo-clone-DESIGN-config-file.md
git commit -m "docs(design): add config file design doc

Schema, precedence, env vars, loader behavior. Cross-links stdlib-portability
and multi-account-config-layout.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Write `docs/design/repo-clone-DESIGN-multi-account-config-layout.md`

**Files:**
- Create: `docs/design/repo-clone-DESIGN-multi-account-config-layout.md`

- [ ] **Step 1: Write the doc**

```markdown
# Multi-account config layout (forward-compatible; not implemented in v1)

## What ships in v1

```
~/.config/shibuido/repo-clone/
└── repo-clone-config.yaml         # the only config file v1 reads
```

The loader also accepts `profiles/default.yaml` as an equivalent alias.
Both are documented; `repo-clone-config.yaml` is the canonical location.

## What v1 does NOT do

* No `--profile NAME` flag.
* No multi-account switching.
* If you need an alternate config for one invocation, use `--config PATH`.

## Forward-compatible layout

When profile support lands (deferred to FUTURE_WORK), the directory grows
**additively** — no migration required:

```
~/.config/shibuido/repo-clone/
├── repo-clone-config.yaml         # equivalent to profiles/default.yaml
└── profiles/
    ├── default.yaml               # default profile
    ├── work.yaml                  # --profile work
    └── personal.yaml              # --profile personal
```

Resolution rules (proposed for future flag):

1. `--config PATH` → that file, ignore everything else.
2. `--profile NAME` → `profiles/NAME.yaml`, ERROR if missing.
3. Default → `repo-clone-config.yaml` if present, else
   `profiles/default.yaml`, else built-in defaults.
4. If `repo-clone-config.yaml` AND `profiles/default.yaml` both exist with
   conflicting contents, WARN and prefer `repo-clone-config.yaml`.

## Why this layout

* **Zeroconf-friendly.** New users have one file to read, one path to
  document.
* **Doesn't punish single-account users with profile boilerplate.**
* **Migration-free.** Existing `repo-clone-config.yaml` stays where it is
  forever; profile support layers on top.

## When to implement

When at least one of these is true:

* Real demand from users with multiple GitHub accounts.
* A second host (GitLab) lands and benefits from per-host credentials per
  profile.
* The single-config + `--config PATH` workflow is demonstrably insufficient.

Until then: YAGNI.
```

- [ ] **Step 2: Commit**

```bash
git add docs/design/repo-clone-DESIGN-multi-account-config-layout.md
git commit -m "docs(design): add multi-account config layout (forward-compat)

Layout designed today so future --profile support is purely additive.
v1 ships single-file; profile flag is FUTURE_WORK.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Write `docs/design/repo-clone-DESIGN-output-prefixes.md`

**Files:**
- Create: `docs/design/repo-clone-DESIGN-output-prefixes.md`

- [ ] **Step 1: Write the doc**

```markdown
# Output prefixes: INFO / WARN / ERROR

Human-mode stdout/stderr lines are prefixed by category. These prefixes are
stable; downstream `grep`/`awk` can rely on them.

| Prefix | Meaning | Visibility | Stream |
|---|---|---|---|
| `INFO:` | Routine progress (cloned, fetched, fork exists, etc.) | Hidden by default; shown at `-v` and above, or when `output.show_info: true` | stderr |
| `WARN:` | Degraded but proceeding (local changes, fork exists, upstream remote points elsewhere, etc.) | Always shown | stderr |
| `ERROR:` | Abort condition; followed by exit | Always shown | stderr |
| `DETAIL:` | Existing `-vv` / `-vvv` output (timing, sizes) | Verbosity-gated | stderr |
| `DEBUG:` | Existing `-vvv` output | Verbosity-gated | stderr |

## Why prefixes (and not colors)

* Pipe-friendly: `repo-clone ... | grep -v INFO:` works.
* No tty-detection required.
* Colors can layer on later (planned: ANSI when `sys.stderr.isatty()`) without
  changing the contract.

## When the script writes to stdout vs stderr

* **Stdout:** plain repo paths and JSONL events. Anything a downstream consumer
  might pipe into another tool.
* **Stderr:** everything human-facing (prefixes, progress, errors).

## Verbosity levels (existing)

| Level | Flag | Adds |
|---|---|---|
| 0 | (default) | WARN, ERROR only |
| 1 | `-v` | + INFO, git commands as run |
| 2 | `-vv` | + timing, sizes, DETAIL |
| 3 | `-vvv` | + DEBUG; raw git output streamed live |

`output.show_info: true` in config promotes INFO to level 0 visibility.
`-v` and config setting are independent — either makes INFO visible.

## See also

* JSONL-mode contract: `repo-clone-DESIGN-jsonl-output.md`.
* Tone for the strings inside these messages: `maintainers/repo-clone-MAINTAINERS-tone-guide.md`.
```

- [ ] **Step 2: Commit**

```bash
git add docs/design/repo-clone-DESIGN-output-prefixes.md
git commit -m "docs(design): document INFO/WARN/ERROR prefix contract

Stable prefixes, stream split, verbosity ladder, link to JSONL counterpart.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Write `docs/design/repo-clone-DESIGN-jsonl-output.md`

**Files:**
- Create: `docs/design/repo-clone-DESIGN-jsonl-output.md`

- [ ] **Step 1: Write the doc**

```markdown
# JSON Lines output mode (`--jsonl`)

When `--jsonl` is passed (or `output.mode: jsonl` is set in config), the tool
emits **one compact JSON object per line on stdout**, and suppresses the
human-mode `INFO:` / `WARN:` / `ERROR:` prefix lines on stderr (the data is
already in the JSONL stream).

Mutually exclusive with the human prefix mode. CLI `--jsonl` overrides config.

## Why

Programmatic consumers (other shell scripts, agents, CI jobs) parse line-by-line
without regex-fragile prefix matching. `jq` works out of the box.

## Schema

Every line is a JSON object with at minimum these fields:

```json
{"event": "<event-name>", "level": "info" | "warn" | "error" | "debug", "ts": "<ISO-8601>", ...}
```

Field rules:

* `event` — stable, kebab-case event name. Additive: new events may appear; old
  events keep their names. Consumers should ignore unknown events.
* `level` — `"info"`, `"warn"`, `"error"`, `"debug"`. Maps 1:1 to the
  human-mode prefixes.
* `ts` — ISO-8601 timestamp with timezone (e.g. `2026-05-06T14:23:01.045+00:00`).
* Per-event fields — additive; new fields may appear in future versions.

## Event catalogue (v1)

| event | level | typical fields |
|---|---|---|
| `parsed-url` | info | `url`, `host`, `org`, `repo`, `is_ssh` |
| `clone-start` | info | `url`, `path`, `recursive`, `shallow` |
| `clone-success` | info | `url`, `path` |
| `clone-failed` | error | `url`, `path`, `git_exit_code` |
| `update-start` | info | `path` |
| `update-fetched` | info | `path` |
| `update-pulled` | info | `path` |
| `update-skipped-dirty` | warn | `path`, `dirty_summary` |
| `submodule-result` | info / warn | `path`, `ok` |
| `gh-preflight-ok` | info | `gh_version` |
| `gh-missing` | error | `hint` |
| `gh-unauthed` | error | `hint` |
| `fork-identity-resolved` | info | `source_org`, `source_repo`, `target_org`, `fork_name`, `template` |
| `fork-exists` | info | `target_org`, `fork_name`, `parent_full_name` |
| `fork-collision` | error | `target_org`, `fork_name`, `parent_full_name`, `expected_parent` |
| `fork-creating` | info | `source`, `target_org`, `fork_name`, `all_branches` |
| `fork-created` | info | `source`, `target_org`, `fork_name`, `clone_url` |
| `fork-creation-failed` | error | `source`, `target_org`, `fork_name`, `gh_exit_code`, `stderr` |
| `upstream-added` | info | `fork_path`, `remote_name`, `upstream_url` |
| `upstream-already-correct` | info | `fork_path`, `remote_name` |
| `upstream-mismatch` | warn | `fork_path`, `remote_name`, `current_url`, `expected_url` |
| `summary` | info | `source_path`, `fork_path` (optional), `actions` (list) |

## Example session (`repo-clone --fork --jsonl https://github.com/acme/widget`)

```jsonl
{"event":"parsed-url","level":"info","ts":"...","url":"https://github.com/acme/widget","host":"github.com","org":"acme","repo":"widget","is_ssh":false}
{"event":"gh-preflight-ok","level":"info","ts":"...","gh_version":"2.45.0"}
{"event":"clone-start","level":"info","ts":"...","url":"https://github.com/acme/widget.git","path":"/home/u/github/acme/widget","recursive":true,"shallow":false}
{"event":"clone-success","level":"info","ts":"...","url":"https://github.com/acme/widget.git","path":"/home/u/github/acme/widget"}
{"event":"fork-identity-resolved","level":"info","ts":"...","source_org":"acme","source_repo":"widget","target_org":"VariousForks","fork_name":"widget-by-acme-fork","template":"{repo}-by-{org}-fork"}
{"event":"fork-creating","level":"info","ts":"...","source":"acme/widget","target_org":"VariousForks","fork_name":"widget-by-acme-fork","all_branches":false}
{"event":"fork-created","level":"info","ts":"...","source":"acme/widget","target_org":"VariousForks","fork_name":"widget-by-acme-fork","clone_url":"https://github.com/VariousForks/widget-by-acme-fork.git"}
{"event":"clone-start","level":"info","ts":"...","url":"https://github.com/VariousForks/widget-by-acme-fork.git","path":"/home/u/github/VariousForks/widget-by-acme-fork","recursive":true,"shallow":false}
{"event":"clone-success","level":"info","ts":"...","url":"https://github.com/VariousForks/widget-by-acme-fork.git","path":"/home/u/github/VariousForks/widget-by-acme-fork"}
{"event":"upstream-added","level":"info","ts":"...","fork_path":"/home/u/github/VariousForks/widget-by-acme-fork","remote_name":"upstream","upstream_url":"https://github.com/acme/widget.git"}
{"event":"summary","level":"info","ts":"...","source_path":"/home/u/github/acme/widget","fork_path":"/home/u/github/VariousForks/widget-by-acme-fork","actions":["cloned-source","created-fork","cloned-fork","added-upstream"]}
```

## Stability promise

* Event names: never renamed.
* Per-event fields: never removed; never have their meaning changed.
* New events and new fields: may appear at any version (additive only).

Consumers should:

* Match on `event` and ignore unknown ones.
* Tolerate unknown fields.
* Not depend on the order of events beyond what's documented above.

## See also

* Output prefixes (human mode): `repo-clone-DESIGN-output-prefixes.md`.
```

- [ ] **Step 2: Commit**

```bash
git add docs/design/repo-clone-DESIGN-jsonl-output.md
git commit -m "docs(design): document JSONL output mode

Event catalogue, schema rules, stability promise, full example session.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 8: Write `docs/design/repo-clone-DESIGN-fork-flag.md`

**Files:**
- Create: `docs/design/repo-clone-DESIGN-fork-flag.md`

- [ ] **Step 1: Write the doc**

This is the central feature design doc. Mirrors spec §3, §4, §8.

```markdown
# `--fork` flag

The `--fork` flag turns `repo-clone <url>` into "clone the source AND create
a fork on the host AND clone the fork side AND wire the upstream remote",
all idempotently.

## CLI surface

```
repo-clone [OPTIONS] <url>

  -F, --fork                  Create a fork on the host, then clone source AND fork.
      --fork-all              Fork copies all branches.        Implies --fork.
      --fork-default          Fork copies only default branch. Implies --fork.
      --fork-org NAME         Override target org for the fork. Implies --fork.
      --fork-name VALUE       Override fork repo name (literal, no templating).
                              If VALUE contains exactly one '/', it splits into
                              --fork-org / --fork-name. Implies --fork.
```

`--fork-all` and `--fork-default` are mutually exclusive (error on both).

## On-disk layout (decision Q2)

After `repo-clone --fork https://github.com/acme/widget`:

```
~/github/acme/widget                            # source clone, origin → acme
~/github/<target_org>/<fork_name>               # fork clone
                                                #   origin   → fork
                                                #   upstream → acme/widget
```

Two directories. The path-tells-you-the-org invariant of `repo-clone` is
preserved on both sides.

## Order of operations

```
1. Parse URL                              (existing)
2. Resolve fork identity                   (URL + config + CLI)
3. gh preflight                            (gh --version, gh auth status)
4. Source-side clone/update                (existing repo-clone behavior)
5. Fork creation (idempotent)              (gh api existence check, gh repo fork)
6. Fork-side clone/update                  (reuse clone path on fork URL)
7. Wire upstream remote                    (idempotent)
8. Emit summary                            (human or JSONL)
```

Source first → bad URL fails before touching GitHub.

## Idempotency

| State | Behavior |
|---|---|
| Fork doesn't exist | Create it |
| Fork exists, parent matches source | Continue (INFO) |
| Fork exists, parent mismatches source | ERROR (exit 11) |
| Fork-side dir doesn't exist | Clone fresh |
| Fork-side dir exists, origin matches expected fork URL | Update (existing path) |
| Fork-side dir exists, origin differs / not a git repo | ERROR (exit 5) |
| `upstream` remote absent on fork clone | Add it |
| `upstream` remote present, URL matches source | Skip |
| `upstream` remote present, URL differs | WARN, do not overwrite |

## Exit codes (additions)

| Code | Meaning |
|---|---|
| 8 | `gh` missing or unauthed |
| 9 | Host not yet supported for `--fork` (non-GitHub) |
| 10 | Fork creation API call failed |
| 11 | Fork name collision with unrelated repo |

## `--fork-name` shorthand

If the value contains exactly one `/`, split:

```
--fork-name greg/widget-experimental
≡
--fork-org greg --fork-name widget-experimental
```

If `--fork-org` is also given with a different value → ERROR.
Multiple `/` or empty either side → ERROR.

## Host support

v1: GitHub only. Other hosts: `--fork` with a non-GitHub URL → exit 9 with a
pointer to FUTURE_WORK and the issue tracker.

## See also

* Config schema: `repo-clone-DESIGN-config-file.md`.
* Output: `repo-clone-DESIGN-output-prefixes.md`, `repo-clone-DESIGN-jsonl-output.md`.
* Help for AI agents: `repo-clone-DESIGN-help-for-agents.md`.
```

- [ ] **Step 2: Commit**

```bash
git add docs/design/repo-clone-DESIGN-fork-flag.md
git commit -m "docs(design): document --fork feature design

CLI surface, on-disk layout, order of operations, idempotency matrix,
exit codes, shorthand parsing, host scope.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 9: Write `docs/design/repo-clone-DESIGN-help-for-agents.md`

**Files:**
- Create: `docs/design/repo-clone-DESIGN-help-for-agents.md`

- [ ] **Step 1: Write the doc**

```markdown
# `--help-for-agents`

A second help mode targeted at LLM agents using `repo-clone` programmatically.
Two equivalent entry points (Q6):

* `repo-clone --help-for-agents` (canonical flag)
* `repo-clone help-for-agents` (subcommand alias)

## Audience

LLM agents reading the output as part of a context window, deciding how to
invoke the tool, predicting failure modes, parsing JSONL, recovering from
errors. **Not humans.**

## Style

* Information density over politeness.
* Pseudocode, tables, equations, fragments preferred over prose.
* No reassuring filler ("This tool helps you...").
* No marketing tone.
* Emoji only where they aid scanning (e.g., 🤖 / ⚠️ delimiters), never decoration.
* No repetition of `--help` content. Cross-link instead.

## Required sections (each independently editable)

| Section | Purpose |
|---|---|
| `IDENTITY` | One-line what + why. |
| `INVOCATION` | Grammar in EBNF-ish form + canonical examples. |
| `EXIT_CODES` | Table of codes and their meanings (1..N). |
| `STATE_MACHINE` | Per-URL outcome decision tree (no fork / fork / re-run). |
| `JSONL_SCHEMA` | Pointer to design doc + brief inline event list. |
| `IDEMPOTENCY` | Re-run guarantees: what's safe, what's not. |
| `FAILURE_RECOVERY` | Retryable errors, network timeouts, expected wall-clock ranges (fork creation 5–30s, clones can take minutes). |
| `CONTEXTUAL_FIT` | Where this tool sits in agent workflows; path-layout invariants; companion tools (`find-snapshot`, `sk`, `fzf`). |
| `TROUBLESHOOTING_POINTERS` | Local + remote URL of `docs/repo-clone-TROUBLESHOOTING.md`. |
| `REPORT_BUGS` | Issue tracker link + brief template. |

## Cross-links

* `--help-for-agents` output mentions `--help` once at the top, for the human
  overview. Do not duplicate `--help` content beyond that pointer.
* `--help` mentions `--help-for-agents` prominently (a 4-line block, easy to
  scan), so an agent reading `--help` learns the better option exists.

## Discoverability in `--help`

The `--help` output includes (early, easy to scan):

```
🤖 AI agents: run `repo-clone --help-for-agents` for an agent-optimized
   briefing (denser format, troubleshooting paths, exit-code semantics,
   JSONL contract).
   Issues / unclear behavior → https://github.com/shibuido/repo-clone/issues
```

## Implementation note

The agent-help text is assembled from small string fragments (one per section)
inside the script, so each section is independently editable. Lives in the
`# ---- Agent help ----` banner in the script.

## Anti-patterns

* Padding sections with prose to look "complete".
* Hedging ("This may sometimes...") — agents need crisp constraints.
* Restating `--help` content.
* Walls of text. Use tables.
```

- [ ] **Step 2: Commit**

```bash
git add docs/design/repo-clone-DESIGN-help-for-agents.md
git commit -m "docs(design): document --help-for-agents style and contract

Audience, style rules, required sections, anti-patterns,
cross-linking with --help.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 10: Write `docs/design/repo-clone-DESIGN-docs-architecture.md`

**Files:**
- Create: `docs/design/repo-clone-DESIGN-docs-architecture.md`

- [ ] **Step 1: Write the doc**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add docs/design/repo-clone-DESIGN-docs-architecture.md
git commit -m "docs(design): document the docs architecture itself

Roles of each doc, cognitive learning curve principle, cross-linking,
why two READMEs, why scoped design docs. Reusable across shibuido tools.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 11: Write `docs/repo-clone-TROUBLESHOOTING.md`

**Files:**
- Create: `docs/repo-clone-TROUBLESHOOTING.md`

- [ ] **Step 1: Write the doc**

```markdown
# Troubleshooting

Symptom-keyed list. Find your symptom, follow the fix.

## "ERROR: gh: command not found"

You hit `--fork`, but the GitHub CLI (`gh`) isn't installed.

```bash
# Arch:
sudo pacman -S github-cli

# Ubuntu/Debian:
sudo apt install gh

# macOS:
brew install gh

# Then authenticate once:
gh auth login
```

## "ERROR: gh is not authenticated"

`gh` is installed but no credentials.

```bash
gh auth login          # interactive
# or
gh auth login --with-token < token.txt
```

Verify: `gh auth status` should print "Logged in to github.com as <you>".

## "ERROR: --fork-all and --fork-default are mutually exclusive"

You passed both. Pick one. Or remove both and let the config / built-in
default decide.

## "ERROR: fork name 'foo/bar/baz' has more than one '/'"

The shorthand parser saw multiple slashes. Use either:

* `--fork-name simple-name` (no slash)
* `--fork-name org/name` (one slash)

## "ERROR: target_org/fork_name exists but parent.full_name is X (expected Y)"

A repo with the templated fork name already lives at the target org, but it
isn't a fork of the source you specified. To resolve:

* Check `gh repo view <target_org>/<fork_name>` and decide whether to keep it.
* Use `--fork-name` to pick a different name.
* Or use `--fork-org` to pick a different org.

## "ERROR: path exists but is not a git repository: ..."

A directory at the expected clone destination exists but has no `.git`.
Move it out of the way (or delete if you're sure it's safe) and re-run.

## "WARN: upstream remote points to X (expected Y)"

The fork-side clone already has an `upstream` remote, but it doesn't match
the source URL. `repo-clone` will not overwrite it. To fix manually:

```bash
cd ~/github/<target_org>/<fork_name>
git remote set-url upstream <source-url>
```

## "WARN: local changes detected, skipping pull"

Existing behavior on the source side. Stash or commit, then re-run.

## Hugging Face: "ERROR: git-lfs is required"

Install git-lfs and re-run. See `repo-clone.README.md` Quickstart.

## Watch mode: "ERROR: No clipboard tool found"

Existing behavior. Install one of: `wl-clipboard`, `xclip`, `xsel`.

## Slow fork creation (5–30s)

Normal. GitHub's fork API can take a few seconds; large repos longer. Use
`-v` to see progress.

## "Fork created but clone of fork 404'd"

GitHub occasionally lags 1–3s between fork-API success and the new repo
becoming clonable. The script retries with backoff for up to ~10s. If it
still fails:

```bash
# Wait a few seconds, then re-run the same command. Idempotent.
repo-clone --fork <url>
```

## Still stuck?

Open an issue with the failing command, the full output (including `-vv`
where helpful), and your `gh --version` and `git --version`:

  https://github.com/shibuido/repo-clone/issues
```

- [ ] **Step 2: Commit**

```bash
git add docs/repo-clone-TROUBLESHOOTING.md
git commit -m "docs: add symptom-keyed troubleshooting page

Covers gh missing/unauthed, --fork-* arg errors, naming collisions,
upstream-remote mismatch, watch mode, fork creation lag, HF git-lfs.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 12: Write `maintainers/repo-clone-MAINTAINERS-tone-guide.md`

**Files:**
- Create: `maintainers/repo-clone-MAINTAINERS-tone-guide.md`

- [ ] **Step 1: Write the doc**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add maintainers/repo-clone-MAINTAINERS-tone-guide.md
git commit -m "docs(maintainers): add tone guide

Four audiences (README/--help, --help-for-agents, error messages,
commit messages) with do/don't and before/after examples.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 13: Write `maintainers/repo-clone-MAINTAINERS-decisions.md`

**Files:**
- Create: `maintainers/repo-clone-MAINTAINERS-decisions.md`

- [ ] **Step 1: Write the doc**

```markdown
# Decisions log

Append-only log of "we chose X because Y". Future maintainers append
entries; never rewrite history. Every entry dated.

---

## 2026-05-05 — gh CLI required for `--fork` (Q1)

**Decision:** Require `gh` CLI for the fork path. Preflight checks `gh
--version` and `gh auth status`.

**Why:** GitHub auth (2FA, SSO, enterprise) is gnarly. `gh` solves it. Stays
consistent with stdlib-portability principle (shell out to a CLI rather than
implement HTTP+auth ourselves).

**Considered:** direct REST API via `urllib`; hybrid (gh-if-present else API).
Rejected: re-implementing auth is the largest source of edge-case complexity
in any tool that talks to GitHub.

---

## 2026-05-05 — Two directories for source + fork (Q2)

**Decision:** Source clone at `$base/$host/$org/$repo`, fork clone at
`$base/$host/$forkOrg/$forkName`. Fork side gets `upstream` remote
auto-wired.

**Why:** Preserves repo-clone's defining invariant — "the path tells you
the host/org/repo of `origin`". Standard fork workflow with `upstream`
remote is well-known.

**Considered:** single dir with two remotes (breaks invariant); fork-only
clone (loses upstream working copy).

---

## 2026-05-05 — Default fork target = authenticated gh user (Q3)

**Decision:** When no `fork.target_org` is configured and no `--fork-org` is
passed, fork to the user's own GitHub account (matches `gh repo fork` default
and the webui default).

**Why:** Zero-friction first run. Power users configure `target_org:
VariousForks` once and forget.

---

## 2026-05-05 — Default branch only for new forks (Q4)

**Decision:** Default = default-branch-only (matches GitHub webui). Both
`--fork-all` and `--fork-default` flags exist; either implies `--fork`;
mutually exclusive.

**Why:** Webui parity surprises nobody. Power users override per-invocation
or in config.

---

## 2026-05-05 — Python `str.format` template + literal `--fork-name` (Q5)

**Decision:** Config carries the template (`{repo}-by-{org}-fork`), CLI
carries a literal override (`--fork-name LITERAL`). `--fork-name org/name`
shorthand splits into org + name.

**Why:** Templating is for "set once" use; per-invocation overrides are
almost always literal renames.

---

## 2026-05-05 — Both `--help-for-agents` flag and subcommand alias (Q6)

**Decision:** Ship both `--help-for-agents` (canonical flag) and
`help-for-agents` (subcommand alias).

**Why:** Best discoverability for agents. Symmetric with `help|-h|--help`
pattern.

---

## 2026-05-05 — Idempotent fork flow + structured logging (Q7)

**Decision:** Re-running `repo-clone --fork <url>` is safe. Existing fork on
GH or matching local dir → update. Unrelated local dir → ERROR. All output
gets `INFO:` / `WARN:` / `ERROR:` prefixes; `--jsonl` for programmatic
consumers.

**Why:** Mirrors existing repo-clone non-fork behavior (re-running updates).
Structured prefixes are pipe-friendly without color-tty machinery.

---

## 2026-05-06 — Spec corrections during plan authoring

**Decision:** Bump new exit codes to 8/9/10/11 (existing 7 = NO_CLIPBOARD_TOOL).
Keep config filename `repo-clone-config.yaml` (not `config.yaml` — preserves
backward compat). Use `WARN:` prefix (existing) not `WARNING:`.

**Why:** Reading the existing script during plan authoring revealed three
discrepancies. Fixing in spec rather than introducing breaking changes.

---

## How to add an entry

```markdown
## YYYY-MM-DD — Short title

**Decision:** What we chose.

**Why:** The reason. Cite issues, PRs, discussions if relevant.

**Considered:** Alternatives that were rejected and why (if material).
```

Append to the bottom of the dated entries. Never edit older entries.
```

- [ ] **Step 2: Commit**

```bash
git add maintainers/repo-clone-MAINTAINERS-decisions.md
git commit -m "docs(maintainers): add decisions log

Seeded with the 7 brainstorming decisions (Q1-Q7) plus the 2026-05-06
spec-correction decision. Append-only convention documented.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 14: Write `maintainers/repo-clone-MAINTAINERS-README.md`

**Files:**
- Create: `maintainers/repo-clone-MAINTAINERS-README.md`
- Delete: `maintainers/.gitkeep`

- [ ] **Step 1: Write the doc**

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git rm maintainers/.gitkeep
git add maintainers/repo-clone-MAINTAINERS-README.md
git commit -m "docs(maintainers): add maintainer orientation README

How to read the repo, code organization, adding features, adding design
docs, running tests, commit conventions.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 15: Add stdlib YAML-subset reader

**Files:**
- Modify: `repo-clone` (extend `load_config` and add a new helper)
- Modify: `repo-clone.DEV_NOTES.md` (add a manual test case)

The existing `load_config` (`repo-clone:267-293`) requires `pyyaml`. We add a
stdlib fallback that handles the documented schema subset.

- [ ] **Step 1: Write the failing smoke test**

Create `/tmp/claude/2026-05-06-rc-fork/test_yaml_subset.py`:

```bash
mkdir -p /tmp/claude/2026-05-06-rc-fork
cat > /tmp/claude/2026-05-06-rc-fork/test_yaml_subset.py <<'EOF'
"""Smoke test for the stdlib YAML-subset reader."""
import sys
sys.path.insert(0, "/home/gw-t490/github/shibuido/repo-clone")

# Import the script as a module by loading it under a clean name.
import importlib.util
spec = importlib.util.spec_from_file_location(
    "repo_clone_mod",
    "/home/gw-t490/github/shibuido/repo-clone/repo-clone",
)
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)

src = """
extra_hosts:
  - gitlab.mycompany.com
  - gitea.internal.org
watch_interval: 0.5
fork:
  target_org: VariousForks
  name_template: "{repo}-by-{org}-fork"
  all_branches: false
output:
  mode: human
  show_info: true
"""

result = mod.parse_yaml_subset(src)
assert result["extra_hosts"] == ["gitlab.mycompany.com", "gitea.internal.org"], result
assert result["watch_interval"] == "0.5", result   # subset reader keeps strings; caller coerces
assert result["fork"]["target_org"] == "VariousForks", result
assert result["fork"]["name_template"] == "{repo}-by-{org}-fork", result
assert result["fork"]["all_branches"] is False, result
assert result["output"]["mode"] == "human", result
assert result["output"]["show_info"] is True, result
print("YAML-subset reader smoke test: PASS")
EOF
python3 /tmp/claude/2026-05-06-rc-fork/test_yaml_subset.py
```

Expected: `AttributeError: module ... has no attribute 'parse_yaml_subset'` (or similar).

- [ ] **Step 2: Add the YAML-subset reader to `repo-clone`**

Insert a new banner section just before `def load_config()` (currently at
line 267). Specifically, insert after the helpers block ending at line 264.
Add this code:

```python
# ---- Config file loading ----
# Tiny stdlib YAML-subset reader. Used as fallback when pyyaml is not
# installed. Handles only the schema documented in
# docs/design/repo-clone-DESIGN-config-file.md:
#   - top-level keys
#   - one level of nested mappings
#   - string and boolean scalars
#   - lists of strings (one item per line, "  - item")
# Anything outside this subset → ValueError("...install pyyaml...").

_YAML_BOOLS = {"true": True, "false": False, "yes": True, "no": False}


def _yaml_strip_value(raw: str) -> object:
    """Strip a YAML scalar value into Python: bool, None, or string."""
    s = raw.strip()
    if not s:
        return ""
    # Strip surrounding quotes if balanced
    if (s[0] == s[-1]) and s[0] in ("'", '"') and len(s) >= 2:
        return s[1:-1]
    low = s.lower()
    if low in _YAML_BOOLS:
        return _YAML_BOOLS[low]
    if low == "null" or low == "~":
        return None
    return s


def parse_yaml_subset(text: str) -> dict:
    """
    Parse a small YAML subset. See module-level comment for scope.

    Raises ValueError on any construct outside the subset.
    """
    result: dict = {}
    current_top: Optional[str] = None         # active nested-mapping key
    current_list_top: Optional[str] = None    # active list key (top-level)

    for raw_line in text.splitlines():
        # Strip comments (only when '#' is at line start or preceded by space)
        line = raw_line.rstrip()
        if not line.strip() or line.lstrip().startswith("#"):
            continue
        # Find the comment portion safely
        if " #" in line:
            line = line.split(" #", 1)[0].rstrip()

        indent = len(line) - len(line.lstrip())
        stripped = line.lstrip()

        # List item: "- value", indented under a top-level list key
        if stripped.startswith("- "):
            item = _yaml_strip_value(stripped[2:])
            if not isinstance(item, str):
                raise ValueError(
                    "config syntax beyond the stdlib subset (non-string list "
                    "item); install pyyaml or simplify"
                )
            if current_list_top is None:
                raise ValueError("list item without preceding key")
            result.setdefault(current_list_top, []).append(item)
            continue

        # Mapping line: "key:" or "key: value"
        if ":" not in stripped:
            raise ValueError(f"unrecognized line: {raw_line!r}")

        key, _, value = stripped.partition(":")
        key = key.strip()
        value = value.strip()

        if indent == 0:
            current_top = None
            current_list_top = None
            if value == "":
                # Could be either a nested-mapping start or a list start.
                # We don't know yet; mark both possibilities and resolve on
                # the next line.
                current_top = key
                current_list_top = key
                # Pre-create an empty dict; will be replaced with [] if a
                # list item appears first.
                result.setdefault(key, {})
            else:
                result[key] = _yaml_strip_value(value)
        elif indent == 2 and current_top is not None:
            # Nested mapping under current_top
            target = result.get(current_top)
            if not isinstance(target, dict):
                # Was tentatively a list; demote to dict.
                target = {}
                result[current_top] = target
            current_list_top = None  # nested mapping → not a list
            if value == "":
                raise ValueError(
                    "config syntax beyond the stdlib subset (mapping nested "
                    "deeper than 1 level); install pyyaml or simplify"
                )
            target[key] = _yaml_strip_value(value)
        else:
            raise ValueError(f"unsupported indentation level: {raw_line!r}")

    # Cleanup: keys that were tentatively dicts but received list items
    # are already handled (setdefault → list). Empty tentative dicts that
    # received nothing stay as {}.
    return result
```

- [ ] **Step 3: Run smoke test, expect PASS**

```bash
python3 /tmp/claude/2026-05-06-rc-fork/test_yaml_subset.py
```

Expected: `YAML-subset reader smoke test: PASS`.

- [ ] **Step 4: Wire fallback into `load_config`**

Modify `load_config` (lines 267-293). Replace the `try: import yaml` block
with a fallback chain:

```python
def load_config(config_path: Optional[Path] = None) -> dict:
    """
    Load configuration from the canonical location, or an explicit path.

    Resolution order:
      1. config_path argument (if provided)
      2. ~/.config/shibuido/repo-clone/repo-clone-config.yaml
      3. ~/.config/shibuido/repo-clone/profiles/default.yaml (forward-compat)

    Returns empty dict if no config file found.

    Loader prefers pyyaml; falls back to parse_yaml_subset on ImportError.
    On parse error, prints ERROR and exits nonzero.
    """
    if config_path is None:
        base = Path.home() / ".config" / "shibuido" / "repo-clone"
        candidates = [
            base / "repo-clone-config.yaml",
            base / "profiles" / "default.yaml",
        ]
        config_path = next((p for p in candidates if p.exists()), None)
        if config_path is None:
            log_debug("No config file found in canonical locations")
            return {}

    if not config_path.exists():
        log_warn(f"Config file not found: {config_path}")
        return {}

    try:
        text = config_path.read_text()
    except OSError as e:
        log_error(f"Cannot read config {config_path}: {e}")
        sys.exit(ExitCode.PERMISSION_ERROR)

    # Prefer pyyaml
    try:
        import yaml  # type: ignore
        try:
            config = yaml.safe_load(text) or {}
            log_debug(f"Loaded config (pyyaml) from {config_path}")
            return config
        except Exception as e:
            log_error(f"Failed to parse config {config_path}: {e}")
            sys.exit(ExitCode.INVALID_URL)  # generic config-error exit
    except ImportError:
        pass

    # Fallback: stdlib YAML-subset
    try:
        config = parse_yaml_subset(text)
        log_debug(f"Loaded config (stdlib subset) from {config_path}")
        return config
    except ValueError as e:
        log_error(
            f"Failed to parse config {config_path}: {e}\n"
            f"  Hint: install pyyaml for richer YAML support, or simplify your config."
        )
        sys.exit(ExitCode.INVALID_URL)
```

- [ ] **Step 5: Re-run smoke test plus a load_config integration test**

```bash
mkdir -p /tmp/claude/2026-05-06-rc-fork/cfgtest/.config/shibuido/repo-clone
cat > /tmp/claude/2026-05-06-rc-fork/cfgtest/.config/shibuido/repo-clone/repo-clone-config.yaml <<'EOF'
fork:
  target_org: VariousForks
  name_template: "{repo}-by-{org}-fork"
output:
  show_info: true
EOF

cat > /tmp/claude/2026-05-06-rc-fork/test_load_config.py <<'EOF'
import os, importlib.util, sys
os.environ["HOME"] = "/tmp/claude/2026-05-06-rc-fork/cfgtest"
spec = importlib.util.spec_from_file_location(
    "repo_clone_mod",
    "/home/gw-t490/github/shibuido/repo-clone/repo-clone",
)
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)
cfg = mod.load_config()
assert cfg["fork"]["target_org"] == "VariousForks", cfg
assert cfg["output"]["show_info"] is True, cfg
print("load_config integration smoke test: PASS")
EOF
python3 /tmp/claude/2026-05-06-rc-fork/test_load_config.py
```

Expected: `load_config integration smoke test: PASS`.

- [ ] **Step 6: Add manual test case to DEV_NOTES**

Append to `repo-clone.DEV_NOTES.md` § Manual Test Cases:

```markdown
### Stdlib YAML-subset config

```bash
# Without pyyaml installed:
pip uninstall -y pyyaml 2>/dev/null

mkdir -p ~/.config/shibuido/repo-clone
cat > ~/.config/shibuido/repo-clone/repo-clone-config.yaml <<'EOF'
fork:
  target_org: VariousForks
output:
  show_info: true
EOF

repo-clone -vvv https://github.com/shibuido/repo-clone 2>&1 | grep "Loaded config (stdlib subset)"
```
```

- [ ] **Step 7: Commit**

```bash
git add repo-clone repo-clone.DEV_NOTES.md
git commit -m "feat: add stdlib YAML-subset reader as pyyaml fallback

* New parse_yaml_subset() handles the documented schema subset:
  top-level keys, one level of nested mappings, strings/bools, list of strings
* load_config() now tries pyyaml first; falls back on ImportError
* load_config() also accepts profiles/default.yaml as forward-compat alias
* Errors on unsupported syntax with a clear pointer to install pyyaml
* DEV_NOTES updated with manual test case

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 16: Add structured output / JSONL emitter

**Files:**
- Modify: `repo-clone` (add new banner section before `# ---- Config file loading ----`)

- [ ] **Step 1: Write the failing smoke test**

```bash
cat > /tmp/claude/2026-05-06-rc-fork/test_emit.py <<'EOF'
import io, json, importlib.util, sys
spec = importlib.util.spec_from_file_location(
    "repo_clone_mod",
    "/home/gw-t490/github/shibuido/repo-clone/repo-clone",
)
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)

# Switch to JSONL mode
mod.set_output_mode("jsonl")
buf = io.StringIO()
mod.set_output_stream(buf)

mod.emit("clone-start", level="info", url="https://x", path="/tmp/x")
mod.emit("clone-success", level="info", url="https://x", path="/tmp/x")

lines = buf.getvalue().strip().splitlines()
assert len(lines) == 2, lines
for line in lines:
    rec = json.loads(line)
    assert "event" in rec and "level" in rec and "ts" in rec, rec
print("emit smoke test: PASS")
EOF
python3 /tmp/claude/2026-05-06-rc-fork/test_emit.py
```

Expected: `AttributeError: module ... has no attribute 'set_output_mode'` (failing).

- [ ] **Step 2: Add the output module**

Insert after the existing `log_*` helpers (after line 113, before the
dataclasses block at line 115). Add this section:

```python
# ---- Output / structured logging ----
# Two output modes: "human" (existing INFO:/WARN:/ERROR: prefixes on stderr)
# and "jsonl" (one JSON object per line on stdout). See
# docs/design/repo-clone-DESIGN-output-prefixes.md and
# docs/design/repo-clone-DESIGN-jsonl-output.md.

import json as _json
from datetime import datetime, timezone

# Module-level state. set_output_mode() and set_output_stream() configure.
_OUTPUT_MODE = "human"
_OUTPUT_STREAM = None  # None → stdout for jsonl, stderr for human
_SHOW_INFO = False     # Promotes INFO to v0 visibility when True


def set_output_mode(mode: str) -> None:
    """Set output mode: 'human' or 'jsonl'."""
    global _OUTPUT_MODE
    if mode not in ("human", "jsonl"):
        raise ValueError(f"output mode must be human or jsonl, got {mode!r}")
    _OUTPUT_MODE = mode


def set_output_stream(stream) -> None:
    """Override the destination stream (used by tests). None → default."""
    global _OUTPUT_STREAM
    _OUTPUT_STREAM = stream


def set_show_info(value: bool) -> None:
    """Promote INFO: lines to verbosity-0 visibility."""
    global _SHOW_INFO
    _SHOW_INFO = bool(value)


def _now_iso() -> str:
    return datetime.now(timezone.utc).isoformat(timespec="milliseconds")


def emit(event: str, *, level: str = "info", **fields) -> None:
    """
    Emit a structured event.

    In human mode: emits a prefixed line on stderr (INFO:/WARN:/ERROR:),
    suppressing INFO unless verbosity >= 1 or show_info is True.

    In jsonl mode: emits one compact JSON object on stdout (or override stream).
    """
    if _OUTPUT_MODE == "jsonl":
        record = {"event": event, "level": level, "ts": _now_iso(), **fields}
        line = _json.dumps(record, separators=(",", ":"), ensure_ascii=False)
        out = _OUTPUT_STREAM if _OUTPUT_STREAM is not None else sys.stdout
        print(line, file=out, flush=True)
        return

    # Human mode: synthesize a readable message from fields
    if level == "info":
        if VERBOSITY < VerboseLevel.COMMANDS and not _SHOW_INFO:
            return
        prefix = "INFO:"
    elif level == "warn":
        prefix = "WARN:"
    elif level == "error":
        prefix = "ERROR:"
    elif level == "debug":
        if VERBOSITY < VerboseLevel.DEBUG:
            return
        prefix = "DEBUG:"
    else:
        prefix = level.upper() + ":"

    # Best-effort one-line render for human mode.
    detail = ", ".join(f"{k}={v}" for k, v in fields.items())
    msg = f"{event}" + (f" — {detail}" if detail else "")
    out = _OUTPUT_STREAM if _OUTPUT_STREAM is not None else sys.stderr
    print(f"{prefix} {msg}", file=out, flush=True)
```

- [ ] **Step 3: Run smoke test, expect PASS**

```bash
python3 /tmp/claude/2026-05-06-rc-fork/test_emit.py
```

Expected: `emit smoke test: PASS`.

- [ ] **Step 4: Commit**

```bash
git add repo-clone
git commit -m "feat: add structured emit() for human and JSONL output modes

* set_output_mode('human'|'jsonl'), set_output_stream(), set_show_info()
* emit(event, level=, **fields) renders a human prefix line OR a one-line
  JSON object with event/level/ts plus the caller's fields
* Existing log_info/log_warn/log_error helpers untouched (keep working);
  new code paths use emit() instead

Implements docs/design/repo-clone-DESIGN-output-prefixes.md and
docs/design/repo-clone-DESIGN-jsonl-output.md.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 17: Add gh preflight helper

**Files:**
- Modify: `repo-clone` (new banner section after Output)

- [ ] **Step 1: Write the failing smoke test**

```bash
cat > /tmp/claude/2026-05-06-rc-fork/test_gh_preflight.py <<'EOF'
import importlib.util, sys
spec = importlib.util.spec_from_file_location(
    "repo_clone_mod",
    "/home/gw-t490/github/shibuido/repo-clone/repo-clone",
)
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)
# This will succeed on this workstation (gh installed and authed).
result = mod.gh_preflight()
assert result is True, result
print("gh_preflight smoke test: PASS")
EOF
python3 /tmp/claude/2026-05-06-rc-fork/test_gh_preflight.py
```

Expected: `AttributeError: ...gh_preflight`.

- [ ] **Step 2: Add the helper**

Insert a new section after the Output block:

```python
# ---- gh preflight ----
# Verify the GitHub CLI is installed and authenticated. Used by --fork.

def gh_preflight() -> bool:
    """
    Verify gh is installed and authenticated.

    Emits an error event and returns False on failure; returns True on success.
    """
    if shutil.which("gh") is None:
        emit(
            "gh-missing",
            level="error",
            hint="install GitHub CLI: https://cli.github.com/  (then run `gh auth login`)",
        )
        return False
    try:
        ver = subprocess.run(
            ["gh", "--version"], capture_output=True, text=True, timeout=10
        )
        if ver.returncode != 0:
            emit("gh-missing", level="error", hint="gh --version exited nonzero")
            return False
    except (subprocess.TimeoutExpired, OSError) as e:
        emit("gh-missing", level="error", hint=f"gh --version: {e}")
        return False

    try:
        auth = subprocess.run(
            ["gh", "auth", "status"], capture_output=True, text=True, timeout=10
        )
        if auth.returncode != 0:
            emit(
                "gh-unauthed",
                level="error",
                hint="run `gh auth login` to authenticate the GitHub CLI",
            )
            return False
    except (subprocess.TimeoutExpired, OSError) as e:
        emit("gh-unauthed", level="error", hint=f"gh auth status: {e}")
        return False

    # Best-effort version line for telemetry
    first_line = ver.stdout.strip().splitlines()[0] if ver.stdout else ""
    emit("gh-preflight-ok", level="info", gh_version=first_line)
    return True


def gh_authed_user() -> Optional[str]:
    """Return the authenticated GitHub username, or None on failure."""
    try:
        result = subprocess.run(
            ["gh", "api", "user", "--jq", ".login"],
            capture_output=True, text=True, timeout=10
        )
        if result.returncode == 0 and result.stdout.strip():
            return result.stdout.strip()
    except (subprocess.TimeoutExpired, OSError):
        pass
    return None
```

- [ ] **Step 3: Run smoke test (gh present on this workstation)**

```bash
python3 /tmp/claude/2026-05-06-rc-fork/test_gh_preflight.py
```

Expected: `gh_preflight smoke test: PASS`.

- [ ] **Step 4: Commit**

```bash
git add repo-clone
git commit -m "feat: add gh preflight and authed-user helpers

* gh_preflight() checks 'gh --version' and 'gh auth status'
  Emits gh-missing or gh-unauthed events on failure; gh-preflight-ok on success
* gh_authed_user() returns 'gh api user --jq .login' for default fork-org

Used by the upcoming --fork flow.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 18: Add new exit codes

**Files:**
- Modify: `repo-clone` (extend `ExitCode` class at lines 44-53)

- [ ] **Step 1: Add four new codes**

Replace lines 44-53 (the `ExitCode` class) with:

```python
# Exit codes
class ExitCode:
    SUCCESS = 0
    INVALID_URL = 1
    CLONE_FAILED = 2
    UPDATE_FAILED = 3
    SUBMODULE_FAILED = 4
    PERMISSION_ERROR = 5
    GIT_LFS_FAILED = 6
    NO_CLIPBOARD_TOOL = 7
    GH_MISSING = 8                 # gh CLI missing or unauthed
    HOST_NOT_SUPPORTED_FORK = 9    # --fork on non-GitHub URL
    FORK_CREATION_FAILED = 10      # gh repo fork failed
    FORK_NAME_COLLISION = 11       # target name exists, parent differs
```

- [ ] **Step 2: Verify the script still parses**

```bash
python3 -c "
import importlib.util
spec = importlib.util.spec_from_file_location('m', '/home/gw-t490/github/shibuido/repo-clone/repo-clone')
m = importlib.util.module_from_spec(spec)
spec.loader.exec_module(m)
assert m.ExitCode.GH_MISSING == 8
assert m.ExitCode.HOST_NOT_SUPPORTED_FORK == 9
assert m.ExitCode.FORK_CREATION_FAILED == 10
assert m.ExitCode.FORK_NAME_COLLISION == 11
print('exit codes: PASS')
"
```

Expected: `exit codes: PASS`.

- [ ] **Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add exit codes 8-11 for fork-flow errors

* 8: GH_MISSING (gh CLI missing or unauthed)
* 9: HOST_NOT_SUPPORTED_FORK (non-GitHub URL with --fork)
* 10: FORK_CREATION_FAILED
* 11: FORK_NAME_COLLISION (target exists, parent.full_name mismatches source)

Exit codes 1-7 unchanged. See docs/design/repo-clone-DESIGN-fork-flag.md.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 19: Add `ForkOptions` dataclass

**Files:**
- Modify: `repo-clone` (add dataclass alongside existing `CloneOptions`)

- [ ] **Step 1: Add the dataclass**

After the `WatchStats` dataclass (currently ending around line 153), add:

```python
@dataclass
class ForkOptions:
    """
    Options controlling --fork behavior. Resolved from CLI + env + config + defaults.
    """
    enabled: bool = False                       # Whether --fork was requested
    target_org: Optional[str] = None            # None → resolved to gh-authed user at runtime
    name_template: str = "{repo}"               # Python str.format template
    name_literal: Optional[str] = None          # --fork-name literal override (no templating)
    all_branches: bool = False                  # Default-branch-only otherwise (matches webui)
    upstream_remote_name: str = "upstream"      # Name of the auto-wired remote on the fork side
```

- [ ] **Step 2: Verify**

```bash
python3 -c "
import importlib.util
spec = importlib.util.spec_from_file_location('m', '/home/gw-t490/github/shibuido/repo-clone/repo-clone')
m = importlib.util.module_from_spec(spec)
spec.loader.exec_module(m)
fo = m.ForkOptions()
assert fo.enabled is False
assert fo.name_template == '{repo}'
assert fo.upstream_remote_name == 'upstream'
print('ForkOptions: PASS')
"
```

Expected: `ForkOptions: PASS`.

- [ ] **Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add ForkOptions dataclass

Holds the resolved --fork settings (enabled, target_org, name_template,
name_literal, all_branches, upstream_remote_name). Populated in main()
from CLI + env + config + defaults.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 20: Add fork identity resolution

**Files:**
- Modify: `repo-clone` (new banner section)

- [ ] **Step 1: Write the failing smoke test**

```bash
cat > /tmp/claude/2026-05-06-rc-fork/test_fork_identity.py <<'EOF'
import importlib.util
spec = importlib.util.spec_from_file_location(
    "repo_clone_mod",
    "/home/gw-t490/github/shibuido/repo-clone/repo-clone",
)
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)

info = mod.parse_git_url("https://github.com/acme/widget")

# Default template (identity)
opts = mod.ForkOptions(enabled=True, target_org="VariousForks")
ident = mod.resolve_fork_identity(info, opts, gh_user="alice")
assert ident.target_org == "VariousForks", ident
assert ident.fork_name == "widget", ident

# Custom template
opts2 = mod.ForkOptions(
    enabled=True, target_org="VariousForks",
    name_template="{repo}-by-{org}-fork",
)
ident2 = mod.resolve_fork_identity(info, opts2, gh_user="alice")
assert ident2.fork_name == "widget-by-acme-fork", ident2

# Literal override beats template
opts3 = mod.ForkOptions(
    enabled=True, target_org="VariousForks",
    name_template="{repo}-by-{org}-fork",
    name_literal="my-experiment",
)
ident3 = mod.resolve_fork_identity(info, opts3, gh_user="alice")
assert ident3.fork_name == "my-experiment", ident3

# Default org = gh user when target_org is None
opts4 = mod.ForkOptions(enabled=True, target_org=None)
ident4 = mod.resolve_fork_identity(info, opts4, gh_user="alice")
assert ident4.target_org == "alice", ident4

print("resolve_fork_identity smoke test: PASS")
EOF
python3 /tmp/claude/2026-05-06-rc-fork/test_fork_identity.py
```

Expected: `AttributeError: ...resolve_fork_identity`.

- [ ] **Step 2: Add the resolver**

Add a new banner section after the gh preflight section:

```python
# ---- Fork identity resolution ----

@dataclass
class ForkIdentity:
    """Resolved (target_org, fork_name, fork_clone_url) for a --fork invocation."""
    target_org: str
    fork_name: str
    fork_clone_url: str           # HTTPS clone URL for the fork (post-creation)
    template_used: str            # The template that produced fork_name (for telemetry)


def split_fork_name_shorthand(value: str) -> Tuple[Optional[str], str]:
    """
    Parse --fork-name VALUE.

    If VALUE contains exactly one '/', split into (org, name).
    Else return (None, VALUE).

    Raises ValueError on multiple '/' or empty either side.
    """
    if value.count("/") == 0:
        return (None, value)
    if value.count("/") == 1:
        org, name = value.split("/", 1)
        if not org or not name:
            raise ValueError(
                f"--fork-name: org or name is empty in {value!r} "
                f"(use 'org/name' or just 'name')"
            )
        return (org, name)
    raise ValueError(
        f"--fork-name: {value!r} has more than one '/'; "
        f"use 'org/name' (one slash) or just 'name'"
    )


def resolve_fork_identity(
    info: RepoInfo,
    opts: ForkOptions,
    gh_user: Optional[str],
) -> ForkIdentity:
    """
    Compute (target_org, fork_name, fork_clone_url) from URL, options, and the
    authed gh user (used as default org if opts.target_org is None).

    Raises ValueError if neither target_org nor gh_user is available.
    """
    # Target org
    target_org = opts.target_org or gh_user
    if not target_org:
        raise ValueError(
            "fork target org could not be resolved: "
            "set fork.target_org in config, pass --fork-org, "
            "or ensure `gh auth status` shows a logged-in user"
        )

    # Fork name
    if opts.name_literal is not None:
        fork_name = opts.name_literal
        template_used = "(literal)"
    else:
        try:
            fork_name = opts.name_template.format(
                repo=info.repo,
                org=info.username,
                host=info.shortcode,
            )
            template_used = opts.name_template
        except (KeyError, IndexError) as e:
            raise ValueError(
                f"fork.name_template uses unknown variable: {e}; "
                f"available: {{repo}}, {{org}}, {{host}}"
            )

    # Fork clone URL (always HTTPS for the v1 GitHub-only path)
    fork_clone_url = f"https://github.com/{target_org}/{fork_name}.git"

    return ForkIdentity(
        target_org=target_org,
        fork_name=fork_name,
        fork_clone_url=fork_clone_url,
        template_used=template_used,
    )
```

- [ ] **Step 3: Run smoke test, expect PASS**

```bash
python3 /tmp/claude/2026-05-06-rc-fork/test_fork_identity.py
```

Expected: `resolve_fork_identity smoke test: PASS`.

- [ ] **Step 4: Test the shorthand splitter**

```bash
python3 -c "
import importlib.util
spec = importlib.util.spec_from_file_location('m', '/home/gw-t490/github/shibuido/repo-clone/repo-clone')
m = importlib.util.module_from_spec(spec)
spec.loader.exec_module(m)

# No slash: (None, value)
assert m.split_fork_name_shorthand('widget') == (None, 'widget')
# One slash: split
assert m.split_fork_name_shorthand('greg/widget-experimental') == ('greg', 'widget-experimental')
# Multiple slashes: error
try:
    m.split_fork_name_shorthand('a/b/c')
    assert False, 'should raise'
except ValueError as e:
    assert 'more than one' in str(e)
# Empty side: error
try:
    m.split_fork_name_shorthand('/widget')
    assert False, 'should raise'
except ValueError as e:
    assert 'empty' in str(e)
print('split_fork_name_shorthand: PASS')
"
```

Expected: `split_fork_name_shorthand: PASS`.

- [ ] **Step 5: Commit**

```bash
git add repo-clone
git commit -m "feat: add fork identity resolution (template + literal + shorthand)

* split_fork_name_shorthand(value): parses 'org/name' shorthand for --fork-name
* resolve_fork_identity(info, opts, gh_user): produces target_org + fork_name
  + fork_clone_url, applying the template or the literal override
* ForkIdentity dataclass holds the resolved triple

Pure functions; covered by inline smoke tests.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 21: Add idempotent fork existence check

**Files:**
- Modify: `repo-clone` (new section)

- [ ] **Step 1: Add the helper**

In a new banner section `# ---- Fork creation (idempotent) ----`:

```python
# ---- Fork creation (idempotent) ----

def fork_exists_on_github(target_org: str, fork_name: str) -> Tuple[bool, Optional[str]]:
    """
    Check whether a repo exists at github.com/{target_org}/{fork_name} and,
    if so, return its parent.full_name (the source repo it forks from).

    Returns:
        (exists: bool, parent_full_name: Optional[str])
        parent_full_name is None if the repo exists but is not a fork.
    """
    try:
        result = subprocess.run(
            ["gh", "api",
             f"repos/{target_org}/{fork_name}",
             "--jq", ".parent.full_name // \"\""],
            capture_output=True, text=True, timeout=15
        )
    except (subprocess.TimeoutExpired, OSError) as e:
        emit("fork-existence-check-failed", level="warn", error=str(e))
        return (False, None)

    if result.returncode == 0:
        parent = result.stdout.strip() or None
        return (True, parent)

    # gh prints "HTTP 404" on stderr for missing repos; treat any non-zero as
    # "not found" only when stderr indicates 404. Otherwise treat as error.
    err = (result.stderr or "")
    if "HTTP 404" in err or "Not Found" in err:
        return (False, None)
    emit(
        "fork-existence-check-failed",
        level="warn",
        gh_exit=result.returncode,
        stderr=err.strip()[:200],
    )
    return (False, None)
```

- [ ] **Step 2: Smoke test against a known repo**

```bash
python3 -c "
import importlib.util
spec = importlib.util.spec_from_file_location('m', '/home/gw-t490/github/shibuido/repo-clone/repo-clone')
m = importlib.util.module_from_spec(spec)
spec.loader.exec_module(m)
m.set_output_mode('human')
exists, parent = m.fork_exists_on_github('shibuido', 'repo-clone')
assert exists is True, (exists, parent)
print('fork_exists_on_github (existing): PASS')

exists2, parent2 = m.fork_exists_on_github('shibuido', 'this-name-does-not-exist-zzz')
assert exists2 is False, (exists2, parent2)
print('fork_exists_on_github (missing): PASS')
"
```

Expected: both PASS lines.

- [ ] **Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add fork_exists_on_github() idempotency probe

* Calls gh api repos/{org}/{name} --jq .parent.full_name
* Returns (exists, parent_full_name) tuple
* parent is None if repo exists but isn't a fork
* 404s detected via stderr pattern; other errors emit fork-existence-check-failed

Used by the upcoming create-or-skip fork flow.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 22: Add fork creation invocation

**Files:**
- Modify: `repo-clone` (extend the fork-creation section)

- [ ] **Step 1: Add the creator**

In the same `# ---- Fork creation (idempotent) ----` section, append:

```python
def create_github_fork(
    info: RepoInfo,
    target_org: str,
    fork_name: str,
    all_branches: bool,
    is_self_org: bool,
    dry_run: bool = False,
) -> int:
    """
    Create a fork via `gh repo fork`. Returns ExitCode.SUCCESS or
    ExitCode.FORK_CREATION_FAILED.

    is_self_org: True if target_org equals the authed gh user. In that case
    `gh repo fork` does not need --org. When forking to a different org, we
    pass --org.
    """
    source = f"{info.username}/{info.repo}"
    cmd = ["gh", "repo", "fork", source, "--clone=false"]

    if not is_self_org:
        cmd.extend(["--org", target_org])

    # Only pass --fork-name if it differs from source repo name (gh quirk:
    # passing --fork-name with the same name still works but is noisy)
    if fork_name != info.repo:
        cmd.extend(["--fork-name", fork_name])

    if not all_branches:
        cmd.append("--default-branch-only")

    if dry_run:
        emit(
            "fork-creating",
            level="info",
            source=source,
            target_org=target_org,
            fork_name=fork_name,
            all_branches=all_branches,
            dry_run=True,
            cmd=" ".join(cmd),
        )
        return ExitCode.SUCCESS

    emit(
        "fork-creating",
        level="info",
        source=source,
        target_org=target_org,
        fork_name=fork_name,
        all_branches=all_branches,
    )

    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=60)
    except (subprocess.TimeoutExpired, OSError) as e:
        emit("fork-creation-failed", level="error",
             source=source, target_org=target_org, fork_name=fork_name,
             error=str(e))
        return ExitCode.FORK_CREATION_FAILED

    if result.returncode != 0:
        emit(
            "fork-creation-failed",
            level="error",
            source=source,
            target_org=target_org,
            fork_name=fork_name,
            gh_exit_code=result.returncode,
            stderr=(result.stderr or "").strip()[:500],
        )
        return ExitCode.FORK_CREATION_FAILED

    clone_url = f"https://github.com/{target_org}/{fork_name}.git"
    emit(
        "fork-created",
        level="info",
        source=source,
        target_org=target_org,
        fork_name=fork_name,
        clone_url=clone_url,
    )
    return ExitCode.SUCCESS
```

- [ ] **Step 2: Dry-run smoke test**

```bash
python3 -c "
import importlib.util
spec = importlib.util.spec_from_file_location('m', '/home/gw-t490/github/shibuido/repo-clone/repo-clone')
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)
mod.set_output_mode('human')
mod.set_show_info(True)
info = mod.parse_git_url('https://github.com/acme/widget')
rc = mod.create_github_fork(info, 'VariousForks', 'widget-by-acme-fork',
                            all_branches=False, is_self_org=False, dry_run=True)
assert rc == 0
print('create_github_fork dry-run: PASS')
"
```

Expected: `INFO: fork-creating ...` line and `PASS`.

- [ ] **Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add create_github_fork() invocation

* Shells out to 'gh repo fork SOURCE --clone=false [--org X] [--fork-name Y]
  [--default-branch-only]'
* Emits fork-creating before, fork-created on success, fork-creation-failed
  on error
* Honors dry_run for preview without API calls
* Returns ExitCode.SUCCESS / ExitCode.FORK_CREATION_FAILED

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 23: Add fork-side clone with backoff for fork-just-created races

**Files:**
- Modify: `repo-clone` (new banner section)

- [ ] **Step 1: Add the helper**

New banner section `# ---- Fork-side clone & upstream wiring ----`:

```python
# ---- Fork-side clone & upstream wiring ----

def clone_fork_with_retry(
    fork_url: str,
    target_org: str,
    fork_name: str,
    options: CloneOptions,
    max_wait_s: float = 10.0,
) -> int:
    """
    Clone (or update) the fork side at $base/$host/$forkOrg/$forkName.

    Reuses the existing parse_git_url + clone_repo path. If the fork was just
    created and GitHub hasn't propagated it yet (HTTP 404), retry with
    exponential backoff up to max_wait_s.
    """
    info = parse_git_url(fork_url)
    waited = 0.0
    delay = 1.0
    while True:
        rc = clone_repo(info, options)
        if rc == ExitCode.SUCCESS:
            return rc
        if waited >= max_wait_s:
            return rc
        # Only retry on a likely "fork not propagated yet" 404
        emit(
            "fork-clone-retrying",
            level="warn",
            target_org=target_org,
            fork_name=fork_name,
            waited_s=round(waited, 2),
        )
        time.sleep(delay)
        waited += delay
        delay = min(delay * 1.5, 4.0)
```

- [ ] **Step 2: Add the upstream-wiring helper**

Append to the same section:

```python
def wire_upstream_remote(
    fork_path: Path,
    upstream_url: str,
    remote_name: str,
    dry_run: bool = False,
) -> None:
    """
    Idempotently ensure `<remote_name>` on the fork-side clone points at
    upstream_url.

    * Absent → add it (emit upstream-added).
    * Present and matches → skip (emit upstream-already-correct).
    * Present and differs → WARN, do not overwrite (emit upstream-mismatch).
    """
    try:
        result = run_git(
            ["remote", "get-url", remote_name],
            cwd=fork_path,
            verbosity=0,
            dry_run=False,  # always actually probe, even on dry-run
            check=False,
        )
        existing = result.stdout.strip() if result.returncode == 0 else None
    except Exception:
        existing = None

    if existing is None:
        if dry_run:
            emit("upstream-added", level="info", fork_path=str(fork_path),
                 remote_name=remote_name, upstream_url=upstream_url, dry_run=True)
            return
        run_git(
            ["remote", "add", remote_name, upstream_url],
            cwd=fork_path, verbosity=0, dry_run=False,
        )
        emit("upstream-added", level="info", fork_path=str(fork_path),
             remote_name=remote_name, upstream_url=upstream_url)
        return

    if existing == upstream_url:
        emit("upstream-already-correct", level="info",
             fork_path=str(fork_path), remote_name=remote_name)
        return

    emit(
        "upstream-mismatch",
        level="warn",
        fork_path=str(fork_path),
        remote_name=remote_name,
        current_url=existing,
        expected_url=upstream_url,
    )
```

- [ ] **Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add fork-side clone (with backoff) and upstream remote wiring

* clone_fork_with_retry() reuses clone_repo() on the fork URL, retrying with
  exponential backoff up to ~10s for fork-just-created propagation races
* wire_upstream_remote() idempotently ensures the upstream remote points at
  the source URL: absent → add, matches → skip, differs → WARN don't overwrite
* Both emit structured events for telemetry

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 24: Add the orchestrating fork flow

**Files:**
- Modify: `repo-clone` (add new top-level function)

- [ ] **Step 1: Add the orchestrator**

Add after the wire_upstream_remote helper:

```python
def repo_clone_fork(url: str, options: CloneOptions, fork_opts: ForkOptions) -> int:
    """
    Orchestrate the --fork flow:
      1. Parse URL  (must be GitHub for v1)
      2. gh preflight
      3. Resolve fork identity
      4. Source-side clone/update
      5. Idempotent fork creation
      6. Fork-side clone (with retry)
      7. Wire upstream remote
      8. Emit summary

    Returns the appropriate ExitCode.
    """
    try:
        info = parse_git_url(url)
    except ValueError as e:
        emit("invalid-url", level="error", url=url, error=str(e))
        return ExitCode.INVALID_URL

    if info.hosting != "github.com":
        emit(
            "host-not-supported-for-fork",
            level="error",
            host=info.hosting,
            hint=("v1 supports GitHub only. PRs welcome to add more hosts; "
                  "see repo-clone.FUTURE_WORK.md and "
                  "https://github.com/shibuido/repo-clone/issues"),
        )
        return ExitCode.HOST_NOT_SUPPORTED_FORK

    if not gh_preflight():
        return ExitCode.GH_MISSING

    gh_user = gh_authed_user()
    try:
        ident = resolve_fork_identity(info, fork_opts, gh_user)
    except ValueError as e:
        emit("fork-identity-failed", level="error", error=str(e))
        return ExitCode.GH_MISSING

    emit(
        "fork-identity-resolved",
        level="info",
        source_org=info.username,
        source_repo=info.repo,
        target_org=ident.target_org,
        fork_name=ident.fork_name,
        template=ident.template_used,
    )

    actions: List[str] = []

    # 1. Source-side clone/update
    src_rc = clone_repo(info, options)
    if src_rc != ExitCode.SUCCESS:
        return src_rc
    actions.append("cloned-source")

    # 2. Idempotent fork creation
    is_self_org = (gh_user is not None and ident.target_org == gh_user)
    exists, parent = fork_exists_on_github(ident.target_org, ident.fork_name)
    expected_parent = f"{info.username}/{info.repo}"
    if exists:
        if parent == expected_parent:
            emit("fork-exists", level="info",
                 target_org=ident.target_org, fork_name=ident.fork_name,
                 parent_full_name=parent)
        else:
            emit(
                "fork-collision",
                level="error",
                target_org=ident.target_org,
                fork_name=ident.fork_name,
                parent_full_name=parent,
                expected_parent=expected_parent,
            )
            return ExitCode.FORK_NAME_COLLISION
    else:
        rc = create_github_fork(
            info, ident.target_org, ident.fork_name,
            fork_opts.all_branches, is_self_org, dry_run=options.dry_run,
        )
        if rc != ExitCode.SUCCESS:
            return rc
        actions.append("created-fork")

    # 3. Fork-side clone
    fork_rc = clone_fork_with_retry(
        ident.fork_clone_url, ident.target_org, ident.fork_name, options
    )
    if fork_rc != ExitCode.SUCCESS:
        return fork_rc
    actions.append("cloned-fork")

    # 4. Wire upstream remote on fork
    base = options.base_dir or Path.home()
    fork_path = base / info.shortcode / ident.target_org / ident.fork_name
    upstream_url = normalize_clone_url(info)
    wire_upstream_remote(
        fork_path,
        upstream_url,
        fork_opts.upstream_remote_name,
        dry_run=options.dry_run,
    )
    actions.append("added-upstream")

    # 5. Summary
    src_path = base / info.shortcode / info.username / info.repo
    emit(
        "summary",
        level="info",
        source_path=str(src_path),
        fork_path=str(fork_path),
        actions=actions,
    )

    return ExitCode.SUCCESS
```

- [ ] **Step 2: Smoke test (dry-run only — no GitHub state changed)**

```bash
python3 -c "
import importlib.util
spec = importlib.util.spec_from_file_location('m', '/home/gw-t490/github/shibuido/repo-clone/repo-clone')
m = importlib.util.module_from_spec(spec)
spec.loader.exec_module(m)
m.set_output_mode('human')
m.set_show_info(True)

# Invalid URL → exit 1
rc = m.repo_clone_fork('not a url', m.CloneOptions(dry_run=True), m.ForkOptions(enabled=True))
assert rc == 1, rc

# Non-GitHub URL → exit 9
rc2 = m.repo_clone_fork('https://gitlab.com/foo/bar', m.CloneOptions(dry_run=True), m.ForkOptions(enabled=True))
assert rc2 == 9, rc2

print('repo_clone_fork error paths: PASS')
"
```

Expected: `repo_clone_fork error paths: PASS`.

- [ ] **Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add repo_clone_fork() orchestrator

Top-level function that runs the full --fork flow:
  parse URL → host gate → gh preflight → resolve identity →
  source clone/update → idempotent fork create → fork clone (with retry) →
  upstream remote wire → summary

Returns the right ExitCode for each failure mode (1, 9, 8, 2/3/4/5/6,
10, 11, 0).

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 25: Add `--help-for-agents` text generator

**Files:**
- Modify: `repo-clone` (new banner section near end of script)

- [ ] **Step 1: Add the generator**

New banner section `# ---- Agent help ----`:

```python
# ---- Agent help ----
# See docs/design/repo-clone-DESIGN-help-for-agents.md for the style contract.
# Each section is a small string fragment so it can be edited independently.

_AGENT_HELP_SECTIONS = {
    "IDENTITY": """\
repo-clone: paste-and-organize git repo cloner; optional --fork to create
+ clone a fork on the host. Path layout: $HOME/$host/$org/$repo (deterministic).
""",
    "INVOCATION": """\
GRAMMAR
  repo-clone <url>                                # clone or update
  repo-clone --fork <url>                         # + create fork on host
  repo-clone --watch                              # clipboard daemon mode
  repo-clone --help                               # human help
  repo-clone --help-for-agents | help-for-agents  # this output

CANONICAL EXAMPLES
  repo-clone https://github.com/acme/widget
  repo-clone -F --fork-org VariousForks https://github.com/acme/widget
  repo-clone --fork --fork-name greg/widget-experimental https://github.com/acme/widget
  repo-clone --fork --jsonl https://github.com/acme/widget | jq -c
""",
    "EXIT_CODES": """\
| code | meaning                                                |
|------|--------------------------------------------------------|
|   0  | success                                                |
|   1  | invalid URL                                            |
|   2  | clone failed                                           |
|   3  | update failed                                          |
|   4  | submodule operation failed                             |
|   5  | permission error / unrelated dir at expected path      |
|   6  | git-lfs missing (Hugging Face)                         |
|   7  | no clipboard tool found (--watch)                      |
|   8  | gh CLI missing or not authenticated                    |
|   9  | host not supported for --fork (non-GitHub in v1)       |
|  10  | gh repo fork failed                                    |
|  11  | fork-name collision: target_org/name exists, not a     |
|      | fork of source                                         |
""",
    "STATE_MACHINE": """\
URL → parse → host_gate(github.com if --fork) → gh_preflight(if --fork) →
  resolve_identity(URL, config, CLI) →
  source_clone_or_update($base/$host/$src_org/$repo) →
  IF --fork:
    fork_existence_check(target_org, fork_name):
      none      → gh repo fork --clone=false
      matches   → INFO continue
      mismatch  → ERROR exit 11
    fork_clone_or_update($base/$host/$target_org/$fork_name) (retry on 404 ≤10s)
    upstream_remote(absent→add | match→skip | differ→WARN)
  emit summary
""",
    "JSONL_SCHEMA": """\
--jsonl emits one compact JSON object per stdout line. Every record has:
  {"event": "<name>", "level": "info"|"warn"|"error"|"debug", "ts": "<ISO-8601>", ...}
Event names are stable; new fields/events are additive only.
Full catalogue: docs/design/repo-clone-DESIGN-jsonl-output.md
""",
    "IDEMPOTENCY": """\
Re-running the SAME invocation is safe:
  source dir exists+repo → fetch+pull (skip pull if dirty)
  fork on GH exists+matches → skip create
  fork dir exists+origin matches → fetch+pull
  upstream remote exists+matches → skip
  upstream remote exists+differs → WARN, do NOT overwrite
""",
    "FAILURE_RECOVERY": """\
TIMING ENVELOPES
  fork creation:   5–30 s typical (GitHub-side; can spike during incidents)
  large clone:     minutes (depends on repo size; --shallow helps)
  gh preflight:    < 2 s

RETRYABLE
  fork clone 404 within ~10s of fork-created  (auto-retried up to ~10s)

NOT AUTO-RETRIED  (treat as terminal; surface to user)
  network failures (transient or persistent — caller should re-run)
  gh auth failures (needs human intervention: `gh auth login`)
  fork collisions (needs human intervention: choose new --fork-name)
""",
    "CONTEXTUAL_FIT": """\
This tool sits inside agent workflows as the bookmark-and-clone step.
Path layout invariant: every clone lives at $base/$host/$org/$repo where
  $base    = options.base_dir or $HOME (overridable: --base-dir, REPO_CLONE_BASE_DIR)
  $host    = shortcode of hosting domain (github, gitlab, codeberg, ...; HF keeps full domain)
  $org     = source URL's org/user
  $repo    = source URL's repo name (no .git suffix)

Companion tools for searching cloned content on disk:
  find-snapshot     # cached daily find index
  sk -f PATTERN     # fuzzy file finder
  fzf               # interactive selector
""",
    "TROUBLESHOOTING_POINTERS": """\
LOCAL:  docs/repo-clone-TROUBLESHOOTING.md
REMOTE: https://github.com/shibuido/repo-clone/blob/master/docs/repo-clone-TROUBLESHOOTING.md
""",
    "REPORT_BUGS": """\
Issue tracker: https://github.com/shibuido/repo-clone/issues
Recommended template:
  Title:   [short symptom]
  Body:    full failing command, full stderr, gh --version, git --version, OS
  Labels:  byAI (if AI-authored)
""",
}


def render_agent_help() -> str:
    """Render the full --help-for-agents text."""
    parts = ["repo-clone — agent help (for human help: `repo-clone --help`)\n"]
    for name, body in _AGENT_HELP_SECTIONS.items():
        parts.append(f"=== {name} ===\n{body.rstrip()}\n")
    return "\n".join(parts)
```

- [ ] **Step 2: Smoke test**

```bash
python3 -c "
import importlib.util
spec = importlib.util.spec_from_file_location('m', '/home/gw-t490/github/shibuido/repo-clone/repo-clone')
m = importlib.util.module_from_spec(spec)
spec.loader.exec_module(m)
text = m.render_agent_help()
for section in ('IDENTITY', 'INVOCATION', 'EXIT_CODES', 'JSONL_SCHEMA', 'CONTEXTUAL_FIT'):
    assert f'=== {section} ===' in text, section
print('render_agent_help: PASS')
"
```

Expected: `render_agent_help: PASS`.

- [ ] **Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add render_agent_help() for --help-for-agents output

Section fragments (IDENTITY, INVOCATION, EXIT_CODES, STATE_MACHINE,
JSONL_SCHEMA, IDEMPOTENCY, FAILURE_RECOVERY, CONTEXTUAL_FIT,
TROUBLESHOOTING_POINTERS, REPORT_BUGS) live in a dict so each section
is independently editable. See
docs/design/repo-clone-DESIGN-help-for-agents.md.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 26: Wire new flags into argparse and main()

**Files:**
- Modify: `repo-clone` `main()` (currently lines 856-967)

This is the biggest single edit. Walk it carefully.

- [ ] **Step 1: Special-case the `help-for-agents` subcommand BEFORE argparse**

In `main()`, immediately after `parser.parse_args()` would be too late.
Add this BEFORE `parser = argparse.ArgumentParser(...)`:

```python
def main() -> None:
    """CLI entry point."""
    # Subcommand alias: `repo-clone help-for-agents` → render and exit.
    if len(sys.argv) >= 2 and sys.argv[1] == "help-for-agents":
        print(render_agent_help())
        sys.exit(0)

    parser = argparse.ArgumentParser(
        ...
```

- [ ] **Step 2: Add new flags to the argparse setup**

After the existing `parser.add_argument("--dry-run", ...)` line (around 901-902),
add:

```python
    parser.add_argument("-F", "--fork", action="store_true",
                        help="Create a fork on the host, then clone source AND fork")
    parser.add_argument("--fork-all", action="store_true",
                        help="Fork copies all branches (implies --fork)")
    parser.add_argument("--fork-default", action="store_true",
                        help="Fork copies only default branch (implies --fork)")
    parser.add_argument("--fork-org", default=None,
                        help="Override target org for the fork (implies --fork)")
    parser.add_argument("--fork-name", default=None,
                        help="Override fork repo name; 'org/name' shorthand "
                             "splits into --fork-org/--fork-name (implies --fork)")
    parser.add_argument("--jsonl", action="store_true",
                        help="Emit one compact JSON object per line on stdout")
    parser.add_argument("--help-for-agents", action="store_true",
                        help="Print agent-optimized help and exit "
                             "(also: `repo-clone help-for-agents`)")
    parser.add_argument("--config", type=Path, default=None,
                        help="Use an alternative config file path")
```

- [ ] **Step 3: Replace the epilog with one that mentions agents**

Replace the existing `epilog="""..."""` (around line 862-879) with:

```python
        epilog="""\
🤖 AI agents: run `repo-clone --help-for-agents` (or `repo-clone help-for-agents`)
   for an agent-optimized briefing — denser format, troubleshooting paths,
   exit-code semantics, JSONL contract.
   Issues / unclear behavior → https://github.com/shibuido/repo-clone/issues

Examples:
  %(prog)s git@github.com:user/repo.git
  %(prog)s https://github.com/user/repo
  %(prog)s --shallow https://github.com/user/large-repo
  %(prog)s --fork https://github.com/acme/widget
  %(prog)s --fork --fork-name greg/widget-mod https://github.com/acme/widget
  %(prog)s --fork --jsonl https://github.com/acme/widget | jq -c
  %(prog)s --watch                    Monitor clipboard continuously
  %(prog)s --watch --selection both   Monitor both clipboards (Linux)

Config file: ~/.config/shibuido/repo-clone/repo-clone-config.yaml
"""
```

- [ ] **Step 4: Add post-parse handling for --help-for-agents flag**

Right after `args = parser.parse_args()`:

```python
    if args.help_for_agents:
        print(render_agent_help())
        sys.exit(0)
```

- [ ] **Step 5: Validate flag combinations**

Right after the help-for-agents handler:

```python
    if args.fork_all and args.fork_default:
        parser.error("--fork-all and --fork-default are mutually exclusive")

    # Any --fork-* flag implies --fork
    fork_implied = (args.fork or args.fork_all or args.fork_default
                    or args.fork_org or args.fork_name)
    args.fork = bool(fork_implied)

    if args.watch and args.fork:
        parser.error("--fork cannot be combined with --watch")
```

- [ ] **Step 6: Wire JSONL output mode early**

Just after the `VERBOSITY` global is set (around line 930), add:

```python
    if args.jsonl:
        set_output_mode("jsonl")
```

- [ ] **Step 7: Build `ForkOptions` from precedence chain**

After loading `config = load_config(args.config)` (currently inside the
`if args.watch:` branch — pull it out so it loads always):

Replace the existing `try: if args.watch: ... else: exit_code = repo_clone(args.url, options)` block with:

```python
    # Always load config so non-watch paths see fork:/output: sections.
    config = load_config(args.config)

    # Apply output.show_info from config (CLI --jsonl already handled above)
    if config.get("output", {}).get("show_info"):
        set_show_info(True)

    # Existing base_dir from config (was watch-only; promote to global)
    if options.base_dir is None and "base_dir" in config:
        bd = config["base_dir"]
        if bd.startswith("~"):
            bd = os.path.expanduser(bd)
        options.base_dir = Path(bd)

    # Resolve --fork-name shorthand (org/name)
    fork_name_org_override: Optional[str] = None
    fork_name_literal: Optional[str] = None
    if args.fork_name is not None:
        try:
            fork_name_org_override, fork_name_literal = split_fork_name_shorthand(args.fork_name)
        except ValueError as e:
            parser.error(str(e))

    # Reconcile --fork-org with shorthand-derived org
    if fork_name_org_override is not None:
        if args.fork_org is not None and args.fork_org != fork_name_org_override:
            parser.error(
                f"--fork-org ({args.fork_org}) conflicts with org from "
                f"--fork-name ({fork_name_org_override})"
            )
        args.fork_org = fork_name_org_override

    # Build ForkOptions: CLI > env > config > default
    cfg_fork = config.get("fork", {}) if isinstance(config.get("fork"), dict) else {}
    target_org = (
        args.fork_org
        or os.environ.get("REPO_CLONE_FORK_ORG")
        or cfg_fork.get("target_org")
    )

    # all_branches: CLI is two-flag mutex; if neither given, use config; else default
    if args.fork_all:
        all_branches = True
    elif args.fork_default:
        all_branches = False
    else:
        all_branches = bool(cfg_fork.get("all_branches", False))

    fork_opts = ForkOptions(
        enabled=args.fork,
        target_org=target_org,
        name_template=cfg_fork.get("name_template", "{repo}"),
        name_literal=fork_name_literal,
        all_branches=all_branches,
        upstream_remote_name=cfg_fork.get("upstream_remote_name", "upstream"),
    )

    try:
        if args.watch:
            watch_config = WatchConfig(
                interval=args.interval if args.interval != 0.5 else config.get("watch_interval", 0.5),
                selection=args.selection if args.selection != "clipboard" else config.get("watch_selection", "clipboard"),
                clipboard_command=config.get("clipboard_command"),
                extra_hosts=config.get("extra_hosts", []),
            )
            exit_code = watch_clipboard(options, watch_config)
        elif fork_opts.enabled:
            exit_code = repo_clone_fork(args.url, options, fork_opts)
        else:
            exit_code = repo_clone(args.url, options)
        sys.exit(exit_code)
    except KeyboardInterrupt:
        print("\nInterrupted", file=sys.stderr)
        sys.exit(130)
```

- [ ] **Step 8: Smoke test — `--help` shows new flags**

```bash
/home/gw-t490/github/shibuido/repo-clone/repo-clone --help 2>&1 | head -50
```

Expected: includes `-F, --fork`, `--fork-all`, `--fork-default`, `--fork-org`,
`--fork-name`, `--jsonl`, `--help-for-agents`, the agent-help block in epilog.

- [ ] **Step 9: Smoke test — `help-for-agents` subcommand works**

```bash
/home/gw-t490/github/shibuido/repo-clone/repo-clone help-for-agents | head -40
```

Expected: starts with `repo-clone — agent help` and shows `=== IDENTITY ===`.

- [ ] **Step 10: Smoke test — flag mutex enforced**

```bash
/home/gw-t490/github/shibuido/repo-clone/repo-clone --fork-all --fork-default https://github.com/foo/bar; echo "exit=$?"
```

Expected: argparse error mentioning mutual exclusion; exit 2 (argparse default).

- [ ] **Step 11: Commit**

```bash
git add repo-clone
git commit -m "feat: wire --fork, --jsonl, --help-for-agents into main()

* New flags: -F/--fork, --fork-all, --fork-default, --fork-org,
  --fork-name (with org/name shorthand), --jsonl, --help-for-agents,
  --config
* Subcommand alias: 'repo-clone help-for-agents' (special-cased before argparse)
* Mutex: --fork-all xor --fork-default; --fork incompatible with --watch
* Any --fork-* flag implies --fork
* config loaded for all paths now (not just --watch)
* ForkOptions built from CLI > env > config > defaults precedence
* repo_clone_fork() invoked when fork.enabled

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 27: End-to-end manual test on a throwaway repo

**Files:**
- Modify: `repo-clone.DEV_NOTES.md` (add manual test cases)

This task uses real GitHub state. Pick a small throwaway repo to fork (the
maintainer's own — e.g., `shibuido/repo-clone` itself can be re-forked into
a test org if one is available, or use `--dry-run` to avoid actual API calls).

- [ ] **Step 1: Dry-run end-to-end**

```bash
mkdir -p /tmp/claude/2026-05-06-rc-fork/e2e
/home/gw-t490/github/shibuido/repo-clone/repo-clone \
  --base-dir /tmp/claude/2026-05-06-rc-fork/e2e \
  --fork --dry-run -v \
  https://github.com/shibuido/repo-clone
```

Expected stderr: shows source clone plan, fork-creating event, fork clone plan,
upstream-added plan, and a summary. No exit nonzero (returns 0 for dry-run).

- [ ] **Step 2: JSONL dry-run**

```bash
/home/gw-t490/github/shibuido/repo-clone/repo-clone \
  --base-dir /tmp/claude/2026-05-06-rc-fork/e2e \
  --fork --jsonl --dry-run \
  https://github.com/shibuido/repo-clone | tee /tmp/claude/2026-05-06-rc-fork/jsonl-out.txt
```

Pipe through `jq -c .` to verify each line is valid JSON:

```bash
cat /tmp/claude/2026-05-06-rc-fork/jsonl-out.txt | jq -c . > /dev/null && echo "all lines valid JSON: PASS"
```

Expected: `all lines valid JSON: PASS`.

- [ ] **Step 3: Non-GitHub URL with --fork → exit 9**

```bash
/home/gw-t490/github/shibuido/repo-clone/repo-clone \
  --base-dir /tmp/claude/2026-05-06-rc-fork/e2e \
  --fork --dry-run \
  https://gitlab.com/inkscape/inkscape; echo "exit=$?"
```

Expected: ERROR mentioning v1 supports GitHub only; `exit=9`.

- [ ] **Step 4: --fork-name shorthand**

```bash
/home/gw-t490/github/shibuido/repo-clone/repo-clone \
  --base-dir /tmp/claude/2026-05-06-rc-fork/e2e \
  --fork --fork-name greg/widget-test --dry-run -v \
  https://github.com/acme/widget 2>&1 | grep -E "target_org|fork_name"
```

Expected: shows `target_org=greg` and `fork_name=widget-test`.

- [ ] **Step 5: Conflicting --fork-org + --fork-name org/name shorthand**

```bash
/home/gw-t490/github/shibuido/repo-clone/repo-clone \
  --fork --fork-org other --fork-name greg/widget --dry-run \
  https://github.com/acme/widget; echo "exit=$?"
```

Expected: argparse error about conflict; `exit=2`.

- [ ] **Step 6: Update DEV_NOTES with these test cases**

Append to `repo-clone.DEV_NOTES.md` § Testing → Manual Test Cases:

```markdown
### --fork end-to-end (dry-run)

```bash
# Basic: source + fork, dry-run
repo-clone --base-dir /tmp/test --fork --dry-run -v https://github.com/shibuido/repo-clone

# JSONL output
repo-clone --base-dir /tmp/test --fork --jsonl --dry-run https://github.com/shibuido/repo-clone | jq -c .

# Non-GitHub host → exit 9
repo-clone --fork --dry-run https://gitlab.com/inkscape/inkscape

# Shorthand --fork-name
repo-clone --fork --fork-name greg/widget-test --dry-run -v https://github.com/acme/widget

# Conflicting flags
repo-clone --fork-all --fork-default https://github.com/foo/bar      # exit 2
repo-clone --fork --fork-org X --fork-name Y/Z https://github.com/foo/bar  # exit 2

# Agent help
repo-clone --help-for-agents | head -40
repo-clone help-for-agents | head -40
```
```

- [ ] **Step 7: Commit**

```bash
git add repo-clone.DEV_NOTES.md
git commit -m "docs(dev-notes): add --fork manual test cases

End-to-end dry-run, JSONL validation, host gate, shorthand, mutex flags,
agent help. Run before any --fork-related PR.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 28: Rewrite `repo-clone.README.md` into tiered structure

**Files:**
- Modify: `repo-clone.README.md`

The existing README is good but needs the tiered structure from spec §9 and
new content for `--fork`, `--jsonl`, agent help.

- [ ] **Step 1: Read the current file to preserve good copy**

```bash
cat /home/gw-t490/github/shibuido/repo-clone/repo-clone.README.md
```

Note in particular the current "Stop organizing repositories. Start exploring
them." pitch and the "GitHub window-shopping" framing — keep them.

- [ ] **Step 2: Replace with the tiered version**

Write `repo-clone.README.md` (full body):

```markdown
# repo-clone

**Stop organizing repositories. Start exploring them.**

Ever been GitHub window-shopping? You find one interesting project, which links
to another, then a dependency you want to inspect, then a fork with extra
features... Before you know it, you've got a dozen browser tabs and the
tedious prospect of:

```bash
mkdir -p ~/repos/someorg && cd ~/repos/someorg && git clone --recursive ...
```

...for each one. Or worse, everything dumped in `~/Downloads` or a flat
`~/projects` folder where nothing is findable later.

**repo-clone** handles the busywork. Drop it in your `$PATH` and paste URLs:

```bash
while true; do read -p "url: " url; repo-clone "$url"; done
```

Each repository lands exactly where you'd expect it:

* `~/github/torvalds/linux`
* `~/gitlab/inkscape/inkscape`
* `~/huggingface.co/TheBloke/Llama-2-7B-GPTQ`

Submodules initialize automatically. Run it again on an existing repo and it
updates cleanly. No thinking required—just copy, paste, explore.

🤖 **AI agents:** see `repo-clone --help-for-agents` for an agent-optimized
briefing.

---

## Quickstart (zeroconf — 30 seconds to first clone)

Requirements: Python 3.9+, `git`. (Optional: `git-lfs` for Hugging Face,
`gh` for `--fork`, `pyyaml` for richer config.)

```bash
# Drop the script in $PATH and make it executable
curl -L https://raw.githubusercontent.com/shibuido/repo-clone/master/repo-clone \
  -o ~/.local/bin/repo-clone && chmod +x ~/.local/bin/repo-clone

# Clone
repo-clone https://github.com/torvalds/linux
# → ~/github/torvalds/linux  (cloned, submodules initialized)
```

Re-run the same command later and it updates cleanly.

→ Next: [Fork tier](#fork-tier-and-fork)

---

## Fork tier — `--fork`

`--fork` creates a fork on the host AND clones both source and fork:

```bash
repo-clone --fork https://github.com/acme/widget
# → ~/github/acme/widget                  (upstream clone, ready to read)
# → ~/github/<you>/widget                  (fork on GitHub, cloned, with
#                                             upstream remote auto-wired)
```

Requires `gh` (GitHub CLI) authenticated:

```bash
gh auth login
```

Re-running is idempotent: existing forks are detected and the local clones
are updated, not re-cloned.

### Fork options

| Flag | What it does |
|---|---|
| `-F`, `--fork` | Enable the fork flow |
| `--fork-all` | Fork copies all branches (overrides webui default; implies `--fork`) |
| `--fork-default` | Fork copies only the default branch (matches webui; implies `--fork`) |
| `--fork-org NAME` | Target a specific org for the fork (e.g., `VariousForks`) |
| `--fork-name VALUE` | Rename the fork. `org/name` shorthand splits into `--fork-org`/`--fork-name`. |

```bash
# Fork into VariousForks org, rename to widget-by-acme-fork
repo-clone --fork --fork-org VariousForks --fork-name widget-by-acme-fork \
  https://github.com/acme/widget

# Same thing, shorter (one-shot rename via shorthand)
repo-clone --fork --fork-name VariousForks/widget-by-acme-fork \
  https://github.com/acme/widget
```

→ Next: [Watch mode](#watch-mode), [Configuration tier](#configuration-tier)

---

## Watch mode

Monitor your clipboard for repository URLs and automatically clone them:

```bash
repo-clone --watch                # Basic
repo-clone --watch --shallow      # Shallow clones
repo-clone --watch -i 0.25        # Faster polling
repo-clone --watch --selection both    # Linux: clipboard + primary selection
repo-clone --watch --dry-run -v        # Test detection without cloning
```

Watch mode recognizes URLs from: GitHub, GitLab, Codeberg, Bitbucket,
Hugging Face, SourceHut. Want another host? Open an issue or PR.

---

## Configuration tier

Create `~/.config/shibuido/repo-clone/repo-clone-config.yaml`:

```yaml
# Watch mode defaults
extra_hosts:
  - gitlab.mycompany.com
watch_interval: 0.5
watch_selection: clipboard

# Fork defaults (used by --fork)
fork:
  target_org: VariousForks
  name_template: "{repo}-by-{org}-fork"
  all_branches: false
  upstream_remote_name: upstream

# Output preferences
output:
  mode: human                  # or "jsonl"
  show_info: false             # if true, INFO: lines visible at -v0
```

Precedence (highest wins): **CLI flags > env vars > config file > defaults.**

`pyyaml` is preferred but not required — a stdlib YAML-subset reader handles
the schema above without it.

→ Full design: [`docs/design/repo-clone-DESIGN-config-file.md`](docs/design/repo-clone-DESIGN-config-file.md)

---

## Advanced

### JSONL output for programmatic consumers

```bash
repo-clone --fork --jsonl https://github.com/acme/widget | jq -c .
```

One compact JSON object per stdout line. Stable event names, additive fields.
→ [`docs/design/repo-clone-DESIGN-jsonl-output.md`](docs/design/repo-clone-DESIGN-jsonl-output.md)

### Agent-optimized help

```bash
repo-clone --help-for-agents
# or
repo-clone help-for-agents
```

Density-optimized output for LLM agents. Tables, pseudocode, exit-code
semantics, JSONL contract, troubleshooting pointers.

### Alternative config

```bash
repo-clone --config /path/to/other-config.yaml ...
```

Multi-account / multi-profile support is on the roadmap.
→ [`docs/design/repo-clone-DESIGN-multi-account-config-layout.md`](docs/design/repo-clone-DESIGN-multi-account-config-layout.md)

### Environment variables

| Var | Maps to |
|---|---|
| `REPO_CLONE_BASE_DIR` | `--base-dir` |
| `REPO_CLONE_VERBOSE` | verbosity (0..3) |
| `REPO_CLONE_FORK_ORG` | `--fork-org` |

---

## Reference

### All options

| Option | Description |
|--------|-------------|
| `-F`, `--fork` | Create a fork on the host, then clone source AND fork |
| `--fork-all` | Fork copies all branches (implies `--fork`) |
| `--fork-default` | Fork copies only default branch (implies `--fork`) |
| `--fork-org NAME` | Override target org for the fork |
| `--fork-name VALUE` | Override fork repo name; `org/name` shorthand splits |
| `--jsonl` | Emit JSONL on stdout |
| `--help-for-agents` | Print agent-optimized help |
| `--config PATH` | Use an alternative config file |
| `-n`, `--no-recursive` | Skip submodule handling |
| `-s`, `--shallow` | Shallow clone (depth 1) |
| `--depth N` | Custom depth |
| `--base-dir PATH` | Override `$HOME` as base directory |
| `-v`, `--verbose` | Increase verbosity (`-v`/`-vv`/`-vvv`) |
| `--dry-run` | Show what would be done without executing |
| `-w`, `--watch` | Watch clipboard mode |
| `-i`, `--interval` | Clipboard polling interval (sec) |
| `--selection` | `clipboard` / `primary` / `both` (Linux) |

### Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Invalid URL |
| 2 | Clone failed |
| 3 | Update failed |
| 4 | Submodule operation failed |
| 5 | Permission error / unrelated dir at expected path |
| 6 | git-lfs missing (Hugging Face) |
| 7 | No clipboard tool found (`--watch`) |
| 8 | `gh` CLI missing or not authenticated |
| 9 | Host not yet supported for `--fork` (non-GitHub) |
| 10 | Fork creation API call failed |
| 11 | Fork name collision (target exists, parent mismatch) |

### URL formats supported

* SSH: `git@github.com:user/repo.git`
* HTTPS: `https://github.com/user/repo.git`
* HTTPS without .git: `https://github.com/user/repo`
* Hugging Face: `https://huggingface.co/user/model`

---

## See also

* [Troubleshooting](docs/repo-clone-TROUBLESHOOTING.md)
* [Design notes](docs/design/) — small, scoped one-topic docs
* [Maintainer notes](maintainers/) — orientation, tone guide, decisions log
* [Developer notes](repo-clone.DEV_NOTES.md) — implementation rationale, edge cases
* [Future work](repo-clone.FUTURE_WORK.md)
* Issues: https://github.com/shibuido/repo-clone/issues
```

- [ ] **Step 3: Verify it renders**

```bash
head -60 /home/gw-t490/github/shibuido/repo-clone/repo-clone.README.md
```

- [ ] **Step 4: Commit**

```bash
git add repo-clone.README.md
git commit -m "docs(readme): rewrite tool README into tiered structure

Tiers: pitch → quickstart → fork tier → watch mode → configuration →
advanced → reference → see also. Each tier self-contained; reader can
stop at any level and have a working setup. New content for --fork,
--jsonl, --help-for-agents, exit codes 8-11.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 29: Write the new repo-root `README.md` (GitHub landing page)

**Files:**
- Create: `README.md` (new, distinct from `repo-clone.README.md`)

- [ ] **Step 1: Write the landing page**

```markdown
# repo-clone

> Stop organizing repositories. Start exploring them.

Paste a URL, get a tidy clone exactly where you'd expect:

```bash
repo-clone https://github.com/torvalds/linux
# → ~/github/torvalds/linux

repo-clone --fork https://github.com/acme/widget
# → ~/github/acme/widget        (upstream, ready to read)
# → ~/github/<you>/widget        (your fork, cloned, with upstream remote)

repo-clone --watch
# paste URLs, they clone themselves. window-shopping mode.
```

## Why

* **Organize on the way in.** Every clone lands at `$HOME/$host/$org/$repo` —
  predictable, greppable, fuzzy-searchable.
* **Idempotent.** Re-run the same command; it updates cleanly. No surprises.
* **Fork-and-experiment loop without leaving the terminal.** `--fork` creates
  the fork, clones both sides, and wires the `upstream` remote.

## Install

```bash
curl -L https://raw.githubusercontent.com/shibuido/repo-clone/master/repo-clone \
  -o ~/.local/bin/repo-clone && chmod +x ~/.local/bin/repo-clone
```

Requires Python 3.9+, `git`. Optional: `git-lfs` (Hugging Face), `gh` (`--fork`),
`pyyaml` (richer config).

## Docs

* **Full guide** → [`repo-clone.README.md`](repo-clone.README.md)
* **Configuration** → [`docs/design/repo-clone-DESIGN-config-file.md`](docs/design/repo-clone-DESIGN-config-file.md)
* **Troubleshooting** → [`docs/repo-clone-TROUBLESHOOTING.md`](docs/repo-clone-TROUBLESHOOTING.md)
* **🤖 AI agents** → `repo-clone --help-for-agents`
* **Design notes** → [`docs/design/`](docs/design/)
* **Contributing** → [`maintainers/`](maintainers/)

## Status

GitHub support today. GitLab / Codeberg / Bitbucket — PRs welcome
([`repo-clone.FUTURE_WORK.md`](repo-clone.FUTURE_WORK.md)).

Issues, ideas, jokes: https://github.com/shibuido/repo-clone/issues

## License

[Insert here once a LICENSE file is added — see FUTURE_WORK.]
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add repo-root README.md as GitHub landing page

Concise pitch, three canonical examples, install one-liner, outbound
links to the comprehensive manual and design/troubleshooting/maintainer
docs. Distinct from repo-clone.README.md (which is the long-form manual).

See docs/design/repo-clone-DESIGN-docs-architecture.md for the role split.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 30: Trim FUTURE_WORK and update DEV_NOTES

**Files:**
- Modify: `repo-clone.FUTURE_WORK.md`
- Modify: `repo-clone.DEV_NOTES.md`

- [ ] **Step 1: Trim FUTURE_WORK**

Edit `repo-clone.FUTURE_WORK.md` — remove "Config file" (now done) and add the
deferred fork-related items.

Replace contents with:

```markdown
# repo-clone Future Work

## Now-implemented items (kept for context; remove after a release cycle)

These items from earlier FUTURE_WORK are now landed:

* ✅ Config file (`~/.config/shibuido/repo-clone/repo-clone-config.yaml`) —
  shipped 2026-05-06 with `fork:` and `output:` sections.

## Investigate

* [ ] GitLab subgroups: `https://gitlab.com/group/subgroup/repo` — verify parsing
* [ ] Bitbucket URL format support
* [ ] SSH config aliases (e.g., `git@gh:user/repo` where `gh` is Host alias)
* [ ] Minimum `gh` version supporting `--default-branch-only` — document the floor

## Consider Adding

* [ ] `--force-pull` — stash local changes, pull, then pop stash
* [ ] `--prune` — prune stale remote tracking branches on update
* [ ] `--jobs N` — parallel submodule fetching (`git fetch --jobs=N`)
* [ ] Progress output for large clones (pass through git's progress)
* [ ] Retry logic for transient network errors (beyond fork-just-created race)

## --fork roadmap

* [ ] `--fork` for GitLab (uses `gh`-equivalent? or `glab`?)
* [ ] `--fork` for Codeberg / Forgejo
* [ ] `--fork` for Bitbucket
* [ ] `--profile NAME` flag + `~/.config/shibuido/repo-clone/profiles/*.yaml`
  (multi-account; layout already forward-compatible — see
  `docs/design/repo-clone-DESIGN-multi-account-config-layout.md`)

## Low Priority

* [ ] `--mirror` mode for backup purposes
* [ ] Fish/Zsh completions
* [ ] Unit tests with pytest (manual integration tests in DEV_NOTES suffice for now)
* [ ] LICENSE file added
```

- [ ] **Step 2: Update DEV_NOTES Files section + Architecture section**

In `repo-clone.DEV_NOTES.md`, update the "Files" table at the bottom:

```markdown
## Files

| File | Purpose |
|------|---------|
| `repo-clone` | Main Python3 script |
| `README.md` | GitHub landing page (concise) |
| `repo-clone.README.md` | Full user manual (tiered) |
| `repo-clone.DEV_NOTES.md` | This file |
| `repo-clone.FUTURE_WORK.md` | Roadmap |
| `docs/design/` | Scoped design rationale (one topic per file) |
| `docs/repo-clone-TROUBLESHOOTING.md` | Symptom-keyed fixes |
| `maintainers/` | Internal notes (orientation, tone, decisions) |
```

And update the "Key Architecture" section:

```markdown
### Key Architecture

```
RepoInfo (dataclass)         - Parsed URL information
CloneOptions (dataclass)     - Runtime clone/update configuration
ForkOptions (dataclass)      - Resolved --fork settings
ForkIdentity (dataclass)     - (target_org, fork_name, fork_clone_url)
WatchConfig / WatchStats     - Watch mode

parse_git_url()              - URL parsing (SSH, HTTPS, HuggingFace)
load_config()                - pyyaml-or-stdlib-subset YAML loader
parse_yaml_subset()          - Stdlib YAML-subset fallback
emit()                       - Structured output (human / JSONL)
gh_preflight()               - 'gh --version' + 'gh auth status'
gh_authed_user()             - 'gh api user --jq .login'
resolve_fork_identity()      - Apply template/literal + default to gh user
fork_exists_on_github()      - 'gh api repos/{org}/{name}' idempotency probe
create_github_fork()         - 'gh repo fork ...'
clone_fork_with_retry()      - Reuse clone path, retry on just-created 404
wire_upstream_remote()       - Idempotent 'git remote add upstream ...'
repo_clone_fork()            - Orchestrator for --fork flow
render_agent_help()          - Build --help-for-agents text
```

The script is structured in literate ordering: small helpers first, composing
functions later, `main()` last. New code added since 2026-05-06 lives in
banner-comment-delimited sections.
```

- [ ] **Step 3: Commit both updates**

```bash
git add repo-clone.FUTURE_WORK.md repo-clone.DEV_NOTES.md
git commit -m "docs: update FUTURE_WORK and DEV_NOTES for --fork release

* FUTURE_WORK: mark config file landed; add multi-host fork roadmap;
  add --profile multi-account work
* DEV_NOTES: add new dataclasses and functions to architecture map;
  expand Files table with new docs/maintainers entries

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 31: Final integration test on a real fork

**Files:** none modified; runs end-to-end against real GitHub.

This is the "really works" gate. Skip if the engineer doesn't have a safe
throwaway fork target available. The dry-run tests in Task 27 cover the
non-network paths.

- [ ] **Step 1: Pick a small public repo + a fork target you control**

Recommended: any tiny public repo + your own GitHub user as target_org.
For the maintainer's setup: a fresh fork to `VariousForks` of a small public
template repo.

- [ ] **Step 2: Run the real fork**

```bash
mkdir -p /tmp/claude/2026-05-06-rc-fork/realtest
/home/gw-t490/github/shibuido/repo-clone/repo-clone \
  --base-dir /tmp/claude/2026-05-06-rc-fork/realtest \
  --fork -v \
  https://github.com/<small-test-repo-owner>/<small-test-repo>
```

Expected:

* Source clone lands at `realtest/github/<owner>/<repo>`
* Fork clone lands at `realtest/github/<your-user>/<repo>`
* `git -C realtest/github/<your-user>/<repo> remote -v` shows `origin` →
  fork URL and `upstream` → source URL
* Exit 0

- [ ] **Step 3: Re-run for idempotency**

```bash
# Same command again
/home/gw-t490/github/shibuido/repo-clone/repo-clone \
  --base-dir /tmp/claude/2026-05-06-rc-fork/realtest \
  --fork -v \
  https://github.com/<small-test-repo-owner>/<small-test-repo>
```

Expected: `INFO: fork-exists ...`, both clones get fetched/pulled, exit 0.
No new fork created on GitHub.

- [ ] **Step 4: Clean up the test fork on GitHub**

```bash
# Optional: delete the test fork
gh repo delete <your-user>/<small-test-repo> --yes
```

- [ ] **Step 5: META commit recording the integration test**

```bash
git commit --allow-empty -m "META: --fork integration test passed end-to-end

* Real fork created via repo-clone --fork against <repo>
* Both clones landed at expected paths
* origin/upstream remotes correctly wired on fork side
* Re-run of same command was idempotent (fork-exists info, both clones updated)
* Exit codes: 0 (success) / 0 (idempotent re-run)

Test artifacts cleaned up.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Self-review checklist (run after writing this plan)

The plan was self-reviewed against the spec on 2026-05-06:

**Spec coverage:**

* §3 CLI surface → Tasks 26 (argparse), 28 (README options table)
* §4 architecture / ordering → Tasks 17, 19-26 build it; Task 14 documents code organization
* §5 config → Tasks 4, 5, 15, 26
* §6 output prefixes → Tasks 6, 16
* §7 --help-for-agents → Tasks 9, 25, 26
* §8 idempotency / exit codes → Tasks 0, 18, 21, 22, 24
* §9 docs scaffold → Tasks 1-14, 28-30
* §10 out of scope → noted in Task 30 (FUTURE_WORK update)
* §11 won't-do → not implemented (correct)
* §12 risks → all addressed: gh-version floor (Task 30 FUTURE_WORK), fork-just-created race (Task 23), naming collision (Task 24), YAML-subset edge cases (Task 15)
* §13 test plan → Tasks 27, 31

**Placeholder scan:** none found; all code blocks contain real code, all doc bodies are written out.

**Type consistency:** `ForkOptions`/`ForkIdentity` field names are consistent across Tasks 19, 20, 24. Function names match between definition and use (`resolve_fork_identity`, `gh_preflight`, `gh_authed_user`, `fork_exists_on_github`, `create_github_fork`, `clone_fork_with_retry`, `wire_upstream_remote`, `repo_clone_fork`, `render_agent_help`, `parse_yaml_subset`, `split_fork_name_shorthand`, `set_output_mode`, `set_show_info`, `emit`).

**Spec discrepancies:** the three discovered defects (exit-code overlap, config filename, prefix casing) are corrected in Task 0 before any code lands. The spec correction is itself a committed change to keep history straight.
