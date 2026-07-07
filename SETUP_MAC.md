# Set up the Opkald Warp fork on macOS (with `?active=true` patch)

This is a hand-off guide. Paste it into Claude in a fresh terminal session on
your Mac and Claude can walk through it step by step. You can also follow it
manually — every command is runnable as-is.

## What you're testing

Symphony fires `warp://launch/<config>?active=true` URIs when it wants to drop a
new tab into Warp. Stock Warp ignores `?active=true` for URI-launched configs
and always opens a brand-new window. Our fork (`feature/launch-uri-active-window`
on `git@github.com:Opkald-AI/warp.git`) changes two things in `app/src/`:

1. `uri/mod.rs` — parses `?active=true` off the URL.
2. `root_view.rs` — when that flag is set, attach the tab to the focused
   workspace; if no window is focused, fall back to any existing workspace
   instead of spawning a new window.

You will:

1. Build the fork as a **local dev channel** of Warp (`WarpLocal.app`, URL scheme
   `warplocal://`). This coexists with the official Warp install — you do **not**
   need to uninstall it.
2. Point your local Symphony at the `warplocal` scheme.
3. Run three test scenarios to confirm the patch behaves as expected.

## Prerequisites

- macOS (Apple Silicon or Intel both work)
- ~40 GB free disk (debug build target dir is large)
- ~30–60 min for the first compile — start it before lunch
- Xcode (full app from the App Store, not just CLT — required by Warp's bootstrap script)
- Homebrew
- An SSH key linked to a GitHub account with read access to `Opkald-AI/warp`

If you don't already have them, install via:

```bash
# Xcode: install from the App Store, then accept the licence
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch

# Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## 1. Clone the fork

Pick a parent folder for the checkout (Symphony's repos live alongside it on
Christian's machine; pick wherever you keep code).

```bash
cd ~/code           # or wherever
git clone git@github.com:Opkald-AI/warp.git opkald-warp
cd opkald-warp
git fetch origin feature/launch-uri-active-window
git checkout feature/launch-uri-active-window
```

Confirm you're on the right branch with the right tip:

```bash
git log --oneline -1
# Should reference the active-window URI change.
git diff master -- app/src/uri/mod.rs app/src/root_view.rs
# Should show the small +/- in those two files.
```

## 2. Run the bootstrap script (one-time)

This installs Rust, brew dependencies (`jq`, `pkgconf`, `llvm`, `clang-format`,
`create-dmg`, `multitime`, `powershell`, `sentry-cli`), `cargo-bundle`, and the
`aarch64-apple-darwin` target. It will prompt for `sudo` once and for
`gcloud auth login` near the end.

```bash
./script/bootstrap
```

If `gcloud` auth fails or you don't have access to the GCP project, that's
fine — codesigning will fall back to ad-hoc (`-`), which is what we want for a
local dev build anyway. You can skip / cancel the gcloud step.

After bootstrap finishes, **start a new terminal session** so `cargo` is on your
PATH, then `cd` back into the repo.

## 3. Build and launch the local dev build

Use the wrapper script — it builds, bundles into a real `.app`, patches the
`Info.plist` to register the `warplocal://` URL scheme, ad-hoc codesigns, and
launches the result.

```bash
./script/run
```

First build is slow (~20–40 min). Subsequent runs are incremental and much
faster.

When it's done, you should have:

```
target/debug/bundle/osx/WarpLocal.app
```

…and a fresh Warp window will open. Notice the bundle name (`WarpLocal`) and
icon variant — that's how you tell it apart from your stable Warp.

## 4. Confirm the `warplocal://` scheme is registered

In any terminal (the new WarpLocal one is fine):

```bash
# Should print a path containing WarpLocal.app
/System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/LaunchServices.framework/Versions/A/Support/lsregister \
  -dump | grep -A 2 "warplocal:" | head
```

Test fire from the shell:

```bash
# Make a tiny launch config first
mkdir -p ~/.warp/launch_configurations
cat > ~/.warp/launch_configurations/smoke_test.yaml <<'YAML'
name: smoke_test
windows:
  - tabs:
      - title: smoke
        layout:
          cwd: ~
YAML

open "warplocal://launch/smoke_test"
open "warplocal://launch/smoke_test?active=true"
```

The first command opens a new window. The second should drop a tab into the
focused WarpLocal window instead of opening another one — that's the patch
working.

## 5. Test plan (the actual thing to verify)

Run these three scenarios in order and watch the behaviour. Each starts with
WarpLocal already running.

### Scenario A — focused window, `?active=true`

1. Click the WarpLocal window to make it focused.
2. From any other terminal: `open "warplocal://launch/smoke_test?active=true"`
3. **Expect:** a new **tab** appears in the focused WarpLocal window.
4. **Regression check:** no new window opened.

### Scenario B — unfocused window, `?active=true` (the bug we're fixing)

1. Click any non-Warp app (Finder, Safari, anything) so WarpLocal is **not** the
   frontmost app, but its window still exists.
2. `open "warplocal://launch/smoke_test?active=true"`
3. **Expect:** the tab attaches to the existing WarpLocal window (the fallback
   path added in `root_view.rs`).
4. **Pre-patch behaviour for comparison:** stock Warp would have opened a new
   window here. Our patch should not.

### Scenario C — `?active=true` omitted (existing behaviour preserved)

1. With WarpLocal running and focused: `open "warplocal://launch/smoke_test"`
2. **Expect:** a brand-new window opens, exactly like stock Warp.
3. This proves we didn't change the default URI-launch path.

If all three behave as described, the patch is good.

## 6. Wire Symphony up to the dev build

Symphony reads `warp_url_scheme` from its config (default `warp`). Point it at
`warplocal` so it fires URIs at your dev build instead of the official Warp.

In your Symphony config (wherever you set `warp_launch_configs_dir` etc.):

```yaml
warp_url_scheme: warplocal
```

Or set it via env var if Symphony picks that up in your setup. Then run a
Symphony flow that opens a launch config with `active_window=True`
(`symphony/warp.py:open_launch_config` and friends construct the URI with
`?active=true` when called that way) and verify the tab lands in your focused
WarpLocal window.

## 7. Iterating on changes

If you make local edits to the Rust code:

```bash
./script/run
```

…rebuilds incrementally and relaunches. If the URL scheme stops working after
a rebuild (rare), re-register Launch Services for the bundle:

```bash
/System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/LaunchServices.framework/Versions/A/Support/lsregister \
  -f target/debug/bundle/osx/WarpLocal.app
```

## 8. Tearing it back down

Nothing on your system is persistently modified outside `~/code/opkald-warp`
and the `WarpLocal.app` bundle inside it. To revert:

- Quit WarpLocal.
- Change Symphony's `warp_url_scheme` back to `warp` (or remove the override).
- Optional: `rm -rf ~/code/opkald-warp` and `rm -rf target/` inside it to free
  disk.

Your stable Warp install is untouched.

## Troubleshooting

- **`cargo: command not found` after bootstrap** — open a new terminal session,
  then `cd` back into the repo. `rustup`/`cargo` only get added to PATH for new
  shells.
- **Build hits a linker error mentioning `Sentry`** — the bootstrap brew step
  didn't finish; rerun `./script/bootstrap`.
- **`open "warplocal://..."` opens the *stable* Warp** — the dev build's
  Launch Services registration didn't stick. Run the `lsregister -f` command
  from section 7, then retry.
- **`?active=true` still opens a new window** — confirm with
  `git log --oneline -1` that you're on `feature/launch-uri-active-window` and
  not on `master`. The patch only exists on the feature branch.
- **gcloud auth prompt in `./script/bootstrap` fails** — safe to skip; it's only
  needed for production codesigning, not local dev.

## File map (so Claude knows where things live)

| Path | Why it matters |
|------|---------------|
| `app/src/uri/mod.rs` | Parses `?active=true` from the URL. |
| `app/src/root_view.rs` | `open_launch_config` — focused-window fallback. |
| `script/run` → `script/macos/run` | Build + bundle + register URL scheme + launch. |
| `script/update_plist` | Inserts `CFBundleURLSchemes` with `$WARP_SCHEME_NAME`. |
| `target/debug/bundle/osx/WarpLocal.app` | The dev build that Launch Services sees. |
