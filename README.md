# packages-index

[![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue)](LICENSE-MIT)

*Clear rules, free as a butterfly.*

The index of [packages.nabi-lang.org](https://packages.nabi-lang.org), the Nabi package registry. As with crates.io-index, this git repository is the registry itself. Each version points to a commit in the package's own git repository. The registry keeps an archive of that commit when the version is published (`GET /api/v1/packages/<name>/<version>/download`) and writes its SHA-256 into the line as `archive`; `nabi` downloads the archive and checks it against that, so a version outlives its repository. Only for a version without `archive` does `nabi` fetch the commit from git, verified by its tree hash.

Lines are written only by the registry: `nabi publish` sends a version to the registry (<https://packages.nabi-lang.org>), which checks it and commits it here. Pull requests that add or change lines are not accepted. The `main` branch is protected: only the registry's own identity may push to it, never force-push, and it cannot be deleted; maintainers change the repository's other files through pull requests to a branch the registry merges.

Website / API: <https://packages.nabi-lang.org>

## Layout

```text
config.nabion          registry configuration and line format
1/<name>              packages with 1-character names
2/<name>              2 characters
3/<c>/<name>          3 characters (<c>: first character)
<ab>/<cd>/<name>      4 or more characters (<ab>: chars 1-2, <cd>: chars 3-4)
```

`-` stays in the path (`nabi-syntax` -> `na/bi/nabi-syntax`). Names differing only in `-` / `_` or case are one package, so a second spelling cannot be published.

`config.nabion` holds `Config { api: "https://packages.nabi-lang.org", format: 1 }`. `api` is the server `nabi` publishes through. It must be `https://` (only `http://127.0.0.1` and `http://localhost` are exempt). `format` is the version of the package line format (below).

## Format

`format` is the version of the package line format below; a change to the line schema raises it. A client or server can read `format` before any package line and stop with a clear message when it does not know it, instead of failing on a line.

## Package files

One file per package, one line per version (publish order, oldest first). Each line is a single-line canonical NABION `Package { .. }`; blank lines are ignored.

```nabion
Package { name: "geom", version: "0.3.1", deps: [{ name: "num", req: "^1.2", kind: "normal" }], source: Dep::git("https://github.com/someone/geom", rev: "4f0c2a9e1b7d3c58a6e09f21d4b8c7a3e5f60123"), tree: "9a1e7c3b5d2f4a6081c9e3b7d5f2a4c6e8b0d1f3", archive: "5c1f0e9d8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d", yanked: false, nabi_version: "0.1.0" }
```

| Field    | Meaning |
|----------|---------|
| `name`   | Lowercase `[a-z0-9_-]+`, starting with a letter, at most 64 characters; used in code with `-` as `_` |
| `version` | Semver (`major.minor.patch[-pre][+build]`) |
| `deps`   | Dependencies: `name`, `req`, `kind` (`normal`, `dev` or `native`), and where needed `registry`, `optional`, `features`, `default_features`, `package` (the package a renamed dependency names) and `target` (a triple or `cfg(..)`); `[]` if none |
| `source` | `Dep::git("<url>", rev: "<40-char commit>")`: an https address on a public host (no IP address, port or user name), the commit on a branch or tag of that repository; repo root is the package |
| `tree`   | Git tree hash of that commit (40 hex digits, as `rev`); `nabi` compares it after checkout |
| `archive` | Written by the registry, never sent by `nabi publish`: SHA-256 (lowercase hex) of the source archive it keeps (`git archive` of the commit, paths under `<name>-<version>/`); `nabi` refuses a download that does not match |
| `yanked` | If `true`, new resolutions skip it (pinned versions in `nabi.lock` keep working) |
| `features` | Optional, the manifest's features |
| `nabi_version` | The oldest toolchain that builds the version, `x.y.z` (the manifest's `nabi_version`); it admits that toolchain and the later ones of its series: `"0.1.3"` is nabi 0.1.3 to 0.1.x, `"1.4.2"` is 1.4.2 to 1.x. The registry refuses a line without it |
| `links` | Optional, the native library the manifest links |
| `official` | Optional, `true` on a version published from the nabi-lang organization; absent means `false` |

Fields come in the order above; optional ones are left out when unset. `official: true`, when present, is always the last field:

```nabion
Package { name: "json", version: "0.1.0", deps: [], source: Dep::git("https://github.com/nabi-lang/json", rev: "<40-char commit>"), tree: "<40-char tree>", archive: "<64-char sha-256>", yanked: false, nabi_version: "0.1.0", official: true }
```

`official` is written only by the registry, on a version published by a trusted publisher in a repository of the [nabi-lang](https://github.com/nabi-lang) organization; a publish that sends it is refused, and `official: false` is never written. It is a bool: a line where it is anything else is malformed. Each version keeps the flag it was published with, so whether a package is official is read from its latest version's line: the highest version that is not yanked, a release over a pre-release (as the site and `nabi` do, the latter to add an official package on `use`). `nabi component add packages` keeps the archives of the official packages' newest versions that its toolchain builds (their `nabi_version` admits it) for builds without a network.

Metadata (description, license, repository, readme) is kept by the server, not in the line.

## Standard library entries

`std`, `core` and `bare`, the standard library's tiers, ship with the toolchain and are never versioned packages. Each has a file here with one line, which the registry shows as an official entry owned by the nabi-lang organization, with a page that links the library's API docs:

```nabion
Library { name: "std", official: true }
```

No version can be published under these names, and `nabi` skips these lines when it resolves. A manifest that lists one as a dependency is an error (`std ships with the toolchain; remove this line`).

A package in the index depends only on packages in an index. A package with `Dep::path` / `Dep::git` dependencies cannot be published; path/git dev dependencies are left out of the entry.

## Seeding

The index starts with `config.nabion` and the three `Library` lines of `std`, `core` and `bare`, committed by hand before the registry first serves. They are needed: the registry refuses a version under a name only when that name's file holds a `Library` line, and `nabi` skips those lines when it resolves. Nothing else is written by hand.

Every `Package` line comes from the registry. The nabi-lang packages are published by the `publish.yml` workflow of each repository, on its `v*` tag, in the order `lib/publish-order.sh` in [nabi-lang/.github](https://github.com/nabi-lang/.github) prints, so a dependency is in the index before the package that needs it. Each needs its trusted publisher on the registry first: the repository, `publish.yml` and the `release` environment.

## Publish / yank

Publishing goes through the server named by `api`, with a registry publish token (not a GitHub token) or from GitHub Actions through trusted publishing. Log in with GitHub at <https://packages.nabi-lang.org/me> to create a token. Publishing is not open to everyone yet: for now only the registry's maintainers get publish tokens and trusted publishers.

```
nabi login                      # save a publish token (read from stdin)
nabi publish --dry-run          # check the entry line
nabi publish                    # send it to the server, which commits and pushes it here
nabi yank <name>@0.3.1          # set yanked: true (--undo: revert)
nabi logout                     # forget the token
```

`nabi publish` builds the entry from the current package's `nabi.nabion`, the `origin` remote URL, and the `HEAD` commit and its tree hash. The work tree must be clean, the package must be the repo root and have a library (`src/lib.nabi`), an executable (`src/main.nabi`) or both, and an existing version is an error. A package with an executable is installed with `nabi install <name>`; the registry's pages say which a package has. The first account to publish a name owns it; the standard library's names (`std`, `core`, `bare`) belong to its entries above; reserved names (`nabi`, `nabic`, `prelude`, ...) and names that look like one already here are refused unless the admins approve them (see the [usage policy](https://packages.nabi-lang.org/policy)). The NABION API (`Authorization: Bearer pkg_..`) is `GET /api/v1/me`, `POST /api/v1/publish`, `POST /api/v1/yank`.

`nabi publish --index-dir <dir>` (and `nabi yank --index-dir <dir>`) writes the line as a commit in a local index checkout and pushes nothing. It is for an index of your own (a private or offline registry, tests), not a way into this one.

## Usage

`nabi` keeps a bare clone of this repository in `$NABI_HOME/cache/registry/db/`. A bare requirement (`"^1.2"`) resolves against this index; `Sources { replace: { "<index>": "<mirror>" } }` in `$NABI_HOME/sources.nabion` fetches it from a mirror instead, and the lock still names this index. For a single dependency from another index, write `Dep::version("^1.2", registry: "<url>")`.
