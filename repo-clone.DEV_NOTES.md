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
RepoInfo (dataclass)     - Parsed URL information
CloneOptions (dataclass) - Runtime configuration
parse_git_url()          - URL parsing (SSH, HTTPS, HuggingFace)
run_git()                - Subprocess wrapper with dry-run support
clone_repo()             - New repository clone
update_repo()            - Existing repository update
init_submodules()        - Submodule sync + init + update
```

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

## Files

| File | Purpose |
|------|---------|
| `repo-clone` | Main Python3 script |
| `repo-clone.bash.bak` | Original bash version (backup) |
| `repo-clone.README.md` | User documentation |
| `repo-clone.DEV_NOTES.md` | This file |

## Future Improvements

Potential enhancements (not implemented):

* `--force-pull` - Pull even with local changes (stash first)
* `--prune` - Prune remote tracking branches
* Config file support - Default options per hosting provider
* Progress indicators - For large clones
* Retry logic - For transient network errors
