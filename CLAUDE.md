# CLAUDE.md

Guidance for Claude Code in this repository.

## Push family context

Shared facts for all Push repos (repo map, git identity, `core/` pinning, cross-repo hardware facts):

@~/.claude/push-family.md

## Project

`keyboard-visualizer` — on-screen piano keyboard for Push 3, driven by the
post-transform notes Live sends to the "Keyboard Viz In" ALSA port. A catalog
hack for [ableton-push-hack](https://github.com/federico-pepe/ableton-push-hack).
Full description, build and deploy steps: [README.md](README.md).

## Rules

- Display-owning hack: draw only through push-manager's `/api/display/*`
  (`core/pmclient`). Never mmap shm. Keep the `/api/display/status`
  dependency watcher.
- Screen strings are ASCII only.
- `core` comes from GitHub, pinned in `src/go.mod`. No local `replace`.
- No `/dev/snd` access before `alsaseq.WaitForBootSettle()`.
- Build: `make` (linux/amd64). Release: bump `hack.json` `version`, push a
  matching `vX.Y.Z` tag; `.github/workflows/release.yml` builds and writes
  `release.json`.
