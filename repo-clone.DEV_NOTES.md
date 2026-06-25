# repo-clone Developer Notes

## Design Decisions

### Why Python3 over Bash?

The original `repo-clone` was a bash script (~135 lines). It was rewritten in Python3 for:

1. **Better URL parsing** - Using `urllib.parse` and regex provides robust handling of edge cases
2. **Cleaner error handling** - Python's try/except is more maintainable than bash trap/set -e
3. **Type safety** - Dataclasses and type hints make the code self-documenting
4. **Submodule complexity** - The submodule handling logic would have been unwieldy in bash
5. **Testability** - Python is easier to unit test

### Key Architecture

```
RepoInfo (dataclass)         - Parsed URL information
CloneOptions (dataclass)     - Runtime clone/update configuration
ForkOptions (dataclass)      - Resolved --fork settings
ForkIdentity (dataclass)     - (target_org, fork_name, fork_clone_url)
WatchConfig / WatchStats     - Watch mode

parse_git_url()              - URL parsing (SSH, HTTPS, HuggingFace)
_parse_gist_path()           - GitHub Gist owner/id parsing
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
run_git()                    - Subprocess wrapper with dry-run support
clone_repo()                 - New repository clone
update_repo()                - Existing repository update
init_submodules()            - Submodule sync + init + update
```

The script is structured in literate ordering: small helpers first, composing
functions later, `main()` last. New code added since 2026-05-06 lives in
banner-comment-delimited sections.

## Git Submodule Handling

### The Problem

Repositories like [FUTO Keyboard](https://github.com/futo-org/android-keyboard) require:

```bash
git clone --recursive https://...
```

If cloned without `--recursive`, submodule directories are empty.

### The Solution

**On initial clone:**

```bash
git clone --recursive <url>
```

**On update (key insight: this also fixes repos cloned without --recursive):**

```bash
git fetch --recurse-submodules
git pull  # if clean
git submodule sync --recursive   # handles URL changes in .gitmodules
git submodule update --init --recursive  # init missing + update all
```

The `--init` flag is crucial - it initializes any submodules that weren't cloned initially.

### Why `git submodule sync`?

If upstream changes submodule URLs in `.gitmodules`, local `.git/config` still has old URLs. Running `git submodule sync --recursive` updates the local config.

## Edge Cases Handled

### URL Parsing

| Case | Example | Handling |
|------|---------|----------|
| SSH with .git | `git@github.com:u/r.git` | Regex match, strip .git |
| SSH without .git | `git@github.com:u/r` | Regex match |
| ssh:// prefix | `ssh://git@github.com/u/r` | Regex handles both |
| HTTPS with .git | `https://github.com/u/r.git` | urlparse, strip .git |
| HTTPS without .git | `https://github.com/u/r` | urlparse |
| HTTP (insecure) | `http://github.com/u/r` | urlparse (works) |
| GitHub Gist with owner | `https://gist.github.com/u/abc123` | Special case, stores at `github/_gist/u/abc123` |
| GitHub Gist id-only | `https://gist.github.com/abc123` | Special case, stores at `github/_gist/anonymous/abc123` |
| Hugging Face | `https://huggingface.co/u/m` | Special case, keeps full domain |

### Repository States

| State | Behavior |
|-------|----------|
| Doesn't exist | Clone with --recursive |
| Exists, is git repo | Update (fetch, pull, submodule) |
| Exists, not git repo | Error with clear message |
| Has local changes | Warn, skip pull, continue with submodules |
| Has submodule URL changes | `git submodule sync` fixes |
| Missing submodules | `git submodule update --init` fixes |

### Shallow Clones

When `--shallow` is used:

```bash
git clone --recursive --depth 1 --shallow-submodules <url>
```

Note: Some deeply nested submodules may fail with shallow clones.

## Testing

### Manual Test Cases

```bash
# Fresh clone
repo-clone --base-dir tmp https://github.com/user/repo

# Update existing
repo-clone --base-dir tmp https://github.com/user/repo

# With local changes (should warn, not fail)
echo "test" >> tmp/github/user/repo/README.md
repo-clone --base-dir tmp https://github.com/user/repo

# Dry run
repo-clone --dry-run -v https://github.com/user/repo

# Shallow clone
repo-clone --shallow --base-dir tmp https://github.com/user/large-repo

# GitHub Gist
repo-clone --dry-run -v --base-dir tmp https://gist.github.com/user/abc123
# Would clone to tmp/github/_gist/user/abc123

# GitHub Gist without owner in URL
repo-clone --dry-run -v --base-dir tmp https://gist.github.com/abc123
# Would clone to tmp/github/_gist/anonymous/abc123

# Version
repo-clone --version
```

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

### Testing Submodule Fix

```bash
# Clone without submodules (simulating old behavior)
mkdir -p tmp/github/user
cd tmp/github/user
git clone https://github.com/user/repo-with-submodules  # no --recursive

# Now run repo-clone - should init submodules
repo-clone --base-dir tmp https://github.com/user/repo-with-submodules
# Verify submodules are now populated
```

### --fork end-to-end (dry-run)

````bash
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
````

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

## Future Improvements

Potential enhancements (not implemented):

* `--force-pull` - Pull even with local changes (stash first)
* `--prune` - Prune remote tracking branches
* Config file support - Default options per hosting provider
* Progress indicators - For large clones
* Retry logic - For transient network errors
