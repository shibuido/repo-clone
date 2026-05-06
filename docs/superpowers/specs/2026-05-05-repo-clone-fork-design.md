# repo-clone `--fork` and surrounding repo upgrades — Design Spec

**Date:** 2026-05-05
**Status:** Approved (brainstorm complete, awaiting plan)
**Scope:** Extend the existing `repo-clone` Python3 single-file script with a
`--fork` feature, structured output, an agent-facing help mode, a config file,
and a documentation/maintainer scaffold.

---

## 1. Motivation

`repo-clone` currently turns a copy-pasted repo URL into an organized local
clone under `$HOME/<host>/<org>/<repo>`. Users (starting with the maintainer's
own workflow) frequently fork repos right after cloning them — opening the
GitHub web UI, clicking *Fork*, picking a target org (`VariousForks`), often
renaming the fork (`<repo>-by-<org>-fork`), then re-cloning. The webui round
trip breaks the keyboard-driven, paste-to-clone flow that makes `repo-clone`
worth using.

`--fork` collapses this round trip into one command, while staying consistent
with the tool's existing invariants (path layout, idempotent re-runs, stdlib
+ shell-out portability).

The same change is the natural moment to land a few small structural upgrades:

* A real config file at `~/.config/shibuido/repo-clone/` so per-user defaults
  (target org, fork name template, branch policy) live outside the codebase.
* Structured output (`INFO:` / `WARNING:` / `ERROR:` prefixes) and an
  opt-in `--jsonl` mode for programmatic consumers.
* A `--help-for-agents` (and `help-for-agents` subcommand alias) producing
  density-optimized output for AI agents using the tool.
* A documentation scaffold (`docs/design/`, `docs/repo-clone-TROUBLESHOOTING.md`,
  repo-root `README.md` distinct from `repo-clone.README.md`, `maintainers/`)
  that future shibuido tools can crib from.

## 2. Decisions locked during brainstorming

| # | Question | Decision |
|---|---|---|
| Q1 | Fork backend & auth | Require `gh` CLI; preflight `gh --version` and `gh auth status` |
| Q2 | On-disk layout for source + fork | Two directories: source at `$base/$host/$org/$repo`, fork at `$base/$host/$forkOrg/$forkName`; fork side gets `upstream` remote auto-wired |
| Q3 | Default fork target org when no config | The authenticated `gh` user (matches webui & `gh repo fork` default) |
| Q4 | Default fork branch policy | Default-branch-only (matches webui). Both `--fork-all` and `--fork-default` exist; either implies `--fork`; mutually exclusive |
| Q5 | Fork name template | Python `str.format` syntax in config; `--fork-name LITERAL` for one-off override; `--fork-name org/name` shorthand splits into org + name |
| Q6 | Agent-help shape | Both `--help-for-agents` flag (canonical) and `help-for-agents` subcommand (alias); flagged prominently in `--help` |
| Q7 | Conflict / collision handling | Idempotent: re-running is safe; existing fork on GH or matching local dir → update; unrelated local dir → ERROR |
| extra | Output | `INFO:`/`WARNING:`/`ERROR:` prefixes, `--verbose` for detail, `--jsonl` for one-JSON-per-line programmatic mode |

## 3. CLI surface

```
repo-clone [OPTIONS] <url>
repo-clone help-for-agents          # subcommand alias for --help-for-agents

New flags introduced by this work:
  -F, --fork                  Create a fork on the host, then clone source AND fork.
      --fork-all              Fork copies all branches.        Implies --fork.
      --fork-default          Fork copies only default branch. Implies --fork.
      --fork-org NAME         Override target org for the fork. Implies --fork.
      --fork-name VALUE       Override fork repo name (literal, no templating).
                              If VALUE contains exactly one '/', it splits into
                              --fork-org / --fork-name. Implies --fork.
      --jsonl                 Emit one compact JSON object per line; mutually
                              exclusive with normal human output.
      --help-for-agents       Print agent-optimized help and exit.
      --config PATH           Use an alternative config file.

Existing flags unchanged: -n/--no-recursive, -s/--shallow, --depth, --base-dir,
-v/--verbose, --dry-run, --watch, -i/--interval, --selection.

Mutually exclusive: --fork-all and --fork-default; --jsonl and the implicit
human output mode (passing --jsonl simply switches modes).
```

`--fork-name` semantics: if the value contains a single `/`, it is parsed as
`<org>/<name>`; left side becomes the fork org (errors if `--fork-org` was
also given with a different value), right side becomes the literal fork name.
Multiple `/` or empty either side → ERROR. Example:
`--fork-name greg/widget-experimental` ≡ `--fork-org greg --fork-name widget-experimental`.

## 4. Architecture

