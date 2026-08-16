# zenvora-admin — design spec

Date: 2026-08-16
Status: revised after 3-agent independent review (security / architecture / scope) — see "Review history" at bottom

## Problem

The user (single operator) manages 6 repos — `homelab-infra` (personal, public) plus 5
private repos under the `ZenvoraAI` org (`day-and-you`, `securevault-framework`,
`family-media`, `aiqiuqi-memorial`, `aws-infrastructure`) — and has lost a unified view
across them. Four concrete pain points, confirmed with the user:

1. **CI/build health** — CI can go silently red across a repo without being noticed
   (see prior incident: a nanoid version-pin miss lit up three signals that nobody saw).
2. **Security posture** — 2FA, secrets, Dependabot alerts are scattered per-repo with
   no aggregate view. The org is on the GitHub Free plan, which cannot enable branch
   protection *or secret scanning* on private repos at all — a hard platform limit, not
   a per-repo gap.
3. **Deployment/runtime drift** — no visibility into whether the containers actually
   running on the Lightsail host match what's pinned in `docker-compose.yml`.
4. **Backlog** — open PRs/issues/stale branches are scattered per-repo with no
   aggregate view.

## Goals

- One authenticated dashboard giving a unified view across all 6 repos for the above
  four concerns.
- Fits on the existing Lightsail host without meaningfully worsening its memory
  pressure (current `docker-compose.yml` already commits 2,286MB of `mem_limit`
  against a ~1.9–2GB host — see "Open risks", this is a real constraint, not a nice-to-have).
- **Proactively notify** (email) when CI goes red or a critical security alert fires —
  a pull-only dashboard doesn't actually fix "nobody noticed," it just relocates the
  failure to "nobody opened the dashboard." This was added after review; see history.

## Non-goals (v1)

- AWS account-level state (backups, cost, IAM) — already tracked separately via the
  `aws-infrastructure` repo and prior AWS reviews; out of scope here.
- Multi-user support — this is a single-operator tool with one whitelisted GitHub
  user (matched by immutable numeric user ID, not username).
- Write actions against GitHub or the docker host (merging PRs, restarting containers,
  rotating secrets) — v1 is read-only observability of those systems. The proactive
  email notification is a side-effect of this app only, not a write against GitHub or
  docker, so it doesn't violate this constraint.

## Repo & hosting

- New repo: `ZenvoraAI/zenvora-admin`, private.
- Deploys as one additional service (plus a docker-socket-proxy sidecar, see
  Architecture) in the existing `homelab-infra` `docker-compose.yml`, reverse-proxied
  by the nginx already running there via a path rule — **no new volume, no nginx
  document-root change** (see "Review history" for why the original static+nginx
  design was dropped).

## Architecture

**Single process, serves both the built static frontend and the API** — chosen over
the original static-frontend-via-nginx split because the existing nginx has no
document root today (only a certbot webroot mount); adding one would require a new
bind mount and an nginx recreate (brief outage for every site on the host) just to
serve this one app. Collapsing to one process avoids that entirely: nginx adds a
single `proxy_pass` location block, no restart-sensitive volume change.

- **Frontend**: SPA (Vite), built at CI time, bundled into the same container image
  and served by the backend process itself as static files. Visual design draws on
  `~/self-growth-dashboard.html`'s language — warm/editorial palette (cream/amber/moss),
  Fraunces+Inter type, card-based grid — adapted to this dashboard's content, not
  copied literally (that file is a personal growth tracker, not a repo dashboard).
