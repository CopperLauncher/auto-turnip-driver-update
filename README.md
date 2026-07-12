# freedreno

Auto-synced fork of [`MojoLauncher/freedreno_turnip-CI`](https://github.com/MojoLauncher/freedreno_turnip-CI)
(itself forked from [`ilhan-athn7/freedreno_turnip-CI`](https://github.com/ilhan-athn7/freedreno_turnip-CI)) —
the script that builds the freedreno/Turnip Vulkan driver as a Magisk
module / AdrenoTools package, structured the way `MojoLauncher/LTW` is
structured (`.github/workflows` for automation, a clean root, artifacts
published via Releases).

**Nothing in `vendor/upstream/` is hand-copied.** A scheduled GitHub
Actions workflow pulls the latest files straight from
`MojoLauncher/freedreno_turnip-CI` every run, diffs them against what's
committed here, and only commits when upstream actually changed. When
MojoLauncher updates their script, this repo picks it up on the next
scheduled run (or on-demand via "Run workflow") — no manual copy/paste,
ever.

## How it works

```
.github/workflows/
  sync-upstream.yml   -> runs scripts/sync-upstream.sh on a schedule (and
                          on manual dispatch), commits changes to
                          vendor/upstream/ if the upstream repo moved
  build-driver.yml     -> runs after a successful sync (or manually),
                          executes the vendored turnip_builder.sh on a
                          fresh Ubuntu runner and publishes the resulting
                          driver package as a GitHub Release, same as
                          upstream's own release flow
  publish-to-copper-android.yml
                        -> runs after a successful build (or manually),
                          extracts libvulkan_freedreno.so from the
                          release and opens a PR against
                          CopperLauncher/Copper-Android replacing
                          app_pojavlauncher/src/main/jniLibs/arm64-v8a/libvulkan_freedreno.so.
                          This is the piece that means you never manually
                          swap that binary again — the only manual step
                          left is reviewing/merging the PR.

scripts/
  sync-upstream.sh     -> the actual sync logic (git-based, no manual
                          file copying — this is the only thing that
                          ever writes to vendor/upstream/)

vendor/
  upstream/            -> auto-populated mirror of upstream's tracked
                          files. Treat this directory as read-only —
                          anything you put here by hand will be
                          overwritten (and diffed away) on the next sync
  UPSTREAM_COMMIT.txt  -> the exact upstream commit SHA currently mirrored,
                          written by the sync script every run
```

## Why this exists instead of just cloning MojoLauncher/freedreno_turnip-CI

Cloning/forking directly means every upstream update has to be manually
pulled and merged. This setup instead treats upstream as a dependency:
the sync workflow is the *only* path files take into this repo, so
staying current is automatic and the history here cleanly shows exactly
when and what changed upstream (each auto-commit references the
upstream SHA it was pulled from).

## Setup

1. Upload this as a new repo (or `git init` it and push).
2. Nothing else needed — `sync-upstream.yml` runs on its own schedule
   (default: daily, matches upstream's release cadence loosely). You can
   also trigger it manually from the Actions tab any time you want an
   immediate check.
3. First run will populate `vendor/upstream/` and commit it. Every run
   after that only commits if something actually changed upstream.
4. `build-driver.yml` picks up from there and builds/publishes whenever
   `vendor/upstream/` changes.

## Getting the driver into Copper-Android

Copper-Android already has Turnip support wired up natively
(`egl_bridge.c`'s `load_turnip_vulkan()`, gated by `ADRENO_POSSIBLE` in
`Android.mk`, which is currently on). It loads a binary named
`libvulkan_freedreno.so` from the app's native lib dir at runtime, and a
copy of that file is already checked into
`app_pojavlauncher/src/main/jniLibs/arm64-v8a/libvulkan_freedreno.so`.
That means there's nothing to build in Copper-Android's Java/native code
— the only thing that goes stale is that one binary.

`publish-to-copper-android.yml` closes that gap: it extracts the freshly
built `.so` and opens a PR against Copper-Android replacing that exact
file. To enable it:

1. Create a fine-grained GitHub PAT with `contents: write` and
   `pull-requests: write` on `CopperLauncher/Copper-Android`.
2. Add it as a repo secret here named `COPPER_ANDROID_PAT`.
3. That's it — once `build-driver.yml` publishes a release, this
   workflow fires automatically and a PR shows up on Copper-Android for
   you to review and merge.

## Configuration

Edit the top of `scripts/sync-upstream.sh` to change:
- `UPSTREAM_REPO` — defaults to `MojoLauncher/freedreno_turnip-CI`
- `UPSTREAM_BRANCH` — defaults to `main`
- `SYNC_PATHS` — which files/dirs get mirrored (defaults to the build
  script + license + workflow files upstream ships)

## License

GPLv3, inherited from upstream `freedreno_turnip-CI` (see `LICENSE`).
Original authorship: [ilhan-athn7](https://github.com/ilhan-athn7),
maintained downstream by [MojoLauncher](https://github.com/MojoLauncher).
This repo only adds the sync/build automation — it claims no authorship
over the vendored driver-build script itself.
