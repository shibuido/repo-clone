# Clipboard Watch Mode Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add `--watch` mode to repo-clone that monitors clipboard for repository URLs and auto-clones them.

**Architecture:** Extend existing single-file script with new dataclasses (WatchConfig, WatchStats), clipboard backend detection via subprocess, and a polling loop that reuses existing `parse_git_url()` and `repo_clone()` functions.

**Tech Stack:** Python 3.9+ stdlib only (subprocess, shutil, signal, dataclasses). Optional PyYAML for config file.

---

## Task 1: Add BUILTIN_HOSTS Constant and Exit Code

**Files:**

* Modify: `repo-clone` (after line 44, add new exit code; after line 56, add hosts list)

**Step 1: Add the constants**

Add after `GIT_LFS_FAILED = 6` (line 44):

```python
    NO_CLIPBOARD_TOOL = 7
```

Add after `VERBOSITY = 0` line (around line 56):

```python
# Built-in recognized repository hosts
BUILTIN_HOSTS = [
    "github.com",
    "gitlab.com",
    "codeberg.org",
    "bitbucket.org",
    "huggingface.co",
    "sr.ht",
    "git.sr.ht",
]
```

**Step 2: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add BUILTIN_HOSTS constant and NO_CLIPBOARD_TOOL exit code

Preparation for clipboard watch mode feature.

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 2: Add WatchConfig and WatchStats Dataclasses

**Files:**

* Modify: `repo-clone` (after CloneOptions dataclass, around line 116)

**Step 1: Add the dataclasses**

Add after the `CloneOptions` dataclass:

```python
@dataclass
class WatchConfig:
    """Configuration for clipboard watch mode."""
    interval: float = 0.5                    # Polling interval in seconds
    selection: str = "clipboard"             # clipboard | primary | both
    clipboard_command: Optional[str] = None  # Override auto-detected command
    extra_hosts: List[str] = field(default_factory=list)  # Additional hosts to recognize


@dataclass
class WatchStats:
    """Runtime statistics for watch mode."""
    cloned: int = 0
    skipped: int = 0
    failed: int = 0
    seen_urls: set = field(default_factory=set)
```

**Step 2: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add WatchConfig and WatchStats dataclasses

Data structures for clipboard watch mode configuration and runtime stats.

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 3: Add Config File Loading Function

**Files:**

* Modify: `repo-clone` (add function after `normalize_clone_url`, around line 227)

**Step 1: Add the function**

```python
def load_config() -> dict:
    """
    Load configuration from ~/.config/shibuido/repo-clone/repo-clone-config.yaml

    Returns empty dict if file doesn't exist or PyYAML not installed.
    """
    config_path = Path.home() / ".config" / "shibuido" / "repo-clone" / "repo-clone-config.yaml"

    if not config_path.exists():
        log_debug(f"No config file at {config_path}")
        return {}

    try:
        import yaml
    except ImportError:
        log_warn(f"Config file found but PyYAML not installed. Using defaults.\n"
                 f"  Install with: pip install pyyaml")
        return {}

    try:
        with open(config_path) as f:
            config = yaml.safe_load(f) or {}
        log_debug(f"Loaded config from {config_path}: {config}")
        return config
    except Exception as e:
        log_warn(f"Failed to parse config file: {e}")
        return {}
```

**Step 2: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add load_config() for YAML configuration file

Loads optional config from ~/.config/shibuido/repo-clone/repo-clone-config.yaml
Gracefully handles missing PyYAML or invalid YAML.

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 4: Add Clipboard Backend Detection

**Files:**

* Modify: `repo-clone` (add imports at top, add function after `load_config`)

**Step 1: Add shutil import**

Add to imports section (around line 26):

```python
import shutil
import signal
```

**Step 2: Add detect_clipboard_backend function**

