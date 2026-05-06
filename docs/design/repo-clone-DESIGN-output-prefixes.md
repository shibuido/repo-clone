# Output prefixes: INFO / WARN / ERROR

Human-mode stdout/stderr lines are prefixed by category. These prefixes are
stable; downstream `grep`/`awk` can rely on them.

| Prefix | Meaning | Visibility | Stream |
|---|---|---|---|
| `INFO:` | Routine progress (cloned, fetched, fork exists, etc.) | Hidden by default; shown at `-v` and above, or when `output.show_info: true` | stderr |
| `WARN:` | Degraded but proceeding (local changes, fork exists, upstream remote points elsewhere, etc.) | Always shown | stderr |
| `ERROR:` | Abort condition; followed by exit | Always shown | stderr |
| `DETAIL:` | Existing `-vv` / `-vvv` output (timing, sizes) | Verbosity-gated | stderr |
| `DEBUG:` | Existing `-vvv` output | Verbosity-gated | stderr |

## Why prefixes (and not colors)

* Pipe-friendly: `repo-clone ... | grep -v INFO:` works.
* No tty-detection required.
* Colors can layer on later (planned: ANSI when `sys.stderr.isatty()`) without
  changing the contract.

## When the script writes to stdout vs stderr

* **Stdout:** plain repo paths and JSONL events. Anything a downstream consumer
  might pipe into another tool.
* **Stderr:** everything human-facing (prefixes, progress, errors).

## Verbosity levels (existing)

| Level | Flag | Adds |
|---|---|---|
| 0 | (default) | WARN, ERROR only |
| 1 | `-v` | + INFO, git commands as run |
| 2 | `-vv` | + timing, sizes, DETAIL |
| 3 | `-vvv` | + DEBUG; raw git output streamed live |

`output.show_info: true` in config promotes INFO to level 0 visibility.
`-v` and config setting are independent — either makes INFO visible.

## See also

* JSONL-mode contract: `repo-clone-DESIGN-jsonl-output.md`.
* Tone for the strings inside these messages: `maintainers/repo-clone-MAINTAINERS-tone-guide.md`.
