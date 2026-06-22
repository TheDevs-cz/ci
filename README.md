# TheDevs shared CI

Reusable GitHub Actions workflows shared across TheDevs app repositories. The standard
*build image → push to GHCR → trigger deploy* pipeline every app uses.

> **This repo MUST stay PUBLIC.** Most app repos are public, and on GitHub a **public** repo can
> only call a reusable workflow that lives in a **public** repo. Making this private would break
> every public caller. There are no secrets here — the per-app secret lives in each caller's repo
> and in a box-only file on the deploy host.

## Workflows

| workflow | what it does | use it when |
|----------|--------------|-------------|
| **`_ship.yml`** | build → push → trigger deploy (→ optionally await result) | **the default** — one line, covers ~all apps |
| `_build.yml` | build + push to GHCR only (fully customisable) | you need a custom step between build and deploy |
| `_deploy.yml` | HMAC deploy trigger + optional result wait only | re-deploy an already-built tag, no rebuild |

`_ship.yml` is a thin orchestrator that calls `_build.yml` then `_deploy.yml`. The split keeps the
security-critical deploy signer in one tiny file that **never holds the GHCR build token**, and lets
you customise the build without touching the signer.

## How a deploy works (end to end)

```
app repo: push to main
  └─ calls TheDevs-cz/ci/.github/workflows/_ship.yml@v1
       build:   build the image, push ghcr.io/<owner>/<app>:sha-<git-sha> (+ main, + latest)
       deploy:  HMAC-POST {"app","tag","job_id","head_sha","nonce"} to
                https://deploy.<deploy_host>/hooks/<app>
                  └─ webhook receiver (non-root) verifies the per-app HMAC, enqueues a job
                       └─ host systemd executor (root) runs /srv/<app>/deploy.sh <tag>
                          (dump_secrets → migrate → zero-downtime rollout)
                          └─ (if await_deploy) publishes a GitHub check-run with the
                             status + redacted logs → this run goes red/green
```

The receiver derives the app from the **hook URL** (`/hooks/<app>`), not the request body, so a
tampered body can never switch which app is deployed. Only the **tag** is honored from the body, and
it must match `^[A-Za-z0-9._-]{1,128}$` (re-validated by the host executor).

## Add it to an app repo

Create `.github/workflows/release.yml` with this **caller snippet** — that's the whole integration:

```yaml
name: Release
on:
  push:
    branches: [main]        # push-to-main ONLY — see the security rule below

permissions:
  contents: read
  packages: write
  checks: read              # needed only if you set await_deploy: true

jobs:
  ship:
    uses: TheDevs-cz/ci/.github/workflows/_ship.yml@v1   # pinned tag (no @master/@main)
    secrets:
      DEPLOY_WEBHOOK_SECRET: ${{ secrets.DEPLOY_WEBHOOK_SECRET }}
    with:
      app: my-app           # == apps/<app>/ on the box == GHCR image name == hook id
      image: ghcr.io/my-owner/my-app
```

The default image is `ghcr.io/<owner>/<repo>`; set `image:` if the GHCR name differs. Use explicit
`secrets:` (not `inherit`) — inherit only works when caller and this repo share an owner, which they
usually don't.

## Customising the Docker build (build args, cache, target, secrets)

The build is parameterised. **Build args** are the common need — e.g. stamping a version:

```yaml
jobs:
  ship:
    uses: TheDevs-cz/ci/.github/workflows/_ship.yml@v1
    secrets:
      DEPLOY_WEBHOOK_SECRET: ${{ secrets.DEPLOY_WEBHOOK_SECRET }}
    with:
      app: my-app
      build_args: |                      # newline-separated KEY=VALUE
        APP_VERSION=${{ github.sha }}
        NODE_ENV=production
      target: runtime                    # multi-stage target (optional)
      cache: true                        # per-image layer cache (default true)
      dockerfile: docker/Dockerfile      # default: Dockerfile
      context: .                         # default: .
```