```python
def detect_clipboard_backend(selection: str = "clipboard") -> Optional[List[List[str]]]:
    """
    Detect available clipboard tool for the current platform/session.

    Args:
        selection: "clipboard", "primary", or "both"

    Returns:
        List of command lists to try (for "both" mode, returns two commands).
        None if no clipboard tool found.
    """
    import platform
    system = platform.system()

    def make_commands(tool: str, sel: str) -> List[str]:
        """Build command list for a tool and selection type."""
        if tool == "wl-paste":
            cmd = ["wl-paste", "--type", "text"]
            if sel == "primary":
                cmd.insert(1, "--primary")
            return cmd
        elif tool == "xclip":
            return ["xclip", "-selection", sel, "-o"]
        elif tool == "xsel":
            flag = "--clipboard" if sel == "clipboard" else "--primary"
            return ["xsel", flag, "--output"]
        elif tool == "pbpaste":
            return ["pbpaste"]
        return []

    def test_tool(cmd: List[str]) -> bool:
        """Test if a clipboard command works."""
        try:
            result = subprocess.run(cmd, capture_output=True, timeout=2)
            return result.returncode == 0
        except (subprocess.TimeoutExpired, FileNotFoundError, OSError):
            return False

    def detect_for_selection(sel: str) -> Optional[List[str]]:
        """Detect working clipboard command for a specific selection."""
        if system == "Darwin":
            cmd = make_commands("pbpaste", sel)
            if shutil.which("pbpaste"):
                return cmd
            return None

        # Linux/BSD: check session type
        session_type = os.environ.get("XDG_SESSION_TYPE", "").lower()
        wayland_display = os.environ.get("WAYLAND_DISPLAY")
        display = os.environ.get("DISPLAY")

        # Try Wayland first if session indicates it
        if session_type == "wayland" or wayland_display:
            if shutil.which("wl-paste"):
                cmd = make_commands("wl-paste", sel)
                if test_tool(cmd):
                    log_debug(f"Using wl-paste for {sel}")
                    return cmd

        # Try X11 tools
        if session_type == "x11" or display:
            if shutil.which("xclip"):
                cmd = make_commands("xclip", sel)
                if test_tool(cmd):
                    log_debug(f"Using xclip for {sel}")
                    return cmd
            if shutil.which("xsel"):
                cmd = make_commands("xsel", sel)
                if test_tool(cmd):
                    log_debug(f"Using xsel for {sel}")
                    return cmd

        return None

    if selection == "both":
        clip_cmd = detect_for_selection("clipboard")
        primary_cmd = detect_for_selection("primary")
        if clip_cmd or primary_cmd:
            return [cmd for cmd in [clip_cmd, primary_cmd] if cmd]
        return None
    else:
        cmd = detect_for_selection(selection)
        return [cmd] if cmd else None
```

**Step 3: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 4: Test detection on current system**

Run: `python3 -c "exec(open('repo-clone').read().split('if __name__')[0]); print(detect_clipboard_backend('clipboard'))"`
Expected: A list like `[['xclip', '-selection', 'clipboard', '-o']]` or similar

**Step 5: Commit**

```bash
git add repo-clone
git commit -m "feat: add detect_clipboard_backend() for cross-platform clipboard access

Auto-detects clipboard tool based on platform and session type:
- macOS: pbpaste
- Linux Wayland: wl-paste
- Linux X11: xclip, xsel (fallback chain)

Supports clipboard, primary, and both selections.

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 5: Add Clipboard Read and URL Extraction Functions

**Files:**

* Modify: `repo-clone` (add after `detect_clipboard_backend`)

**Step 1: Add the functions**

```python
def read_clipboard(commands: List[List[str]]) -> Optional[str]:
    """
    Read text from clipboard using detected backend.

    Args:
        commands: List of command lists to try (from detect_clipboard_backend)

    Returns:
        Clipboard text or None on error/empty.
    """
    for cmd in commands:
        try:
            result = subprocess.run(cmd, capture_output=True, text=True, timeout=2)
            if result.returncode == 0 and result.stdout:
                return result.stdout
        except (subprocess.TimeoutExpired, FileNotFoundError, OSError) as e:
            log_debug(f"Clipboard read failed with {cmd[0]}: {e}")
            continue
    return None


