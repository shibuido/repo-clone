# Multi-account config layout (forward-compatible; not implemented in v1)

## What ships in v1

```
~/.config/shibuido/repo-clone/
└── repo-clone-config.yaml         # the only config file v1 reads
```

The loader also accepts `profiles/default.yaml` as an equivalent alias.
Both are documented; `repo-clone-config.yaml` is the canonical location.

## What v1 does NOT do

* No `--profile NAME` flag.
* No multi-account switching.
* If you need an alternate config for one invocation, use `--config PATH`.

## Forward-compatible layout

When profile support lands (deferred to FUTURE_WORK), the directory grows
**additively** — no migration required:

```
~/.config/shibuido/repo-clone/
├── repo-clone-config.yaml         # equivalent to profiles/default.yaml
└── profiles/
    ├── default.yaml               # default profile
    ├── work.yaml                  # --profile work
    └── personal.yaml              # --profile personal
```

Resolution rules (proposed for future flag):

1. `--config PATH` → that file, ignore everything else.
2. `--profile NAME` → `profiles/NAME.yaml`, ERROR if missing.
3. Default → `repo-clone-config.yaml` if present, else
   `profiles/default.yaml`, else built-in defaults.
4. If `repo-clone-config.yaml` AND `profiles/default.yaml` both exist with
   conflicting contents, WARN and prefer `repo-clone-config.yaml`.

## Why this layout

* **Zeroconf-friendly.** New users have one file to read, one path to
  document.
* **Doesn't punish single-account users with profile boilerplate.**
* **Migration-free.** Existing `repo-clone-config.yaml` stays where it is
  forever; profile support layers on top.

## When to implement

When at least one of these is true:

* Real demand from users with multiple GitHub accounts.
* A second host (GitLab) lands and benefits from per-host credentials per
  profile.
* The single-config + `--config PATH` workflow is demonstrably insufficient.

Until then: YAGNI.
