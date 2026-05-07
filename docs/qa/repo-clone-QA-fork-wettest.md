# QA: `repo-clone --fork` wet-run integration test

End-to-end QA procedure that exercises `repo-clone --fork` against real
GitHub state — the gate that dry-run smoke tests cannot cover. Validates:

* Custom `--config PATH` loading
* `fork.target_org` override (forking into a non-default org)
* `fork.name_template` engine (custom rename of the fork)
* Real `gh repo fork` API call + propagation handling
* Idempotent re-run (`fork-exists` skip path)
* On-disk and on-GitHub state correctness

This file is the **canonical, executable QA procedure**. Run it before any
merge that touches the `--fork` code path. It is dated and parameterized;
see "Re-running this procedure" below.

**First execution:** 2026-05-07 against `pikvm/pi-builder` →
`VariousForks/pi-builder-by-pikvm-fork`. All 10 pass/fail criteria met. Log
artifacts archived at `~/wettest-logs-2026-05-07/` on the runner's machine.

## Re-running this procedure

To re-run on a future date or different source repo:

1. Replace `2026-05-07` everywhere with today's date (`date +%Y-%m-%d`).
2. Replace `pikvm/pi-builder` with any small public repo (≤ 50 MB recommended).
3. Replace `pi-builder-by-pikvm-fork` with the templated name your config
   produces — for the default `{repo}-by-{org}-fork` template, that's
   `<source-repo>-by-<source-org>-fork`.
4. Verify your authed `gh` user is a member of `VariousForks` org:
   `gh api user/orgs --jq '.[].login' | grep VariousForks`. If not, either
   join the org or change `target_org` in the test config to an org you
   control (your own GitHub username works as a fallback).

Everything else is mechanical. The procedure self-cleans at Step 6c — re-runs
leave no persistent state on either GitHub or local disk.

## Pre-flight (one-time, before each run)

```bash
# Confirm the source repo exists and is reasonably small
gh api repos/pikvm/pi-builder --jq '{size_kb: .size, default_branch, fork: .fork, archived}'
# expect: size_kb < 50000, fork=false, archived=false

# Confirm the target fork doesn't already exist
gh api repos/VariousForks/pi-builder-by-pikvm-fork 2>&1 | head -3
# expect: HTTP 404 — clean slate

# Confirm gh auth status
gh auth status
# expect: Logged in to github.com as <your-user>

# Confirm membership in target org
gh api user/orgs --jq '.[].login' | grep -x VariousForks && echo "ok: in VariousForks" || echo "FAIL: not in VariousForks; switch target_org"
```

If any check fails, fix before proceeding.

## The procedure (6 steps)

### Step 0 — Pre-state snapshot

```bash
mkdir -p /tmp/claude/2026-05-07-rc-fork-wettest
ls /tmp/claude/2026-05-07-rc-fork-wettest/                              # expect: empty
gh api repos/VariousForks/pi-builder-by-pikvm-fork 2>&1 | head -3       # expect: HTTP 404
```

### Step 1 — Write the custom test config

```bash
cat > /tmp/claude/2026-05-07-rc-fork-wettest/repo-clone-config.yaml <<'EOF'
# Custom config for the wet-run integration test.
fork:
  target_org: VariousForks
  name_template: "{repo}-by-{org}-fork"
  all_branches: false
  upstream_remote_name: upstream

output:
  mode: human
  show_info: false
EOF
```

### Step 2 — Verbose dry-run with custom config

```bash
cd ~/github/shibuido/repo-clone
./repo-clone -vvv --fork --dry-run \
  --config /tmp/claude/2026-05-07-rc-fork-wettest/repo-clone-config.yaml \
  --base-dir /tmp/claude/2026-05-07-rc-fork-wettest \
  https://github.com/pikvm/pi-builder \
  2>&1 | tee /tmp/claude/2026-05-07-rc-fork-wettest/02-dryrun-vvv.log
echo "exit=${PIPESTATUS[0]}"
```

**Events to grep for in the log** (each must appear):

| Event | Expected fields |
|---|---|
| `DEBUG ... Loaded config (...) from /tmp/.../repo-clone-config.yaml` | (no fields — proves explicit-config path) |
| `INFO: gh-preflight-ok` | `gh_version=...` |
| `INFO: fork-identity-resolved` | `target_org=VariousForks`, `fork_name=pi-builder-by-pikvm-fork`, `template={repo}-by-{org}-fork` |
| `INFO: fork-creating` | `dry_run=True`, `cmd=gh repo fork pikvm/pi-builder --clone=false --org VariousForks --fork-name pi-builder-by-pikvm-fork --default-branch-only` |
| `INFO: upstream-added` | `dry_run=True`, `remote_name=upstream`, `upstream_url=https://github.com/pikvm/pi-builder.git` |
| `INFO: summary` | `actions=['cloned-source', 'created-fork', 'cloned-fork', 'added-upstream']` |

