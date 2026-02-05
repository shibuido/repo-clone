# Clipboard Watch Mode Design

**Date:** 2026-02-05
**Status:** Approved

## Overview

Add clipboard monitoring mode to repo-clone that watches for repository URLs and automatically clones them. Enables frictionless workflow: copy a repo URL anywhere, it gets cloned automatically.

## CLI Interface

```
repo-clone --watch [OPTIONS]

Watch mode options:
  --watch, -w          Enable clipboard monitoring mode
  --interval, -i FLOAT Polling interval in seconds (default: 0.5)
  --selection SEL      Linux clipboard: clipboard|primary|both (default: clipboard)
```

Existing flags work with watch mode:

* `--shallow`, `--depth` - apply to all clones during session
* `--base-dir` - target directory for organized clones
* `-v, --verbose` - show detected URLs, skip reasons, timing
* `--dry-run` - detect URLs but don't clone
* `--no-recursive` - skip submodules for all clones

Example usage:

```bash
# Basic watch mode
repo-clone --watch

# Watch with shallow clones
repo-clone --watch --shallow

# Watch primary selection (Linux middle-click)
repo-clone --watch --selection primary

# Watch both clipboards, custom interval
repo-clone --watch --selection both -i 0.25

# Dry run to test URL detection
repo-clone --watch --dry-run -v
```

Mutual exclusivity: `--watch` cannot be combined with a positional URL argument.

## Clipboard Detection & Access

**Supported platforms:** Linux, macOS, FreeBSD, OpenBSD

**Detection sequence:**

1. Check config file for `clipboard_command` override - use if set, skip detection

2. Detect platform & session:

   * macOS (Darwin) - pbpaste (always available)
   * Linux/BSD:
     * Check `XDG_SESSION_TYPE` env var (most reliable): "wayland" or "x11"
     * If unset, check `WAYLAND_DISPLAY` (set = Wayland) or `DISPLAY` (set = X11)
     * Verify tool works (test read, check exit code)
     * If fails, try next in chain

3. Cache working tool for session

4. Fail with platform-specific install message if none works

**Tool commands:**

| Session | Tool | Command |
|---------|------|---------|
| Wayland | wl-paste | `wl-paste --type text` |
| Wayland (primary) | wl-paste | `wl-paste --primary --type text` |
| X11 | xclip | `xclip -selection clipboard -o` |
| X11 (primary) | xclip | `xclip -selection primary -o` |
| X11 | xsel | `xsel --clipboard --output` |
| macOS | pbpaste | `pbpaste` |

**`--selection both` implementation:** Poll clipboard first, then primary, alternate each cycle. Dedupe across both.

**Error on missing tool:**

```
ERROR: No clipboard tool found.
Install one of: wl-clipboard (Wayland), xclip, xsel (X11)
  Arch:   pacman -S wl-clipboard   # or xclip
  Ubuntu: apt install xclip
```

## URL Pattern Matching

**Built-in recognized hosts:**

```python
BUILTIN_HOSTS = [
    "github.com",
    "gitlab.com",
    "codeberg.org",
    "bitbucket.org",
    "huggingface.co",
    "sr.ht",           # SourceHut
    "git.sr.ht",       # SourceHut git
]
```

**URL patterns matched:**

```
# HTTPS formats
https://{host}/{user}/{repo}
https://{host}/{user}/{repo}.git
https://{host}/{user}/{repo}/...  (any subpath - strips to repo)

# SSH formats
git@{host}:{user}/{repo}.git
git@{host}:{user}/{repo}
ssh://git@{host}/{user}/{repo}
```

**Matching logic:**

1. Extract hostname from clipboard text
2. Check against: built-in list + `extra_hosts` from config
3. If match, attempt to parse with existing `parse_git_url()`
4. If parse succeeds, valid repo URL, trigger clone
5. If parse fails, log at `-vv`, skip

**Config extension:**

```yaml
extra_hosts:
  - gitlab.mycompany.com
  - gitea.internal.org
```

**Edge cases:**

* Clipboard contains multiple lines: check first line only
* URL has query params or anchors: strip before parsing
* Whitespace around URL: trim

