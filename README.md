# rokstones.io-index

The package (**stone**) registry index for [rok](https://rokstones.io). As with crates.io-index, this git repository is the registry itself. There is no server hosting source tarballs: each version points to a commit in the stone's own git repository, and `rok` fetches that commit and verifies it by tree hash.

Website / API: <https://rokstones.io>

## Layout

```text
config.rokon          registry configuration
1/<name>              stones with 1-character names
2/<name>              2 characters
3/<c>/<name>          3 characters (<c>: first character)
<ab>/<cd>/<name>      4 or more characters (<ab>: characters 1-2, <cd>: characters 3-4)
```

`-` stays in the path (`rok-syntax` → `ro/k-/rok-syntax`). Names that differ only in `-` / `_` or case are one stone (`rok-syntax`, `rok_syntax`), so a second spelling cannot be published.

## `config.rokon`

```rokon
Registry {
    name: "rokstones.io",        // registry name
    dl: "git",                   // source policy: fetch from each version's `source` (git commit)
    api: "https://rokstones.io", // web / API
}
```

## Stone files

One file per stone, one line per version (in publish order, oldest first). Each line is a single-line canonical ROKON value `Stone { .. }`. Blank lines are ignored.

```rokon
Stone { name: "geom", vers: "0.3.1", deps: [{ name: "num", req: "^1.2", kind: "normal" }], source: Dep::git("https://github.com/someone/geom", rev: "4f0c2a9e1b7d3c58a6e09f21d4b8c7a3e5f60123"), cksum: "9a1e7c3b5d2f4a6081c9e3b7d5f2a4c6e8b0d1f3", yanked: false }
```

| Field    | Meaning                                                                                        |
|----------|------------------------------------------------------------------------------------------------|
| `name`   | Stone name: lowercase `[a-z0-9_-]+`, starting with a letter. Code names it with `-` as `_` (`use rok_syntax::..`) |
| `vers`   | Semver version (`major.minor.patch[-pre][+build]`)                                             |
| `deps`   | Dependency stones (below). `[]` if none                                                        |
| `source` | `Dep::git("<url>", rev: "<40-character commit>")`. The repository root is the package (`lib.rok`, `rok.rokon`) |
| `cksum`  | Git tree hash of that commit. `rok` compares it after checkout and errors on mismatch          |
| `yanked` | If `true`, new resolutions skip it (versions already pinned in `rok.lock` keep working)        |
| `description`, `license`, `repository` | Optional, from the manifest (`rok.rokon`), after `yanked` |

Each entry of `deps`:

| Field      | Meaning                                                                                     |
|------------|---------------------------------------------------------------------------------------------|
| `name`     | Dependency stone name                                                                       |
| `req`      | Version requirement (`^1.2`, `~1.2.3`, `=1.0.0`, `>=1.2, <2`, `1.*`, `*`; a bare version means `^`) |
| `kind`     | `"normal"` or `"dev"` (dev is for the stone's own tests and is not followed)                |
| `registry` | Optional. URL of another index. Defaults to this index                                      |

Stones depend only on stones. A package with `Dep::path` / `Dep::git` dependencies cannot be published, and path / git dev dependencies are left out of the entry.

## Publish / yank

There is no server; publishing means adding a commit to this repository.

```
rok publish                                          # print the entry line (dry run)
rok publish --index <checkout of this repo> --commit # append the line to the file and commit
rok yank --vers 0.3.1 --index <checkout> [name]      # set yanked: true and commit (--undo: revert)
```

`rok publish` builds the entry from the current package's `rok.rokon` (`name`, `version`, `deps`), the URL of git remote `origin`, and the `HEAD` commit and its tree hash. The work tree must be clean, the package must be the repository root, and an existing version is an error. Pushing the commit to `origin` is up to the publisher. Send the resulting index commit to this repository as a pull request.

## Usage

`rok` keeps a bare clone of this repository in `$ROK_HOME/cache/registry/db/` and reads from it. The index location is the `ROK_REGISTRY` environment variable, else `Config { registry: "<url>" }` in `$ROK_HOME/config.rokon`, else the default (this repository's GitHub URL). To use another index for a single dependency, write `Dep::version("^1.2", registry: "<url>")`.