The script remains one file (`repo-clone`, no extension, executable, Python 3.9+,
stdlib only — `pyyaml` remains an optional upgrade for richer configs).
Code stays in literate ordering: small helpers first, composing functions
later, `main()` last.

New logical sections added inside the script (each preceded by a banner
comment):

* `# ---- Config file loading ----` — tiny stdlib YAML-subset reader; merges
  defaults ⇽ config-file ⇽ env vars ⇽ CLI flags; reads the new `fork:` and
  `output:` sections.
* `# ---- Output / structured logging ----` — `info()`, `warn()`, `err()`,
  `event()` helpers; `--jsonl` flips the backend; `--verbose` controls
  `INFO:` visibility.
* `# ---- gh preflight ----` — checks `gh --version` and `gh auth status`;
  exits with code 7 on failure.
* `# ---- Fork identity resolution ----` — combines URL, config, and CLI
  to resolve `(target_org, fork_name)` plus the templated fork URL.
* `# ---- Fork creation (idempotent) ----` — checks via `gh api` whether the
  fork already exists; calls `gh repo fork` if not; verifies `parent.full_name`
  matches the source on existing forks (mismatch → ERROR).
* `# ---- Fork-side clone & upstream wiring ----` — reuses the existing
  clone/update path on the fork URL; idempotently adds the `upstream` remote
  pointing at the source URL; warns if `upstream` exists but points
  somewhere unexpected (does not overwrite).
* `# ---- Agent help ----` — assembles the `--help-for-agents` text from
  small fragments so each section is independently editable.

The order of operations for `repo-clone --fork <url>`:

```
1. Parse URL (existing parse_git_url)
2. Resolve fork identity from URL + config + CLI
3. gh preflight (only on fork path)
4. Source-side clone/update (existing behavior, unchanged)
5. Fork creation (idempotent; gh api check, then gh repo fork if missing)
6. Fork-side clone/update (reuse existing logic against fork URL)
7. Wire upstream remote on fork (idempotent; never overwrites)
8. Emit summary (human or JSONL)
```

Source side runs *before* fork side so a bad URL fails fast without touching
GitHub. If fork creation fails, the source clone is left as-is (independently
useful; no rollback).

## 5. Config file

**Location:** `~/.config/shibuido/repo-clone/repo-clone-config.yaml` (respects `$XDG_CONFIG_HOME`).

**Loaded if present, ignored if absent.** A tiny stdlib YAML-subset reader
handles top-level keys, one level of nested mappings, strings, booleans, and
lists of strings. Anything richer requires `pyyaml`; if the reader sees
syntax it can't parse, it errors with a clear pointer.

**Forward-compatible directory layout** (multi-profile support is a future
non-goal but the layout is designed so it slots in additively):

```
~/.config/shibuido/repo-clone/
├── config.yaml                  # v1: single config loaded today
└── profiles/                    # future: --profile NAME → profiles/NAME.yaml
    └── default.yaml             # future: equivalent alias for config.yaml
```

v1 loader: try `repo-clone-config.yaml`, else try `profiles/default.yaml`, else use
built-in defaults. If both exist with different contents, WARNING.

**Schema:**

```yaml
# Existing watch-mode keys remain (extra_hosts, watch_interval, watch_selection,
# clipboard_command). New sections:

fork:
  target_org: VariousForks                    # default: gh-authed user
  name_template: "{repo}-by-{org}-fork"       # default: "{repo}"
  all_branches: false                         # default: false (matches webui)
  upstream_remote_name: upstream              # default: "upstream"

output:
  mode: human                                 # human | jsonl
  show_info: false                            # if false, INFO: is suppressed
                                              # unless --verbose
```

**Precedence (highest wins):** CLI flags > env vars > config file >
built-in defaults.

**Env vars added:** `REPO_CLONE_FORK_ORG`. Existing `REPO_CLONE_BASE_DIR`,
`REPO_CLONE_VERBOSE` retained.

**Multi-account today:** single config is v1; `--config PATH` points at an
alternative file for the rare power-user case.

## 6. Output: prefixes and `--jsonl`

**Human mode:**

```
INFO:  routine progress; suppressed unless --verbose or output.show_info
WARN:  degraded but proceeding (e.g., fork already exists, upstream remote
       points elsewhere and we left it alone)
ERROR: abort condition; followed by exit
```

`--verbose` keeps its existing role (shows git commands) and additionally
unmasks `INFO:` lines.

**JSONL mode (`--jsonl`):** one compact JSON object per stdout line.
Mutually exclusive with the human prefixes (no `INFO:` strings; instead each
event gets `"level": "info"|"warn"|"error"` and a stable `"event"` name).
Schema lives in `docs/design/repo-clone-DESIGN-jsonl-output.md`. Sketch:

```json
{"event":"clone-start","level":"info","url":"https://github.com/acme/widget","path":"~/github/acme/widget"}
{"event":"fork-created","level":"info","source":"acme/widget","fork":"VariousForks/widget-by-acme-fork"}
{"event":"upstream-added","level":"info","fork_path":"~/github/VariousForks/widget-by-acme-fork","upstream_url":"https://github.com/acme/widget"}
{"event":"summary","level":"info","source_path":"...","fork_path":"...","actions":["cloned-source","created-fork","cloned-fork","added-upstream"]}
```

Stable event names + additive fields = consumers can rely on the contract.

## 7. `--help-for-agents`

Both forms work: `repo-clone --help-for-agents` (canonical flag, fits argparse
natively) and `repo-clone help-for-agents` (subcommand alias; one positional
string special-cased before URL parsing).

`--help` includes a prominent block routing AI agents to the agent help mode
and the issue tracker.

The agent-help output is density-optimized: tables, pseudocode, equations and
fragments preferred over prose. Fixed sections (each independently editable):

* `IDENTITY` — one-line what + why
* `INVOCATION` — grammar + canonical examples
* `EXIT_CODES` — table
* `STATE_MACHINE` — per-URL outcome decision tree
* `JSONL_SCHEMA` — event types + fields
* `IDEMPOTENCY` — re-run guarantees
* `FAILURE_RECOVERY` — retryable errors, network timeouts, expected wall-clock
  ranges (fork creation 5–30s typical, large clones minutes)
* `CONTEXTUAL_FIT` — where this tool sits in agent workflows; path layout
  invariants; companion tools (`find-snapshot`, `sk`, `fzf`)
* `TROUBLESHOOTING_POINTERS` — local + remote URL of
  `docs/repo-clone-TROUBLESHOOTING.md`
* `REPORT_BUGS` — issue tracker link + brief template

Style rules (codified in the design doc): no marketing tone, no reassuring
filler, emoji only where they aid scanning, no repetition of `--help` content.
Cross-link to `--help` for the human-oriented overview.

## 8. Idempotency, exit codes, error matrix

**Re-running `repo-clone --fork URL` is safe.**

| Situation | Behavior | Exit |
|---|---|---|
| `gh` missing or unauthed | ERROR with actionable hint | 8 (new) |
| Non-GitHub URL with `--fork` | ERROR pointing at FUTURE_WORK | 9 (new) |
| Source clone fails | ERROR; abort before touching GitHub | 2 |
| Fork already exists on GH and `parent.full_name` matches source | INFO; continue to clone/update fork side | 0 |
| Fork already exists on GH but `parent` mismatches source | ERROR (name collision with unrelated repo) | 11 (new) |
| Local fork dir exists, `origin` matches expected fork URL | Update (existing behavior) | 0 |
| Local fork dir exists, `origin` differs or not a git repo | ERROR with specific path | 5 |
| Fork creation API call fails | ERROR; source clone is left as-is (no rollback) | 10 (new) |
| Existing exit codes 1–6 unchanged | — | 1–6 |

**Newly minted codes:** 8 (`gh` missing/unauth), 9 (host not yet supported for fork), 10 (fork creation API failure), 11 (fork name collision with unrelated repo). Existing codes 1–7 retain their current meanings (7 = NO_CLIPBOARD_TOOL); we do not re-purpose them for new error classes.

## 9. Documentation & maintainer scaffold

```
repo-clone/
├── README.md                                 (NEW) GitHub landing page
├── repo-clone                                Python script (grows in place)
├── repo-clone.README.md                      (REWRITTEN) tiered user manual
├── repo-clone.DEV_NOTES.md                   (lightly updated)
├── repo-clone.FUTURE_WORK.md                 (items moved as they land)
├── docs/
│   ├── design/                               10 small design docs
│   │   ├── repo-clone-DESIGN-fork-flag.md
│   │   ├── repo-clone-DESIGN-config-file.md
│   │   ├── repo-clone-DESIGN-help-for-agents.md
│   │   ├── repo-clone-DESIGN-jsonl-output.md
│   │   ├── repo-clone-DESIGN-output-prefixes.md
│   │   ├── repo-clone-DESIGN-stdlib-portability.md
│   │   ├── repo-clone-DESIGN-greppable-filenames.md
│   │   ├── repo-clone-DESIGN-docs-architecture.md
│   │   ├── repo-clone-DESIGN-multi-account-config-layout.md
│   │   └── (plus this spec lives under docs/superpowers/specs/)
│   └── repo-clone-TROUBLESHOOTING.md
└── maintainers/
    ├── repo-clone-MAINTAINERS-README.md
    ├── repo-clone-MAINTAINERS-tone-guide.md
    └── repo-clone-MAINTAINERS-decisions.md
```

**Two distinct READMEs by design:**