def extract_repo_url(text: str, extra_hosts: List[str] = None) -> Optional[str]:
    """
    Extract a repository URL from clipboard text if it matches known hosts.

    Args:
        text: Raw clipboard text
        extra_hosts: Additional hosts to recognize beyond BUILTIN_HOSTS

    Returns:
        Cleaned URL string or None if not a recognized repo URL.
    """
    if not text:
        return None

    # Get first line, strip whitespace
    line = text.strip().split('\n')[0].strip()
    if not line:
        return None

    # Strip query params and anchors for matching
    clean_url = line.split('?')[0].split('#')[0]

    # Build host list
    all_hosts = BUILTIN_HOSTS + (extra_hosts or [])

    # Check SSH format: git@host:user/repo
    ssh_match = re.match(r'^(?:ssh://)?git@([^:/]+)[:/]', clean_url)
    if ssh_match:
        host = ssh_match.group(1)
        if host in all_hosts:
            log_debug(f"Matched SSH URL for host: {host}")
            return clean_url

    # Check HTTPS format
    if clean_url.startswith(('http://', 'https://')):
        try:
            parsed = urlparse(clean_url)
            if parsed.netloc in all_hosts:
                log_debug(f"Matched HTTPS URL for host: {parsed.netloc}")
                return clean_url
        except Exception:
            pass

    return None
```

**Step 2: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add read_clipboard() and extract_repo_url() functions

- read_clipboard(): reads from detected clipboard backend
- extract_repo_url(): extracts repo URL if host matches known list

Handles SSH and HTTPS formats, strips query params and anchors.

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 6: Add Status Line Printing Function

**Files:**

* Modify: `repo-clone` (add after `extract_repo_url`)

**Step 1: Add the function**

```python
def print_status_line(stats: WatchStats, message: str = "") -> None:
    """
    Print/update the in-place status line.

    Args:
        stats: Current watch statistics
        message: Optional additional message
    """
    status = f"\rWatching clipboard... (cloned: {stats.cloned}, skipped: {stats.skipped}, failed: {stats.failed})"
    if message:
        status += f" {message}"
    status += " Ctrl+C to stop"
    # Clear to end of line and print
    print(f"{status}\033[K", end="", flush=True, file=sys.stderr)
