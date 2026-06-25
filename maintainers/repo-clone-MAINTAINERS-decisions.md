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

## 2026-06-25 — GitHub Gist path layout

**Decision:** Clone GitHub Gists under
`$base/github/_gist/<user-or-anonymous>/<gist-id>`.

**Why:** Gists are Git repositories, but `gist.github.com/<id>` does not always
carry an owner segment and should not collide with normal GitHub repositories
under `$base/github/<user>/<repo>`. The `_gist` segment is visibly synthetic and
keeps all Gists browsable in one place.

**Considered:** `$base/gist.github.com/<user>/<id>` preserves the literal host
but separates Gists from the GitHub base. `$base/github/<user>/_gist/<id>` keeps
per-user locality but puts a synthetic namespace inside every normal user/org
directory. The centralized `_gist` namespace is clearer and lower risk.

---

## How to add an entry

```markdown
## YYYY-MM-DD — Short title

**Decision:** What we chose.

**Why:** The reason. Cite issues, PRs, discussions if relevant.

**Considered:** Alternatives that were rejected and why (if material).
```

Append to the bottom of the dated entries. Never edit older entries.