- **Backend**: one process (Bun preferred over Node for memory footprint — Node +
  Octokit + TLS + a docker client realistically lands 70–110MB RSS on its own, which
  doesn't leave room for the SQLite layer and static-file serving added below).
  Responsibilities:
  1. GitHub App user-to-server OAuth-like login handshake (see Auth).
  2. Serve the built SPA static files.
  3. Proxy + cache GitHub REST/GraphQL calls, backed by SQLite (see Data flow).
  4. Query docker container state **via a docker-socket-proxy sidecar**, not a direct
     socket mount (see below), and compare against GHCR data.
  5. Run the periodic check that triggers the proactive email alert.
- **docker-socket-proxy sidecar**: a small dedicated container (e.g.
  `tecnativa/docker-socket-proxy`, ~10–15m) that has the real `docker.sock` mount and
  exposes only `GET /containers/*` over an internal-only HTTP API. The backend talks
  to this proxy, never to the raw socket. This replaces the original design's
  `docker.sock:ro` mount, which does **not** actually restrict the Docker API's
  write/exec surface — the `ro` bind-mount flag only affects filesystem `open()`
  semantics on the socket file, not the API traffic carried over it. A compromised
  backend with a raw socket mount is root-equivalent on a host running 6 other
  production services; the proxy is the actual boundary.
- SQLite (embedded, no server process) for sessions, a persisted API-response cache,
  and alert-state tracking — see Data flow. Target `mem_limit`: ~128m for the backend,
  ~16m for the socket-proxy sidecar. This raises the host's committed total from
  2,286MB to roughly 2,430MB — see "Open risks," this needs to be checked against
  actual free memory before deploying, not assumed to fit.

## Auth

- **GitHub App** with fine-grained, read-only permissions (Contents, Metadata,
  Actions, Dependabot alerts, Secret-scanning alerts: read-only; org Administration:
  read-only for 2FA status and Actions-minutes billing) — **not** a classic OAuth App.
  Classic OAuth's `repo` scope grants full read/write across all repos, which would
  make this "read-only observability" tool a credential capable of destructive writes
  across the whole org if it were ever compromised. The GitHub App's user-to-server
  flow preserves the "log in as yourself" UX while actually being read-only.
- Login flow includes a CSRF `state` parameter, generated per attempt and validated on
  callback (missing from the original design — this is standard OAuth-callback
  hygiene, not optional).
- On callback, the backend checks two things before issuing a session, both fail-closed:
  1. The authenticated GitHub user's **immutable numeric ID** (not username — usernames
     can be released and reclaimed by a different account) matches the single
     whitelisted ID (env var).
  2. The GitHub user's `two_factor_authentication` field is `true`. If the operator's
     own GitHub account doesn't have 2FA enabled, login is rejected outright — this
     also happens to close an existing gap noted in the org's security posture (2FA
     was off account-wide).