```

**Step 2: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add print_status_line() for in-place status updates

Shows cloned/skipped/failed counts, updates in-place using carriage return.

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 7: Add Main Watch Loop Function

**Files:**

* Modify: `repo-clone` (add after `print_status_line`)

**Step 1: Add the watch_clipboard function**

```python
def watch_clipboard(options: CloneOptions, watch_config: WatchConfig) -> int:
    """
    Main clipboard watch loop.

    Args:
        options: Clone options to apply to all clones
        watch_config: Watch mode configuration

    Returns:
        Exit code (0 if any success or no attempts, 1 if all failed)
    """
    # Detect clipboard backend
    if watch_config.clipboard_command:
        # User override from config
        import shlex
        clipboard_commands = [shlex.split(watch_config.clipboard_command)]
        log_info(f"Using configured clipboard command: {watch_config.clipboard_command}")
    else:
        clipboard_commands = detect_clipboard_backend(watch_config.selection)
        if not clipboard_commands:
            log_error("No clipboard tool found.\n"
                     "Install one of: wl-clipboard (Wayland), xclip, xsel (X11)\n"
                     "  Arch:   pacman -S wl-clipboard   # or xclip\n"
                     "  Ubuntu: apt install xclip")
            return ExitCode.NO_CLIPBOARD_TOOL

    stats = WatchStats()
    last_content = ""
    running = True

    def handle_sigint(signum, frame):
        nonlocal running
        running = False

    signal.signal(signal.SIGINT, handle_sigint)

    print("Watching clipboard for repo URLs... (Ctrl+C to stop)", file=sys.stderr)
    print_status_line(stats)

    while running:
        content = read_clipboard(clipboard_commands)

        if not content or content == last_content:
            time.sleep(watch_config.interval)
            continue

        last_content = content
        url = extract_repo_url(content, watch_config.extra_hosts)

        if not url:
            time.sleep(watch_config.interval)
            continue

        # Check for duplicate in this session
        if url in stats.seen_urls:
            log_info(f"Skipping duplicate: {url}")
            stats.skipped += 1
            print_status_line(stats)
            time.sleep(watch_config.interval)
            continue

        stats.seen_urls.add(url)

        # Try to parse and clone
        try:
            info = parse_git_url(url)
            print(f"\n", file=sys.stderr)  # Clear status line
            result = clone_repo(info, options)
            if result == ExitCode.SUCCESS:
                stats.cloned += 1
            else:
                stats.failed += 1
                log_error(f"Clone failed for: {url}")
        except ValueError as e:
            log_debug(f"Could not parse URL: {url} - {e}")
            stats.failed += 1

        print_status_line(stats)
        # No sleep after action - immediate re-check

    # Print summary
    print(f"\nStopped. Cloned: {stats.cloned}, Skipped: {stats.skipped}, Failed: {stats.failed}",
          file=sys.stderr)

    # Exit code: 0 if any success or no attempts, 1 if all failed
    total_attempts = stats.cloned + stats.failed
    if total_attempts > 0 and stats.cloned == 0:
        return 1
    return 0
```

**Step 2: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 3: Commit**

```bash
git add repo-clone
git commit -m "feat: add watch_clipboard() main loop function

Core watch mode implementation:
- Polls clipboard at configured interval
- Detects repo URLs matching known hosts
- Session-scoped deduplication with INFO logging
- In-place status line updates
- Graceful SIGINT handling with summary
- Smart exit codes (0 if any success, 1 if all failed)

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 8: Update CLI Argument Parser

**Files:**

* Modify: `repo-clone` (modify `main()` function, around line 530)

**Step 1: Update argument parser**

Replace the current `parser.add_argument("url", ...)` line with:

```python
    parser.add_argument("url", nargs="?", default=None,
                        help="Git repository URL (SSH or HTTPS)")
    parser.add_argument("-w", "--watch", action="store_true",
                        help="Watch clipboard for repo URLs (continuous mode)")
    parser.add_argument("-i", "--interval", type=float, default=0.5,
                        help="Clipboard polling interval in seconds (default: 0.5)")
    parser.add_argument("--selection", choices=["clipboard", "primary", "both"],
                        default="clipboard",
                        help="Linux clipboard selection (default: clipboard)")
```

**Step 2: Add validation after args = parser.parse_args()**

Add after `args = parser.parse_args()`:

```python
    # Validate: either --watch or url required, not both
    if args.watch and args.url:
        parser.error("Cannot use --watch with a URL argument")
    if not args.watch and not args.url:
        parser.error("Either provide a URL or use --watch mode")
```

**Step 3: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 4: Test help output**

Run: `./repo-clone --help`
Expected: Shows new --watch, --interval, --selection options

**Step 5: Commit**

