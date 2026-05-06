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
