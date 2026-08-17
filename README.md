# homelab-infra

Docker Compose orchestration for the shared Lightsail box
(`ubuntu@ip-172-26-13-172`). This public repository contains only Compose,
nginx configuration, and deploy-support scripts. It never contains runtime env
files, certificates, database credentials, or GHCR tokens.

The checkout on the host is `/opt/homelab-infra`. Application source code and
release workflows remain in their own repositories.

## Services and routing

| Compose service | Container | Host port | Public hostnames | `mem_limit` |
| --- | --- | ---: | --- | ---: |
| `nginx` | `homelab-nginx` | 80 / 443 | all hosts below | 64m |
| `api` | `homelab-family-api` | 4000 | `api.family.valtou.com` | 640m |
| `securevault-api` | `homelab-securevault-api` | 3000 | `api.valtou.com` | 256m |
| `dayandyou-prod` | `homelab-dayandyou-prod` | 3003 | `dayandyou.com`, `www.dayandyou.com` | 400m |
| `dayandyou-staging` | `homelab-dayandyou-staging` | 3002 | `staging.dayandyou.com` | 350m |
| `memorial-api` | `homelab-memorial-api` | 3001 | `api.aiqiuqi.com` | 192m |
| `memorial-worker` | `homelab-memorial-worker` | — | none (no listener) | 384m |

Every service uses `network_mode: host`; the application ports must remain
firewalled from the public internet. Docker nginx is the only public reverse
proxy. `memorial-worker` is the one service with no port and no vhost — it
processes media out of band.

Those limits sum to 2286 MiB on a host measured at 1.9 GiB, so the box is
deliberately oversubscribed by roughly 18%. A `mem_limit` is a ceiling, not a
reservation, and no service sits near its own ceiling — but the arithmetic means
several services peaking together is an OOM, not a slowdown. Check
`docker stats --no-stream` before raising any limit, and treat the total as the
budget being spent rather than headroom being claimed.

## Images and tags

Images come from GHCR under the `zenvoraai` namespace. The application
repositories moved from the `qclawchang` personal account to the `ZenvoraAI`
organization on 2026-08-15, and **GHCR packages do not follow a repository
transfer** — the old `ghcr.io/qclawchang/*` packages still exist but nothing
pushes to them any more, so pinning to one would pull a frozen image for ever
with no error to notice.

The organization is spelled `ZenvoraAI`, but every image reference here says
`zenvoraai`: a Docker image reference must be lower case or the client rejects
it outright with `repository name must be lowercase`, before the request ever
reaches the registry.

Four services accept a pinned tag so a deploy can name the exact image it built
rather than racing `:latest`:

| Variable | Service | Default |
| --- | --- | --- |
| `GHCR_OWNER` | all | `zenvoraai` |
| `FAMILY_API_TAG` | `api` | `latest` |
| `SECUREVAULT_API_TAG` | `securevault-api` | `latest` |
| `MEMORIAL_API_TAG` | `memorial-api` | `latest` |
| `MEMORIAL_WORKER_TAG` | `memorial-worker` | `latest` |

Day and You pins by channel instead — `dayandyou:release` and
`dayandyou:staging` — so it has no tag variable.

## Normal deployment model

Each product's GitHub Actions workflow builds and pushes its own GHCR image,
runs any required migration, then SSHes to this host to update exactly one named
service:

```bash
cd /opt/homelab-infra
git pull --ff-only
export FAMILY_API_TAG=<the sha this run built>   # or the service's own tag var
docker compose pull <service>
docker compose up -d <service>
```

Pass the tag the run just built rather than letting it default to `latest`.
`:latest` moves, so a deploy that pulls it can ship an image a different run
pushed a moment earlier — the deploy reports success for a release it did not
build.

For an existing service, never run an unqualified `docker compose up -d` during
a product deploy. It risks restarting unrelated sites. The application workflow
is responsible for migrations; this repository is responsible for runtime
orchestration only.

## Day-to-day operations

