# mesa-mirror

Read-only mirror of [Mesa](https://gitlab.freedesktop.org/mesa/mesa) (branches and tags), so CI can clone Mesa from GitHub instead of gitlab.freedesktop.org.

This `mirror-sync` branch is the default branch and holds only the sync tooling. All other branches and tags are copied straight from upstream. Don't commit to them, because the next sync overwrites them.

## Using it in CI

Redirect the upstream URL before cloning submodules. No changes to `.gitmodules` are needed:

```bash
git config --global url."https://github.com/Lange-Studios/mesa-mirror".insteadOf https://gitlab.freedesktop.org/mesa/mesa
git submodule update --init --recursive --depth 1
```

## How the sync works

[`.github/workflows/sync.yml`](.github/workflows/sync.yml) runs daily at 04:17 UTC, the hour with the fewest merges to Mesa's `main`. Start it by hand from the Actions tab (`workflow_dispatch`) when you need a newer commit before then, for example right after godot-nir-static bumps its Mesa pin:

1. Seeds the existing history from this repo, so only new objects come from upstream.
2. Fetches branches and tags from freedesktop, trying up to 5 times. If freedesktop can't be reached, it syncs branches from Chromium's mirror (`chromium.googlesource.com/external/gitlab.freedesktop.org/mesa/mesa`) and leaves tags as they are.
3. Skips about 66 malformed CVS-era annotated tags (such as `R300_DRIVER_0` and `mesa_3_1`). Their tagger line has no space before the date, or no date at all. Git accepts them, but GitHub rejects any push that contains them. No commits are affected.
4. Pushes with `--prune` over SSH using the `MESA_MIRROR_DEPLOY_KEY` secret, whose public half is a write-enabled deploy key. A deploy key is needed because Mesa contains `.github/workflows/` files, and the built-in `GITHUB_TOKEN` isn't allowed to push those.
5. Disables any workflow other than the sync (for example Mesa's own `macos.yml`) and cancels any runs those workflows started.