- **Session storage**: SQLite, not a cookie-embedded token. A session row holds an
  opaque session ID, the encrypted GitHub token, the user ID, and an expiry (~12h).
  The cookie holds only the opaque session ID (httpOnly, secure, `SameSite=Lax`).
  Revocation is a row delete — this is the fix for the original design's biggest gap,
  where a leaked cookie was a live, unrevocable GitHub token for up to 12h. The
  encryption key for the token column comes from an env var; rotating it invalidates
  all existing sessions (acceptable — they're short-lived and re-auth is one click).
- Every panel is gated behind a valid, unexpired session; nothing renders
  unauthenticated.

## Feature panels (all five are in scope for v1, plus one background job)

1. **Repo grid** — entry view. One card per repo (all 6): name, visibility, last push,
   primary language, description.
2. **CI health** — latest workflow run status per repo, failures surfaced prominently
   (this is the direct fix for the "silent red CI" failure mode), plus GitHub Actions
   minutes used this billing cycle against a quota-warning threshold (default: 80% of
   the monthly included minutes — adjustable later, not a hard requirement).
3. **Security posture** — account-level 2FA status, Dependabot alerts per repo,
   repo-level secrets count. Both branch protection **and secret scanning** render as
   "unavailable on this plan" rather than a false negative, since GitHub Free cannot
   enable either on private repos (secret scanning requires GHAS) — this was missed
   for secret scanning in the original design and is the same false-signal class the
   design already handled for branch protection.
4. **Deployment drift** — modeled **per running container**, not per repo (6 repos map
   to 7 containers today; two repos have none, one has two; the 7th container, nginx,
   isn't a deployed product image and isn't tracked here). Reads container state via
   the docker-socket-proxy and reports two *distinct* signals, not one conflated
   "drift" flag. This is a **staleness** comparison, not a pin comparison — there is
   no durable "what should be running" value anywhere on this host to check against.
   An earlier version of this design tried comparing the running image's tag against
   a value mirroring `docker-compose.yml`'s own per-service tag variables
   (`FAMILY_API_TAG` etc.), on the assumption those were stable pins; tracing the
   actual deploy process (each product's own CI workflow SSHing in and running
   `export FAMILY_API_TAG=<sha>; docker compose up -d <service>` as a one-off) showed
   that value never persists anywhere `zenvora-admin`'s own, separately-running
   container could read it. Comparing to GHCR build recency instead needs no such
   external state:
   - **Upgrade-available signal**: GHCR has a newer image version than the one
     currently running (found by listing package versions ordered by creation — GHCR
     has no literal "latest tag" endpoint, so "latest" is inferred as most-recently-
     created), and that newer version is recent (within a staleness threshold,
     default 48h). Informational — a deploy just hasn't happened yet, normal.
   - **Fault signal**: same as above, but the newer version has existed *past* the
     staleness threshold and still isn't running — likely a stuck or forgotten
     deploy, not just "hasn't happened yet." This is the one that should look
     alarming, and must be visually distinct from the upgrade-available signal above.
5. **Backlog** — open PRs awaiting the user's review, open issues assigned to the
   user, stale branches (default: no commits in 90 days), aggregated across all 6
   repos into one list.
6. **Proactive alert (background job, not a panel)** — a periodic in-process check
   (independent of anyone loading the page) re-evaluates CI health and security alerts
   against the last-known state persisted in SQLite. On a transition from OK to bad —
   not on every check, to avoid spamming the same red build repeatedly — it sends one
   email via SMTP (credentials via env var).

## Data flow

- Frontend calls `/api/repos`, `/api/ci`, `/api/security`, `/api/deployment`,
  `/api/backlog` independently, in parallel — each panel loads on its own.
- Backend resolves the session from the cookie's opaque ID against SQLite, decrypts
  the stored GitHub token, calls GitHub REST/GraphQL.
- Responses are cached in SQLite (not purely in-memory) with a 60s freshness window.
  Using SQLite instead of a pure in-memory TTL means: (a) a cached "last good" value
  survives a backend restart instead of the cache going cold, and (b) the same table
  doubles as the state history the proactive-alert job diffs against.
- Deployment-drift additionally queries the docker-socket-proxy and GHCR's package
  versions API, per container.
- GitHub API load: roughly 40–60 requests per full page load across all panels,
  capped at ~60 full-loads/hour by the 60s cache — far under the 5,000/hr authenticated
  REST limit. The one limit worth watching is the separate Search API limit (30/min),
  used by the backlog panel's issue/PR search.

## Error handling

- Each panel fails independently: a GitHub rate-limit or API error shows "couldn't
  load, retry" on that one card only, not a full-page crash. The SQLite-backed last-
  good value is shown if available, clearly marked as stale — this now actually works
  across restarts, unlike a pure in-memory cache which would have nothing to fall back
  on immediately after a restart.
- Docker-socket-proxy errors (proxy unreachable, permission denied) render an explicit
  error on the deployment panel — this tool exists specifically to surface problems,
  so no panel silently renders empty on failure.
- Auth failures (wrong user ID, 2FA not enabled, invalid/expired `state`) render a
  clear rejection reason on the login page, not a generic error.

## Testing

- Unit tests for GitHub-response mapping functions and the two drift-comparison
  signals (fault vs upgrade-available).
- Integration tests for the API routes with mocked GitHub API responses.
- OAuth callback tests: valid login; wrong user ID rejected; 2FA-disabled account
  rejected; invalid/missing `state` rejected.
- Explicit test that no panel endpoint returns data without a valid session.
- Proactive-alert job test: fires once on an OK→bad transition, does not re-fire on
  a subsequent check that's still bad (no repeat spam).
- No automated end-to-end test against the real docker-socket-proxy (requires the
  real host) — covered by a manual verification step after deploy instead.

## Open risks / things to verify during implementation

- **Host memory — verified 2026-08-16, proceeds.** Measured on the real host:
  `free -h` shows 1.9Gi total, 906Mi used, **752Mi available**; `docker stats
  --no-stream` shows actual combined container usage is only ~660MB (vs. the
  2,286MB their `mem_limit` ceilings sum to — those ceilings are far from being hit
  simultaneously in practice). zenvora-admin's backend (~128m) plus the
  socket-proxy sidecar (~16m) is ~144MB, comfortably inside the 752MB available.
  Two things to keep an eye on, not blockers: `homelab-memorial-api` is already at
  89.5% of its own 192m ceiling (171.9MiB/192MiB) — the container closest to an
  OOM-kill if it grows, independent of this project; and the worst-case ceiling
  arithmetic (every container simultaneously at its own cap) rises from 2,286MB to
  ~2,430MB against the 1.9GB host, which the 2.8GB of free swap would absorb as
  slowdown rather than a crash, consistent with the README's framing of `mem_limit`
  as a ceiling rather than a reservation.
- GitHub App installation covers both the org (`ZenvoraAI`) and the personal account
  (`homelab-infra`) — confirm the user-to-server token actually carries read access to
  both during implementation; App installation scope across a personal account vs an
  org has different setup steps.
- SMTP/email-sending credential for the proactive alert needs to be provisioned
  (env var) — not yet chosen.

## Review history

This spec was revised after an independent 3-agent review (security / architecture /
scope-completeness) of the first draft. All three agents independently flagged the
classic-OAuth `repo`-scope and the `docker.sock:ro` mount as critical — those are the
two biggest changes reflected above. Other changes (SQLite session store, per-
container drift modeling, secret-scanning plan limitation, single-process serving,
proactive email alert, login-time 2FA requirement) came from a mix of review findings
and follow-up decisions made with the user afterward.
