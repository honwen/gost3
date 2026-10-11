# AGENTS.md

Guidance for AI agents (and humans) working in this repository.
For the internals of the `gost` module see [CLAUDE.md](CLAUDE.md).

## Overview

This repository is the **gost3 fork** (`honwen/gost3`) of
[go-gost/gost](https://github.com/go-gost/gost) — GOST v3, a multi-protocol
tunnel/proxy. `master` tracks upstream; releases of the fork are cut from
`master` with the fork's own revision suffix (`vX.Y.Z-N`).

| Field | Value |
|-------|-------|
| Module path | `github.com/go-gost/gost` (unchanged from upstream) |
| Go version | `1.26` (see `go.mod`) |
| Binary | `gost` |

## Build & test

```bash
go build ./cmd/gost/...          # build the CLI
go vet ./...
gofmt -l .                       # keep the tree formatted, it is clean today
go test ./tests/e2e/...          # integration tests, they need Docker (testcontainers)
make linux-amd64                 # cross-compile into bin/ via the Makefile
goreleaser check                 # validate the release config
goreleaser release --snapshot --clean --skip=publish   # local build of all targets
```

There is no unit-test suite and no test CI job: `tests/e2e/` drives the built
binary in containers and is too heavy to run on every push.

## Release

Only `.github/workflows/release.yml` (GoReleaser) is kept, and it runs on tag
pushes:

1. bump `cmd/gost/version.go`, commit on `master`,
2. tag `vX.Y.Z-N` and push it — the tag is the release trigger.

Details that are easy to get wrong:

- **The tag is the version.** `.goreleaser.yaml` builds with
  `-X 'main.version={{ .Tag }}'`, so `gost -V` prints the tag, e.g.
  `gost v3.3.0-1`. `cmd/gost/version.go` only affects local builds; bump both
  so they agree.
- **`project_name: gost`** is set on purpose: the repository directory is
  `gost3`, and without the override GoReleaser names the archives *and the
  binary* `gost3_*` / `gost3`, which contradicts the Makefile, the README and
  `gost.service`.
- **`release.prerelease: false`** on purpose: `v3.3.0-1` is a semver
  pre-release, so `auto` would hide the release from `/releases/latest`.
- **Keep an older tag on the remote** (e.g. `v3.3.0`). GoReleaser derives the
  changelog from the previous tag; with no tag at all the release notes fall
  back to an arbitrary range of old commits.
- GoReleaser v2 refuses configuration files without `version: 2`.
- The fork has no `DOCKER_*` secrets and does not publish images; the docker
  and nightly workflows were removed.

## Dependencies

- `github.com/go-gost/core` — interfaces
- `github.com/go-gost/x` — implementations, config, registry
- `github.com/judwhite/go-svc` — OS service framework (daemon, SIGHUP/SIGTERM)

Bumping `x` is the usual way to pick up new features; the version lives in
`go.mod` only.
