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
| Go version | `1.27` (see `go.mod`) |
| Binary | `gost` |

## Build & test

```bash
go build ./...                   # build all packages
go vet ./...
gofmt -l .                       # keep the tree formatted, it is clean today
go test ./cmd/gost/...           # unit tests (version resolution)
go test ./tests/e2e/...          # integration tests, they need Docker (testcontainers)
make linux-amd64                 # cross-compile into bin/ via the Makefile
goreleaser check                 # validate the release config
goreleaser release --snapshot --clean --skip=publish   # local build of all targets
```

`cmd/gost` has a small unit-test suite; there is no test CI job. `tests/e2e/`
drives the built binary in containers, takes about 12 minutes, and has one
known pre-existing failure (see below).

## Following upstream

`o` is the upstream remote (`go-gost/gost`); `origin` is the fork. To cut a
fork release from a new upstream release:

1. `git fetch --no-tags o && git fetch o --tags`, then pick the newest stable
   tag. Nightly tags often point at the same commit as the release (`v3.3.3`
   and `v3.3.4-nightly.20261102` were both `3d514c5`).
2. `git rebase <tag>`. Expect modify/delete conflicts on the files the fork
   deliberately removed (`.github/workflows/buildx.yml`,
   `.github/workflows/trigger-nightly.yml`, `Dockerfile`); resolve them by
   keeping the deletions (`git rm`).
3. `cmd/gost/version.go` conflicts: take upstream's implementation and set
   `buildVersionFallback` to the fork version (`X.Y.Z-N`).
4. Squash all fork commits into one commit on top of the upstream tag:
   `git reset --soft <tag>`, then commit as `release vX.Y.Z-N`.
5. Keep the `go` directive in `go.mod` on the latest stable Go (check
   `https://go.dev/dl/?mode=json`); the release workflow reads it through
   `go-version-file`.
6. Verify locally: `go build ./...`, `go vet ./...`, `go test ./cmd/gost/...`,
   and optionally `go test ./tests/e2e/...` (Docker required, ~12 minutes).
7. `git push --force-with-lease origin master`, tag `vX.Y.Z-N`, push the tag,
   watch the goreleaser workflow, then confirm the release is not
   draft/pre-release and that the downloaded linux/amd64 binary reports
   `gost X.Y.Z-N`.

## Release

Only `.github/workflows/release.yml` (GoReleaser) is kept, and it runs on tag
pushes:

1. bump `buildVersionFallback` in `cmd/gost/version.go`, commit on `master`,
2. tag `vX.Y.Z-N` and push it — the tag is the release trigger.

Details that are easy to get wrong:

- **The tag is the version.** `.goreleaser.yaml` builds with
  `-X 'main.version={{ .Tag }}'`, so `gost -V` prints the tag, e.g.
  `gost 3.3.3-1` (the leading `v` is stripped). `cmd/gost/version.go` only
  affects local builds through the `buildVersionFallback` const; bump both so
  they agree.
- **`project_name: gost`** is set on purpose: the repository directory is
  `gost3`, and without the override GoReleaser names the archives *and the
  binary* `gost3_*` / `gost3`, which contradicts the Makefile, the README and
  `gost.service`.
- **`release.prerelease: false`** on purpose: `v3.3.3-1` is a semver
  pre-release, so `auto` would hide the release from `/releases/latest`.
- **Keep an older tag on the remote** (e.g. `v3.3.0`). GoReleaser derives the
  changelog from the previous tag; with no tag at all the release notes fall
  back to an arbitrary range of old commits.
- GoReleaser v2 refuses configuration files without `version: 2`.
- The fork has no `DOCKER_*` secrets and does not publish images; the docker
  and nightly workflows were removed.

## Known issues and gotchas

- **Never run `go get -u ./...`.** It upgrades indirect dependencies past the
  versions pinned by `go-gost/x` and breaks the build: the latest
  `gvisor.dev/gvisor` has a `stack`/`bridge` package clash, `gobwas/glob`
  v1.1.0 removed the `glob.Glob` type used by `x/internal/matcher`, and
  quic-go v0.65 no longer matches `webtransport-go` v0.13. Follow upstream's
  versions instead. Test-only direct dependencies (`moby/moby/client`,
  `testcontainers-go`) can be bumped on their own; they do not affect the
  binary.
- **`gost -V` strips the leading `v`** from the injected tag (`cleanVersion`
  in `cmd/gost/version.go`), so it prints e.g. `gost 3.3.3-1`, not
  `gost v3.3.3-1`.
- **Known e2e failure.** `TestForwardSuite/TestTCPForwardSniffingBypass`
  fails at least since `v3.3.1`: curl gets an empty reply instead of the
  expected 403 from the sniffer bypass. It is not caused by rebases or
  dependency bumps; check it against the tag being rebased from before
  investigating.
- **Overwriting a release.** Delete the existing GitHub release first
  (`gh release delete <tag> --yes`, leaving the tag), then `git tag -f <tag>`
  and `git push --force origin <tag>`. With `release.mode: keep-existing`,
  GoReleaser keeps the old notes and assets, so a clean overwrite needs the
  release to be gone before the workflow runs.

## Dependencies

- `github.com/go-gost/core` — interfaces
- `github.com/go-gost/x` — implementations, config, registry
- `github.com/judwhite/go-svc` — OS service framework (daemon, SIGHUP/SIGTERM)

Bumping `x` is the usual way to pick up new features; the version lives in
`go.mod` only.
