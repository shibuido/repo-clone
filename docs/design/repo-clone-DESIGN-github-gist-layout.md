# GitHub Gist layout

## Decision

GitHub Gists clone under the GitHub base with a reserved `_gist` namespace:

```text
$BASE/github/_gist/<user-or-anonymous>/<gist-id>
```

Examples:

```text
https://gist.github.com/1234567890abcdef
→ ~/github/_gist/anonymous/1234567890abcdef

https://gist.github.com/torvalds/1234567890abcdef
→ ~/github/_gist/torvalds/1234567890abcdef
```

The clone URL remains a real Gist URL:

```text
https://gist.github.com/<gist-id>.git
https://gist.github.com/<user>/<gist-id>.git
```

## Why

Gists are Git repositories, but their URL shape differs from normal GitHub
repositories: `gist.github.com/<id>` may have no owner segment. Putting them
under `github/_gist` keeps GitHub-related material together while avoiding
collisions with real repositories under `github/<user>/<repo>`.

The leading underscore marks `_gist` as a repo-clone-managed synthetic namespace.
It is intentionally centralized:

```text
~/github/_gist/alice/<id>
```

instead of:

```text
~/github/alice/_gist/<id>
```

That keeps `~/github/alice/*` reserved for normal repositories owned by Alice.

## Parsing

Supported URL forms:

* `https://gist.github.com/<gist-id>`
* `https://gist.github.com/<user>/<gist-id>`
* `git@gist.github.com:<gist-id>.git`
* `git@gist.github.com:<user>/<gist-id>.git`

Id-only URLs use `anonymous` as the owner bucket because the URL itself does not
carry an owner. This is a path bucket, not a claim about the Gist's actual owner.

## Related

* User manual: `../../repo-clone.README.md`
* URL parser: `../../repo-clone`
* Filename convention: `repo-clone-DESIGN-greppable-filenames.md`
