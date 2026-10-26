# rokstones.io-index

The package (stone) registry index for [Rok](https://rokstones.io). As with crates.io-index, this git repository is the registry itself: no server hosts source tarballs. Each version points to a commit in the stone's own git repository, and `rok` fetches that commit and verifies it by tree hash.

Website / API: <https://rokstones.io>

## Layout

```text
config.rokon          registry configuration
1/<name>              stones with 1-character names
2/<name>              2 characters
3/<c>/<name>          3 characters (<c>: first character)
<ab>/<cd>/<name>      4 or more characters (<ab>: chars 1-2, <cd>: chars 3-4)
```

`-` stays in the path (`rok-syntax` -> `ro/k-/rok-syntax`). Names differing only in `-` / `_` or case are one stone, so a second spelling cannot be published.

`config.rokon` holds `Config { api: "https://rokstones.io" }`: the server `rok` publishes through. It must be `https://` (only `http://127.0.0.1` and `http://localhost` are exempt).

## Stone files

One file per stone, one line per version (publish order, oldest first). Each line is a single-line canonical ROKON `Stone { .. }`; blank lines are ignored.

```rokon
Stone { name: "geom", vers: "0.3.1", deps: [{ name: "num", req: "^1.2", kind: "normal" }], source: Dep::git("https://github.com/someone/geom", rev: "4f0c2a9e1b7d3c58a6e09f21d4b8c7a3e5f60123"), cksum: "9a1e7c3b5d2f4a6081c9e3b7d5f2a4c6e8b0d1f3", yanked: false }
```

| Field    | Meaning |
|----------|---------|
| `name`   | Lowercase `[a-z0-9_-]+`, starting with a letter; used in code with `-` as `_` |
| `vers`   | Semver (`major.minor.patch[-pre][+build]`) |
| `deps`   | Dependency stones (`name`, `req`, `kind` normal/dev, optional `registry`); `[]` if none |
| `source` | `Dep::git("<url>", rev: "<40-char commit>")`; repo root is the package |
| `cksum`  | Git tree hash of that commit; `rok` compares it after checkout |
| `yanked` | If `true`, new resolutions skip it (pinned versions in `rok.lock` keep working) |
| `description`, `license`, `repository` | Optional, from the manifest |

Stones depend only on stones. A package with `Dep::path` / `Dep::git` dependencies cannot be published; path/git dev dependencies are left out of the entry.

## Publish / yank

Publishing goes through the server named by `api`, with a rokstones.io publish token (not a GitHub token). Log in with GitHub at <https://rokstones.io/me> to create one. For now only admins get publish tokens; everyone else sends the line as a pull request.

```
rok login                      # save a publish token (read from stdin)
rok publish                    # print the entry line (dry run)
rok publish --commit           # send to the server, which commits and pushes it here
rok yank --vers 0.3.1 [name]   # set yanked: true (--undo: revert)
rok logout                     # forget the token
```

`rok publish` builds the entry from the current package's `rok.rokon`, the `origin` remote URL, and the `HEAD` commit and its tree hash. The work tree must be clean, the package must be the repo root, and an existing version is an error. The first account to publish a name owns it. The ROKON API (`Authorization: Bearer rks_..`) is `GET /api/v1/me`, `POST /api/v1/publish`, `POST /api/v1/yank`.

Without a token, add the line yourself and open a pull request: `rok publish --index <checkout> --commit` commits to a local checkout of a fork and pushes nothing.

## Usage

`rok` keeps a bare clone of this repository in `$ROK_HOME/cache/registry/db/`. The index location is `ROK_REGISTRY`, else `Config { registry: "<url>" }` in `$ROK_HOME/config.rokon`, else this repository's GitHub URL. For a single dependency from another index, write `Dep::version("^1.2", registry: "<url>")`.
