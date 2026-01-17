# repo-clone

**Stop organizing repositories. Start exploring them.**

Ever been GitHub window-shopping? You find one interesting project, which links to another, then a dependency you want to inspect, then a fork with extra features... Before you know it, you've got a dozen browser tabs and the tedious prospect of:

```bash
mkdir -p ~/repos/someorg && cd ~/repos/someorg && git clone --recursive ...
```

...for each one. Or worse, everything dumped in `~/Downloads` or a flat `~/projects` folder where nothing is findable later.

**repo-clone** handles the busywork. Drop it in your `$PATH` and paste URLs:

```bash
while true; do read -p "url: " url; repo-clone "$url"; done
```

Each repository lands exactly where you'd expect it:

* `~/github/torvalds/linux`
* `~/gitlab/inkscape/inkscape`
* `~/huggingface.co/TheBloke/Llama-2-7B-GPTQ`

Submodules initialize automatically. Run it again on an existing repo and it updates cleanly. No thinking required—just copy, paste, explore.

---

## Overview

`repo-clone` automatically organizes cloned repositories under a structured directory hierarchy:

```
$HOME/<hosting>/<username>/<repo>
```

For example:

* `git@github.com:user/project.git` → `~/github/user/project`
* `https://gitlab.com/org/repo` → `~/gitlab/org/repo`
* `https://huggingface.co/TheBloke/model` → `~/huggingface.co/TheBloke/model`

## Features

* **Automatic submodule support** - Uses `--recursive` by default
* **Fixes repos cloned without submodules** - Running on existing repo initializes missing submodules
* **Smart updates** - Fetches, pulls (if clean), syncs and updates submodules
* **Local change protection** - Warns and skips pull when local changes exist
* **Hugging Face support** - Automatically sets up git-lfs
* **Shallow clone option** - For large repositories

## Usage

```bash
repo-clone [OPTIONS] <url>
```

### Options

| Option | Description |
|--------|-------------|
| `-n, --no-recursive` | Skip submodule handling |
| `-s, --shallow` | Shallow clone (depth 1) |
| `--depth N` | Custom depth for shallow clone |
| `--base-dir PATH` | Override $HOME as base directory |
| `-v, --verbose` | Show git commands being executed |
| `--dry-run` | Show what would be done without executing |

### Environment Variables

| Variable | Description |
|----------|-------------|
| `REPO_CLONE_BASE_DIR` | Alternative to `--base-dir` |
| `REPO_CLONE_VERBOSE` | Set to "1" for verbose output |

## Examples

### Basic clone

```bash
repo-clone https://github.com/user/repo
repo-clone git@github.com:user/repo.git
```

### Clone with shallow depth

```bash
repo-clone --shallow https://github.com/user/large-repo
repo-clone --depth 10 https://github.com/user/repo
```

### Skip submodules

```bash
repo-clone --no-recursive https://github.com/user/repo
```

### Preview what would happen

```bash
repo-clone --dry-run -v https://github.com/user/repo
```

### Hugging Face models

```bash
repo-clone https://huggingface.co/TheBloke/wizard-mega-13B-GPTQ
```

### Custom base directory

```bash
repo-clone --base-dir /data/repos https://github.com/user/repo
# or
REPO_CLONE_BASE_DIR=/data/repos repo-clone https://github.com/user/repo
```

## Behavior

### New Repository

1. Creates directory structure: `$HOME/<hosting>/<user>/`
2. Runs `git clone --recursive <url>`
3. For Hugging Face: runs `git lfs install` first

### Existing Repository

1. `git fetch --recurse-submodules`
2. Checks for local changes
3. If clean: `git pull`
4. If dirty: warns and skips pull
5. `git submodule sync --recursive` (handles URL changes)
6. `git submodule update --init --recursive` (init missing + update all)

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Invalid URL |
| 2 | Clone failed |
| 3 | Update failed |
| 4 | Submodule operation failed |
| 5 | Permission error |
| 6 | Git LFS not installed (Hugging Face) |

## URL Formats Supported

* SSH: `git@github.com:user/repo.git`
* HTTPS: `https://github.com/user/repo.git`
* HTTPS (no .git): `https://github.com/user/repo`
* Hugging Face: `https://huggingface.co/user/model`

## Requirements

* Python 3.9+
* git
* git-lfs (only for Hugging Face repositories)

## See Also

* `repo-clone.DEV_NOTES.md` - Developer notes and implementation details