**Expected exit:** 0. **Expected on-disk side effects:** none — the only file
in the temp dir at this point is `repo-clone-config.yaml` from Step 1.

### Step 3 — Verbose wet run (real fork creation)

```bash
./repo-clone -vvv --fork \
  --config /tmp/claude/2026-05-07-rc-fork-wettest/repo-clone-config.yaml \
  --base-dir /tmp/claude/2026-05-07-rc-fork-wettest \
  https://github.com/pikvm/pi-builder \
  2>&1 | tee /tmp/claude/2026-05-07-rc-fork-wettest/03-wet-vvv.log
echo "exit=${PIPESTATUS[0]}"
```

**Events to grep for** (in addition to dry-run set):

| Event | Expected fields |
|---|---|
| `INFO: fork-creating` | NO `dry_run=True` field |
| `INFO: fork-created` | `clone_url=https://github.com/VariousForks/pi-builder-by-pikvm-fork.git` |
| `INFO: upstream-added` | NO `dry_run=True` field |

**Possibly:** one or more `WARN: fork-clone-retrying` events if GitHub
propagation lags 1-3s. Acceptable; the retry-on-CLONE_FAILED guard handles it.

**Expected timing:** source clone 3-30s (depending on network), fork clone
1-5s (GitHub usually propagates fast). Total 5-60s.

**Expected exit:** 0.

### Step 4 — Verify on-disk and on-GitHub state

```bash
echo "=== source clone remotes ==="
git -C /tmp/claude/2026-05-07-rc-fork-wettest/github/pikvm/pi-builder remote -v

echo "=== fork clone remotes ==="
git -C /tmp/claude/2026-05-07-rc-fork-wettest/github/VariousForks/pi-builder-by-pikvm-fork remote -v

echo "=== fork on GitHub ==="
gh api repos/VariousForks/pi-builder-by-pikvm-fork \
  --jq '{full_name, fork, parent: .parent.full_name, default_branch, created_at}'
```

**Expected output:**

* Source clone has only `origin` pointing at `pikvm/pi-builder` (URL form
  may be `https://...` or `git@github.com:...` depending on user's
  `url.insteadOf` config — only the `pikvm/pi-builder` part is contractual)
* Fork clone has TWO remotes: `origin` → `VariousForks/pi-builder-by-pikvm-fork`
  AND `upstream` → `pikvm/pi-builder`
* GitHub API confirms `VariousForks/pi-builder-by-pikvm-fork` exists with
  `parent.full_name=pikvm/pi-builder`, `fork=true`, `default_branch=master`

**Save the `created_at` timestamp** — you'll re-check it in Step 5.

### Step 5 — Idempotent re-run

```bash
./repo-clone -vvv --fork \
  --config /tmp/claude/2026-05-07-rc-fork-wettest/repo-clone-config.yaml \
  --base-dir /tmp/claude/2026-05-07-rc-fork-wettest \
  https://github.com/pikvm/pi-builder \
  2>&1 | tee /tmp/claude/2026-05-07-rc-fork-wettest/05-rerun-vvv.log
echo "exit=${PIPESTATUS[0]}"

# Verify GitHub fork wasn't recreated:
gh api repos/VariousForks/pi-builder-by-pikvm-fork --jq '.created_at'
# expect: same timestamp as Step 4
```

**Key differences from Step 3** (the diagnostic events):

| Event | Status | Meaning |
|---|---|---|
| `Updating existing repository` (×2) | both clones present | reused, not re-cloned |
| `INFO: fork-exists` (NOT `fork-creating`) | with `parent_full_name=pikvm/pi-builder` | gh-api existence probe matched, fork creation skipped |
| `git fetch --recurse-submodules` (×2) | runs for both | update path |
| `git pull` (×2) | runs unless dirty | clean update |
| `INFO: upstream-already-correct` OR `WARN: upstream-mismatch` | one or the other | mismatch is a known-cosmetic issue when `url.insteadOf` rewrites: see "Known cosmetic gotchas" below |
| `INFO: summary` actions | NO `created-fork`; `added-upstream` may still appear (see gotchas) | proves the fork-creation step was skipped |

**Expected exit:** 0. **GitHub `created_at` MUST match Step 4** — proves no
new fork was created on GitHub.

### Step 6 — Save logs, commit (this) QA doc, cleanup

#### 6a — Preserve logs

```bash
mkdir -p ~/wettest-logs-2026-05-07
cp /tmp/claude/2026-05-07-rc-fork-wettest/*.log ~/wettest-logs-2026-05-07/
cp /tmp/claude/2026-05-07-rc-fork-wettest/repo-clone-config.yaml ~/wettest-logs-2026-05-07/
ls ~/wettest-logs-2026-05-07/
```

#### 6b — (Already done — this file IS the committed QA doc)

This file lives at `docs/qa/repo-clone-QA-fork-wettest.md`. It's committed
to master alongside the design docs. New QA procedures for other features
go in the same directory with `repo-clone-QA-<topic>.md` filenames.