**Caching.** With `cache: true` (default), builds use `mode=max` (all layers, including
multi-stage intermediates) across **two** per-image backends: GitHub Actions cache (fast) and a
persistent `ghcr.io/<owner>/<app>:buildcache` registry tag (no eviction, survives across branches).
The `:buildcache` tag is overwritten each build (it doesn't accumulate) and is never pulled by the
deploy host — it only deploys `main`/`sha-*`. Set `cache: false` to disable both.

In your `Dockerfile`:

```dockerfile
ARG APP_VERSION=dev
ARG NODE_ENV=production
ENV APP_VERSION=${APP_VERSION}
```

> ⚠️ **Build args are NOT secret.** They are baked into image metadata/history and readable by anyone
> who can pull the image. Never put tokens/passwords in `build_args`.

**Secret build inputs** (private package registry token, etc.) go through `BUILD_SECRETS`, which feeds
BuildKit secret *mounts* (not baked into layers):

```yaml
    secrets:
      DEPLOY_WEBHOOK_SECRET: ${{ secrets.DEPLOY_WEBHOOK_SECRET }}
      BUILD_SECRETS: |
        npm_token=${{ secrets.NPM_TOKEN }}
    with:
      app: my-app
```

```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN="$(cat /run/secrets/npm_token)" npm ci
```

Build inputs: `build_args`, `cache`, `provenance`, `target`, `dockerfile`, `context`, `platforms`,
`image` (and `BUILD_SECRETS`). See the table below.

## Deploy status + logs in your CI

By default the deploy is fire-and-forget: the run goes green as soon as the box **accepts** the
trigger (it does not yet know if the deploy succeeded). Set **`await_deploy: true`** to make the run
reflect the **real** deploy outcome:

```yaml
jobs:
  ship:
    uses: TheDevs-cz/ci/.github/workflows/_ship.yml@v1
    permissions:
      contents: read
      packages: write
      checks: read                       # required for await_deploy
    secrets:
      DEPLOY_WEBHOOK_SECRET: ${{ secrets.DEPLOY_WEBHOOK_SECRET }}
    with:
      app: my-app
      await_deploy: true                 # ⟵ wait for the deploy to actually finish
      deploy_sha: true                    # (recommended with await) deploy the exact image just built
```

With `await_deploy: true`:

- **Status** — the run **fails (red)** if `deploy.sh` fails (bad image, failed migration, rollout
  health-check abort/revert, executor timeout) and **passes (green) only** if the deploy finished
  `rc=0`. It is **fail-closed**: no success proof ⇒ failure.
- **Where to see it** — three places:
  1. A **check-run named `deploy`** on the commit/PR (the repo's **Checks** tab).
  2. **Inline in this run** — the await step echoes the deploy log into a collapsible group.
  3. The run's overall **red/green**.
- **Logs** — the check + inline log show a **redacted tail** of the deploy output (secrets stripped;
  by default only status is public and full logs stay on the box — see below). The **full,
  unredacted** logs live on the deploy host and are queryable in **Grafana/Loki** (label `app="deploy"`,
  filter by your app) with 30-day retention, and via `journalctl -u lily-deploy-runner` on the box.

> **Correlation is exact.** Each run sends a unique `job_id` + `nonce` and the box reports back on
> *that* nonce + *your* commit SHA, so concurrent runs, retries, and queued deploys never show each
> other's result.

`await_timeout_seconds` (default `2700`) bounds the wait. The box runs **one deploy at a time**, so an
app queued behind others waits longer — keep the timeout above the box executor limit (1800s) plus
expected queue depth, or a healthy-but-queued deploy will false-red.

## One deploy at a time (concurrency)

`_deploy.yml` sets, per app + host:

```yaml
concurrency:
  group: deploy-<app>-<host>
  cancel-in-progress: false
```

Behavior (this is exactly "never cancel a running deploy; supersede stale pending ones"):

- An **in-progress** deploy is **never cancelled mid-flight** — it always finishes.
- A new push while one is in progress becomes **`pending`** (waits its turn).
- A still-newer push **replaces the older pending** one (you deploy the newest commit, not stale ones).

This is a **two-layer lock**: the GitHub concurrency group serializes per app at the CI level, and the
box's `flock` serializes **all** deploys globally on the host. Even if a CI run were cancelled, an
already-started box deploy still finishes (the box doesn't depend on the CI connection).

> Prefer to deploy **every** push in order instead of dropping superseded ones? That's a GitHub-native
> `concurrency: { queue: max }` at your caller level — ask and we'll document the variant.

## Inputs (`_ship.yml`)

| input                   | required | default                   | meaning |
|-------------------------|----------|---------------------------|---------|
| `app`                   | **yes**  | —                         | app id == hook id == `apps/<app>/` == GHCR image |
| `image`                 | no       | `ghcr.io/<owner>/<repo>`  | fully-qualified GHCR image to build/push |
| `deploy_host`           | no       | `lily.srv.thedevs.cz`     | server FQDN; webhook vhost is `deploy.<deploy_host>` |
| `deploy_tag`            | no       | `main`                    | tag sent to the receiver to deploy |
| `deploy_sha`            | no       | `false`                   | deploy the immutable `sha-<git-sha>` tag just built |
| `await_deploy`          | no       | `false`                   | wait for the real deploy result; fail the run if it failed |
| `await_timeout_seconds` | no       | `2700`                    | max seconds to await the result (> 1800 + queue) |
| `context`               | no       | `.`                       | Docker build context |
| `dockerfile`            | no       | `Dockerfile`              | path to the Dockerfile (relative to `context`) |
| `platforms`             | no       | `linux/amd64`             | build platform(s); the box is amd64-only |
| `target`                | no       | `''`                      | multi-stage build target |
| `build_args`            | no       | `''`                      | newline `KEY=VALUE` build args (**non-secret**) |
| `cache`                 | no       | `true`                    | per-image layer cache: GHA + GHCR `:buildcache`, `mode=max` |
| `provenance`            | no       | `true`                    | SLSA provenance attestation |

Secrets: **`DEPLOY_WEBHOOK_SECRET`** (required), **`BUILD_SECRETS`** (optional, BuildKit mounts).

## Composing the halves directly (advanced)

For a step between build and deploy (image scan, matrix, etc.), call the halves yourself:

```yaml
jobs:
  build:
    permissions: { contents: read, packages: write }
    uses: TheDevs-cz/ci/.github/workflows/_build.yml@v1
    with:
      image: ghcr.io/my-owner/my-app
      build_args: APP_VERSION=${{ github.sha }}
  # ... your own job(s) here, e.g. scan ${{ needs.build.outputs.digest }} ...
  deploy:
    needs: build
    permissions: { contents: read, checks: read }
    uses: TheDevs-cz/ci/.github/workflows/_deploy.yml@v1
    secrets:
      DEPLOY_WEBHOOK_SECRET: ${{ secrets.DEPLOY_WEBHOOK_SECRET }}
    with:
      app: my-app
      deploy_tag: ${{ needs.build.outputs.sha_tag }}
      await_deploy: true
```

## Required secret: `DEPLOY_WEBHOOK_SECRET`

A **per-app** HMAC secret in the app repo's GitHub secrets that matches this app's hook secret on the
box (`/srv/webhook/secrets.env`). It signs the deploy trigger (HMAC-SHA256 over the raw body, sent as
`X-Hub-Signature-256: sha256=<hex>`).

It is **privilege-decoupled by design**: if it leaks, the only thing an attacker can do is re-trigger
a deploy of **this one app's** own image tag. It is not an SSH key, has no docker socket, and cannot
touch `/srv` or any other app.

To rotate: set a new value in `/srv/webhook/secrets.env`, re-run the box's `/srv/webhook/deploy.sh`
(hot-reloads the hooks file), and update the app repo's `DEPLOY_WEBHOOK_SECRET` to the same value.

## The push-to-main-only rule (security)

The caller workflow MUST trigger on **`push` to `main` only**. Do **not** use `pull_request_target`,
and never check out untrusted PR code in a workflow that holds `DEPLOY_WEBHOOK_SECRET` or
`packages:write` — a fork PR could otherwise exfiltrate the secret or push a malicious image. IP
allow-listing / rate-limiting is handled at the box's Traefik public edge (CrowdSec + rate-limit + TLS).

## What is produced

Image tags pushed to `ghcr.io/<owner>/<app>`:

- `sha-<git-sha>` — the **immutable** tag (full git sha), kept for traceability / rollback / immutable deploys.
- `main` — the branch tag, **deployed** by default.
- `latest` — convenience pointer, **only** on the default branch.

The deploy trigger sends `deploy_tag` (default `main`, or the `sha-<git-sha>` when `deploy_sha: true`).

## Versioning

- Callers pin `@v1`. All current inputs are **additive with defaults that reproduce prior behavior**,
  so the `v1` tag is moved forward and existing callers keep working. `_build.yml` and `_deploy.yml`
  are reachable under the **same** `v1` tag.
- Cut a **new major** only for a breaking change to the deploy **handshake** (body shape, header name,
  tag regex, `QUEUED` marker) or if `await_deploy`'s default ever flips to `true`.
- Keep third-party actions **pinned** (a version tag; bump deliberately, never `@master`).
- Keep this repo **public**.
