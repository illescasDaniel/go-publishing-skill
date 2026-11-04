---
name: go-publishing
description: Set up and run publishing for a Go module or CLI with a GitHub Actions release workflow (semver tag push, tag/CHANGELOG check, tests, GoReleaser binaries, GitHub release with the CHANGELOG notes) plus CI, so it appears on pkg.go.dev. Use when asked to publish or release a Go package/module/CLI, get it on pkg.go.dev, cut a Go release or bump a version, add a release workflow or GoReleaser, or fix a package missing from pkg.go.dev.
---

# Go publishing

Go has no upload step. **The tag is the release:** the module proxy (and so `go install` and pkg.go.dev) serves
whatever `vX.Y.Z` points at, and once fetched it never changes. The workflow runs on a pushed `v*` tag, refuses to
build binaries or create the GitHub release unless the tag has a `CHANGELOG.md` entry and the tests pass; it then
runs GoReleaser and uses that entry as the release notes. It cannot gate the module itself: the tag is public once
pushed, so the real checks happen **before** tagging (step 3).

Templates are in `templates/` (copy them, replace `{{COMMAND}}` and `{{CMD_PATH}}`; leave GoReleaser's own `{{.Version}}` as is):

| Template | Goes to |
|---|---|
| `.github/workflows/release.yml` | same path. Runs on `v*` tag push |
| `.github/workflows/ci.yml` | same path. gofmt, build, vet, test on push and PRs |
| `.goreleaser.yaml` | same path. Skip it for a library with no `main` package; use the `gh release create` step noted in `release.yml` |
| `scripts/release_notes.sh` | same path. Prints the `CHANGELOG.md` section for a tag |
| `CHANGELOG.md` | same path. `## x.y.z` sections, newest first |

Action versions are pinned by commit SHA with the version in a comment. Keep it that way when updating them.

## 1. Prepare the module

- `go.mod` module path **equals the public repo URL** (`github.com/<owner>/<repo>`). From v2 on the path needs a
  `/v2` suffix and the tag `v2.x.y`; do not tag v2 without it.
- Public repo with a `LICENSE` that pkg.go.dev recognises (MIT, Apache-2.0, BSD...). Without one it hides the docs.
- Docs: a `// Package x ...` comment on each importable package, and for a CLI a `// Command x ...` comment above
  `package main` (usage, install line, env vars, flags). That text is the pkg.go.dev page. Add a README badge:
  `[![Go Reference](https://pkg.go.dev/badge/<module>.svg)](https://pkg.go.dev/<module>)`.
- CLI version: do not hardcode it. Use `var version = "dev"`, set by GoReleaser's ldflags
  (`-X main.version={{.Version}}`), and when it is still `"dev"` fall back to
  `debug.ReadBuildInfo().Main.Version` (skip it if empty or `(devel)`), because `go install ...@vX.Y.Z` does not run
  ldflags. ldflags give `1.2.3`, `go install` gives `v1.2.3`: trim the `v` if the output must match.
- Put the CLI under `cmd/<name>` so `go install <module>/cmd/<name>@latest` works.
- Copy the templates, fill the placeholders, write the first `CHANGELOG.md` entry.
- `go.mod` must have no `replace` or `exclude` directives, or `go install <module>/cmd/<name>@vX.Y.Z` fails. Run
  `go mod tidy`. Add `dist/` to the project's `.gitignore`.
- Keep `release.yml`'s setup and test steps in sync with `ci.yml` (e.g. a tool CI installs first, like gopls).
- Check it builds: `go build ./... && go vet ./... && go test ./...`.

## 2. One-time setup

- No registry account, token or trusted publisher is needed. `GITHUB_TOKEN` creates the release, so the workflow
  only needs `permissions: contents: write`.
- Push `main` when authorized and wait for CI to be green.

## 3. Cut a release

1. Add the `## x.y.z` section to `CHANGELOG.md` (bump the version only if the code hardcodes it). Commit, push
   `main` when authorized, and check `scripts/release_notes.sh vX.Y.Z` prints the section. Wait until CI is green on
   that exact commit: `gh run list --workflow ci.yml --branch main --limit 1 --json headSha,conclusion` shows
   `headSha` equal to `git rev-parse HEAD` and `conclusion` `success`.
2. Obtain explicit authorization for that version: pushing the tag publishes it, and proxy versions cannot be
   changed or truly deleted. Existing authorization in the task counts; do not ask again.
3. `git tag vX.Y.Z && git push origin vX.Y.Z` (the tag only). The release workflow starts.
4. Watch it with `gh run watch <id> --exit-status`. A full test suite can take many minutes; the GitHub release
   appears only when it finishes.
5. Ask the proxy to index the tag so pkg.go.dev shows it within minutes (anyone running `go install` or visiting
   the page also triggers it):

```bash
cd "$(mktemp -d)" && GOPRIVATE= GONOPROXY= GOPROXY=https://proxy.golang.org go list -m <module>@vX.Y.Z
```

6. Verify: `gh release list` shows it as Latest with binaries attached, and
   `go install <module>/cmd/<name>@vX.Y.Z && <name> --version` prints the right version.

The tag is public once pushed, even if the run fails. Re-run (`gh run rerun <id> --failed`) only for transient
failures. If code or the CHANGELOG must change, fix `main` and release the next patch; never move the tag. If a bad
version went out, add a `retract vX.Y.Z` directive to `go.mod` and release a new patch version.

## Rules

- Never push a tag, publish or create a release without the user's explicit go-ahead for that version.
- Tag only a commit that is on `origin/main` with green CI and a CHANGELOG entry; check both before asking for the
  go-ahead.
- Never move or re-push an existing tag; release a new patch version instead.
- Never put tokens or secrets in the repo or a workflow; `GITHUB_TOKEN` is enough.
- Report outcomes faithfully: say a release run is still in progress rather than that it failed or is missing.
