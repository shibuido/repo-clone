# repo-clone Future Work

## Investigate

* [ ] GitLab subgroups: `https://gitlab.com/group/subgroup/repo` - verify parsing works
* [ ] Bitbucket URL format support
* [ ] SSH config aliases (e.g., `git@gh:user/repo` where `gh` is Host alias)

## Consider Adding

* [ ] `--force-pull` - stash local changes, pull, then pop stash
* [ ] `--prune` - prune stale remote tracking branches on update
* [ ] `--jobs N` - parallel submodule fetching (`git fetch --jobs=N`)
* [ ] Progress output for large clones (pass through git's progress)
* [ ] Retry logic for transient network errors

## Low Priority

* [ ] Config file (`~/.config/repo-clone.toml`) for default options per host
* [ ] `--mirror` mode for backup purposes
* [ ] Fish/Zsh completions
* [ ] Unit tests with pytest
