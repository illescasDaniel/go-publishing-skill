# go-publishing

Publish a Go module or CLI the safe way, and have your AI agent set it up for you. The skill adds a tag-triggered
GitHub Actions release workflow (CHANGELOG check, tests, GoReleaser binaries, GitHub release with your notes), CI,
and walks the agent through preparing the module, getting it onto [pkg.go.dev](https://pkg.go.dev) and each release.

It is an [Agent Skill](https://agentskills.io) (a `SKILL.md` plus workflow templates), so it works in Claude Code,
Cursor and any other agent that loads skills.

## How it works

| Step | What happens |
|---|---|
| **Prepare** | Module path matches the repo URL, license, package/command doc comments for pkg.go.dev, a version that `go install` reports correctly, templates added |
| **One-time setup** | Nothing to register: Go has no upload step and `GITHUB_TOKEN` creates the release |
| **Release** | Add the changelog entry, then (after your go-ahead) push a `vX.Y.Z` tag; the workflow does the rest |
| **Index** | Ask the module proxy for the tag so pkg.go.dev shows it within minutes, then verify the release and `go install` |

The release workflow (`release.yml`) refuses to create the GitHub release if the tag has no `CHANGELOG.md` entry or
the tests fail. Because the tag itself publishes the module, the agent checks both, and waits for green CI, before
tagging. It never tags, pushes or publishes without your explicit go-ahead for that version, and never moves a tag.

What the templates give you:

- `release.yml`: release on tag push, as above
- `ci.yml`: gofmt, `go mod tidy` check, build, vet and test on push and PRs
- `.goreleaser.yaml`: linux/macOS/Windows binaries for amd64 and arm64, with checksums
- `scripts/release_notes.sh`: prints a changelog section for a tag
- `CHANGELOG.md`: the format `release_notes.sh` expects

## Install

### Any agent: the `skills` CLI

```bash
npx skills add illescasDaniel/go-publishing-skill
```

Non-interactive: `npx skills add illescasDaniel/go-publishing-skill -a claude-code -y`, plus `-g` for global.

### By hand

Copy the `skills/go-publishing` folder into your agent's skills directory.

| | Project (shared with your team) | All your projects |
|---|---|---|
| Codex | `.codex/skills/go-publishing/` | `~/.codex/skills/go-publishing/` |
| Claude Code | `.claude/skills/go-publishing/` | `~/.claude/skills/go-publishing/` |
| Cursor | `.cursor/skills/go-publishing/` | `~/.cursor/skills/go-publishing/` |

## Contributing

Issues and pull requests are welcome. The skill is deliberately short: if a change makes `SKILL.md` longer, it should
earn it.

## License

MIT, see [LICENSE](LICENSE).