* `README.md` (repo root, NEW) — GitHub landing page. Concise, fits a screen;
  pitch + one canonical example block (plain clone, `--fork` clone, `--watch`)
  + install + an outbound docs index. Job: hook visitors in 30 seconds and
  route to deeper docs.
* `repo-clone.README.md` (existing, rewritten) — comprehensive man-page-quality
  user manual. Tiered: zeroconf quickstart → `--fork` tier → watch mode tier
  → configuration tier → advanced (env vars, alt config, JSONL) → reference
  (full options, exit codes, URL formats) → see-also.

Each tier of the manual is self-contained — the reader can stop at any level
and have a working setup. Cognitive learning curve principle: every section
introduces at most one new concept; ends with an explicit "next-step" link.

**Design docs are short, scoped, reusable** (≤3 screens each). The whole
`docs/design/` directory should be readable in one sitting. Filenames follow
the greppable convention (`<tool>-DESIGN-<topic>.md` and
`<tool>-MAINTAINERS-<topic>.md`) so `sk -f DESIGN | head` and equivalents
work cleanly.

**Tone guide** (`maintainers/repo-clone-MAINTAINERS-tone-guide.md`) covers
three audiences with do/don't lists and before/after examples:

* README + `--help` (humans): builder voice, CLI-native, concise, gentle
  humor allowed; show-don't-tell with examples.
* `--help-for-agents`: density over politeness; pseudocode + tables welcome;
  no prose padding; cite tracker + troubleshooting paths.
* Error messages: state what failed, where, and the next action; no
  apologies; exit codes documented.
* Commit messages: defer to `ai_docs/small_descriptive_commits.md` and
  `ai_docs/meta_commit_motivation.md`; do not duplicate.

**Decisions log** (`maintainers/repo-clone-MAINTAINERS-decisions.md`) is
seeded with the eight Q1–Q7-plus-output decisions from this brainstorm,
each dated 2026-05-05. Future maintainers append entries; never rewrite
history.

## 10. Out of scope (this spec) → FUTURE_WORK

* GitLab / Codeberg / Bitbucket fork support (hooks designed in; not
  implemented — non-GitHub URLs with `--fork` ERROR with exit 8 and a
  pointer)
* Multiple-account / profile-aware config switching (`--profile NAME`).
  Layout is forward-compatible; flag itself ships later.
* `--mirror`, `--prune`, `--force-pull`, parallel submodule jobs (already
  in FUTURE_WORK)
* Shell completions
* pytest unit tests (manual integration tests in DEV_NOTES remain v1)

## 11. Won't do (deliberate non-features)

* Auto-detecting which account to fork as based on URL — single config +
  flag is clearer
* Rolling back source clone if fork creation fails — source clone is
  independently useful and shouldn't be deleted
* Fork-without-clone mode — `gh repo fork` already does that; users
  wanting it should call `gh` directly

## 12. Risks / things to verify during implementation

* Minimum `gh` version supporting `--default-branch-only` semantics —
  document the floor; preflight should warn if older
* Race between `gh repo fork` returning and the fork being clonable
  (GitHub usually instant, occasionally lags 1–3s) — small retry-with-backoff
  on the fork-side clone if it 404s within 10s
* Naming-collision detection: when target org already hosts a repo with the
  templated name that *isn't* a fork of the source, detect via
  `gh api repos/{org}/{name} --jq .parent.full_name` and ERROR (exit 4)
  rather than silently treating it as the existing fork
* YAML-subset reader edge cases — keep coverage to documented subset; if a
  user's config triggers the subset's limits, the error should suggest
  installing pyyaml

## 13. Test plan (manual; pytest suite is FUTURE_WORK)

To be expanded into the implementation plan, but at minimum:

* `--fork` against a small public repo, default config (forks to authed
  user) → both clones land, `upstream` wired, summary correct
* Re-run same command → idempotent; INFO says fork exists; clones updated
* Same with `fork.target_org` and `name_template` configured → fork lands
  at templated location
* `--fork-name org/name` shorthand → splits correctly
* `--fork-name foo/bar` together with `--fork-org other` → ERROR
* `--fork-all` and `--fork-default` together → ERROR
* `--jsonl` produces one valid JSON per line; piping to `jq` works
* GitLab URL with `--fork` → ERROR exit 8 with FUTURE_WORK pointer
* `gh` not installed → ERROR exit 7 with install hint
* Local dir at fork path that isn't a git repo → ERROR exit 5
* `--dry-run --fork` → prints planned actions, makes no API calls

## 14. References

* Existing repo-clone source, README, DEV_NOTES, FUTURE_WORK
* `gh` CLI: <https://cli.github.com/manual/gh_repo_fork>
* shibuido conventions: `ai_docs/small_descriptive_commits.md`,
  `ai_docs/meta_commit_motivation.md`,
  `ai_docs/github_issues_cli_best_practices.md`
