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

```bash
mkdir -p ~/.config/shibuido/repo-clone
```

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
