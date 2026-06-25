# repo-clone

> Stop organizing repositories. Start exploring them.

Paste a URL, get a tidy clone exactly where you'd expect:

```bash
repo-clone https://github.com/torvalds/linux
# → ~/github/torvalds/linux

repo-clone https://gist.github.com/torvalds/1234567890abcdef
# → ~/github/_gist/torvalds/1234567890abcdef

repo-clone --fork https://github.com/acme/widget
# → ~/github/acme/widget        (upstream, ready to read)
# → ~/github/<you>/widget        (your fork, cloned, with upstream remote)

repo-clone --watch
# paste URLs, they clone themselves. window-shopping mode.
```

## Why

* **Organize on the way in.** Every clone lands at `$HOME/$host/$org/$repo`
  or, for Gists, `$HOME/github/_gist/$user/$gist_id` — predictable,
  greppable, fuzzy-searchable.
* **Idempotent.** Re-run the same command; it updates cleanly. No surprises.
* **Fork-and-experiment loop without leaving the terminal.** `--fork` creates
  the fork, clones both sides, and wires the `upstream` remote.

## Install

```bash
curl -L https://raw.githubusercontent.com/shibuido/repo-clone/master/repo-clone \
  -o ~/.local/bin/repo-clone && chmod +x ~/.local/bin/repo-clone
```

Requires Python 3.9+, `git`. Optional: `git-lfs` (Hugging Face), `gh` (`--fork`),
`pyyaml` (richer config).

## Docs

* **Full guide** → [`repo-clone.README.md`](repo-clone.README.md)
* **Configuration** → [`docs/design/repo-clone-DESIGN-config-file.md`](docs/design/repo-clone-DESIGN-config-file.md)
* **Troubleshooting** → [`docs/repo-clone-TROUBLESHOOTING.md`](docs/repo-clone-TROUBLESHOOTING.md)
* **🤖 AI agents** → `repo-clone --help-for-agents`
* **Design notes** → [`docs/design/`](docs/design/)
* **Contributing** → [`maintainers/`](maintainers/)

## Status

GitHub support today. GitLab / Codeberg / Bitbucket — PRs welcome
([`repo-clone.FUTURE_WORK.md`](repo-clone.FUTURE_WORK.md)).

Issues, ideas, jokes: https://github.com/shibuido/repo-clone/issues

## License

[Insert here once a LICENSE file is added — see FUTURE_WORK.]
