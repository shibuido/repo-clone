# repo-clone Future Work

## Now-implemented items (kept for context; remove after a release cycle)

These items from earlier FUTURE_WORK are now landed:

* ✅ Config file (`~/.config/shibuido/repo-clone/repo-clone-config.yaml`) —
  shipped 2026-05-06 with `fork:` and `output:` sections.
* ✅ GitLab nested subgroup parsing + `{namespace}` template vars + per-host
  `fork.<shortcode>.*` config sections — shipped 2026-05-14. `RepoInfo` now
  carries `namespace_path: List[str]`; on-disk layout mirrors the full chain
  (`~/gitlab/group/sub/repo`). The `/-/` UI-resource sentinel is stripped.
  Template vars: `{namespace}`, `{namespace_slash}`, `{parent}`, `{top}`,
  `{host}` (plus `{org}` as legacy alias).
* ✅ `--fork` for GitLab via `glab` — shipped 2026-05-15. Mirrors the gh
  flow: `glab_preflight`, `glab_authed_user`, `fork_exists_on_gitlab`,
  `create_gitlab_fork`, `update_fork_about_gitlab`. `glab repo fork` has
  no flag to target a different namespace, so fork creation drives
  `POST projects/:id/fork` via `glab api` directly with `namespace_path`.
  New exit code `13` (`GLAB_MISSING`).

## Investigate
* [ ] Bitbucket URL format support
* [ ] SSH config aliases (e.g., `git@gh:user/repo` where `gh` is Host alias)
* [ ] Minimum `gh` version supporting `--default-branch-only` — document the floor
* [ ] `wire_upstream_remote` URL comparison: handle `url.insteadOf` rewrites
  so repeat runs don't WARN when stored remote URL is the rewritten form
  (e.g. `git@github.com:org/repo` vs `https://github.com/org/repo`)
* [ ] `summary.actions` reports `added-upstream` even on re-runs where
  `wire_upstream_remote` took the WARN-mismatch (skip) path. Cosmetic only;
  the actions list should distinguish `added` / `confirmed` / `mismatched`.
  Discovered during the 2026-05-07 wet-run integration test (see
  `docs/qa/repo-clone-QA-fork-wettest.md`).

## Consider Adding

* [ ] `--force-pull` — stash local changes, pull, then pop stash
* [ ] `--prune` — prune stale remote tracking branches on update
* [ ] `--jobs N` — parallel submodule fetching (`git fetch --jobs=N`)
* [ ] Progress output for large clones (pass through git's progress)
* [ ] Retry logic for transient network errors (beyond fork-just-created race)

## --fork roadmap

* [ ] `--fork` for Codeberg / Forgejo
* [ ] `--fork` for Bitbucket
* [ ] `--profile NAME` flag + `~/.config/shibuido/repo-clone/profiles/*.yaml`
  (multi-account; layout already forward-compatible — see
  `docs/design/repo-clone-DESIGN-multi-account-config-layout.md`)

## Low Priority

* [ ] `--mirror` mode for backup purposes
* [ ] Fish/Zsh completions
* [ ] Unit tests with pytest (manual integration tests in DEV_NOTES suffice for now)
* [ ] LICENSE file added