```bash
cd /opt/homelab-infra
docker compose ps
docker compose logs --tail=100 <service>
docker compose restart <service>
docker stats --no-stream
```

Health checks on the host:

```bash
curl -fsS http://127.0.0.1:4000/health  # family API
curl -fsS http://127.0.0.1:3000/health  # SecureVault API
curl -fsS http://127.0.0.1:3001/health  # memorial API
curl -fsS http://127.0.0.1:3003/         # Day and You production
curl -fsS http://127.0.0.1:3002/         # Day and You staging
```

`memorial-worker` has no listener, so there is nothing to curl. Check it through
Compose instead:

```bash
docker compose ps memorial-worker
docker compose logs --tail=50 memorial-worker
```

Use `docker compose config` before applying a Compose or nginx configuration
change. Check nginx syntax without replacing the running container:

```bash
docker compose run --rm --no-deps nginx nginx -t
```

`run` matters here, not `exec`. It creates a fresh container, so the bind mounts
resolve to the files currently on disk; `exec` reuses the running container's
mounts, which can be stale — see below.

### Changing `nginx/nginx.conf` needs a recreate, not a reload

`nginx/conf.d` is a **directory** mount, so edits inside it are visible to the
running container and `nginx -s reload` picks them up.

`nginx/nginx.conf` is a **single-file** mount, and `git pull` replaces files
rather than editing them in place. The container keeps pointing at the old
inode, so no amount of reloading will ever load a new `nginx.conf`:

```bash
docker compose up -d --force-recreate nginx   # brief outage for every site
```

This bites in a confusing way. On 2026-08-13 a correct `limit_req_zone` line was
on disk and `nginx -t` still failed with `zero size shared memory zone
"email_ep"` — conf.d (directory mount, fresh) referenced a zone that
nginx.conf (single-file mount, stale) had not yet defined. The file was right;
the mount was old. `docker compose run` validated the same config successfully.

The running nginx keeps its loaded config until a reload succeeds, so an invalid
file on disk breaks nothing. Validate, then recreate.

## Secrets and state

- Family Media persists its API env only at
  `/opt/secrets/family-media/.env`; Compose mounts it read-only at `/app/.env`.
  It must be readable by the Compose user and mode `0600`.
- The memorial services read `/opt/secrets/aiqiuqi-memorial/api.env` and
  `/opt/secrets/aiqiuqi-memorial/worker.env` through `env_file`. Do not
  hand-edit either one — see `scripts/` below.
- SecureVault and Day and You receive runtime variables from their deployment
  workflows; do not add their values to this repository.
- PostgreSQL runs natively on Lightsail. Its backup, archival, and restore
  tooling lives in the private `aws-infrastructure` repository, not here, and
  it is not a Compose service.
- Certbot state lives under `/etc/letsencrypt`; ACME challenge files live at
  `/var/lib/homelab-acme` and are mounted read-only into Docker nginx.

## `scripts/` — host tooling

| Script | What it does |
| --- | --- |
| `docker-prune.sh` | Removes images no container references and older than 30 days. Deploys pin exact commit-SHA tags, so every deploy across every project leaves a tagged image behind for ever; unchecked, that fills the disk, which it did on 2026-08-12. The 30-day floor keeps a recent rollback target reachable. |
| `refresh-memorial-aws-credentials.sh` | Pulls the memorial API and worker AWS credentials from SSM Parameter Store into their env files. |
| `refresh-memorial-secrets.sh` | Pulls the CloudFront media signing key pair from SSM into the memorial API env file. |
| `refresh-zenvora-admin-secrets.sh` | Pulls zenvora-admin's 12 secrets (GitHub App credentials, SES SMTP credentials, token-encryption key, whitelisted user ID, alert-check PAT) from SSM into its env file. Same pattern as the memorial scripts, for the same reason. |
| `reload-nginx-after-cert-renewal.sh` | Certbot deploy hook; reloads the `nginx` container after a renewal. |

