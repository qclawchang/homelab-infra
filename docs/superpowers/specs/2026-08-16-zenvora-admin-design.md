# zenvora-admin — design spec

Date: 2026-08-16
Status: approved for implementation planning

## Problem

The user (single operator) manages 6 repos — `homelab-infra` (personal, public) plus 5
private repos under the `ZenvoraAI` org (`day-and-you`, `securevault-framework`,
`family-media`, `aiqiuqi-memorial`, `aws-infrastructure`) — and has lost a unified view
across them. Four concrete pain points, confirmed with the user:

1. **CI/build health** — CI can go silently red across a repo without being noticed
   (see prior incident: a nanoid version-pin miss lit up three signals that nobody saw).
2. **Security posture** — 2FA, secrets, Dependabot alerts are scattered per-repo with
   no aggregate view. The org is on the GitHub Free plan, which cannot enable branch
   protection on private repos at all — a hard platform limit, not a per-repo gap.
3. **Deployment/runtime drift** — no visibility into whether the containers actually
   running on the Lightsail host match the latest images pushed to GHCR.
4. **Backlog** — open PRs/issues/stale branches are scattered per-repo with no
   aggregate view.

## Goals

- One authenticated dashboard giving a unified view across all 6 repos for the above
  four concerns.
- Fits on the existing Lightsail host without meaningfully worsening its memory
  pressure (current `docker-compose.yml` already commits 2,286MB of `mem_limit`
  against a ~1.9–2GB host).

## Non-goals (v1)

- AWS account-level state (backups, cost, IAM) — already tracked separately via the
  `aws-infrastructure` repo and prior AWS reviews; out of scope here.
- Multi-user support — this is a single-operator tool with one whitelisted GitHub
  username.
- Any persistent database or server-side session store.
- Write actions (merging PRs, restarting containers, rotating secrets) — v1 is
  read-only observability. Actions are a possible future iteration, not part of this
  spec.

## Repo & hosting

- New repo: `ZenvoraAI/zenvora-admin`, private.
- Deploys as one additional service in the existing `homelab-infra`
  `docker-compose.yml`, reverse-proxied by the nginx already running there.
- No new reverse proxy, no new TLS termination, no new persistent volume beyond what's
  needed for the container itself.

## Architecture

**Static frontend + minimal API backend**, chosen over a single server-rendered
process specifically because the frontend can be served by the nginx that's already
resident on the host at effectively zero additional runtime memory, while the backend
stays small enough to add to an already-oversubscribed host.

- **Frontend**: static SPA (Vite), built at CI time, served as static files by the
  existing nginx. Visual design draws on `~/self-growth-dashboard.html`'s language —
  warm/editorial palette (cream/amber/moss), Fraunces+Inter type, card-based grid —
  adapted to this dashboard's content, not copied literally (that file is a personal
  growth tracker, not a repo dashboard).
- **Backend**: one process (Node or Bun), three responsibilities only:
  1. GitHub OAuth handshake.
  2. Proxy + cache GitHub REST/GraphQL calls (60s in-memory TTL; single-user, so no
     cache-key collision risk).
  3. Read local docker state via a **read-only** docker socket mount, and compare
     against GHCR's latest-tag API.
- No database anywhere. Target `mem_limit`: 64–96m for the backend service.

## Auth

- GitHub OAuth only (no separate password to manage).
- OAuth scope requested: repo/org read + `read:packages` (needed for the
  deployment-drift panel to query GHCR tags).
- Callback exchanges the code for a token, checks the returned username against a
  single whitelisted username (env var).
- On success, the backend signs an httpOnly, secure session cookie whose payload is an
  **encrypted copy of the GitHub token itself** — this is the literal mechanism by
  which "no database" is upheld: nothing is stored server-side, the cookie *is* the
  session. Short expiry (~12h); re-authenticate via GitHub after that.
- Every panel is gated behind a valid session; nothing renders unauthenticated.

## Feature panels (all five are in scope for v1)

1. **Repo grid** — entry view. One card per repo (all 6): name, visibility, last push,
   primary language, description.
2. **CI health** — latest workflow run status per repo, failures surfaced prominently
   (this is the direct fix for the "silent red CI" failure mode), plus GitHub Actions
   minutes used this billing cycle with a quota-warning threshold.
3. **Security posture** — account-level 2FA status, Dependabot alerts per repo,
   secret-scanning alerts, repo-level secrets count. Branch protection renders as
   "unavailable on this plan" rather than a false "unprotected" flag, since the Free
   org tier cannot enable it on private repos at all.
4. **Deployment drift** — reads `docker ps`/`docker inspect` on the Lightsail host via
   the read-only socket mount, cross-references each running container's image tag
   against the latest tag pushed to GHCR for that repo, flags drift.
5. **Backlog** — open PRs awaiting the user's review, open issues assigned to the
   user, stale branches, aggregated across all 6 repos into one list.

## Data flow

- Frontend calls `/api/repos`, `/api/ci`, `/api/security`, `/api/deployment`,
  `/api/backlog` independently, in parallel — each panel loads on its own.
- Backend decrypts the token from the session cookie per request, calls GitHub
  REST/GraphQL, serves from the 60s in-memory cache when fresh.
- Deployment-drift additionally reads the docker socket and calls GHCR's package API.

## Error handling

- Each panel fails independently: a GitHub rate-limit or API error shows "couldn't
  load, retry" on that one card only, not a full-page crash. Last-good cached value is
  shown if available, clearly marked as stale.
- Docker-socket errors (not mounted, permission denied) render an explicit error on
  the deployment panel — this tool exists specifically to surface problems, so no
  panel silently renders empty on failure.

## Testing

- Unit tests for GitHub-response mapping functions and the drift-comparison logic.
- Integration tests for the API routes with mocked GitHub API responses.
- OAuth callback test with a mocked token exchange.
- No automated end-to-end test against the real docker socket (requires the real
  host) — covered by a manual verification step after deploy instead.

## Open risks / things to verify during implementation

- Confirm actual free memory on the Lightsail host (`free -h`) before wiring the new
  service into `docker-compose.yml` — the 2,286MB `mem_limit` figure is committed
  capacity, not necessarily peak real usage, but headroom should be checked, not
  assumed.
- Confirm GitHub OAuth App scopes needed for GHCR package reads work as expected for
  a personal-account token against org-owned packages (`read:packages` on a
  classic/OAuth token vs fine-grained PAT behavior can differ).
