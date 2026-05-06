# `--help-for-agents`

A second help mode targeted at LLM agents using `repo-clone` programmatically.
Two equivalent entry points (Q6):

* `repo-clone --help-for-agents` (canonical flag)
* `repo-clone help-for-agents` (subcommand alias)

## Audience

LLM agents reading the output as part of a context window, deciding how to
invoke the tool, predicting failure modes, parsing JSONL, recovering from
errors. **Not humans.**

## Style

* Information density over politeness.
* Pseudocode, tables, equations, fragments preferred over prose.
* No reassuring filler ("This tool helps you...").
* No marketing tone.
* Emoji only where they aid scanning (e.g., 🤖 / ⚠️ delimiters), never decoration.
* No repetition of `--help` content. Cross-link instead.

## Required sections (each independently editable)

| Section | Purpose |
|---|---|
| `IDENTITY` | One-line what + why. |
| `INVOCATION` | Grammar in EBNF-ish form + canonical examples. |
| `EXIT_CODES` | Table of codes and their meanings (1..N). |
| `STATE_MACHINE` | Per-URL outcome decision tree (no fork / fork / re-run). |
| `JSONL_SCHEMA` | Pointer to design doc + brief inline event list. |
| `IDEMPOTENCY` | Re-run guarantees: what's safe, what's not. |
| `FAILURE_RECOVERY` | Retryable errors, network timeouts, expected wall-clock ranges (fork creation 5–30s, clones can take minutes). |
| `CONTEXTUAL_FIT` | Where this tool sits in agent workflows; path-layout invariants; companion tools (`find-snapshot`, `sk`, `fzf`). |
| `TROUBLESHOOTING_POINTERS` | Local + remote URL of `docs/repo-clone-TROUBLESHOOTING.md`. |
| `REPORT_BUGS` | Issue tracker link + brief template. |

## Cross-links

* `--help-for-agents` output mentions `--help` once at the top, for the human
  overview. Do not duplicate `--help` content beyond that pointer.
* `--help` mentions `--help-for-agents` prominently (a 4-line block, easy to
  scan), so an agent reading `--help` learns the better option exists.

## Discoverability in `--help`

The `--help` output includes (early, easy to scan):

```
🤖 AI agents: run `repo-clone --help-for-agents` for an agent-optimized
   briefing (denser format, troubleshooting paths, exit-code semantics,
   JSONL contract).
   Issues / unclear behavior → https://github.com/shibuido/repo-clone/issues
```

## Implementation note

The agent-help text is assembled from small string fragments (one per section)
inside the script, so each section is independently editable. Lives in the
`# ---- Agent help ----` banner in the script.

## Anti-patterns

* Padding sections with prose to look "complete".
* Hedging ("This may sometimes...") — agents need crisp constraints.
* Restating `--help` content.
* Walls of text. Use tables.
