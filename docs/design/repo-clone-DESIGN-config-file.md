# Config file

**Path:** `~/.config/shibuido/repo-clone/repo-clone-config.yaml`
(respects `$XDG_CONFIG_HOME`; if set, replaces `~/.config`).

Loaded if present, ignored if absent. Forward-compatible layout (see
`repo-clone-DESIGN-multi-account-config-layout.md`).

## Schema

```yaml
# Existing keys (watch mode):
extra_hosts:                       # Additional hosts beyond BUILTIN_HOSTS
  - gitlab.mycompany.com
watch_interval: 0.5                # Polling interval (seconds)
watch_selection: clipboard         # clipboard | primary | both
clipboard_command: null            # Override auto-detected clipboard tool
base_dir: ~/repos                  # Override $HOME for clone destinations

# New: fork section
fork:
  target_org: VariousForks         # Default: gh-authed user
  name_template: "{repo}-by-{org}-fork"   # Default: "{repo}"
  all_branches: false              # Default: false (matches GitHub webui)
  upstream_remote_name: upstream   # Default: "upstream"
  about_suffix: ""                 # Appended to fork "About" description.
                                   # Default: "(forked using repo-clone CLI https://github.com/shibuido/repo-clone)"
                                   # Set to "" or false to disable.

# New: output section
output:
  mode: human                      # human | jsonl
  show_info: false                 # If true, INFO: visible at -v0
```

## Precedence (highest wins)

```
CLI flag  >  env var  >  config file  >  built-in default
```

## Env vars

| Var | Maps to |
|---|---|
| `REPO_CLONE_BASE_DIR` | `base_dir` |
| `REPO_CLONE_VERBOSE` | verbosity (0..3) |
| `REPO_CLONE_FORK_ORG` | `fork.target_org` |

## Loader behavior

1. If `~/.config/shibuido/repo-clone/repo-clone-config.yaml` exists, parse it.
2. Else, if `~/.config/shibuido/repo-clone/profiles/default.yaml` exists,
   parse it (forward-compat path; see multi-account layout doc).
3. Else, use built-in defaults.
4. Try `pyyaml` first; on `ImportError`, fall back to the stdlib YAML-subset
   reader (`repo-clone-DESIGN-stdlib-portability.md`).
5. On parse failure, print `ERROR:` and exit nonzero (do not silently use
   defaults — config errors should be loud).

## Why one file, not multiple profiles (yet)

v1 keeps it simple: one user, one config. Power users with multi-account
needs use `--config PATH` to point at an alternate file. Profile support is
forward-compatible (see multi-account layout doc) but **not implemented**.

## Next step

* Loader implementation: see `repo-clone` source, section
  `# ---- Config file loading ----`.
* Multi-account layout: `repo-clone-DESIGN-multi-account-config-layout.md`.