```bash
git add repo-clone
git commit -m "feat: add --watch, --interval, --selection CLI arguments

- --watch/-w: enable clipboard monitoring mode
- --interval/-i: polling interval in seconds (float, default 0.5)
- --selection: clipboard|primary|both for Linux (default: clipboard)
- URL now optional (nargs='?') with mutual exclusivity validation

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 9: Add Watch Mode Branch in main()

**Files:**

* Modify: `repo-clone` (modify end of `main()` function)

**Step 1: Add watch mode handling**

Replace the `try: exit_code = repo_clone(args.url, options)` block with:

```python
    try:
        if args.watch:
            # Load config file
            config = load_config()

            # Build WatchConfig with precedence: CLI > config > defaults
            watch_config = WatchConfig(
                interval=args.interval if args.interval != 0.5 else config.get("watch_interval", 0.5),
                selection=args.selection if args.selection != "clipboard" else config.get("watch_selection", "clipboard"),
                clipboard_command=config.get("clipboard_command"),
                extra_hosts=config.get("extra_hosts", []),
            )

            # Override CloneOptions from config if not set by CLI
            if options.base_dir is None and "base_dir" in config:
                base_dir_str = config["base_dir"]
                if base_dir_str.startswith("~"):
                    base_dir_str = os.path.expanduser(base_dir_str)
                options.base_dir = Path(base_dir_str)

            exit_code = watch_clipboard(options, watch_config)
        else:
            exit_code = repo_clone(args.url, options)
        sys.exit(exit_code)
    except KeyboardInterrupt:
        print("\nInterrupted", file=sys.stderr)
        sys.exit(130)
```

**Step 2: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 3: Test watch mode starts**

Run: `timeout 2 ./repo-clone --watch 2>&1 || true`
Expected: Shows "Watching clipboard..." message

**Step 4: Commit**

```bash
git add repo-clone
git commit -m "feat: integrate watch mode into main() with config precedence

- Loads config file and builds WatchConfig
- CLI flags override config file values
- Config file overrides built-in defaults
- Branches to watch_clipboard() when --watch is used

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 10: Update Docstring and Examples

**Files:**

* Modify: `repo-clone` (update module docstring at top and epilog in main())

**Step 1: Update module docstring**

Replace lines 2-21 with:

```python
"""
Repo Clone Tool - Clone and organize git repositories with submodule support.

Usage:
    repo-clone [OPTIONS] <url>        Clone/update a single repository
    repo-clone --watch [OPTIONS]      Monitor clipboard for repo URLs

This script clones repositories organized under:
    $HOME/<hosting_shortcode>/<username>/<repo>

Examples:
    repo-clone git@github.com:user/repo.git
    repo-clone https://github.com/user/repo
    repo-clone https://huggingface.co/TheBloke/model
    repo-clone --shallow https://github.com/user/large-repo
    repo-clone --watch                    # Monitor clipboard
    repo-clone --watch --shallow          # Monitor with shallow clones
    repo-clone --watch --selection both   # Monitor both clipboards (Linux)

Features:
    - Automatic submodule initialization (--recursive by default)
    - Updates existing repos with fetch/pull and submodule sync
    - Fixes repos that were cloned without submodules
    - Special handling for Hugging Face (git lfs)
    - Clipboard watch mode for continuous cloning
"""
```

**Step 2: Update epilog in main()**

Update the epilog in argparse to:

```python
        epilog="""
Examples:
  %(prog)s git@github.com:user/repo.git
  %(prog)s https://github.com/user/repo
  %(prog)s https://huggingface.co/TheBloke/model
  %(prog)s --shallow https://github.com/user/large-repo
  %(prog)s --no-recursive https://github.com/user/repo
  %(prog)s --dry-run -v https://github.com/user/repo

Watch mode:
  %(prog)s --watch                    Monitor clipboard continuously
  %(prog)s --watch --shallow          Shallow clone detected repos
  %(prog)s --watch -i 0.25            Faster polling (250ms)
  %(prog)s --watch --selection both   Monitor both clipboards (Linux)
  %(prog)s --watch --dry-run -v       Test URL detection

Config file: ~/.config/shibuido/repo-clone/repo-clone-config.yaml
        """
```

**Step 3: Verify syntax**

Run: `python3 -m py_compile repo-clone`
Expected: No output (success)

**Step 4: Test help shows new examples**

Run: `./repo-clone --help`
Expected: Shows watch mode examples

