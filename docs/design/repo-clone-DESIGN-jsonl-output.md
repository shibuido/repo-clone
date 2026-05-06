# JSON Lines output mode (`--jsonl`)

When `--jsonl` is passed (or `output.mode: jsonl` is set in config), the tool
emits **one compact JSON object per line on stdout**, and suppresses the
human-mode `INFO:` / `WARN:` / `ERROR:` prefix lines on stderr (the data is
already in the JSONL stream).

Mutually exclusive with the human prefix mode. CLI `--jsonl` overrides config.

## Why

Programmatic consumers (other shell scripts, agents, CI jobs) parse line-by-line
without regex-fragile prefix matching. `jq` works out of the box.

## Schema

Every line is a JSON object with at minimum these fields:

```json
{"event": "<event-name>", "level": "info" | "warn" | "error" | "debug", "ts": "<ISO-8601>", ...}
```

Field rules:

* `event` — stable, kebab-case event name. Additive: new events may appear; old
  events keep their names. Consumers should ignore unknown events.
* `level` — `"info"`, `"warn"`, `"error"`, `"debug"`. Maps 1:1 to the
  human-mode prefixes.
* `ts` — ISO-8601 timestamp with timezone (e.g. `2026-05-06T14:23:01.045+00:00`).
* Per-event fields — additive; new fields may appear in future versions.

## Event catalogue (v1)

| event | level | typical fields |
|---|---|---|
| `parsed-url` | info | `url`, `host`, `org`, `repo`, `is_ssh` |
| `clone-start` | info | `url`, `path`, `recursive`, `shallow` |
| `clone-success` | info | `url`, `path` |
| `clone-failed` | error | `url`, `path`, `git_exit_code` |
| `update-start` | info | `path` |
| `update-fetched` | info | `path` |
| `update-pulled` | info | `path` |
| `update-skipped-dirty` | warn | `path`, `dirty_summary` |
| `submodule-result` | info / warn | `path`, `ok` |
| `gh-preflight-ok` | info | `gh_version` |
| `gh-missing` | error | `hint` |
| `gh-unauthed` | error | `hint` |
| `fork-identity-resolved` | info | `source_org`, `source_repo`, `target_org`, `fork_name`, `template` |
| `fork-exists` | info | `target_org`, `fork_name`, `parent_full_name` |
| `fork-collision` | error | `target_org`, `fork_name`, `parent_full_name`, `expected_parent` |
| `fork-creating` | info | `source`, `target_org`, `fork_name`, `all_branches` |
| `fork-created` | info | `source`, `target_org`, `fork_name`, `clone_url` |
| `fork-creation-failed` | error | `source`, `target_org`, `fork_name`, `gh_exit_code`, `stderr` |
| `upstream-added` | info | `fork_path`, `remote_name`, `upstream_url` |
| `upstream-already-correct` | info | `fork_path`, `remote_name` |
| `upstream-mismatch` | warn | `fork_path`, `remote_name`, `current_url`, `expected_url` |
| `summary` | info | `source_path`, `fork_path` (optional), `actions` (list) |

## Example session (`repo-clone --fork --jsonl https://github.com/acme/widget`)

```jsonl
{"event":"parsed-url","level":"info","ts":"...","url":"https://github.com/acme/widget","host":"github.com","org":"acme","repo":"widget","is_ssh":false}
{"event":"gh-preflight-ok","level":"info","ts":"...","gh_version":"2.45.0"}
{"event":"clone-start","level":"info","ts":"...","url":"https://github.com/acme/widget.git","path":"/home/u/github/acme/widget","recursive":true,"shallow":false}
{"event":"clone-success","level":"info","ts":"...","url":"https://github.com/acme/widget.git","path":"/home/u/github/acme/widget"}
{"event":"fork-identity-resolved","level":"info","ts":"...","source_org":"acme","source_repo":"widget","target_org":"VariousForks","fork_name":"widget-by-acme-fork","template":"{repo}-by-{org}-fork"}
{"event":"fork-creating","level":"info","ts":"...","source":"acme/widget","target_org":"VariousForks","fork_name":"widget-by-acme-fork","all_branches":false}
{"event":"fork-created","level":"info","ts":"...","source":"acme/widget","target_org":"VariousForks","fork_name":"widget-by-acme-fork","clone_url":"https://github.com/VariousForks/widget-by-acme-fork.git"}
{"event":"clone-start","level":"info","ts":"...","url":"https://github.com/VariousForks/widget-by-acme-fork.git","path":"/home/u/github/VariousForks/widget-by-acme-fork","recursive":true,"shallow":false}
{"event":"clone-success","level":"info","ts":"...","url":"https://github.com/VariousForks/widget-by-acme-fork.git","path":"/home/u/github/VariousForks/widget-by-acme-fork"}
{"event":"upstream-added","level":"info","ts":"...","fork_path":"/home/u/github/VariousForks/widget-by-acme-fork","remote_name":"upstream","upstream_url":"https://github.com/acme/widget.git"}
{"event":"summary","level":"info","ts":"...","source_path":"/home/u/github/acme/widget","fork_path":"/home/u/github/VariousForks/widget-by-acme-fork","actions":["cloned-source","created-fork","cloned-fork","added-upstream"]}
```

## Stability promise

* Event names: never renamed.
* Per-event fields: never removed; never have their meaning changed.
* New events and new fields: may appear at any version (additive only).

Consumers should:

* Match on `event` and ignore unknown ones.
* Tolerate unknown fields.
* Not depend on the order of events beyond what's documented above.

## See also

* Output prefixes (human mode): `repo-clone-DESIGN-output-prefixes.md`.
