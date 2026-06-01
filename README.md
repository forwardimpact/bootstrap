# FIT Bootstrap

Opinionated FIT environment bootstrap for GitHub Actions. Sets up Bun,
restores a single environment cache (CLI tools in `~/.local` plus
`node_modules` and `generated`), installs anything missing, optionally
checks out the wiki, and runs `./scripts/bootstrap.sh`.

Single source of truth for the FIT CI environment. The monorepo's local
`bootstrap` action and every FIT sibling action (e.g. `kata-agent`) call
this one — version-pinned at `@v1` — so they never drift.

## Usage

```yaml
- uses: actions/checkout@v4
- uses: forwardimpact/fit-bootstrap@v1
  with:
    token: ${{ steps.ci-app.outputs.token }}   # optional, enables wiki checkout
    app-slug: kata-agent-team                   # optional, sets git identity
    app-id: ${{ secrets.KATA_APP_ID }}          # optional, sets git identity
```

Cold-cache runtime is ~3 minutes; warm-cache is ~15-20 seconds.

## Prerequisites

The consumer repo must follow FIT conventions:

- `scripts/install-deps.sh` — installs `just`, `apm`, `gh` into
  `$HOME/.local`. Supports `--paths` (prints the paths it manages, so the
  action caches exactly those); its hash is part of the cache key.
- `scripts/bootstrap.sh` — invoked after the environment is ready. Receives
  `BOOTSTRAP_WORKSPACE_CACHE_HIT={true|false}` (skip install/codegen on a warm
  cache) and `BOOTSTRAP_SKIP_SYNC=true` (the action already rebased onto
  `origin/main`, so don't fetch+rebase again). Handles wiki init/pull when a
  `token:` is provided.
- `bun.lock` — its hash is part of the cache key.

## Inputs

| Input         | Required | Default    | Description                                                                              |
| ------------- | -------- | ---------- | ---------------------------------------------------------------------------------------- |
| `token`       | No       | `""`       | GitHub token with read access to the wiki. When provided, the wiki is checked out into `./wiki`. Pushing back is the caller's job — see [`forwardimpact/fit-wiki@v1`](https://github.com/forwardimpact/fit-wiki). |
| `app-slug`    | No       | `""`       | GitHub App slug for git identity (e.g. `kata-agent-team`).                              |
| `app-id`      | No       | `""`       | GitHub App ID for the git identity email.                                                |
| `bun-version` | No       | `"1.3.11"` | Bun version to install.                                                                  |

## Caching

One **environment cache** holds both the CLI tools and the workspace, so the
critical path makes a single `actions/cache` restore:

- **Paths** — the tool paths `scripts/install-deps.sh --paths` declares (each
  tool's lib dir + bin symlink, which keeps unrelated `~/.local` tooling out
  of the cache), plus `node_modules`, `generated`, and
  `libraries/*/src/generated`.
- **Key** — `env-v2-<os>-<hash>`, where the hash covers everything that
  changes what gets installed or generated: `scripts/install-deps.sh`,
  `bun.lock`, `**/*.proto`, and the `libcodegen` sources. The action rebases
  the workspace onto `origin/main` *before* hashing, so the key reflects the
  tree `scripts/bootstrap.sh` actually runs against — a feature branch caught
  behind a release commit lands on the same key as a fresh build of main, not
  a stale snapshot.
- **Version prefix** — bump `env-vN` (v2 → v3 → …) when the cached layout
  changes in a way the hash can't see, e.g. the move to relative generated
  symlinks that v2 introduced.

A `restore-keys` fallback warms the cache from the most recent build on a key
miss; `cache-hit` stays `'false'` there, so `scripts/bootstrap.sh` runs
`bun install` + `just codegen` to reconcile. On a full hit it skips both —
`generated` and its **relative** `libraries/*/src/generated` symlinks restore
intact. (Relative links survive `actions/cache` extraction; the old absolute
ones did not, which is why codegen used to re-run on every warm cache.)

## Wiki

When `token` is provided, the wiki is checked out into `./wiki` before
`scripts/bootstrap.sh` runs, so agents can read and write agent memory
during the job. When `token` is empty, the checkout is skipped — the
action is safe to use in jobs that don't need the wiki (e.g. pure CI
checks).

Pushing the wiki back is **not** this action's job. The token minted at
job start expires after one hour, so a cleanup-time push fails on long
agent runs. Push with [`forwardimpact/fit-wiki@v1`](https://github.com/forwardimpact/fit-wiki)
as an `always()` step after the agent — it mints a fresh token first.