**Step 5: Commit**

```bash
git add repo-clone
git commit -m "docs: update docstring and help text with watch mode examples

Adds comprehensive examples for watch mode usage including:
- Basic watch mode
- Shallow clones in watch mode
- Custom polling interval
- Linux clipboard selection
- Dry run for testing

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 11: Manual Integration Test

**Files:**

* None (manual testing)

**Step 1: Test basic watch mode**

Run in terminal 1:
```bash
./repo-clone --watch --dry-run -v
```

In terminal 2, copy a GitHub URL:
```bash
echo "https://github.com/anthropics/anthropic-sdk-python" | xclip -selection clipboard
```

Expected: Terminal 1 shows URL detection and dry-run clone output

**Step 2: Test duplicate detection**

Copy the same URL again in terminal 2.
Expected: Terminal 1 shows "INFO: Skipping duplicate: ..."

**Step 3: Test Ctrl+C exit**

Press Ctrl+C in terminal 1.
Expected: Shows summary "Stopped. Cloned: 0, Skipped: 1, Failed: 0"

**Step 4: Test actual clone (optional)**

```bash
./repo-clone --watch --shallow
```
Copy a small repo URL. Verify it clones successfully.

**Step 5: Commit test results as META commit**

```bash
git commit --allow-empty -m "META: Manual integration test of watch mode

TESTED:
- Basic watch mode starts and displays status line
- URL detection works for GitHub HTTPS URLs
- Duplicate detection skips repeated URLs with INFO message
- Ctrl+C shows summary and exits cleanly
- Dry-run mode prevents actual cloning

ENVIRONMENT: Linux X11, xclip

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Task 12: Update README with Watch Mode Documentation

**Files:**

* Modify: `README.md`

**Step 1: Add Watch Mode section**

Add after the main usage section:

```markdown
## Watch Mode

Monitor your clipboard for repository URLs and automatically clone them:

```bash
# Basic watch mode
repo-clone --watch

# With shallow clones
repo-clone --watch --shallow

# Custom polling interval (250ms)
repo-clone --watch -i 0.25

# Monitor both clipboards on Linux (primary + clipboard)
repo-clone --watch --selection both

# Test URL detection without cloning
repo-clone --watch --dry-run -v
```

Watch mode recognizes URLs from: GitHub, GitLab, Codeberg, Bitbucket, Hugging Face, SourceHut.

**Want support for another host?** Open an issue or PR!

### Configuration

Create `~/.config/shibuido/repo-clone/repo-clone-config.yaml`:

```yaml
# Additional hosts to recognize
extra_hosts:
  - gitlab.mycompany.com
  - gitea.internal.org

# Watch mode defaults
watch_interval: 0.5
watch_selection: clipboard  # clipboard | primary | both

# Override clipboard command (optional)
# clipboard_command: "xclip -selection clipboard -o"
```

Requires `pyyaml` for config file support: `pip install pyyaml`
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add Watch Mode section to README

Documents:
- Basic and advanced watch mode usage
- Supported repository hosts
- Configuration file format and location
- Encourages community contributions for new hosts

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
```

---

## Summary

| Task | Description | New Lines |
|------|-------------|-----------|
| 1 | BUILTIN_HOSTS + exit code | ~12 |
| 2 | WatchConfig + WatchStats dataclasses | ~18 |
| 3 | load_config() function | ~28 |
| 4 | detect_clipboard_backend() | ~75 |
| 5 | read_clipboard() + extract_repo_url() | ~55 |
| 6 | print_status_line() | ~12 |
| 7 | watch_clipboard() main loop | ~75 |
| 8 | CLI argument updates | ~15 |
| 9 | main() watch mode branch | ~25 |
| 10 | Docstring updates | ~20 |
| 11 | Manual integration test | (testing) |
| 12 | README documentation | ~40 |

**Total new code:** ~315 lines (within 200-300 estimate from design)
