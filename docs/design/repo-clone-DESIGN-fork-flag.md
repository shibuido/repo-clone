# `--fork` flag

The `--fork` flag turns `repo-clone <url>` into "clone the source AND create
a fork on the host AND clone the fork side AND wire the upstream remote",
all idempotently.

## CLI surface

```
repo-clone [OPTIONS] <url>

  -F, --fork                  Create a fork on the host, then clone source AND fork.
      --fork-all              Fork copies all branches.        Implies --fork.
      --fork-default          Fork copies only default branch. Implies --fork.
      --fork-org NAME         Override target org for the fork. Implies --fork.
      --fork-name VALUE       Override fork repo name (literal, no templating).
                              If VALUE contains exactly one '/', it splits into
                              --fork-org / --fork-name. Implies --fork.
```

`--fork-all` and `--fork-default` are mutually exclusive (error on both).

## On-disk layout (decision Q2)

After `repo-clone --fork https://github.com/acme/widget`:

```
~/github/acme/widget                            # source clone, origin → acme
~/github/<target_org>/<fork_name>               # fork clone
                                                #   origin   → fork
                                                #   upstream → acme/widget
```

Two directories. The path-tells-you-the-org invariant of `repo-clone` is
preserved on both sides.

## Order of operations

```
1. Parse URL                              (existing)
2. Resolve fork identity                   (URL + config + CLI)
3. gh preflight                            (gh --version, gh auth status)
4. Source-side clone/update                (existing repo-clone behavior)
5. Fork creation (idempotent)              (gh api existence check, gh repo fork)
6. Fork-side clone/update                  (reuse clone path on fork URL)
7. Wire upstream remote                    (idempotent)
8. Emit summary                            (human or JSONL)
```

Source first → bad URL fails before touching GitHub.

## Idempotency

| State | Behavior |
|---|---|
| Fork doesn't exist | Create it |
| Fork exists, parent matches source | Continue (INFO) |
| Fork exists, parent mismatches source | ERROR (exit 11) |
| Fork-side dir doesn't exist | Clone fresh |
| Fork-side dir exists, origin matches expected fork URL | Update (existing path) |
| Fork-side dir exists, origin differs / not a git repo | ERROR (exit 5) |
| `upstream` remote absent on fork clone | Add it |
| `upstream` remote present, URL matches source | Skip |
| `upstream` remote present, URL differs | WARN, do not overwrite |

## Exit codes (additions)

| Code | Meaning |
|---|---|
| 8 | `gh` missing or unauthed |
| 9 | Host not yet supported for `--fork` (non-GitHub) |
| 10 | Fork creation API call failed |
| 11 | Fork name collision with unrelated repo |

## `--fork-name` shorthand

If the value contains exactly one `/`, split:

```
--fork-name greg/widget-experimental
≡
--fork-org greg --fork-name widget-experimental
```

If `--fork-org` is also given with a different value → ERROR.
Multiple `/` or empty either side → ERROR.

## Host support

v1: GitHub only. Other hosts: `--fork` with a non-GitHub URL → exit 9 with a
pointer to FUTURE_WORK and the issue tracker.

## See also

* Config schema: `repo-clone-DESIGN-config-file.md`.
* Output: `repo-clone-DESIGN-output-prefixes.md`, `repo-clone-DESIGN-jsonl-output.md`.
* Help for AI agents: `repo-clone-DESIGN-help-for-agents.md`.
