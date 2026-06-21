# TheDevs shared CI

Reusable GitHub Actions workflows shared across TheDevs app repositories. Today it ships one
workflow: **`_ship.yml`** — the standard *build image → push to GHCR → trigger deploy* pipeline
every app uses.

> **This repo MUST stay PUBLIC.** Most app repos are public, and on GitHub a **public** repo can
> only call a reusable workflow that lives in a **public** repo. Making this private would break
> every public caller. There are no secrets here — the per-app secret lives in each caller's repo
> and in a box-only file on the deploy host.

## How a deploy works (end to end)

```
app repo: push to main
  └─ calls TheDevs-cz/ci/.github/workflows/_ship.yml@v1
       1. build the image, push ghcr.io/<owner>/<app>:sha-<short>  (+ main, + latest on default branch)
       2. HMAC-POST {"app":"<app>","tag":"main"} to
          https://deploy.<deploy_host>/hooks/<app>
            └─ webhook receiver on the box (non-root) verifies the per-app HMAC,
               validates the tag, enqueues a job
                 └─ host systemd executor (root) runs /srv/<app>/deploy.sh <tag>
                    (dump_secrets → migrate → zero-downtime rollout)
```

The receiver derives the app from the **hook URL** (`/hooks/<app>`), not from the request body, so
a tampered body can never switch which app is deployed. Only the **tag** is taken from the request,
and it must match `^[A-Za-z0-9._-]{1,128}$` (re-validated again by the host executor).

**We deploy `main` for now** (simpler; fewer distinct images kept on the box). The immutable
`sha-<short>` tag is still built and pushed to GHCR for traceability/rollback — flip a caller to
immutable deploys later by passing `deploy_tag: sha-<short>` (one-line change, no box edit).

## Add it to an app repo

Create `.github/workflows/release.yml` in the app repo with this **caller snippet** — that's the
whole integration:

```yaml
name: Release
on:
  push:
    branches: [main]        # push-to-main ONLY — see the rule below

permissions:
  contents: read
  packages: write

jobs:
  ship:
    uses: TheDevs-cz/ci/.github/workflows/_ship.yml@v1   # pinned tag (no @master/@main)
    # Pass the secret EXPLICITLY — `secrets: inherit` does NOT cross owners, and
    # most app repos (e.g. janmikes/*) are a different owner than TheDevs-cz.
    # GITHUB_TOKEN is provided to the called workflow automatically.
    secrets:
      DEPLOY_WEBHOOK_SECRET: ${{ secrets.DEPLOY_WEBHOOK_SECRET }}
    with:
      app: my-app           # == apps/<app>/ on the box == GHCR image name == hook id
      image: ghcr.io/my-owner/my-app
      deploy_host: lily.srv.thedevs.cz
```

The default image is `ghcr.io/<owner>/<repo>`; set `image:` if the GHCR name differs from the repo.
**Note:** use explicit `secrets:` (not `inherit`) — inherit only works when caller and the
shared-CI repo share an owner, which they usually don't here.

### Inputs

| input         | required | default                              | meaning                                              |
|---------------|----------|--------------------------------------|------------------------------------------------------|
| `app`         | **yes**  | —                                    | app id == hook id == `apps/<app>/` == GHCR image     |
| `image`       | no       | `ghcr.io/<owner>/<repo>` (lowercase) | fully-qualified GHCR image to build/push             |
| `deploy_host` | no       | `lily.srv.thedevs.cz`                | server FQDN; webhook vhost is `deploy.<deploy_host>` |
| `deploy_tag`  | no       | `main`                               | tag sent to the receiver to deploy (set to a `sha-<short>` for immutable) |
| `dockerfile`  | no       | `Dockerfile`                         | path to the Dockerfile (relative to `context`)       |
| `context`     | no       | `.`                                  | Docker build context                                 |
| `platforms`   | no       | `linux/amd64`                        | build platform(s); the box is amd64-only             |

### Required secret: `DEPLOY_WEBHOOK_SECRET`

A **per-app** HMAC secret, set in the app repo's GitHub secrets, that matches this app's hook secret
on the box (a line in the box-only file `/srv/webhook/secrets.env`). It signs the deploy trigger
(HMAC-SHA256 over the raw request body, sent as `X-Hub-Signature-256: sha256=<hex>`).

It is **privilege-decoupled by design**: if it leaks, the only thing an attacker can do is
re-trigger a deploy of **this one app's** own image tag. It is not an SSH key, has no docker socket,
and cannot touch `/srv` or any other app.

To rotate: set a new value in the box file `/srv/webhook/secrets.env`, re-run the box's
`/srv/webhook/deploy.sh` (hot-reloads the hooks file), and update the app repo's
`DEPLOY_WEBHOOK_SECRET` to the same value.

## The push-to-main-only rule (security)

The caller workflow MUST trigger on **`push` to `main` only**. Do **not** use
`pull_request_target`, and never check out untrusted PR code in a workflow that holds
`DEPLOY_WEBHOOK_SECRET` or `packages:write` — a PR from a fork could otherwise exfiltrate the
secret or push a malicious image. IP allow-listing / rate-limiting is handled at the box's Traefik
public edge (CrowdSec + rate-limit + TLS), not in this workflow.

## What `_ship.yml` produces

Image tags pushed to `ghcr.io/<owner>/<app>`:

- `sha-<short>` — the **immutable** tag, kept for traceability / rollback.
- `main` — the branch tag that is **deployed** by default (`deploy_tag`).
- `latest` — convenience pointer, **only** on the default branch.

The deploy trigger sends `deploy_tag` (default `main`); the box pulls that tag.

## Conventions for changing this repo

- Keep third-party actions **pinned** (a version tag today; bump deliberately, never `@master`).
- Do not break the handshake contract (endpoint, header name `X-Hub-Signature-256`, body shape
  `{"app","tag"}`, tag regex) — the box's receiver and executor are written against these exact values.
- Callers pin `_ship.yml@v1`; move/update the `v1` tag for backward-compatible changes, or cut a new
  tag and bump callers for breaking ones.
- Keep this repo **public**.