#### 6c — Cleanup test state

```bash
gh repo delete VariousForks/pi-builder-by-pikvm-fork --yes
gh api repos/VariousForks/pi-builder-by-pikvm-fork 2>&1 | head -3   # expect HTTP 404
rm -rf /tmp/claude/2026-05-07-rc-fork-wettest
```

## Pass/fail criteria (10 checkboxes)

The wet test passes when ALL of these hold:

- [ ] Custom config loaded — Step 2 log: `Loaded config (...) from /tmp/.../repo-clone-config.yaml`
- [ ] target_org override applied — Step 2 log: `fork-identity-resolved` shows `target_org=VariousForks`
- [ ] name_template applied — Step 2 log: `fork_name=pi-builder-by-pikvm-fork`, `template={repo}-by-{org}-fork`
- [ ] Source clone exists with correct origin — Step 4: `git remote -v` on `.../pikvm/pi-builder` points at pikvm/pi-builder
- [ ] Fork clone exists with correct origin — Step 4: `git remote -v` shows `origin → VariousForks/pi-builder-by-pikvm-fork`
- [ ] Fork clone has upstream remote — Step 4: `git remote -v` shows `upstream → pikvm/pi-builder`
- [ ] Real fork on GitHub — Step 4: `gh api repos/VariousForks/pi-builder-by-pikvm-fork --jq .parent.full_name` returns `pikvm/pi-builder`
- [ ] Idempotent re-run — Step 5: exit=0, NO `fork-creating` event, `fork-exists` event present, GitHub `created_at` unchanged
- [ ] `-vvv` actually verbose — All logs contain `DEBUG [...]:` lines + git command echoes (`$ git ...`) + timing (`-> completed in X.XXs`)
- [ ] Cleanup leaves no trace — Step 6c: `gh api` returns 404, `/tmp/claude/2026-05-07-rc-fork-wettest/` removed

## Known cosmetic gotchas (NOT defects)

These appear during the test but do not constitute failures:

* **`url.insteadOf` rewriting** — if your `~/.gitconfig` has rules like
  `url."git@github.com:".insteadOf = "https://github.com/"`, git stores
  remote URLs in the rewritten SSH form. `wire_upstream_remote` then sees
  the rewritten form on probe and emits `WARN: upstream-mismatch` on every
  re-run (current_url is SSH, expected_url is HTTPS). The remote is **not**
  overwritten — non-overwriting policy is preserved. Logged in
  `repo-clone.FUTURE_WORK.md` as a URL-canonicalization-needed item.

* **`summary.actions` includes `added-upstream` on re-runs** even when
  `wire_upstream_remote` took the WARN-mismatch (skip) path. Cosmetic only;
  the actions list is informational, not the source of truth. Will be
  refined in a follow-up to distinguish `added` / `confirmed` / `mismatched`.

* **ED25519 randomart appears during clone** — git's first SSH connection
  to a host displays the host key fingerprint as ASCII art. Cosmetic;
  results from the `url.insteadOf` rewrite forcing SSH for HTTPS-input URLs.

## Rollback / interruption recovery

If the test fails partway:

```bash
# If a fork was created on GitHub but local clones failed, delete the fork:
gh repo delete VariousForks/pi-builder-by-pikvm-fork --yes

# Remove any partial local state:
rm -rf /tmp/claude/2026-05-07-rc-fork-wettest

# Then triage — return to plan, don't push fixes ad-hoc.
```

## Cross-references

* Design contract being validated: [`../design/repo-clone-DESIGN-fork-flag.md`](../design/repo-clone-DESIGN-fork-flag.md)
* Output mode contract: [`../design/repo-clone-DESIGN-output-prefixes.md`](../design/repo-clone-DESIGN-output-prefixes.md), [`../design/repo-clone-DESIGN-jsonl-output.md`](../design/repo-clone-DESIGN-jsonl-output.md)
* Config file contract: [`../design/repo-clone-DESIGN-config-file.md`](../design/repo-clone-DESIGN-config-file.md)
* Filename convention (why this file is named `repo-clone-QA-...`): [`../design/repo-clone-DESIGN-greppable-filenames.md`](../design/repo-clone-DESIGN-greppable-filenames.md)
* Implementation spec: [`../superpowers/specs/2026-05-05-repo-clone-fork-design.md`](../superpowers/specs/2026-05-05-repo-clone-fork-design.md)
* Original wet-run plan that produced this procedure: `~/.claude/plans/please-verbose-wet-run-luminous-parasol.md` (transient — captured here for permanence)

## See also

* [`repo-clone-TROUBLESHOOTING.md`](../repo-clone-TROUBLESHOOTING.md) — symptom-keyed fixes (covers the same `gh` / fork issues from a user perspective)
* [`../../repo-clone.DEV_NOTES.md`](../../repo-clone.DEV_NOTES.md) — manual test cases (dry-run only; this QA doc is the wet counterpart)
