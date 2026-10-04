# auto-preview

Label a pull request, get a live preview URL. Remove the label, it's gone.

It runs your stack with `docker compose` on a GitHub-hosted runner and exposes it through free
[Cloudflare Quick Tunnels](https://try.cloudflare.com) (no Cloudflare account or token). Everything
specific to your app lives in one compose file; the workflow knows nothing about it.

## Add it to your repo

**1. Add a workflow** at `.github/workflows/preview.yml`:

```yaml
name: Preview

on:
  pull_request:
    types: [labeled, unlabeled, synchronize, reopened, closed]

permissions:
  contents: read
  pull-requests: write

jobs:
  preview:
    uses: fourcores/auto-preview/.github/workflows/preview.yml@main   # pin a tag or SHA once you have one
    with:
      expose: "frontend:3000,api:8000"    # services that need a public URL, as name:port
    secrets:
      PREVIEW_ENV: ${{ secrets.PREVIEW_ENV }}   # optional, see "Good to know" below
```

**2. Describe your stack** in `.preview/docker-compose.yml`. This is an example; swap in your own services:

```yaml
services:
  api:                                      # example backend
    build: { context: .., dockerfile: api/Dockerfile }
    ports: ["8000:8000"]                    # must match expose: api:8000
    environment:
      DATABASE_URL: postgresql://app:app@db:5432/app
      FRONTEND_URL: ${PREVIEW_URL_FRONTEND} # public URL of the frontend (for CORS)
    depends_on: { db: { condition: service_healthy } }
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:8000/health || exit 1"]
      interval: 5s
      retries: 30

  frontend:                                 # example frontend
    build:
      context: ../frontend
      args:
        API_URL: ${PREVIEW_URL_API}         # public URL of the API, known before the build
    ports: ["3000:3000"]                    # must match expose: frontend:3000
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:3000/ || exit 1"]
      interval: 5s
      retries: 30

  db:                                       # example backing service (empty and throwaway per preview)
    image: postgres:16-alpine
    environment: { POSTGRES_USER: app, POSTGRES_PASSWORD: app, POSTGRES_DB: app }
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 3s
      retries: 20
```

**3. Create the label** (it must exist before anyone can apply it):

```sh
gh label create preview --color 0E8A16 --description "Run a preview environment"
```

**4. Open a PR from a branch in the same repo and add the `preview` label.** A comment appears on the
PR and turns into the URLs when the stack is up (a few minutes).

`template/` holds the same two files with more comments.
[fourcores/auto-preview-sample](https://github.com/fourcores/auto-preview-sample) is a complete
working repo using this exact setup; fork or copy it to try it first.

## How it works

The label is **state**: *the PR has the label → a preview should be running.*

| Event on the PR | Result |
|---|---|
| `preview` label added | Preview built from the PR's head commit |
| Push while labelled | Old build cancelled, rebuilt |
| `preview:<name>` added or removed while `preview` is on | Rebuilt with compose profile `<name>` on or off |
| `preview` label removed, or PR closed | Stopped |
| TTL elapsed (default 2 h) | Expired |
| `preview:<name>` with **no** `preview` label | Nothing: profile labels only modify a preview, never start one |
| Fork PRs, other labels, pushes to unlabelled PRs | Ignored (no runner started) |

One comment on the PR is edited in place: *building → live (URLs) → stopped / expired / failed*.

Under the hood: tunnels start first, so each service's public URL exists before anything builds and can
be baked into builds and config with no restarts. Then `docker compose up --build --wait` starts your
stack, and the workflow waits for each public URL to answer before posting it. Stopping works through a
per-PR concurrency group: a teardown or a newer deploy cancels the running job, and the runner is discarded.

## Tailor it to your project

**Compose rules**
- Publish each exposed service's port on the host (`ports: ["8000:8000"]`); the left side must match `expose:`.
- Give every service a `healthcheck:`. The workflow waits on them, and `depends_on` with
  `condition: service_healthy` orders startup.
- Each exposed URL must answer `/` with a non-5xx status before it's announced.
- Build contexts are relative to the compose file.

**Variables available in the compose file** (as `${NAME}`)

| Variable | Value |
|---|---|
| `PREVIEW_URL_<NAME>` | Public URL of each exposed service: `api` → `PREVIEW_URL_API`, `my-api` → `PREVIEW_URL_MY_API` |
| `PREVIEW_PR_NUMBER`, `PREVIEW_SHA` | The PR number and commit being previewed |
| `PREVIEW_ENV_FILE` | Path to your `PREVIEW_ENV` secret as a file, for `env_file:` |

**Adding more services**: the workflow never lists them, so just add them to the compose file.
- *Internal only* (cache, queue, worker): add it with a healthcheck. No workflow change.
- *Needs its own public URL*: publish its port and add `name:port` to `expose:`. It gets its own tunnel and `PREVIEW_URL_<NAME>`.
- *Optional*: set `profiles: ["extras"]` and switch it on with a `preview:extras` label
  (`gh label create "preview:extras" --color 5319E7`). Profile services stay internal unless exposed.
- Tunnels are HTTP only, so raw TCP services (databases you want to connect to) can't be exposed.

**Good to know**
- Frontend and API sit on different hostnames, so treat them as cross-site: cookies need
  `SameSite=None; Secure`, or use bearer tokens.
- Databases start empty every time. Run migrations or seed data in the entrypoint, or in a one-shot
  service the API waits on (`depends_on: { migrate: { condition: service_completed_successfully } }`).
- **Secrets:** store one repo secret `PREVIEW_ENV` as `KEY=value` lines and pass it to services with
  `env_file: [${PREVIEW_ENV_FILE}]`. Use throwaway values only (see Security).
- The compose file can live anywhere: set the `compose-file` input. It must be a single file.

**Inputs** (under `with:`)

| Input | Default | |
|---|---|---|
| `expose` | *required* | `name:port` pairs to tunnel |
| `compose-file` | `.preview/docker-compose.yml` | |
| `label` | `preview` | `<label>:<name>` selects profile `<name>` |
| `ttl-minutes` | `120` | 5 to 330 |
| `start-timeout-seconds` | `600` | limit for `docker compose up --wait` |
| `runner` | `ubuntu-latest` | must be linux/x86_64 |

Don't add a `concurrency:` block to your caller: the reusable workflow owns the per-PR group and a clash cancels it.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| No workflow run at all | PR is from a fork; label isn't named exactly `preview`; or the PR branch lacks the workflow file |
| Gate ran, other jobs skipped (`Decision: none`) | Only a `preview:<name>` label is on the PR (add `preview`), or the PR isn't open |
| `startup_failure` / "workflow not found" | Wrong `uses:` path or ref; for a private repo, enable Settings → Actions → General → Access |
| Comment says *failed* | Read the **Dump logs** step in the run (`docker compose ps` and logs) |
| "Not reachable" | `/` returns 5xx, or the service doesn't publish the port named in `expose` |
| `PREVIEW_URL_X` is blank | Name mismatch with `expose` (`my-api` becomes `PREVIEW_URL_MY_API`) |
| Tunnel URL missing | Transient cloudflared failure; remove and re-add the label |

## Security and limits

- **Same-repo branches only.** Fork PRs get no secrets and are ignored. Don't work around this with `pull_request_target`.
- **PR code runs with your preview secrets** (the compose file and Dockerfiles come from the PR), so anyone who can push a branch can read `PREVIEW_ENV`. Use throwaway credentials, never production ones.
- **URLs are public and unauthenticated** and are posted in the PR comment. No real customer data or third-party credentials.
- Anyone with triage access or above can trigger it, since they can apply labels.
- Previews use Actions minutes while up (private repos), are capped by the 6 h job limit, and builds aren't layer-cached yet.
- Quick Tunnels have no SLA and aren't for production traffic.
- Needs a stack that runs under docker compose on linux/x86_64 (no GPU or cloud-only services).

## Status

Verified on real GitHub with [auto-preview-sample](https://github.com/fourcores/auto-preview-sample): deploy on label, rebuild on push and on profile
change, a newer run cancelling an older one, teardown on label removal, a profile label alone doing
nothing, and the head-commit build. **Not yet confirmed:** an unrelated label leaving a live preview
alone, TTL expiry, the *failed* comment, closing the PR, and fork PRs being ignored.
