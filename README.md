# mesa-mirror-sync

Keeps [Lange-Studios/mesa-mirror](https://github.com/Lange-Studios/mesa-mirror), a read-only mirror of [Mesa](https://gitlab.freedesktop.org/mesa/mesa), in sync with upstream, so CI can clone Mesa from GitHub instead of gitlab.freedesktop.org.

The sync lives in this separate repo so Actions can stay **disabled** on `mesa-mirror`. Mesa ships its own GitHub workflows (such as `.github/workflows/macos.yml`), and with Actions off they never run in the mirror.

## Using the mirror in CI

Redirect the upstream URL before cloning submodules. No changes to `.gitmodules` are needed:

```bash
git config --global url."https://github.com/Lange-Studios/mesa-mirror".insteadOf https://gitlab.freedesktop.org/mesa/mesa
git submodule update --init --recursive --depth 1
```

## How the sync works

[`.github/workflows/sync.yml`](.github/workflows/sync.yml) runs daily at 04:17 UTC, the hour with the fewest merges to Mesa's `main`. Start it by hand from the Actions tab (`workflow_dispatch`) when you need a newer commit before then, for example right after godot-nir-static bumps its Mesa pin:

1. Seeds the existing history from `mesa-mirror`, so only new objects come from upstream.
2. Fetches branches and tags from freedesktop, trying up to 5 times. If freedesktop can't be reached, it syncs branches from Chromium's mirror (`chromium.googlesource.com/external/gitlab.freedesktop.org/mesa/mesa`) and leaves tags as they are.
3. Skips about 66 malformed CVS-era annotated tags (such as `R300_DRIVER_0` and `mesa_3_1`). Their tagger line has no space before the date, or no date at all. Git accepts them, but GitHub rejects any push that contains them. No commits are affected.
4. Pushes to `mesa-mirror` with `--prune` over SSH.

## Credentials

The push uses an SSH deploy key:

- **Public half:** a deploy key with write access on `mesa-mirror`.
- **Private half:** the `MESA_MIRROR_DEPLOY_KEY` Actions secret in this repo.

This repo's built-in `GITHUB_TOKEN` can't push to another repo, and a token also isn't allowed to push Mesa's `.github/workflows/` files, so a deploy key is needed. To rotate it, generate a new key pair, replace both halves, and delete the old deploy key.

GitHub disables scheduled workflows in public repos after 60 days without commits, and nothing commits here. The workflow's last step re-enables itself through the API on every run, which resets that timer.