Community contributions for new default hosts encouraged via GitHub issues/PRs.

## Watch Loop & Runtime Behavior

**Main watch loop:**

1. Startup:

   * Detect clipboard backend (fail fast if none)
   * Load config file if exists
   * Print: "Watching clipboard for repo URLs... (Ctrl+C to stop)"
   * Initialize: `seen_urls = set()`, `stats = {cloned: 0, skipped: 0, failed: 0}`

2. Poll cycle:

   * Read clipboard content
   * If empty or read error: `sleep(interval)`, continue
   * Trim whitespace, extract first line
   * If URL not in `seen_urls` AND matches known host:
     * Add to `seen_urls`
     * Attempt `parse_git_url()`
     * If valid: clone, update stats, continue immediately (no sleep)
     * If parse fails: `log_debug()`, sleep
   * If URL in `seen_urls`: `log_info("Skipping duplicate: {url}")`, sleep
   * If no match: sleep

3. On Ctrl+C (SIGINT):

   * Print newline (clear status line)
   * Print summary: "Stopped. Cloned: N, Skipped: N, Failed: N"
   * Exit with appropriate code

**Status line (updated in-place):**

```
Watching clipboard... (cloned: 3, skipped: 1, failed: 0) Ctrl+C to stop
```

Uses `\r` + clear to end of line for in-place update.

**Timing:** Sleep only on "no action" cycles. Immediate re-poll after clone attempt.

## Config File

**Location:**

```
~/.config/shibuido/repo-clone/repo-clone-config.yaml
```

Follows XDG Base Directory spec. File is optional.

**Full schema:**

```yaml
# Additional hosts to recognize (extends built-in list)
extra_hosts:
  - gitlab.mycompany.com
  - gitea.internal.org

# Watch mode defaults
watch_interval: 0.5              # seconds (float)
watch_selection: clipboard       # clipboard | primary | both

# Override clipboard command (bypasses auto-detection)
clipboard_command: "xclip -selection clipboard -o"

# Base clone settings (can still be overridden by CLI flags)
shallow: false
recursive: true
base_dir: ~/repos              # expands ~ at runtime
```

**Precedence (highest to lowest):**

1. CLI flags (`--interval 0.25`)
2. Environment variables (`REPO_CLONE_BASE_DIR`)
3. Config file values
4. Built-in defaults

**PyYAML dependency:** Config file is optional feature. If PyYAML not installed and config file exists, warn and continue with defaults.

## Deduplication

**Session-scoped:** Track URLs seen this session in a set. On duplicate:

```
INFO: Skipping duplicate: https://github.com/user/repo
```

**Cross-session:** Relies on existing clone-or-update logic (clone new repos, fetch+pull existing).

## Error Handling

* Clone failure: Print `ERROR:` to stderr, increment failed count, keep watching
* Clipboard read error: Log at debug level, continue polling
* Invalid URL from known host: Log at debug level, skip

## Exit Codes

| Code | Condition |
|------|-----------|
| 0 | Any success OR zero attempts (including Ctrl+C) |
| 1 | All attempts failed (attempts > 0) |

## Code Organization

New code (~200-250 lines) appended to existing `repo-clone` script:

* `BUILTIN_HOSTS` - list constant
* `WatchConfig` - dataclass for watch settings
* `WatchStats` - dataclass for runtime stats
* `load_config()` - YAML config loading
* `detect_clipboard_backend()` - platform detection
* `make_clipboard_reader()` - returns callable for reading clipboard
* `is_repo_url()` - host matching
* `extract_repo_url()` - URL extraction and cleanup
* `watch_clipboard()` - main watch loop
* `print_status_line()` - in-place status update
* `main()` modifications - new args, branch to watch mode

Single file stays single file. Reuses existing `parse_git_url()`, `repo_clone()`, logging.

## Dependencies

* Required: Python 3.9+, git
* Optional: PyYAML (for config file)
* System: One of wl-paste, xclip, xsel (Linux/BSD) or pbpaste (macOS)

## README Updates

* New "Watch Mode" section with examples
* Encourage host additions via issues/PRs
* Config file documentation