The two refresh scripts exist because both memorial containers ran for weeks
with the literal placeholder text from the setup notes as their
`AWS_ACCESS_KEY_ID`. Every media upload failed and nothing said so — the admin
UI had no upload entry point yet, so nobody exercised the path. Hand-copying a
secret through a terminal is the failure mode (it is also what corrupted the
CloudFront private key), so the scripts remove the hand.

Fetching from SSM only moves *where* a value is typed. The validation is the
actual guard: each script refuses to write a value that is not shaped like the
credential it claims to be, and it validates **both** files before writing
either. A per-file check as it goes would leave the API holding a fresh key and
the worker holding a placeholder — harder to notice than either file being
plainly wrong.

Both run under `sudo` while Compose does not, so they restore the original
owner after rewriting; a root-owned env file would lock Compose out of a file it
could read a moment earlier.

## `tests/`

These run on a workstation or on the host and assert properties of this
repository, not of a live deployment:

```bash
bash tests/verify-container-log-limits.sh    # every service bounds its log
bash tests/verify-email-rate-limits.sh       # nginx limits + TRUST_PROXY reaches the container
bash tests/verify-certbot-webroot.sh         # the webroot renewal path is intact
bash tests/verify-memorial-credential-refresh.sh  # the refresh script refuses a bad value
```

`verify-container-log-limits.sh` asserts per service rather than grepping for
the `x-logging` anchor, so a service added later without a `logging:` key fails
the test instead of quietly growing an unbounded log.

## Container logs are bounded in two places

Docker's `json-file` driver has no size limit by default, and a full disk takes
down every site at once because they all share one box. The disk already filled
on 2026-08-12 from image accumulation — `scripts/docker-prune.sh` handles that
half; unbounded logs are the same failure by another route.

| Where | Covers | Takes effect |
| --- | --- | --- |
| `x-logging` anchor in `docker-compose.yml` | every service here | when a container is next recreated, which every deploy does |
| `/etc/docker/daemon.json` on the host | anything started outside Compose | when the Docker daemon next restarts |

Both say `max-size: 10m`, `max-file: 3` — 30 MB per container, ~210 MB in total
against a 58 GB disk. An explicit setting in Compose wins over the daemon
default, and they agree anyway.

Neither was applied by restarting the Docker daemon. A daemon restart bounces
every container, and log settings are baked in at container creation, so a
restart would have caused an outage *and* still left the existing containers
unbounded until each was recreated. Letting each service pick the limit up on
its next deploy costs nothing.

Validate a `daemon.json` edit before it can break anything — a malformed file
stops Docker from starting at all:

```bash
sudo dockerd --validate --config-file=/etc/docker/daemon.json
```

Check what a running container actually got:

```bash
docker inspect <container> --format '{{.HostConfig.LogConfig}}'
```

## Certificate renewal (Certbot webroot)

The Certbot webroot migration completed on 2026-08-02. Docker nginx serves
HTTP-01 challenges from `/var/lib/homelab-acme` and the Certbot deploy hook
reloads the `nginx` container after renewal.

Normal verification:

```bash
sudo certbot renew --dry-run
cd /opt/homelab-infra
docker compose logs --since 10m nginx
```

Do not use `certbot --nginx`: the host nginx service is intentionally stopped.
Do not repeat the migration's `certbot certonly --force-renewal` commands during
normal maintenance.

## Onboarding a new service

1. Add the Compose service, image, `mem_limit`, `logging: *default-logging`,
   and any needed nginx server block in this repository. Do not put secrets in
   Compose. Subtract the new limit from the memory budget above before choosing
   it — the host is already oversubscribed.
2. Validate locally with `docker compose config`, nginx syntax validation, and
   `bash tests/verify-container-log-limits.sh`, which fails on a service added
   without a `logging:` key.
3. Merge and pull this repository on the host.
4. Ensure the product repository's workflow builds/pushes its image under
   `ghcr.io/zenvoraai/`, passes the built SHA as the service's tag variable, and
   starts only its named service.
5. Add its hostname to the Certbot webroot procedure and verify a local ACME
   probe before requesting a certificate.
