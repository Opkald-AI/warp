# CLAUDE.md - warp (Opkald-AI fork of warpdotdev/warp)

Upstream's agent guide is [WARP.md](WARP.md) (build, test, lint, architecture). It covers everything about
the codebase itself. This file only covers what's specific to the OpkaldAI fork.

## Why the fork exists

[opkald-symphony](https://github.com/Opkald-AI/opkald-symphony) opens a Warp tab per kanban autopilot task
with `warp://launch/<config>?active=true`. Stock Warp ignores `?active=true` and opens a new window for each
one. This fork makes those URIs stack as tabs in the running window, and adds a Symphony toggle to the UI.
It's a developer tool on Christian's Mac: not deployed, not part of the call-handling product.

## State (2026-10)

- Default branch: **`master`** (not `main`). It is upstream `master` as of the fork point plus two
  OpkaldAI commits, and ~2,370 commits behind upstream.
  - `922cc643` (#3): "Autopilot" button in the workspace top bar. Runs `symphony autopilot toggle` and
    reads `~/.symphony/control.json` for its on/off label (`app/src/workspace/view.rs`, `util.rs`).
  - `4dffab27` (#4): single-instance forwarding enabled without the `release_bundle` cfg, so dev/OSS builds
    deliver launch URIs to the running instance (`app/src/app_services/windows/mod.rs`, `app/src/lib.rs`).
- **Open PR #2** `feature/launch-uri-active-window`: the actual `?active=true` handling
  (`app/src/uri/mod.rs`, `app/src/root_view.rs`) plus `SETUP_MAC.md` (build `WarpLocal.app`, scheme
  `warplocal://`, test scenarios). Until it merges, build from that branch to get tab stacking.
- `feature/symphony-toggle` also carries `SETUP_MAC.md`.

## Commands

See WARP.md. Short version: `./script/bootstrap`, `cargo run`, `./script/presubmit`.
Not verified here (no Rust toolchain in this environment; the repo is ~5k files).

## Rules

- Keep OpkaldAI changes small and in their own commits so upstream syncs stay feasible.
- Symphony picks the URL scheme with `warp_url_scheme` in `~/.symphony/config.yaml`
  (`warp`, `warplocal`, `warposs`, `warpdev`). It must match the build you're running.
- Inherited upstream workflows (`.github/workflows/*`) target warpdotdev infrastructure and secrets.
  Don't expect them to work here.
