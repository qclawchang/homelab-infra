# zenvora-admin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `ZenvoraAI/zenvora-admin`, a single-operator dashboard giving a unified, authenticated view across the operator's 6 repos for CI health, security posture, deployment drift on the Lightsail host, and backlog — plus a proactive email alert so regressions don't require someone to open the page.

**Architecture:** One Bun/Hono process serves both a built React SPA and a JSON API, backed by SQLite (sessions, response cache, alert-state — no separate database server). Login is a GitHub App user-to-server OAuth flow gated by an immutable-user-ID whitelist and a GitHub-account-2FA requirement. Docker container state is read through a `docker-socket-proxy` sidecar restricted to `GET /containers/*`, never a raw socket mount. Deploys as two more services in the existing `homelab-infra` `docker-compose.yml`, reverse-proxied by the existing nginx under `admin.valtou.com`.

**Tech Stack:** Bun + TypeScript + Hono (backend), `bun:sqlite`, `@octokit/rest`, `nodemailer`; Vite + React (frontend); `bun:test` + `@testing-library/react` for tests; Docker multi-stage build; GitHub Actions for CI/build.

**Spec:** `docs/superpowers/specs/2026-08-16-zenvora-admin-design.md` — this plan implements that spec exactly; read both together. Task boundaries below quote the spec's constraints verbatim where they constrain a task's implementation.

## Global Constraints

- Runtime is Bun, not Node — the spec sizes the backend `mem_limit` at ~128m, which depends on Bun's smaller footprint (Node + Octokit + TLS alone measured 70–110MB RSS in review).
- No database server process anywhere — `bun:sqlite` (embedded) is the only persistence.
- Auth is a **GitHub App** with fine-grained read-only permissions, never a classic OAuth App (`repo` scope grants write access, which the spec explicitly rejects).
- Every OAuth callback validates a CSRF `state` parameter generated per login attempt.
- Login requires both: (1) the authenticated GitHub user's **immutable numeric ID** matches a single whitelisted ID, and (2) that GitHub account's `two_factor_authentication` field is `true`. Both fail closed.
- Sessions live in SQLite; the cookie holds only an opaque session ID (httpOnly, secure, `SameSite=Strict`), never the GitHub token itself. Revocation is a row delete.
- Docker state is read only via the `docker-socket-proxy` sidecar's `GET /containers/*` — the backend never mounts `/var/run/docker.sock` directly.
- Deployment drift compares **tags**, not content digests (the repo's SHA-pinned services use the SHA as the tag itself), and reports two distinct signals per container, never conflated: `fault` (running tag ≠ pinned tag — only possible when a pin exists) and `upgrade_available` (GHCR has something newer than what's running). Two of the six containers (`dayandyou-staging`/`-prod`) have no pin mechanism at all and can only ever report `ok`/`upgrade_available`, never `fault`.
- Branch protection and secret scanning both render `unavailable_on_plan`, not a false negative — GitHub Free cannot enable either on private repos.
- GitHub API responses are cached in SQLite with a 60-second freshness window; a stale cached value is served (marked stale) if a live fetch fails, rather than the panel going blank.
- The proactive alert job fires an email only on an OK→bad transition, never repeatedly while a check stays bad, and covers both CI health and security alerts (Dependabot) — one check throwing must never silence the rest.
- Panel routes (Tasks 8–11, 14) register at paths *relative* to their `/api` mount point (e.g. `/repos`, not `/api/repos`) — Task 19 mounts the whole sub-app under `/api` via `app.route('/api', api)`, so an absolute path there would double the prefix.
- Every GitHub-facing endpoint response is wrapped as `{ data, stale }`, and the frontend actually renders the stale case — this is the concrete reason SQLite backs the cache instead of a plain in-memory TTL.
- No task in this plan performs a write action against GitHub or the Docker host — v1 is read-only observability plus the one email side-effect.
- New repo `ZenvoraAI/zenvora-admin` is private. It deploys into the existing `homelab-infra` `docker-compose.yml`/nginx, hostname `admin.valtou.com`, following that repo's existing "Onboarding a new service" checklist (`homelab-infra/README.md`).

---

## File Structure

```
zenvora-admin/                          (new repo)
  package.json, tsconfig.json, .env.example, Dockerfile
  .github/workflows/ci.yml
  src/
    types.ts                    Hono Variables type shared across routes
    server.ts                   app entrypoint — wires every route + starts the alert scheduler
    static.ts                   serves the built frontend
    db/
      schema.sql                 sessions / oauth_states / cache_entries / alert_state
      client.ts                  openDb()
    auth/
      crypto.ts                  encryptToken / decryptToken (AES-256-GCM)
      oauth.ts                   login + callback routes, GitHubOAuthClient
      session.ts                 requireAuth middleware, logout route
    github/
      client.ts                  createOctokit(token)
      cache.ts                   cachedFetch() — SQLite-backed, stale-on-error
      repos.ts                   repo grid
      ci.ts                      CI health + Actions quota
      security.ts                 2FA / Dependabot / secrets / plan-unavailable flags
      backlog.ts                  PRs / issues / stale branches
    deployment/
      dockerProxy.ts              docker-socket-proxy HTTP client
      drift.ts                    compareDrift() pure logic + GHCR version lookup
      route.ts                    /api/deployment, combines the two above
    alerts/
      job.ts                      recordCheckAndGetTransition() — SQLite-backed state diff
      email.ts                    sendAlertEmail() via nodemailer
      scheduler.ts                 runAlertChecks() / startAlertScheduler()
  tests/                          mirrors src/, one test file per module above
  frontend/
    package.json, vite.config.ts, index.html, bunfig.toml, happy-dom-register.ts
    src/
      main.tsx, App.tsx, api.ts
      hooks/useApiData.ts
      styles/tokens.css
      components/
        RepoGrid.tsx, CiHealth.tsx, SecurityPosture.tsx, DeploymentDrift.tsx, Backlog.tsx, LoginGate.tsx
        __tests__/panels.test.tsx, __tests__/loginGate.test.tsx

homelab-infra/                          (this repo — modified, not created)
  docker-compose.yml                    + docker-socket-proxy, + zenvora-admin services
  nginx/conf.d/admin.valtou.com.conf     (new)
  README.md                              services table + hostname list updated
```

---

### Task 1: Repo scaffold, health check, shared types, env template

**Files:**
- Create: `zenvora-admin/package.json`, `zenvora-admin/tsconfig.json`, `zenvora-admin/.env.example`
- Create: `zenvora-admin/src/types.ts`
- Create: `zenvora-admin/src/server.ts`
- Test: `zenvora-admin/tests/server.test.ts`

**Interfaces:**
- Produces: `AppVariables { githubToken: string; userId: number }` (from `src/types.ts`) — every later route module types its `Hono` instance as `Hono<{ Variables: AppVariables }>`.
- Produces: `createApp(): Hono<{ Variables: AppVariables }>` (from `src/server.ts`) — Task 19 changes this signature to `{ app, db }`; every task before Task 19 uses `new Hono()` directly in its own tests rather than calling `createApp()`.

- [ ] **Step 1: Create the GitHub repo**

```bash
gh repo create ZenvoraAI/zenvora-admin --private --description "Single-operator admin dashboard for ZenvoraAI repos and the Lightsail host"
git clone git@github.com:ZenvoraAI/zenvora-admin.git
cd zenvora-admin
```

- [ ] **Step 2: Add `package.json`**

```json
{
  "name": "zenvora-admin",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "bun run --watch src/server.ts",
    "start": "bun run src/server.ts",
    "test": "bun test"
  },
  "dependencies": {
    "hono": "^4.6.0",
    "@octokit/rest": "^21.0.0",
    "nodemailer": "^6.9.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "bun-types": "^1.1.0",
    "@types/nodemailer": "^6.4.0"
  }
}
```

- [ ] **Step 3: Add `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["bun-types"]
  },
  "include": ["src", "tests"]
}
```

- [ ] **Step 4: Add `.env.example`** — every env var any later task reads, defined once now

```bash
PORT=3100
DB_PATH=data/zenvora-admin.sqlite

GITHUB_APP_CLIENT_ID=
GITHUB_APP_CLIENT_SECRET=
GITHUB_APP_REDIRECT_URI=https://admin.valtou.com/auth/callback
WHITELISTED_GITHUB_USER_ID=
TOKEN_ENCRYPTION_KEY=
ALERT_CHECK_TOKEN=

DOCKER_PROXY_URL=http://127.0.0.1:2375

SMTP_HOST=
SMTP_PORT=587
SMTP_SECURE=true
SMTP_USER=
SMTP_PASS=
ALERT_EMAIL_FROM=
ALERT_EMAIL_TO=

FAMILY_API_TAG=
SECUREVAULT_API_TAG=
MEMORIAL_API_TAG=
MEMORIAL_WORKER_TAG=
```

These four reuse the *exact same env var names* `docker-compose.yml` already substitutes into the sibling services' own image lines (`${FAMILY_API_TAG:-latest}` etc.) — when zenvora-admin's `environment:` passthrough resolves them from the same project `.env`/shell source, its notion of "what's pinned" and the actual running deploy's pin are automatically the same value, with no separate copy to keep in sync. There are no `DAYANDYOU_*_TAG` entries: `dayandyou-staging`/`dayandyou-prod` use static, unpinned tags (`:staging`/`:release`) with no per-deploy override variable in `docker-compose.yml` at all — Task 13/19 treat those two as having no pin to check.

- [ ] **Step 5: Install dependencies**

```bash
bun install
```

- [ ] **Step 6: Create `src/types.ts`**

```ts
export interface AppVariables {
  githubToken: string;
  userId: number;
}
```

- [ ] **Step 7: Write the failing test for the health check**

```ts
// tests/server.test.ts
import { describe, test, expect } from 'bun:test';
import { createApp } from '../src/server';

describe('GET /healthz', () => {
  test('returns ok status', async () => {
    const app = createApp();
    const res = await app.request('/healthz');
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ status: 'ok' });
  });
});
```

- [ ] **Step 8: Run it, confirm it fails**

Run: `bun test tests/server.test.ts`
Expected: FAIL — `src/server.ts` does not exist yet.

- [ ] **Step 9: Create `src/server.ts`**

```ts
import { Hono } from 'hono';
import type { AppVariables } from './types';

export function createApp() {
  const app = new Hono<{ Variables: AppVariables }>();
  app.get('/healthz', (c) => c.json({ status: 'ok' }));
  return app;
}

if (import.meta.main) {
  const app = createApp();
  const port = Number(process.env.PORT ?? 3100);
  Bun.serve({ fetch: app.fetch, port });
  console.log(`zenvora-admin listening on :${port}`);
}
```

- [ ] **Step 10: Run it, confirm it passes**

Run: `bun test tests/server.test.ts`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat: scaffold zenvora-admin with a health check endpoint"
```

---

### Task 2: SQLite schema and db client

**Files:**
- Create: `src/db/schema.sql`
- Create: `src/db/client.ts`
- Test: `tests/db/client.test.ts`

**Interfaces:**
- Produces: `openDb(path?: string): Database` (from `bun:sqlite`) — every later task that touches persistence calls this, passing `':memory:'` in tests.

- [ ] **Step 1: Write the failing test**

```ts
// tests/db/client.test.ts
import { describe, test, expect } from 'bun:test';
import { openDb } from '../../src/db/client';

describe('openDb', () => {
  test('creates all four tables on a fresh in-memory database', () => {
    const db = openDb(':memory:');
    const tables = db
      .query("SELECT name FROM sqlite_master WHERE type = 'table' ORDER BY name")
      .all() as { name: string }[];
    expect(tables.map((t) => t.name)).toEqual([
      'alert_state',
      'cache_entries',
      'oauth_states',
      'sessions',
    ]);
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/db/client.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/db/schema.sql`**

```sql
CREATE TABLE IF NOT EXISTS sessions (
  id TEXT PRIMARY KEY,
  user_id INTEGER NOT NULL,
  encrypted_token TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  expires_at INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS oauth_states (
  state TEXT PRIMARY KEY,
  created_at INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS cache_entries (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  fetched_at INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS alert_state (
  check_key TEXT PRIMARY KEY,
  status TEXT NOT NULL,
  updated_at INTEGER NOT NULL
);
```

- [ ] **Step 4: Write `src/db/client.ts`**

```ts
import { Database } from 'bun:sqlite';
import { readFileSync } from 'node:fs';
import { fileURLToPath } from 'node:url';
import { dirname, join } from 'node:path';

const SCHEMA_PATH = join(dirname(fileURLToPath(import.meta.url)), 'schema.sql');

export function openDb(path: string = process.env.DB_PATH ?? 'data/zenvora-admin.sqlite'): Database {
  const db = new Database(path, { create: true });
  db.exec('PRAGMA journal_mode = WAL;');
  db.exec(readFileSync(SCHEMA_PATH, 'utf-8'));
  return db;
}
```

- [ ] **Step 5: Run it, confirm it passes**

Run: `bun test tests/db/client.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: add SQLite schema and db client"
```

---

### Task 3: Token encryption helpers

**Files:**
- Create: `src/auth/crypto.ts`
- Test: `tests/auth/crypto.test.ts`

**Interfaces:**
- Produces: `encryptToken(plaintext: string): string`, `decryptToken(encoded: string): string` — used by Task 5 (session creation) and Task 6 (session reads).

- [ ] **Step 1: Write the failing test**

```ts
// tests/auth/crypto.test.ts
import { describe, test, expect, beforeAll } from 'bun:test';
import { randomBytes } from 'node:crypto';
import { encryptToken, decryptToken } from '../../src/auth/crypto';

beforeAll(() => {
  process.env.TOKEN_ENCRYPTION_KEY = randomBytes(32).toString('base64');
});

describe('token encryption', () => {
  test('round-trips a token', () => {
    const token = 'ghu_exampleUserToServerToken1234567890';
    const encrypted = encryptToken(token);
    expect(encrypted).not.toBe(token);
    expect(decryptToken(encrypted)).toBe(token);
  });

  test('rejects a tampered ciphertext', () => {
    const encrypted = encryptToken('ghu_anotherToken');
    const raw = Buffer.from(encrypted, 'base64');
    raw[raw.length - 1] ^= 0xff;
    expect(() => decryptToken(raw.toString('base64'))).toThrow();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/auth/crypto.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/auth/crypto.ts`**

```ts
import { randomBytes, createCipheriv, createDecipheriv } from 'node:crypto';

const ALGORITHM = 'aes-256-gcm';

function loadKey(): Buffer {
  const b64 = process.env.TOKEN_ENCRYPTION_KEY;
  if (!b64) throw new Error('TOKEN_ENCRYPTION_KEY is not set');
  const key = Buffer.from(b64, 'base64');
  if (key.length !== 32) throw new Error('TOKEN_ENCRYPTION_KEY must decode to 32 bytes');
  return key;
}

export function encryptToken(plaintext: string): string {
  const key = loadKey();
  const iv = randomBytes(12);
  const cipher = createCipheriv(ALGORITHM, key, iv);
  const ciphertext = Buffer.concat([cipher.update(plaintext, 'utf-8'), cipher.final()]);
  const authTag = cipher.getAuthTag();
  return Buffer.concat([iv, authTag, ciphertext]).toString('base64');
}

export function decryptToken(encoded: string): string {
  const key = loadKey();
  const raw = Buffer.from(encoded, 'base64');
  const iv = raw.subarray(0, 12);
  const authTag = raw.subarray(12, 28);
  const ciphertext = raw.subarray(28);
  const decipher = createDecipheriv(ALGORITHM, key, iv);
  decipher.setAuthTag(authTag);
  return Buffer.concat([decipher.update(ciphertext), decipher.final()]).toString('utf-8');
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/auth/crypto.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add AES-256-GCM token encryption helpers"
```

---

### Task 4: OAuth login route

**Files:**
- Create: `src/auth/oauth.ts` (login portion — callback added in Task 5)
- Test: `tests/auth/oauth.test.ts` (login portion)

**Interfaces:**
- Consumes: `openDb` (Task 2).
- Produces: `createOAuthLoginRoute(app: Hono, db: Database): void`, `pruneExpiredStates(db: Database): void` — Task 5 adds to the same file and reuses `pruneExpiredStates`.

- [ ] **Step 1: Write the failing test**

```ts
// tests/auth/oauth.test.ts
import { describe, test, expect, beforeEach } from 'bun:test';
import { Hono } from 'hono';
import { openDb } from '../../src/db/client';
import { createOAuthLoginRoute } from '../../src/auth/oauth';

describe('GET /auth/login', () => {
  beforeEach(() => {
    process.env.GITHUB_APP_CLIENT_ID = 'Iv1.testclientid';
    process.env.GITHUB_APP_REDIRECT_URI = 'https://admin.valtou.com/auth/callback';
  });

  test('redirects to GitHub authorize URL with a fresh state', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    createOAuthLoginRoute(app, db);

    const res = await app.request('/auth/login', { redirect: 'manual' });

    expect(res.status).toBe(302);
    const location = new URL(res.headers.get('location')!);
    expect(location.origin + location.pathname).toBe('https://github.com/login/oauth/authorize');
    expect(location.searchParams.get('client_id')).toBe('Iv1.testclientid');
    expect(location.searchParams.get('state')).toMatch(/^[0-9a-f]{48}$/);

    const stored = db
      .query('SELECT state FROM oauth_states WHERE state = ?')
      .get(location.searchParams.get('state')) as { state: string } | null;
    expect(stored?.state).toBe(location.searchParams.get('state'));

    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/auth/oauth.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/auth/oauth.ts`**

```ts
import type { Hono } from 'hono';
import type { Database } from 'bun:sqlite';
import { randomBytes } from 'node:crypto';

const STATE_TTL_MS = 10 * 60 * 1000;

export function pruneExpiredStates(db: Database) {
  db.query('DELETE FROM oauth_states WHERE created_at < ?').run(Date.now() - STATE_TTL_MS);
}

export function createOAuthLoginRoute(app: Hono, db: Database) {
  app.get('/auth/login', (c) => {
    const state = randomBytes(24).toString('hex');
    db.query('INSERT INTO oauth_states (state, created_at) VALUES (?, ?)').run(state, Date.now());

    const clientId = process.env.GITHUB_APP_CLIENT_ID;
    if (!clientId) return c.text('GITHUB_APP_CLIENT_ID is not configured', 500);

    const authorizeUrl = new URL('https://github.com/login/oauth/authorize');
    authorizeUrl.searchParams.set('client_id', clientId);
    authorizeUrl.searchParams.set('redirect_uri', process.env.GITHUB_APP_REDIRECT_URI ?? '');
    authorizeUrl.searchParams.set('state', state);

    return c.redirect(authorizeUrl.toString());
  });
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/auth/oauth.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add OAuth login route with CSRF state issuance"
```

---

### Task 5: OAuth callback route

**Files:**
- Modify: `src/auth/oauth.ts` (append callback + client factory)
- Modify: `tests/auth/oauth.test.ts` (append callback tests)

**Interfaces:**
- Consumes: `pruneExpiredStates` (this file, Task 4), `encryptToken` (Task 3), `openDb` (Task 2).
- Produces: `GitHubOAuthClient { exchangeCode(code): Promise<{accessToken}>, fetchUser(accessToken): Promise<{id, login, two_factor_authentication}> }`, `createRealGitHubOAuthClient(): GitHubOAuthClient`, `createOAuthCallbackRoute(app, db, client): void` — Task 19 wires `createRealGitHubOAuthClient()` into the real app.

- [ ] **Step 1: Write the failing tests** (append to `tests/auth/oauth.test.ts`)

```ts
import { randomBytes } from 'node:crypto';
import { createOAuthCallbackRoute, type GitHubOAuthClient } from '../../src/auth/oauth';

function fakeClient(user: { id: number; login: string; two_factor_authentication: boolean | null }): GitHubOAuthClient {
  return {
    async exchangeCode() { return { accessToken: 'ghu_fake' }; },
    async fetchUser() { return user; },
  };
}

describe('GET /auth/callback', () => {
  beforeEach(() => {
    process.env.TOKEN_ENCRYPTION_KEY = randomBytes(32).toString('base64');
    process.env.WHITELISTED_GITHUB_USER_ID = '4242';
  });

  test('creates a session for the whitelisted user with 2FA on', async () => {
    const db = openDb(':memory:');
    db.query('INSERT INTO oauth_states (state, created_at) VALUES (?, ?)').run('good-state', Date.now());

    const app = new Hono();
    createOAuthCallbackRoute(app, db, fakeClient({ id: 4242, login: 'qclawchang', two_factor_authentication: true }));

    const res = await app.request('/auth/callback?code=abc&state=good-state', { redirect: 'manual' });

    expect(res.status).toBe(302);
    expect(res.headers.get('location')).toBe('/');
    expect(res.headers.get('set-cookie') ?? '').toContain('zenvora_session=');
    expect(res.headers.get('set-cookie') ?? '').toContain('HttpOnly');
    expect(db.query('SELECT * FROM sessions').all()).toHaveLength(1);

    db.close();
  });

  test('rejects a non-whitelisted user id by redirecting to the login page with a reason', async () => {
    const db = openDb(':memory:');
    db.query('INSERT INTO oauth_states (state, created_at) VALUES (?, ?)').run('good-state', Date.now());
    const app = new Hono();
    createOAuthCallbackRoute(app, db, fakeClient({ id: 9999, login: 'someone-else', two_factor_authentication: true }));

    const res = await app.request('/auth/callback?code=abc&state=good-state', { redirect: 'manual' });

    expect(res.status).toBe(302);
    const location = new URL(res.headers.get('location')!, 'https://admin.valtou.com');
    expect(location.pathname).toBe('/login');
    expect(location.searchParams.get('error')).toContain('not authorized');
    expect(db.query('SELECT * FROM sessions').all()).toHaveLength(0);
    db.close();
  });

  test('rejects a whitelisted user without 2FA enabled by redirecting to the login page with a reason', async () => {
    const db = openDb(':memory:');
    db.query('INSERT INTO oauth_states (state, created_at) VALUES (?, ?)').run('good-state', Date.now());
    const app = new Hono();
    createOAuthCallbackRoute(app, db, fakeClient({ id: 4242, login: 'qclawchang', two_factor_authentication: false }));

    const res = await app.request('/auth/callback?code=abc&state=good-state', { redirect: 'manual' });

    expect(res.status).toBe(302);
    const location = new URL(res.headers.get('location')!, 'https://admin.valtou.com');
    expect(location.pathname).toBe('/login');
    expect(location.searchParams.get('error')).toContain('two-factor');
    expect(db.query('SELECT * FROM sessions').all()).toHaveLength(0);
    db.close();
  });

  test('rejects an invalid or already-used state by redirecting to the login page with a reason', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    createOAuthCallbackRoute(app, db, fakeClient({ id: 4242, login: 'qclawchang', two_factor_authentication: true }));

    const res = await app.request('/auth/callback?code=abc&state=never-issued', { redirect: 'manual' });

    expect(res.status).toBe(302);
    const location = new URL(res.headers.get('location')!, 'https://admin.valtou.com');
    expect(location.pathname).toBe('/login');
    expect(location.searchParams.get('error')).toContain('state');
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/auth/oauth.test.ts`
Expected: FAIL — `createOAuthCallbackRoute` not exported yet.

- [ ] **Step 3: Append to `src/auth/oauth.ts`**

```ts
import { encryptToken } from './crypto';

export interface GitHubOAuthClient {
  exchangeCode(code: string): Promise<{ accessToken: string }>;
  fetchUser(accessToken: string): Promise<{ id: number; login: string; two_factor_authentication: boolean | null }>;
}

export function createRealGitHubOAuthClient(): GitHubOAuthClient {
  return {
    async exchangeCode(code) {
      const res = await fetch('https://github.com/login/oauth/access_token', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
        body: JSON.stringify({
          client_id: process.env.GITHUB_APP_CLIENT_ID,
          client_secret: process.env.GITHUB_APP_CLIENT_SECRET,
          code,
          redirect_uri: process.env.GITHUB_APP_REDIRECT_URI,
        }),
      });
      const body = (await res.json()) as { access_token?: string; error?: string };
      if (!body.access_token) throw new Error(`GitHub token exchange failed: ${body.error ?? 'unknown error'}`);
      return { accessToken: body.access_token };
    },
    async fetchUser(accessToken) {
      const res = await fetch('https://api.github.com/user', {
        headers: { Authorization: `Bearer ${accessToken}`, Accept: 'application/vnd.github+json' },
      });
      if (!res.ok) throw new Error(`GitHub user lookup failed: ${res.status}`);
      return (await res.json()) as { id: number; login: string; two_factor_authentication: boolean | null };
    },
  };
}

function loginRedirect(reason: string) {
  return `/login?error=${encodeURIComponent(reason)}`;
}

export function createOAuthCallbackRoute(app: Hono, db: Database, client: GitHubOAuthClient) {
  app.get('/auth/callback', async (c) => {
    const code = c.req.query('code');
    const state = c.req.query('state');
    if (!code || !state) return c.redirect(loginRedirect('missing code or state'));

    pruneExpiredStates(db);
    const stateRow = db.query('SELECT state FROM oauth_states WHERE state = ?').get(state);
    if (!stateRow) return c.redirect(loginRedirect('invalid or expired login state, please try again'));
    db.query('DELETE FROM oauth_states WHERE state = ?').run(state);

    let accessToken: string;
    try {
      ({ accessToken } = await client.exchangeCode(code));
    } catch {
      return c.redirect(loginRedirect('GitHub token exchange failed, please try again'));
    }

    const user = await client.fetchUser(accessToken);

    const whitelistedId = Number(process.env.WHITELISTED_GITHUB_USER_ID);
    if (user.id !== whitelistedId) {
      return c.redirect(loginRedirect('this GitHub account is not authorized for zenvora-admin'));
    }
    if (user.two_factor_authentication !== true) {
      return c.redirect(loginRedirect('two-factor authentication must be enabled on this GitHub account'));
    }

    const sessionId = randomBytes(24).toString('hex');
    const now = Date.now();
    const twelveHoursMs = 12 * 60 * 60 * 1000;
    db.query(
      'INSERT INTO sessions (id, user_id, encrypted_token, created_at, expires_at) VALUES (?, ?, ?, ?, ?)'
    ).run(sessionId, user.id, encryptToken(accessToken), now, now + twelveHoursMs);

    c.header(
      'Set-Cookie',
      `zenvora_session=${sessionId}; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=${twelveHoursMs / 1000}`
    );
    return c.redirect('/');
  });
}
```

`SameSite=Strict` rather than `Lax`: this is a single-operator tool with no legitimate cross-site entry point, so there's no login flow `Strict` would break, and it's strictly better CSRF protection than `Lax`.

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/auth/oauth.test.ts`
Expected: PASS — 5 tests (1 login + 4 callback).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add OAuth callback with user-ID whitelist and 2FA gate"
```

---

### Task 6: Auth middleware and logout

**Files:**
- Create: `src/auth/session.ts`
- Test: `tests/auth/session.test.ts`

**Interfaces:**
- Consumes: `decryptToken` (Task 3), `openDb` (Task 2).
- Produces: `requireAuth(db: Database)` (Hono middleware, sets `githubToken`/`userId` on context), `createLogoutRoute(app, db)` — Task 19 mounts `requireAuth` in front of every `/api/*` route.

- [ ] **Step 1: Write the failing test**

```ts
// tests/auth/session.test.ts
import { describe, test, expect, beforeEach } from 'bun:test';
import { randomBytes } from 'node:crypto';
import { Hono } from 'hono';
import { openDb } from '../../src/db/client';
import { requireAuth, createLogoutRoute } from '../../src/auth/session';
import { encryptToken } from '../../src/auth/crypto';

describe('requireAuth middleware', () => {
  beforeEach(() => {
    process.env.TOKEN_ENCRYPTION_KEY = randomBytes(32).toString('base64');
  });

  test('rejects a request with no session cookie', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    app.use('/protected', requireAuth(db));
    app.get('/protected', (c) => c.text('ok'));

    const res = await app.request('/protected');
    expect(res.status).toBe(401);
    db.close();
  });

  test('rejects an expired session', async () => {
    const db = openDb(':memory:');
    db.query(
      'INSERT INTO sessions (id, user_id, encrypted_token, created_at, expires_at) VALUES (?, ?, ?, ?, ?)'
    ).run('sess-1', 4242, encryptToken('ghu_x'), Date.now() - 1000, Date.now() - 500);

    const app = new Hono();
    app.use('/protected', requireAuth(db));
    app.get('/protected', (c) => c.text('ok'));

    const res = await app.request('/protected', { headers: { cookie: 'zenvora_session=sess-1' } });
    expect(res.status).toBe(401);
    db.close();
  });

  test('allows a valid session through and exposes the token', async () => {
    const db = openDb(':memory:');
    db.query(
      'INSERT INTO sessions (id, user_id, encrypted_token, created_at, expires_at) VALUES (?, ?, ?, ?, ?)'
    ).run('sess-2', 4242, encryptToken('ghu_x'), Date.now(), Date.now() + 60_000);

    const app = new Hono();
    app.use('/protected', requireAuth(db));
    app.get('/protected', (c) => c.text(`token:${c.get('githubToken')}`));

    const res = await app.request('/protected', { headers: { cookie: 'zenvora_session=sess-2' } });
    expect(res.status).toBe(200);
    expect(await res.text()).toBe('token:ghu_x');
    db.close();
  });
});

describe('POST /auth/logout', () => {
  test('deletes the session row and clears the cookie', async () => {
    process.env.TOKEN_ENCRYPTION_KEY = randomBytes(32).toString('base64');
    const db = openDb(':memory:');
    db.query(
      'INSERT INTO sessions (id, user_id, encrypted_token, created_at, expires_at) VALUES (?, ?, ?, ?, ?)'
    ).run('sess-3', 4242, encryptToken('ghu_x'), Date.now(), Date.now() + 60_000);

    const app = new Hono();
    createLogoutRoute(app, db);

    const res = await app.request('/auth/logout', {
      method: 'POST',
      headers: { cookie: 'zenvora_session=sess-3' },
    });

    expect(res.status).toBe(200);
    expect(db.query('SELECT * FROM sessions WHERE id = ?').get('sess-3')).toBeNull();
    expect(res.headers.get('set-cookie')).toContain('Max-Age=0');
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/auth/session.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/auth/session.ts`**

```ts
import type { Context, Next, Hono } from 'hono';
import type { Database } from 'bun:sqlite';
import { decryptToken } from './crypto';

interface SessionRow {
  id: string;
  user_id: number;
  encrypted_token: string;
  created_at: number;
  expires_at: number;
}

function parseCookie(header: string | undefined, name: string): string | undefined {
  if (!header) return undefined;
  for (const part of header.split(';')) {
    const [key, ...rest] = part.trim().split('=');
    if (key === name) return rest.join('=');
  }
  return undefined;
}

export function requireAuth(db: Database) {
  return async (c: Context, next: Next) => {
    const sessionId = parseCookie(c.req.header('cookie'), 'zenvora_session');
    if (!sessionId) return c.text('not authenticated', 401);

    const row = db.query('SELECT * FROM sessions WHERE id = ?').get(sessionId) as SessionRow | null;
    if (!row || row.expires_at < Date.now()) return c.text('session expired or invalid', 401);

    c.set('githubToken', decryptToken(row.encrypted_token));
    c.set('userId', row.user_id);
    await next();
  };
}

export function createLogoutRoute(app: Hono, db: Database) {
  app.post('/auth/logout', (c) => {
    const sessionId = parseCookie(c.req.header('cookie'), 'zenvora_session');
    if (sessionId) db.query('DELETE FROM sessions WHERE id = ?').run(sessionId);
    c.header('Set-Cookie', 'zenvora_session=; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=0');
    return c.text('logged out');
  });
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/auth/session.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add session-gating middleware and logout route"
```

---

### Task 7: SQLite-backed GitHub cache and authenticated client factory

**Files:**
- Create: `src/github/cache.ts`
- Create: `src/github/client.ts`
- Test: `tests/github/cache.test.ts`
- Test: `tests/github/client.test.ts`

**Interfaces:**
- Produces: `cachedFetch<T>(db, key, fetcher): Promise<{data: T, stale: boolean}>` — every panel endpoint (Tasks 8–11, 14) wraps its GitHub calls in this.
- Produces: `createOctokit(token: string): Octokit` — used everywhere an authenticated GitHub call is made.

- [ ] **Step 1: Write the failing cache tests**

```ts
// tests/github/cache.test.ts
import { describe, test, expect, mock } from 'bun:test';
import { openDb } from '../../src/db/client';
import { cachedFetch } from '../../src/github/cache';

describe('cachedFetch', () => {
  test('returns a fresh cached value without calling the fetcher', async () => {
    const db = openDb(':memory:');
    db.query('INSERT INTO cache_entries (key, value, fetched_at) VALUES (?, ?, ?)').run(
      'repos', JSON.stringify({ count: 6 }), Date.now()
    );
    const fetcher = mock(async () => ({ count: 999 }));

    const result = await cachedFetch(db, 'repos', fetcher);

    expect(result).toEqual({ data: { count: 6 }, stale: false });
    expect(fetcher).not.toHaveBeenCalled();
    db.close();
  });

  test('calls the fetcher and caches the result when there is no entry', async () => {
    const db = openDb(':memory:');
    const fetcher = mock(async () => ({ count: 6 }));

    const result = await cachedFetch(db, 'repos', fetcher);

    expect(result).toEqual({ data: { count: 6 }, stale: false });
    expect(fetcher).toHaveBeenCalledTimes(1);
    const row = db.query('SELECT value FROM cache_entries WHERE key = ?').get('repos') as { value: string };
    expect(JSON.parse(row.value)).toEqual({ count: 6 });
    db.close();
  });

  test('falls back to a stale cached value when the fetcher throws', async () => {
    const db = openDb(':memory:');
    db.query('INSERT INTO cache_entries (key, value, fetched_at) VALUES (?, ?, ?)').run(
      'repos', JSON.stringify({ count: 6 }), Date.now() - 120_000
    );
    const fetcher = mock(async () => { throw new Error('rate limited'); });

    const result = await cachedFetch(db, 'repos', fetcher);

    expect(result).toEqual({ data: { count: 6 }, stale: true });
    db.close();
  });

  test('rethrows when the fetcher fails and there is nothing cached', async () => {
    const db = openDb(':memory:');
    const fetcher = mock(async () => { throw new Error('rate limited'); });

    await expect(cachedFetch(db, 'repos', fetcher)).rejects.toThrow('rate limited');
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/github/cache.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/github/cache.ts`**

```ts
import type { Database } from 'bun:sqlite';

const FRESHNESS_MS = 60_000;

export interface CachedFetchResult<T> {
  data: T;
  stale: boolean;
}

export async function cachedFetch<T>(
  db: Database,
  key: string,
  fetcher: () => Promise<T>
): Promise<CachedFetchResult<T>> {
  const row = db.query('SELECT value, fetched_at FROM cache_entries WHERE key = ?').get(key) as
    | { value: string; fetched_at: number }
    | null;

  if (row && Date.now() - row.fetched_at < FRESHNESS_MS) {
    return { data: JSON.parse(row.value) as T, stale: false };
  }

  try {
    const data = await fetcher();
    db.query(
      'INSERT INTO cache_entries (key, value, fetched_at) VALUES (?, ?, ?) ' +
        'ON CONFLICT(key) DO UPDATE SET value = excluded.value, fetched_at = excluded.fetched_at'
    ).run(key, JSON.stringify(data), Date.now());
    return { data, stale: false };
  } catch (err) {
    if (row) return { data: JSON.parse(row.value) as T, stale: true };
    throw err;
  }
}
```

- [ ] **Step 4: Run cache test, confirm it passes**

Run: `bun test tests/github/cache.test.ts`
Expected: PASS

- [ ] **Step 5: Write the failing client test**

```ts
// tests/github/client.test.ts
import { describe, test, expect } from 'bun:test';
import { createOctokit } from '../../src/github/client';

describe('createOctokit', () => {
  test('creates an Octokit instance configured with the given token', () => {
    const client = createOctokit('ghu_example');
    expect(client).toBeDefined();
    expect(typeof client.request).toBe('function');
  });
});
```

- [ ] **Step 6: Write `src/github/client.ts`**

```ts
import { Octokit } from '@octokit/rest';

export function createOctokit(token: string): Octokit {
  return new Octokit({ auth: token });
}
```

- [ ] **Step 7: Run all, confirm pass**

Run: `bun test tests/github/`
Expected: PASS — 5 tests total.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: add SQLite-backed GitHub response cache and Octokit factory"
```

---

### Task 8: Repo grid endpoint

**Files:**
- Create: `src/github/repos.ts`
- Test: `tests/github/repos.test.ts`

**Interfaces:**
- Consumes: `cachedFetch` (Task 7).
- Produces: `RepoSummary`, `fetchRepoGrid(octokit, repos): Promise<RepoSummary[]>`, `createReposRoute(app, db, octokitFor, repos)` — mounted at `/repos` (Task 19 mounts this sub-app under `/api`, giving `/api/repos` — routes here use paths *relative to that mount*, not the full `/api/...` path, otherwise Task 19's `app.route('/api', api)` would double the prefix).

- [ ] **Step 1: Write the failing test**

```ts
// tests/github/repos.test.ts
import { describe, test, expect } from 'bun:test';
import { Hono } from 'hono';
import { openDb } from '../../src/db/client';
import { createReposRoute } from '../../src/github/repos';

function fakeOctokit() {
  return {
    repos: {
      get: async ({ owner, repo }: { owner: string; repo: string }) => {
        const fixtures: Record<string, any> = {
          'qclawchang/homelab-infra': { name: 'homelab-infra', full_name: 'qclawchang/homelab-infra', private: false, pushed_at: '2026-08-16T10:47:14Z', language: 'Shell', description: null },
          'ZenvoraAI/day-and-you': { name: 'day-and-you', full_name: 'ZenvoraAI/day-and-you', private: true, pushed_at: '2026-08-16T10:48:16Z', language: 'TypeScript', description: null },
        };
        return { data: fixtures[`${owner}/${repo}`] };
      },
    },
  };
}

describe('GET /repos', () => {
  test('returns exactly the passed-in repos, not the operator\'s full account', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); c.set('userId', 4242); await next(); });
    createReposRoute(app, db, () => fakeOctokit() as any, [
      { owner: 'qclawchang', repo: 'homelab-infra' },
      { owner: 'ZenvoraAI', repo: 'day-and-you' },
    ]);

    const res = await app.request('/repos');
    const body = await res.json();

    expect(res.status).toBe(200);
    expect(body.data.map((r: any) => r.fullName)).toEqual([
      'qclawchang/homelab-infra',
      'ZenvoraAI/day-and-you',
    ]);
    expect(body.stale).toBe(false);
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/github/repos.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/github/repos.ts`**

```ts
import type { Hono } from 'hono';
import type { Database } from 'bun:sqlite';
import type { Octokit } from '@octokit/rest';
import { cachedFetch } from './cache';

export interface RepoSummary {
  name: string;
  fullName: string;
  private: boolean;
  pushedAt: string | null;
  language: string | null;
  description: string | null;
}

export async function fetchRepoGrid(octokit: Octokit, repos: { owner: string; repo: string }[]): Promise<RepoSummary[]> {
  const results = await Promise.all(repos.map(({ owner, repo }) => octokit.repos.get({ owner, repo })));

  return results.map(({ data: repo }: any) => ({
    name: repo.name,
    fullName: repo.full_name,
    private: repo.private,
    pushedAt: repo.pushed_at,
    language: repo.language,
    description: repo.description,
  }));
}

export function createReposRoute(
  app: Hono,
  db: Database,
  octokitFor: (token: string) => Octokit,
  repos: { owner: string; repo: string }[]
) {
  app.get('/repos', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, `repos:${c.get('userId')}`, () => fetchRepoGrid(octokit, repos));
    return c.json({ data: result.data, stale: result.stale });
  });
}
```

Fetching each of the 6 repos explicitly via `repos.get` (rather than listing everything the operator owns/belongs to) keeps this panel scoped to exactly the fixed repo set the rest of the plan uses — the previous draft ignored its own `repos` parameter and would have silently listed the operator's entire account.

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/github/repos.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add repo grid endpoint"
```

---

### Task 9: CI health endpoint

**Files:**
- Create: `src/github/ci.ts`
- Test: `tests/github/ci.test.ts`

**Interfaces:**
- Consumes: `cachedFetch` (Task 7).
- Produces: `RepoCiStatus`, `CiHealth`, `fetchCiHealth(octokit, repos)`, `createCiRoute(app, db, octokitFor, repos)` — mounted at `/ci` (becomes `/api/ci` once Task 19 mounts this sub-app under `/api` — see Task 8's note on relative paths).

- [ ] **Step 1: Write the failing test**

```ts
// tests/github/ci.test.ts
import { describe, test, expect } from 'bun:test';
import { Hono } from 'hono';
import { openDb } from '../../src/db/client';
import { createCiRoute } from '../../src/github/ci';

function fakeOctokit(runsByRepo: Record<string, any>) {
  return {
    actions: {
      listWorkflowRunsForRepo: async ({ owner, repo }: { owner: string; repo: string }) => ({
        data: { workflow_runs: runsByRepo[`${owner}/${repo}`] ?? [] },
      }),
    },
    request: async (route: string) => {
      if (route === 'GET /organizations/{org}/settings/billing/usage') {
        return { data: { usageItems: [{ product: 'actions', unitType: 'Minutes', quantity: 1700 }] }, includedMinutes: 2000 } as any;
      }
      throw new Error(`unexpected request: ${route}`);
    },
  };
}

describe('GET /ci', () => {
  test('reports latest run status per repo and flags quota warning past 80%', async () => {
    const db = openDb(':memory:');
    const octokit = fakeOctokit({
      'ZenvoraAI/day-and-you': [{ conclusion: 'failure', status: 'completed', html_url: 'https://x/1' }],
      'qclawchang/homelab-infra': [{ conclusion: 'success', status: 'completed', html_url: 'https://x/2' }],
    });

    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); await next(); });
    createCiRoute(app, db, () => octokit as any, [
      { owner: 'ZenvoraAI', repo: 'day-and-you' },
      { owner: 'qclawchang', repo: 'homelab-infra' },
    ]);

    const res = await app.request('/ci');
    const body = await res.json();

    expect(body.data.repos.find((r: any) => r.repo === 'ZenvoraAI/day-and-you').conclusion).toBe('failure');
    expect(body.data.quotaWarning).toBe(true);
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/github/ci.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/github/ci.ts`**

```ts
import type { Hono } from 'hono';
import type { Database } from 'bun:sqlite';
import type { Octokit } from '@octokit/rest';
import { cachedFetch } from './cache';

const QUOTA_WARNING_RATIO = 0.8;

export interface RepoCiStatus {
  repo: string;
  conclusion: string | null;
  status: string;
  htmlUrl: string | null;
}

export interface CiHealth {
  repos: RepoCiStatus[];
  actionsMinutesUsed: number;
  actionsMinutesIncluded: number;
  quotaWarning: boolean;
}

export async function fetchCiHealth(octokit: Octokit, repos: { owner: string; repo: string }[]): Promise<CiHealth> {
  const runs = await Promise.all(
    repos.map(async ({ owner, repo }) => {
      const { data } = await octokit.actions.listWorkflowRunsForRepo({ owner, repo, per_page: 1 });
      const latest = data.workflow_runs[0];
      return {
        repo: `${owner}/${repo}`,
        conclusion: latest?.conclusion ?? null,
        status: latest?.status ?? 'no_runs',
        htmlUrl: latest?.html_url ?? null,
      };
    })
  );

  // GitHub's old `GET /orgs/{org}/settings/billing/actions` (the typed
  // `octokit.billing.getGithubActionsBillingOrg` helper) now returns 410 Gone —
  // it was replaced by the "enhanced billing" usage API. Using the untyped
  // `.request()` escape hatch here since there may not be a typed Octokit
  // helper for the new endpoint yet. VERIFY THE RESPONSE SHAPE against a real
  // API call early in implementation — the `usageItems` filter/summation below
  // is a best-effort reading of GitHub's documented shape, not independently
  // confirmed, and `actionsMinutesUsed`/`actionsMinutesIncluded` may need
  // adjusting once checked against a real response.
  const billingResponse = (await octokit.request('GET /organizations/{org}/settings/billing/usage', {
    org: 'ZenvoraAI',
  })) as any;
  const actionsMinutesUsed = (billingResponse.data.usageItems ?? [])
    .filter((item: any) => item.product === 'actions' && item.unitType === 'Minutes')
    .reduce((sum: number, item: any) => sum + item.quantity, 0);
  const actionsMinutesIncluded = billingResponse.includedMinutes ?? 2000;

  return {
    repos: runs,
    actionsMinutesUsed,
    actionsMinutesIncluded,
    quotaWarning: actionsMinutesUsed >= actionsMinutesIncluded * QUOTA_WARNING_RATIO,
  };
}

export function createCiRoute(
  app: Hono,
  db: Database,
  octokitFor: (token: string) => Octokit,
  repos: { owner: string; repo: string }[]
) {
  app.get('/ci', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, 'ci-health', () => fetchCiHealth(octokit, repos));
    return c.json({ data: result.data, stale: result.stale });
  });
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/github/ci.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add CI health endpoint with Actions quota warning"
```

---

### Task 10: Security posture endpoint

**Files:**
- Create: `src/github/security.ts`
- Test: `tests/github/security.test.ts`

**Interfaces:**
- Consumes: `cachedFetch` (Task 7).
- Produces: `RepoSecurityStatus`, `SecurityPosture`, `fetchSecurityPosture(octokit, repos)`, `fetchRepoSecurityStatus(octokit, owner, repo)` (single-repo helper, factored out so Task 19's alert checks can reuse it without duplicating the Octokit calls), `createSecurityRoute(app, db, octokitFor, repos)` — mounted at `/security` (becomes `/api/security` once Task 19 mounts this sub-app under `/api`).

- [ ] **Step 1: Write the failing test**

```ts
// tests/github/security.test.ts
import { describe, test, expect } from 'bun:test';
import { Hono } from 'hono';
import { openDb } from '../../src/db/client';
import { createSecurityRoute } from '../../src/github/security';

function fakeOctokit() {
  return {
    users: { getAuthenticated: async () => ({ data: { two_factor_authentication: true } }) },
    dependabot: {
      listAlertsForRepo: async () => ({ data: [{ id: 1 }] }),
    },
    actions: {
      listRepoSecrets: async () => ({ data: { total_count: 3 } }),
    },
  };
}

describe('GET /security', () => {
  test('reports 2FA status, alert counts, and marks unsupported checks unavailable-on-plan', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); await next(); });
    createSecurityRoute(app, db, () => fakeOctokit() as any, [{ owner: 'ZenvoraAI', repo: 'day-and-you' }]);

    const res = await app.request('/security');
    const body = await res.json();

    expect(body.data.twoFactorEnabled).toBe(true);
    expect(body.data.repos[0]).toEqual({
      repo: 'ZenvoraAI/day-and-you',
      openDependabotAlerts: 1,
      secretsCount: 3,
      branchProtection: 'unavailable_on_plan',
      secretScanning: 'unavailable_on_plan',
    });
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/github/security.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/github/security.ts`**

```ts
import type { Hono } from 'hono';
import type { Database } from 'bun:sqlite';
import type { Octokit } from '@octokit/rest';
import { cachedFetch } from './cache';

export interface RepoSecurityStatus {
  repo: string;
  openDependabotAlerts: number;
  secretsCount: number;
  branchProtection: 'unavailable_on_plan';
  secretScanning: 'unavailable_on_plan';
}

export interface SecurityPosture {
  twoFactorEnabled: boolean;
  repos: RepoSecurityStatus[];
}

export async function fetchRepoSecurityStatus(octokit: Octokit, owner: string, repo: string): Promise<RepoSecurityStatus> {
  const [alerts, secrets] = await Promise.all([
    octokit.dependabot.listAlertsForRepo({ owner, repo, state: 'open', per_page: 100 }),
    octokit.actions.listRepoSecrets({ owner, repo, per_page: 1 }),
  ]);
  return {
    repo: `${owner}/${repo}`,
    openDependabotAlerts: alerts.data.length,
    secretsCount: secrets.data.total_count,
    branchProtection: 'unavailable_on_plan' as const,
    secretScanning: 'unavailable_on_plan' as const,
  };
}

export async function fetchSecurityPosture(
  octokit: Octokit,
  repos: { owner: string; repo: string }[]
): Promise<SecurityPosture> {
  const { data: user } = await octokit.users.getAuthenticated();

  const repoStatuses = await Promise.all(
    repos.map(({ owner, repo }) => fetchRepoSecurityStatus(octokit, owner, repo))
  );

  return {
    twoFactorEnabled: Boolean((user as { two_factor_authentication?: boolean }).two_factor_authentication),
    repos: repoStatuses,
  };
}

export function createSecurityRoute(
  app: Hono,
  db: Database,
  octokitFor: (token: string) => Octokit,
  repos: { owner: string; repo: string }[]
) {
  app.get('/security', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, 'security-posture', () => fetchSecurityPosture(octokit, repos));
    return c.json({ data: result.data, stale: result.stale });
  });
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/github/security.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add security posture endpoint with plan-unavailable handling"
```

---

### Task 11: Backlog endpoint

**Files:**
- Create: `src/github/backlog.ts`
- Test: `tests/github/backlog.test.ts`

**Interfaces:**
- Consumes: `cachedFetch` (Task 7).
- Produces: `BacklogItem`, `Backlog`, `fetchBacklog(octokit, username, repos)`, `createBacklogRoute(app, db, octokitFor, username, repos)` — mounted at `/backlog` (becomes `/api/backlog` once Task 19 mounts this sub-app under `/api`).

- [ ] **Step 1: Write the failing test**

```ts
// tests/github/backlog.test.ts
import { describe, test, expect } from 'bun:test';
import { Hono } from 'hono';
import { openDb } from '../../src/db/client';
import { createBacklogRoute } from '../../src/github/backlog';

function fakeOctokit() {
  const oldCommitDate = new Date(Date.now() - 200 * 24 * 60 * 60 * 1000).toISOString();
  return {
    search: {
      issuesAndPullRequests: async ({ q }: { q: string }) => ({
        data: {
          items: q.includes('is:pr')
            ? [{ repository_url: 'https://api.github.com/repos/ZenvoraAI/day-and-you', title: 'Fix login', html_url: 'https://x/pr/1', updated_at: '2026-08-16T00:00:00Z' }]
            : [{ repository_url: 'https://api.github.com/repos/ZenvoraAI/day-and-you', title: 'Bug report', html_url: 'https://x/issue/1', updated_at: '2026-08-16T00:00:00Z' }],
        },
      }),
    },
    repos: {
      listBranches: async () => ({ data: [{ name: 'old-feature', commit: { sha: 'abc' } }] }),
      getCommit: async () => ({ data: { commit: { committer: { date: oldCommitDate } } } }),
    },
  };
}

describe('GET /backlog', () => {
  test('aggregates review-requested PRs, assigned issues, and stale branches', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); await next(); });
    createBacklogRoute(app, db, () => fakeOctokit() as any, 'qclawchang', [{ owner: 'ZenvoraAI', repo: 'day-and-you' }]);

    const res = await app.request('/backlog');
    const body = await res.json();

    expect(body.data.reviewRequestedPrs).toHaveLength(1);
    expect(body.data.assignedIssues).toHaveLength(1);
    expect(body.data.staleBranches).toEqual([{ repo: 'ZenvoraAI/day-and-you', branch: 'old-feature', lastCommitAt: expect.any(String) }]);
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/github/backlog.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/github/backlog.ts`**

```ts
import type { Hono } from 'hono';
import type { Database } from 'bun:sqlite';
import type { Octokit } from '@octokit/rest';
import { cachedFetch } from './cache';

const STALE_BRANCH_DAYS = 90;

export interface BacklogItem {
  repo: string;
  title: string;
  url: string;
  updatedAt: string;
}

export interface Backlog {
  reviewRequestedPrs: BacklogItem[];
  assignedIssues: BacklogItem[];
  staleBranches: { repo: string; branch: string; lastCommitAt: string }[];
}

function toItem(item: any): BacklogItem {
  return {
    repo: item.repository_url.split('/repos/')[1],
    title: item.title,
    url: item.html_url,
    updatedAt: item.updated_at,
  };
}

export async function fetchBacklog(
  octokit: Octokit,
  username: string,
  repos: { owner: string; repo: string }[]
): Promise<Backlog> {
  const [prSearch, issueSearch] = await Promise.all([
    octokit.search.issuesAndPullRequests({ q: `is:pr is:open review-requested:${username}` }),
    octokit.search.issuesAndPullRequests({ q: `is:issue is:open assignee:${username}` }),
  ]);

  const staleCutoff = Date.now() - STALE_BRANCH_DAYS * 24 * 60 * 60 * 1000;

  const staleBranchesByRepo = await Promise.all(
    repos.map(async ({ owner, repo }) => {
      const { data: branches } = await octokit.repos.listBranches({ owner, repo, per_page: 100 });
      const checked = await Promise.all(
        branches.map(async (branch) => {
          const { data: commit } = await octokit.repos.getCommit({ owner, repo, ref: branch.commit.sha });
          const commitDate = commit.commit.committer?.date;
          if (commitDate && new Date(commitDate).getTime() < staleCutoff) {
            return { repo: `${owner}/${repo}`, branch: branch.name, lastCommitAt: commitDate };
          }
          return null;
        })
      );
      return checked.filter((entry): entry is NonNullable<typeof entry> => entry !== null);
    })
  );

  return {
    reviewRequestedPrs: prSearch.data.items.map(toItem),
    assignedIssues: issueSearch.data.items.map(toItem),
    staleBranches: staleBranchesByRepo.flat(),
  };
}

export function createBacklogRoute(
  app: Hono,
  db: Database,
  octokitFor: (token: string) => Octokit,
  username: string,
  repos: { owner: string; repo: string }[]
) {
  app.get('/backlog', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, 'backlog', () => fetchBacklog(octokit, username, repos));
    return c.json({ data: result.data, stale: result.stale });
  });
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/github/backlog.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add backlog endpoint (PRs, issues, stale branches)"
```

---

### Task 12: docker-socket-proxy client

**Files:**
- Create: `src/deployment/dockerProxy.ts`
- Test: `tests/deployment/dockerProxy.test.ts`

**Interfaces:**
- Produces: `RunningContainer { name, image, imageTag }`, `listRunningContainers(proxyBaseUrl?): Promise<RunningContainer[]>` — consumed by Task 14. `imageTag` is parsed out of the container's `Image` reference string (e.g. `ghcr.io/zenvoraai/family-media-api:abc123` → `abc123`), not a content digest — see Task 13 for why: this repo's services are pinned by SHA-as-tag (immutable per build), not by manifest digest, and 2 of the 6 don't have a pin mechanism at all.

- [ ] **Step 1: Write the failing test**

```ts
// tests/deployment/dockerProxy.test.ts
import { describe, test, expect, mock, afterEach } from 'bun:test';
import { listRunningContainers } from '../../src/deployment/dockerProxy';

const originalFetch = globalThis.fetch;
afterEach(() => { globalThis.fetch = originalFetch; });

describe('listRunningContainers', () => {
  test('maps the docker-socket-proxy response into RunningContainer records, extracting the tag', async () => {
    globalThis.fetch = mock(async () =>
      new Response(
        JSON.stringify([
          { Names: ['/homelab-family-api'], Image: 'ghcr.io/zenvoraai/family-media-api:abc123' },
        ]),
        { status: 200 }
      )
    ) as any;

    const result = await listRunningContainers('http://fake-proxy:2375');

    expect(result).toEqual([
      { name: 'homelab-family-api', image: 'ghcr.io/zenvoraai/family-media-api:abc123', imageTag: 'abc123' },
    ]);
  });

  test('defaults the tag to "latest" when the image reference has none', async () => {
    globalThis.fetch = mock(async () =>
      new Response(
        JSON.stringify([{ Names: ['/homelab-memorial-api'], Image: 'ghcr.io/zenvoraai/aiqiuqi-memorial-api' }]),
        { status: 200 }
      )
    ) as any;

    const result = await listRunningContainers('http://fake-proxy:2375');

    expect(result).toEqual([
      { name: 'homelab-memorial-api', image: 'ghcr.io/zenvoraai/aiqiuqi-memorial-api', imageTag: 'latest' },
    ]);
  });

  test('throws when the proxy responds with an error status', async () => {
    globalThis.fetch = mock(async () => new Response('forbidden', { status: 403 })) as any;
    await expect(listRunningContainers('http://fake-proxy:2375')).rejects.toThrow('403');
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/deployment/dockerProxy.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/deployment/dockerProxy.ts`**

```ts
export interface RunningContainer {
  name: string;
  image: string;
  imageTag: string;
}

function parseImageTag(image: string): string {
  const colonIndex = image.lastIndexOf(':');
  const slashIndex = image.lastIndexOf('/');
  // a colon only counts as a tag separator if it comes after the last slash —
  // otherwise it's part of a registry host:port, e.g. "localhost:5000/name"
  const hasTag = colonIndex > slashIndex;
  return hasTag ? image.slice(colonIndex + 1) : 'latest';
}

export async function listRunningContainers(
  proxyBaseUrl: string = process.env.DOCKER_PROXY_URL ?? 'http://127.0.0.1:2375'
): Promise<RunningContainer[]> {
  const res = await fetch(`${proxyBaseUrl}/containers/json`);
  if (!res.ok) throw new Error(`docker-socket-proxy returned ${res.status}`);

  const containers = (await res.json()) as Array<{ Names: string[]; Image: string }>;

  return containers.map((c) => ({
    name: c.Names[0]?.replace(/^\//, '') ?? 'unknown',
    image: c.Image,
    imageTag: parseImageTag(c.Image),
  }));
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/deployment/dockerProxy.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add docker-socket-proxy client for reading container state"
```

---

### Task 13: GHCR tag comparison (drift signals)

**Files:**
- Create: `src/deployment/drift.ts`
- Test: `tests/deployment/drift.test.ts`

**Interfaces:**
- Produces: `PinnedContainer { name, ghcrPackage, pinnedTag }`, `GhcrVersion { tag, createdAt }`, `DriftSignal` (discriminated union: `ok` | `fault` | `upgrade_available`), `compareDrift(pinned, runningTag, latestGhcrVersion): DriftSignal`, `fetchLatestGhcrVersion(octokit, packageName): Promise<GhcrVersion | null>` — both consumed by Task 14.

This compares **tags**, not content digests. `family-api`, `securevault-api`, `memorial-api`, and `memorial-worker` are pinned in `docker-compose.yml` via a tag env var (`${FAMILY_API_TAG:-latest}` etc.) whose value is a commit SHA — the SHA *is* the tag, immutable per build, not a separate `sha256:` manifest digest. `dayandyou-staging` and `dayandyou-prod` use static `:staging`/`:release` tags with **no** per-deploy pin override in `docker-compose.yml` at all — for those two, `pinnedTag` is `null`, and `compareDrift` must never report `fault` for them (there is nothing to have drifted from), only `ok`/`upgrade_available`.

- [ ] **Step 1: Write the failing test**

```ts
// tests/deployment/drift.test.ts
import { describe, test, expect } from 'bun:test';
import { compareDrift, fetchLatestGhcrVersion } from '../../src/deployment/drift';

describe('compareDrift', () => {
  const pinned = { name: 'family-api', ghcrPackage: 'family-media-api', pinnedTag: 'abc123' };

  test('flags a fault when the running tag does not match the pin', () => {
    const signal = compareDrift(pinned, 'oldtag', { tag: 'abc123', createdAt: '2026-08-16T00:00:00Z' });
    expect(signal).toEqual({ kind: 'fault', runningTag: 'oldtag', pinnedTag: 'abc123' });
  });

  test('flags upgrade-available when the pin is correct but GHCR has something newer', () => {
    const signal = compareDrift(pinned, 'abc123', { tag: 'def456', createdAt: '2026-08-16T00:00:00Z' });
    expect(signal).toEqual({ kind: 'upgrade_available', pinnedTag: 'abc123', latestTag: 'def456' });
  });

  test('reports ok when the running tag matches the pin and nothing newer exists', () => {
    const signal = compareDrift(pinned, 'abc123', { tag: 'abc123', createdAt: '2026-08-16T00:00:00Z' });
    expect(signal).toEqual({ kind: 'ok' });
  });

  test('never reports a fault for a container with no pin mechanism (e.g. dayandyou)', () => {
    const unpinned = { name: 'dayandyou-staging', ghcrPackage: 'dayandyou', pinnedTag: null };

    const matching = compareDrift(unpinned, 'staging', { tag: 'staging', createdAt: '2026-08-16T00:00:00Z' });
    expect(matching).toEqual({ kind: 'ok' });

    const withUpgrade = compareDrift(unpinned, 'staging-old', { tag: 'staging-new', createdAt: '2026-08-16T00:00:00Z' });
    expect(withUpgrade.kind).toBe('upgrade_available');
  });
});

describe('fetchLatestGhcrVersion', () => {
  test('returns the most recently created version and its tag', async () => {
    const octokit = {
      packages: {
        getAllPackageVersionsForPackageOwnedByOrg: async () => ({
          data: [{ metadata: { container: { tags: ['abc123'] } }, created_at: '2026-08-16T00:00:00Z' }],
        }),
      },
    };
    expect(await fetchLatestGhcrVersion(octokit as any, 'family-media-api')).toEqual({ tag: 'abc123', createdAt: '2026-08-16T00:00:00Z' });
  });

  test('returns a null tag when the version has no tags (untagged/dangling)', async () => {
    const octokit = {
      packages: {
        getAllPackageVersionsForPackageOwnedByOrg: async () => ({
          data: [{ metadata: { container: { tags: [] } }, created_at: '2026-08-16T00:00:00Z' }],
        }),
      },
    };
    expect(await fetchLatestGhcrVersion(octokit as any, 'family-media-api')).toEqual({ tag: null, createdAt: '2026-08-16T00:00:00Z' });
  });

  test('returns null when the package has no versions', async () => {
    const octokit = { packages: { getAllPackageVersionsForPackageOwnedByOrg: async () => ({ data: [] }) } };
    expect(await fetchLatestGhcrVersion(octokit as any, 'family-media-api')).toBeNull();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/deployment/drift.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/deployment/drift.ts`**

```ts
import type { Octokit } from '@octokit/rest';

export interface PinnedContainer {
  name: string;
  ghcrPackage: string;
  /** null for containers with no per-deploy pin mechanism (dayandyou-staging/-prod) */
  pinnedTag: string | null;
}

export interface GhcrVersion {
  tag: string | null;
  createdAt: string;
}

export type DriftSignal =
  | { kind: 'ok' }
  | { kind: 'fault'; runningTag: string; pinnedTag: string }
  | { kind: 'upgrade_available'; pinnedTag: string | null; latestTag: string };

export function compareDrift(
  pinned: PinnedContainer,
  runningTag: string,
  latestGhcrVersion: GhcrVersion | null
): DriftSignal {
  if (pinned.pinnedTag !== null && runningTag !== pinned.pinnedTag) {
    return { kind: 'fault', runningTag, pinnedTag: pinned.pinnedTag };
  }
  if (latestGhcrVersion?.tag && latestGhcrVersion.tag !== runningTag) {
    return { kind: 'upgrade_available', pinnedTag: pinned.pinnedTag, latestTag: latestGhcrVersion.tag };
  }
  return { kind: 'ok' };
}

export async function fetchLatestGhcrVersion(octokit: Octokit, packageName: string): Promise<GhcrVersion | null> {
  const { data: versions } = await octokit.packages.getAllPackageVersionsForPackageOwnedByOrg({
    org: 'ZenvoraAI',
    package_type: 'container',
    package_name: packageName,
    per_page: 1,
  });

  const latest = versions[0] as any;
  if (!latest) return null;
  // A container package version's tags live at `metadata.container.tags` per
  // GitHub's documented package-version shape — VERIFY THIS against a real API
  // call early in implementation (Task 21's GitHub App setup step is a natural
  // point to do this), since it wasn't independently confirmed during review.
  const tag: string | null = latest.metadata?.container?.tags?.[0] ?? null;
  return { tag, createdAt: latest.created_at };
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/deployment/drift.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add drift comparison logic (fault vs upgrade-available signals)"
```

---

### Task 14: Deployment drift endpoint

**Files:**
- Create: `src/deployment/route.ts`
- Test: `tests/deployment/route.test.ts`

**Interfaces:**
- Consumes: `listRunningContainers` (Task 12), `compareDrift`/`fetchLatestGhcrVersion`/`PinnedContainer` (Task 13), `cachedFetch` (Task 7).
- Produces: `fetchDeploymentDrift(octokit, pinned)`, `createDeploymentRoute(app, db, octokitFor, pinned)` — mounted at `/deployment` (becomes `/api/deployment` once Task 19 mounts this sub-app under `/api`).

- [ ] **Step 1: Write the failing test**

```ts
// tests/deployment/route.test.ts
import { describe, test, expect, mock } from 'bun:test';

mock.module('../../src/deployment/dockerProxy', () => ({
  listRunningContainers: async () => [
    { name: 'homelab-family-api', image: 'ghcr.io/zenvoraai/family-media-api:abc123', imageTag: 'abc123' },
  ],
}));

const { Hono } = await import('hono');
const { openDb } = await import('../../src/db/client');
const { createDeploymentRoute } = await import('../../src/deployment/route');

describe('GET /deployment', () => {
  test('reports a container matching its pin as ok', async () => {
    const db = openDb(':memory:');
    const octokit = {
      packages: {
        getAllPackageVersionsForPackageOwnedByOrg: async () => ({
          data: [{ metadata: { container: { tags: ['abc123'] } }, created_at: '2026-08-16T00:00:00Z' }],
        }),
      },
    };

    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); await next(); });
    createDeploymentRoute(app, db, () => octokit as any, [
      { name: 'homelab-family-api', ghcrPackage: 'family-media-api', pinnedTag: 'abc123' },
    ]);

    const res = await app.request('/deployment');
    const body = await res.json();

    expect(body.data).toEqual([{ container: 'homelab-family-api', signal: { kind: 'ok' } }]);
    db.close();
  });

  test('reports a container with no pin mechanism (dayandyou) as ok when tags match, never as a fault', async () => {
    const db = openDb(':memory:');
    const octokit = {
      packages: {
        getAllPackageVersionsForPackageOwnedByOrg: async () => ({
          data: [{ metadata: { container: { tags: ['staging'] } }, created_at: '2026-08-16T00:00:00Z' }],
        }),
      },
    };

    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); await next(); });
    createDeploymentRoute(app, db, () => octokit as any, [
      { name: 'homelab-family-api', ghcrPackage: 'dayandyou', pinnedTag: null },
    ]);

    const res = await app.request('/deployment');
    const body = await res.json();

    expect(body.data[0].signal.kind).not.toBe('fault');
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/deployment/route.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/deployment/route.ts`**

```ts
import type { Hono } from 'hono';
import type { Database } from 'bun:sqlite';
import type { Octokit } from '@octokit/rest';
import { cachedFetch } from '../github/cache';
import { listRunningContainers } from './dockerProxy';
import { compareDrift, fetchLatestGhcrVersion, type PinnedContainer } from './drift';

export async function fetchDeploymentDrift(octokit: Octokit, pinned: PinnedContainer[]) {
  const running = await listRunningContainers();

  return Promise.all(
    pinned.map(async (p) => {
      const runningContainer = running.find((r) => r.name === p.name);
      const latest = await fetchLatestGhcrVersion(octokit, p.ghcrPackage);
      // a container that isn't running at all has no real tag to compare — 'unknown'
      // guarantees a fault for pinned services (correct: it's not running what it
      // should be) and just an informational upgrade_available for unpinned ones
      const signal = compareDrift(p, runningContainer?.imageTag ?? 'unknown', latest);
      return { container: p.name, signal };
    })
  );
}

export function createDeploymentRoute(
  app: Hono,
  db: Database,
  octokitFor: (token: string) => Octokit,
  pinned: PinnedContainer[]
) {
  app.get('/deployment', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, 'deployment-drift', () => fetchDeploymentDrift(octokit, pinned));
    return c.json({ data: result.data, stale: result.stale });
  });
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/deployment/route.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add deployment drift endpoint (per-container fault vs upgrade signal)"
```

---

### Task 15: Proactive alert job (state diff, email, scheduler)

**Files:**
- Create: `src/alerts/job.ts`
- Create: `src/alerts/email.ts`
- Create: `src/alerts/scheduler.ts`
- Test: `tests/alerts/job.test.ts`
- Test: `tests/alerts/scheduler.test.ts`

**Interfaces:**
- Produces: `recordCheckAndGetTransition(db, checkKey, status): AlertTransition | null`, `sendAlertEmail(email)`, `AlertCheck { key, describe, evaluate }`, `runAlertChecks(db, checks, notify?)`, `startAlertScheduler(db, checks, intervalMs?)` — Task 19 builds the real `AlertCheck[]` list and calls `startAlertScheduler`.

- [ ] **Step 1: Write the failing job test**

```ts
// tests/alerts/job.test.ts
import { describe, test, expect } from 'bun:test';
import { openDb } from '../../src/db/client';
import { recordCheckAndGetTransition } from '../../src/alerts/job';

describe('recordCheckAndGetTransition', () => {
  test('returns a transition the first time a check goes bad', () => {
    const db = openDb(':memory:');
    expect(recordCheckAndGetTransition(db, 'ci:day-and-you', 'bad')).toEqual({ checkKey: 'ci:day-and-you', from: 'unknown', to: 'bad' });
    db.close();
  });

  test('does not repeat a transition on a second bad check in a row', () => {
    const db = openDb(':memory:');
    recordCheckAndGetTransition(db, 'ci:day-and-you', 'bad');
    expect(recordCheckAndGetTransition(db, 'ci:day-and-you', 'bad')).toBeNull();
    db.close();
  });

  test('reports a transition back to ok', () => {
    const db = openDb(':memory:');
    recordCheckAndGetTransition(db, 'ci:day-and-you', 'bad');
    expect(recordCheckAndGetTransition(db, 'ci:day-and-you', 'ok')).toEqual({ checkKey: 'ci:day-and-you', from: 'bad', to: 'ok' });
    db.close();
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/alerts/job.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/alerts/job.ts`**

```ts
import type { Database } from 'bun:sqlite';

export type CheckStatus = 'ok' | 'bad';

export interface AlertTransition {
  checkKey: string;
  from: CheckStatus | 'unknown';
  to: CheckStatus;
}

export function recordCheckAndGetTransition(db: Database, checkKey: string, status: CheckStatus): AlertTransition | null {
  const row = db.query('SELECT status FROM alert_state WHERE check_key = ?').get(checkKey) as { status: CheckStatus } | null;

  db.query(
    'INSERT INTO alert_state (check_key, status, updated_at) VALUES (?, ?, ?) ' +
      'ON CONFLICT(check_key) DO UPDATE SET status = excluded.status, updated_at = excluded.updated_at'
  ).run(checkKey, status, Date.now());

  const from = row?.status ?? 'unknown';
  if (from === status) return null;
  return { checkKey, from, to: status };
}
```

- [ ] **Step 4: Run job test, confirm it passes**

Run: `bun test tests/alerts/job.test.ts`
Expected: PASS

- [ ] **Step 5: Write `src/alerts/email.ts`**

```ts
import nodemailer from 'nodemailer';

export interface AlertEmail {
  subject: string;
  body: string;
}

export async function sendAlertEmail(email: AlertEmail) {
  const transport = nodemailer.createTransport({
    host: process.env.SMTP_HOST,
    port: Number(process.env.SMTP_PORT ?? 587),
    secure: process.env.SMTP_SECURE === 'true',
    auth: { user: process.env.SMTP_USER, pass: process.env.SMTP_PASS },
  });

  await transport.sendMail({
    from: process.env.ALERT_EMAIL_FROM,
    to: process.env.ALERT_EMAIL_TO,
    subject: email.subject,
    text: email.body,
  });
}
```

- [ ] **Step 6: Write the failing scheduler test**

```ts
// tests/alerts/scheduler.test.ts
import { describe, test, expect, mock } from 'bun:test';
import { openDb } from '../../src/db/client';
import { runAlertChecks } from '../../src/alerts/scheduler';

describe('runAlertChecks', () => {
  test('emails once on an ok-to-bad transition and not again while still bad', async () => {
    const db = openDb(':memory:');
    const notify = mock(async () => {});
    const check = { key: 'ci:day-and-you', describe: 'CI for day-and-you', evaluate: async () => 'bad' as const };

    await runAlertChecks(db, [check], notify);
    await runAlertChecks(db, [check], notify);

    expect(notify).toHaveBeenCalledTimes(1);
    db.close();
  });

  test("one check's evaluate() throwing does not prevent the next check from running", async () => {
    const db = openDb(':memory:');
    const notify = mock(async () => {});
    const failingCheck = {
      key: 'ci:broken-repo',
      describe: 'CI for broken-repo',
      evaluate: async () => { throw new Error('GitHub API unreachable'); },
    };
    const healthyCheck = { key: 'ci:day-and-you', describe: 'CI for day-and-you', evaluate: async () => 'bad' as const };

    await runAlertChecks(db, [failingCheck, healthyCheck], notify);

    expect(notify).toHaveBeenCalledTimes(1);
    expect(notify.mock.calls[0][0].subject).toContain('day-and-you');
    db.close();
  });
});
```

- [ ] **Step 7: Run it, confirm it fails**

Run: `bun test tests/alerts/scheduler.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 8: Write `src/alerts/scheduler.ts`**

```ts
import type { Database } from 'bun:sqlite';
import { recordCheckAndGetTransition, type CheckStatus } from './job';
import { sendAlertEmail } from './email';

export interface AlertCheck {
  key: string;
  describe: string;
  evaluate: () => Promise<CheckStatus>;
}

export async function runAlertChecks(db: Database, checks: AlertCheck[], notify = sendAlertEmail) {
  for (const check of checks) {
    try {
      const status = await check.evaluate();
      const transition = recordCheckAndGetTransition(db, check.key, status);
      if (transition && transition.to === 'bad') {
        await notify({
          subject: `zenvora-admin alert: ${check.describe}`,
          body: `${check.describe} went from ${transition.from} to bad at ${new Date().toISOString()}.`,
        });
      }
    } catch (err) {
      // one check's failure (e.g. a transient GitHub API error) must never
      // silence every other check in the same run — that's exactly the
      // silent-failure mode this whole tool exists to prevent
      console.error(`alert check "${check.key}" failed`, err);
    }
  }
}

export function startAlertScheduler(db: Database, checks: AlertCheck[], intervalMs = 5 * 60 * 1000) {
  return setInterval(() => {
    runAlertChecks(db, checks).catch((err) => console.error('alert check failed', err));
  }, intervalMs);
}
```

- [ ] **Step 9: Run scheduler test, confirm it passes**

Run: `bun test tests/alerts/scheduler.test.ts`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat: add proactive alert job (state diff, email, scheduler)"
```

---

### Task 16: Static frontend serving

**Files:**
- Create: `src/static.ts`
- Test: `tests/static.test.ts`

**Interfaces:**
- Produces: `mountStaticFrontend(app, root?)` — mounted by Task 19, points at `frontend/dist` (built by Task 17's `vite build`, bundled by Task 20's Dockerfile).

- [ ] **Step 1: Write the failing test**

```ts
// tests/static.test.ts
import { describe, test, expect, beforeAll, afterAll } from 'bun:test';
import { mkdtempSync, writeFileSync, rmSync } from 'node:fs';
import { join, relative } from 'node:path';
import { Hono } from 'hono';
import { mountStaticFrontend } from '../src/static';

describe('mountStaticFrontend', () => {
  let dir: string;
  let relativeDir: string;

  beforeAll(() => {
    // hono/bun's serveStatic resolves `root` relative to the process cwd, not
    // as an absolute path — create the fixture dir under cwd rather than the
    // OS tmpdir so this test reflects that.
    dir = mkdtempSync(join(process.cwd(), 'zenvora-static-'));
    relativeDir = relative(process.cwd(), dir);
    writeFileSync(join(dir, 'index.html'), '<html><body>zenvora-admin</body></html>');
  });

  afterAll(() => { rmSync(dir, { recursive: true, force: true }); });

  test('serves index.html at the root path', async () => {
    const app = new Hono();
    mountStaticFrontend(app, relativeDir);

    const res = await app.request('/');
    expect(res.status).toBe(200);
    expect(await res.text()).toContain('zenvora-admin');
  });
});
```

- [ ] **Step 2: Run it, confirm it fails**

Run: `bun test tests/static.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write `src/static.ts`**

```ts
import type { Hono } from 'hono';
import { serveStatic } from 'hono/bun';

export function mountStaticFrontend(app: Hono, root: string = 'frontend/dist') {
  app.get('/', serveStatic({ path: `${root}/index.html` }));
  app.use('/*', serveStatic({ root }));
}
```

- [ ] **Step 4: Run it, confirm it passes**

Run: `bun test tests/static.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: serve the built frontend as static files"
```

---

### Task 17: Frontend scaffold, API client, design tokens

**Files:**
- Create: `frontend/package.json`, `frontend/vite.config.ts`, `frontend/index.html`, `frontend/tsconfig.json`
- Create: `frontend/bunfig.toml`, `frontend/happy-dom-register.ts`
- Create: `frontend/src/api.ts`
- Create: `frontend/src/styles/tokens.css`
- Create: `frontend/src/main.tsx`, `frontend/src/App.tsx` (placeholder shell — panels added in Task 18)
- Test: `frontend/src/api.test.ts`

**Interfaces:**
- Produces: `apiGet<T>(path): Promise<T>`, `ApiError` — consumed by every panel component in Task 18 via the `useApiData` hook.

- [ ] **Step 1: Write `frontend/package.json`**

```json
{
  "name": "zenvora-admin-frontend",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "test": "bun test"
  },
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.0",
    "vite": "^5.4.0",
    "typescript": "^5.6.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@testing-library/react": "^16.0.0",
    "@happy-dom/global-registrator": "^15.0.0"
  }
}
```

- [ ] **Step 2: Write `frontend/vite.config.ts`, `frontend/tsconfig.json`, `frontend/index.html`**

```ts
// frontend/vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: { outDir: 'dist' },
});
```

```json
// frontend/tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "lib": ["ES2022", "DOM"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

```html
<!-- frontend/index.html -->
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>zenvora-admin</title>
    <link rel="stylesheet" href="/src/styles/tokens.css" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

- [ ] **Step 3: Write `frontend/bunfig.toml` and `frontend/happy-dom-register.ts`** — registers a DOM for `bun:test` since these are component tests

```toml
# frontend/bunfig.toml
[test]
preload = ["./happy-dom-register.ts"]
```

```ts
// frontend/happy-dom-register.ts
import { GlobalRegistrator } from '@happy-dom/global-registrator';
GlobalRegistrator.register();
```

- [ ] **Step 4: Write `frontend/src/styles/tokens.css`** — palette adapted from `~/self-growth-dashboard.html`

```css
:root {
  --bg: #f1eadb;
  --paper: #fbf6ec;
  --ink: #1f2a26;
  --ink-soft: #4a554f;
  --ink-mute: #8a8675;
  --line: #d8cfb8;
  --ember: #c9612a;
  --moss: #4a6a4f;
  --gold: #d4a341;
  --success: #5a8a5a;
  --danger: #b0492f;
  --r-md: 14px;
  --shadow-sm: 0 1px 2px rgba(31,42,38,.05), 0 2px 6px rgba(31,42,38,.04);
  --font-display: "Fraunces", "Noto Serif SC", Georgia, serif;
  --font-body: "Inter", -apple-system, "PingFang SC", sans-serif;
}

body {
  background: var(--bg);
  color: var(--ink);
  font-family: var(--font-body);
  margin: 0;
}

.card {
  background: var(--paper);
  border: 1px solid var(--line);
  border-radius: var(--r-md);
  box-shadow: var(--shadow-sm);
  padding: 16px;
}

.status-fault { color: var(--danger); font-weight: 600; }
.status-ok { color: var(--success); }
.status-upgrade { color: var(--gold); }
```

- [ ] **Step 5: Write the failing test for the API client**

```ts
// frontend/src/api.test.ts
import { describe, test, expect, mock, afterEach } from 'bun:test';
import { apiGet, ApiError } from './api';

const originalFetch = globalThis.fetch;
afterEach(() => { globalThis.fetch = originalFetch; });

describe('apiGet', () => {
  test('returns parsed JSON on success', async () => {
    globalThis.fetch = mock(async () => new Response(JSON.stringify({ ok: true }), { status: 200 })) as any;
    expect(await apiGet('/api/repos')).toEqual({ ok: true });
  });

  test('throws ApiError on a non-2xx response', async () => {
    globalThis.fetch = mock(async () => new Response('error', { status: 500 })) as any;
    await expect(apiGet('/api/repos')).rejects.toBeInstanceOf(ApiError);
  });
});
```

- [ ] **Step 6: Run it, confirm it fails**

Run: `cd frontend && bun test src/api.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 7: Write `frontend/src/api.ts`**

```ts
export class ApiError extends Error {
  constructor(public status: number, message: string) { super(message); }
}

export async function apiGet<T>(path: string): Promise<T> {
  const res = await fetch(path, { credentials: 'include' });
  if (res.status === 401) {
    window.location.href = '/auth/login';
    throw new ApiError(401, 'not authenticated');
  }
  if (!res.ok) throw new ApiError(res.status, `request to ${path} failed`);
  return (await res.json()) as T;
}
```

- [ ] **Step 8: Run it, confirm it passes**

Run: `cd frontend && bun test src/api.test.ts`
Expected: PASS

- [ ] **Step 9: Write the placeholder shell** (real panels added in Task 18)

```tsx
// frontend/src/App.tsx
export function App() {
  return (
    <main>
      <h1>zenvora-admin</h1>
    </main>
  );
}
```

```tsx
// frontend/src/main.tsx
import { createRoot } from 'react-dom/client';
import { App } from './App';

createRoot(document.getElementById('root')!).render(<App />);
```

- [ ] **Step 10: Install and commit**

```bash
cd frontend && bun install && cd ..
git add -A
git commit -m "feat: scaffold frontend with API client and design tokens"
```

---

### Task 18: Frontend panels (repo grid, CI, security, deployment drift, backlog)

**Files:**
- Create: `frontend/src/hooks/useApiData.ts`
- Create: `frontend/src/components/RepoGrid.tsx`, `CiHealth.tsx`, `SecurityPosture.tsx`, `DeploymentDrift.tsx`, `Backlog.tsx`, `LoginGate.tsx`
- Modify: `frontend/src/App.tsx` (render `LoginGate` at `/login`, otherwise all 5 panels)
- Test: `frontend/src/components/__tests__/panels.test.tsx`

**Interfaces:**
- Consumes: `apiGet`, `ApiError` (Task 17); every backend route now responds `{ data, stale }` (Tasks 8–11, 14, after the stale-surfacing fix), so `useApiData`'s success variant is `{ status: 'success', data: T, stale: boolean }`.

- [ ] **Step 1: Write `frontend/src/hooks/useApiData.ts`**

```tsx
import { useEffect, useState } from 'react';
import { apiGet, ApiError } from '../api';

export type ApiDataState<T> =
  | { status: 'loading' }
  | { status: 'error'; message: string }
  | { status: 'success'; data: T; stale: boolean };

interface ApiEnvelope<T> {
  data: T;
  stale: boolean;
}

export function useApiData<T>(path: string): ApiDataState<T> {
  const [state, setState] = useState<ApiDataState<T>>({ status: 'loading' });

  useEffect(() => {
    let cancelled = false;
    apiGet<ApiEnvelope<T>>(path)
      .then((envelope) => { if (!cancelled) setState({ status: 'success', data: envelope.data, stale: envelope.stale }); })
      .catch((err) => {
        if (cancelled) return;
        setState({ status: 'error', message: err instanceof ApiError ? err.message : 'unexpected error' });
      });
    return () => { cancelled = true; };
  }, [path]);

  return state;
}
```

- [ ] **Step 2: Write the failing panel tests**

```tsx
// frontend/src/components/__tests__/panels.test.tsx
import { describe, test, expect, mock, afterEach } from 'bun:test';
import { render, screen, waitFor } from '@testing-library/react';
import { RepoGrid } from '../RepoGrid';
import { DeploymentDrift } from '../DeploymentDrift';

const originalFetch = globalThis.fetch;
afterEach(() => { globalThis.fetch = originalFetch; });

describe('RepoGrid', () => {
  test('renders repo cards once data loads', async () => {
    globalThis.fetch = mock(async () => new Response(JSON.stringify({
      data: [{ name: 'homelab-infra', fullName: 'qclawchang/homelab-infra', private: false, pushedAt: null, language: 'Shell', description: null }],
      stale: false,
    }), { status: 200 })) as any;

    render(<RepoGrid />);
    await waitFor(() => expect(screen.getByText('homelab-infra')).toBeDefined());
  });

  test("shows a retry message on failure", async () => {
    globalThis.fetch = mock(async () => new Response('error', { status: 500 })) as any;
    render(<RepoGrid />);
    await waitFor(() => expect(screen.getByText(/couldn't load repos/)).toBeDefined());
  });

  test('shows a stale indicator when the API served a cached value after a live-fetch failure', async () => {
    globalThis.fetch = mock(async () => new Response(JSON.stringify({
      data: [{ name: 'homelab-infra', fullName: 'qclawchang/homelab-infra', private: false, pushedAt: null, language: 'Shell', description: null }],
      stale: true,
    }), { status: 200 })) as any;

    render(<RepoGrid />);
    await waitFor(() => expect(screen.getByText(/may be out of date/)).toBeDefined());
  });
});

describe('DeploymentDrift', () => {
  test('visually distinguishes a fault from an upgrade-available signal', async () => {
    globalThis.fetch = mock(async () => new Response(JSON.stringify({
      data: [
        { container: 'homelab-family-api', signal: { kind: 'fault', runningTag: 'oldtag', pinnedTag: 'abc123' } },
        { container: 'homelab-securevault-api', signal: { kind: 'upgrade_available', pinnedTag: 'abc123', latestTag: 'def456' } },
      ],
      stale: false,
    }), { status: 200 })) as any;

    render(<DeploymentDrift />);
    await waitFor(() => expect(screen.getByText(/DRIFT/)).toBeDefined());
    expect(screen.getByText(/newer image available/)).toBeDefined();
  });
});
```

- [ ] **Step 3: Run it, confirm it fails**

Run: `cd frontend && bun test src/components/__tests__/panels.test.tsx`
Expected: FAIL — components don't exist yet.

- [ ] **Step 4: Write `frontend/src/components/RepoGrid.tsx`**

```tsx
import { useApiData } from '../hooks/useApiData';

interface RepoSummary {
  name: string; fullName: string; private: boolean;
  pushedAt: string | null; language: string | null; description: string | null;
}

export function RepoGrid() {
  const state = useApiData<RepoSummary[]>('/api/repos');

  if (state.status === 'loading') return <section className="card">loading repos…</section>;
  if (state.status === 'error') return <section className="card status-fault">couldn't load repos, retry</section>;

  return (
    <section>
      <h2>Repos</h2>
      {state.stale && <p className="status-upgrade">showing cached data, may be out of date</p>}
      <div style={{ display: 'grid', gap: '12px', gridTemplateColumns: 'repeat(auto-fill, minmax(220px, 1fr))' }}>
        {state.data.map((repo) => (
          <article key={repo.fullName} className="card">
            <strong>{repo.name}</strong>
            <div>{repo.private ? 'private' : 'public'}</div>
            <div>{repo.language ?? '—'}</div>
            <div>{repo.description ?? ''}</div>
          </article>
        ))}
      </div>
    </section>
  );
}
```

- [ ] **Step 5: Write `frontend/src/components/CiHealth.tsx`**

```tsx
import { useApiData } from '../hooks/useApiData';

interface CiHealth {
  repos: { repo: string; conclusion: string | null; status: string; htmlUrl: string | null }[];
  actionsMinutesUsed: number; actionsMinutesIncluded: number; quotaWarning: boolean;
}

export function CiHealth() {
  const state = useApiData<CiHealth>('/api/ci');

  if (state.status === 'loading') return <section className="card">loading CI health…</section>;
  if (state.status === 'error') return <section className="card status-fault">couldn't load CI health, retry</section>;

  return (
    <section>
      <h2>CI health</h2>
      {state.stale && <p className="status-upgrade">showing cached data, may be out of date</p>}
      {state.data.quotaWarning && (
        <p className="status-fault">Actions minutes at {state.data.actionsMinutesUsed}/{state.data.actionsMinutesIncluded} — over 80% of quota.</p>
      )}
      <ul>
        {state.data.repos.map((r) => (
          <li key={r.repo} className={r.conclusion === 'failure' ? 'status-fault' : 'status-ok'}>
            {r.repo}: {r.conclusion ?? r.status}
          </li>
        ))}
      </ul>
    </section>
  );
}
```

- [ ] **Step 6: Write `frontend/src/components/SecurityPosture.tsx`**

```tsx
import { useApiData } from '../hooks/useApiData';

interface SecurityPosture {
  twoFactorEnabled: boolean;
  repos: { repo: string; openDependabotAlerts: number; secretsCount: number; branchProtection: string; secretScanning: string }[];
}

export function SecurityPosture() {
  const state = useApiData<SecurityPosture>('/api/security');

  if (state.status === 'loading') return <section className="card">loading security posture…</section>;
  if (state.status === 'error') return <section className="card status-fault">couldn't load security posture, retry</section>;

  return (
    <section>
      <h2>Security posture</h2>
      {state.stale && <p className="status-upgrade">showing cached data, may be out of date</p>}
      <p className={state.data.twoFactorEnabled ? 'status-ok' : 'status-fault'}>
        Account 2FA: {state.data.twoFactorEnabled ? 'enabled' : 'disabled'}
      </p>
      <ul>
        {state.data.repos.map((r) => (
          <li key={r.repo} className={r.openDependabotAlerts > 0 ? 'status-fault' : 'status-ok'}>
            {r.repo}: {r.openDependabotAlerts} Dependabot alerts, {r.secretsCount} secrets
            (branch protection: {r.branchProtection === 'unavailable_on_plan' ? 'unavailable on plan' : r.branchProtection},
             secret scanning: {r.secretScanning === 'unavailable_on_plan' ? 'unavailable on plan' : r.secretScanning})
          </li>
        ))}
      </ul>
    </section>
  );
}
```

- [ ] **Step 7: Write `frontend/src/components/DeploymentDrift.tsx`**

```tsx
import { useApiData } from '../hooks/useApiData';

type DriftSignal =
  | { kind: 'ok' }
  | { kind: 'fault'; runningTag: string; pinnedTag: string }
  | { kind: 'upgrade_available'; pinnedTag: string | null; latestTag: string };

interface DeploymentRow { container: string; signal: DriftSignal; }

export function DeploymentDrift() {
  const state = useApiData<DeploymentRow[]>('/api/deployment');

  if (state.status === 'loading') return <section className="card">loading deployment status…</section>;
  if (state.status === 'error') return <section className="card status-fault">couldn't load deployment status, retry</section>;

  return (
    <section>
      <h2>Deployment drift</h2>
      {state.stale && <p className="status-upgrade">showing cached data, may be out of date</p>}
      <ul>
        {state.data.map((row) => (
          <li key={row.container} className={row.signal.kind === 'fault' ? 'status-fault' : row.signal.kind === 'upgrade_available' ? 'status-upgrade' : 'status-ok'}>
            {row.container}: {row.signal.kind === 'fault' ? 'DRIFT — running image does not match pin' : row.signal.kind === 'upgrade_available' ? 'newer image available on GHCR' : 'matches pin'}
          </li>
        ))}
      </ul>
    </section>
  );
}
```

- [ ] **Step 8: Write `frontend/src/components/Backlog.tsx`**

```tsx
import { useApiData } from '../hooks/useApiData';

interface Backlog {
  reviewRequestedPrs: { repo: string; title: string; url: string; updatedAt: string }[];
  assignedIssues: { repo: string; title: string; url: string; updatedAt: string }[];
  staleBranches: { repo: string; branch: string; lastCommitAt: string }[];
}

export function Backlog() {
  const state = useApiData<Backlog>('/api/backlog');

  if (state.status === 'loading') return <section className="card">loading backlog…</section>;
  if (state.status === 'error') return <section className="card status-fault">couldn't load backlog, retry</section>;

  return (
    <section>
      <h2>Backlog</h2>
      {state.stale && <p className="status-upgrade">showing cached data, may be out of date</p>}
      <h3>PRs awaiting review</h3>
      <ul>{state.data.reviewRequestedPrs.map((pr) => <li key={pr.url}><a href={pr.url}>{pr.repo}: {pr.title}</a></li>)}</ul>
      <h3>Assigned issues</h3>
      <ul>{state.data.assignedIssues.map((issue) => <li key={issue.url}><a href={issue.url}>{issue.repo}: {issue.title}</a></li>)}</ul>
      <h3>Stale branches (90+ days)</h3>
      <ul>{state.data.staleBranches.map((b) => <li key={`${b.repo}/${b.branch}`}>{b.repo}: {b.branch} (last commit {b.lastCommitAt})</li>)}</ul>
    </section>
  );
}
```

- [ ] **Step 9: Write `frontend/src/components/LoginGate.tsx` and its test** — the spec requires rejection reasons to render on a login page (Task 5's callback now redirects to `/login?error=...` instead of returning a plain-text 403); this is that page.

```tsx
// frontend/src/components/__tests__/loginGate.test.tsx
import { describe, test, expect } from 'bun:test';
import { render, screen } from '@testing-library/react';
import { LoginGate } from '../LoginGate';

describe('LoginGate', () => {
  test('shows a login link with no error text when there is no ?error= param', () => {
    window.history.pushState({}, '', '/login');
    render(<LoginGate />);
    expect(screen.getByText(/log in with github/i)).toBeDefined();
  });

  test('shows the decoded rejection reason when ?error= is present', () => {
    window.history.pushState({}, '', '/login?error=' + encodeURIComponent('two-factor authentication must be enabled on this GitHub account'));
    render(<LoginGate />);
    expect(screen.getByText(/two-factor authentication must be enabled/)).toBeDefined();
  });
});
```

```tsx
// frontend/src/components/LoginGate.tsx
export function LoginGate() {
  const error = new URLSearchParams(window.location.search).get('error');

  return (
    <main className="card">
      <h1>zenvora-admin</h1>
      {error && <p className="status-fault">{error}</p>}
      <a href="/auth/login">Log in with GitHub</a>
    </main>
  );
}
```

Run: `cd frontend && bun test src/components/__tests__/loginGate.test.tsx`
Expected: PASS

- [ ] **Step 10: Update `frontend/src/App.tsx`** to show `LoginGate` at `/login` and the five panels everywhere else

```tsx
import { RepoGrid } from './components/RepoGrid';
import { CiHealth } from './components/CiHealth';
import { SecurityPosture } from './components/SecurityPosture';
import { DeploymentDrift } from './components/DeploymentDrift';
import { Backlog } from './components/Backlog';
import { LoginGate } from './components/LoginGate';

export function App() {
  if (window.location.pathname === '/login') {
    return <LoginGate />;
  }

  return (
    <main>
      <h1>zenvora-admin</h1>
      <RepoGrid />
      <CiHealth />
      <SecurityPosture />
      <DeploymentDrift />
      <Backlog />
    </main>
  );
}
```

Any panel's `apiGet` still redirects to `/auth/login` on a 401 exactly as before (Task 17) — `LoginGate` only covers the case where the operator lands on `/login` directly, e.g. after Task 5's callback redirects there with a rejection reason.

- [ ] **Step 11: Run it, confirm it passes**

Run: `cd frontend && bun test src/components/__tests__/panels.test.tsx src/components/__tests__/loginGate.test.tsx`
Expected: PASS

- [ ] **Step 12: Run `bun run dev` and check the golden path in a browser** — per this project's own standard, don't claim a UI change works without seeing it render. Log in via a real GitHub App test installation if credentials are available yet; otherwise confirm each panel's loading/error states render correctly with the dev server up and `/api/*` returning 401, and that `/login?error=...` renders the reason text.

- [ ] **Step 13: Commit**

```bash
git add -A
git commit -m "feat: add the five dashboard panels"
```

---

### Task 19: Wire the application entrypoint

**Files:**
- Modify: `src/server.ts` (replace the Task 1 stub with the full wiring; `createApp()` now returns `{ app, db }`)
- Modify: `tests/server.test.ts` (update to the new return shape)
- Test: `tests/server.integration.test.ts` (new — verifies routing end to end)

**Interfaces:**
- Consumes: every route-creator and the scheduler from Tasks 4–16.
- Produces: `createApp(): { app: Hono, db: Database }` — this is what `bun run src/server.ts` (and the Dockerfile's `CMD`, Task 20) actually starts.

- [ ] **Step 1: Update the existing health check test for the new return shape**

```ts
// tests/server.test.ts — replace the body of the existing test
import { describe, test, expect } from 'bun:test';
import { createApp } from '../src/server';

describe('GET /healthz', () => {
  test('returns ok status', async () => {
    const { app } = createApp();
    const res = await app.request('/healthz');
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ status: 'ok' });
  });
});
```

- [ ] **Step 2: Write the failing integration test**

```ts
// tests/server.integration.test.ts
import { describe, test, expect, beforeAll } from 'bun:test';
import { randomBytes } from 'node:crypto';
import { encryptToken } from '../src/auth/crypto';

describe('createApp wiring', () => {
  beforeAll(() => {
    process.env.DB_PATH = ':memory:';
    process.env.TOKEN_ENCRYPTION_KEY = randomBytes(32).toString('base64');
    process.env.GITHUB_APP_CLIENT_ID = 'test-client-id';
    process.env.GITHUB_APP_REDIRECT_URI = 'https://admin.valtou.com/auth/callback';
  });

  test('exposes /healthz without auth and gates /api/* behind requireAuth', async () => {
    const { createApp } = await import('../src/server');
    const { app } = createApp();

    expect((await app.request('/healthz')).status).toBe(200);
    expect((await app.request('/api/repos')).status).toBe(401);
  });

  test('a valid session actually reaches the mounted panel routes, not a 404', async () => {
    // This specifically guards against a route-prefix mismatch (e.g. a panel
    // registering itself at an absolute "/api/repos" and then getting mounted
    // under "/api" again, becoming unreachable at "/api/api/repos") — the
    // 401-only check above would pass even with that bug, since requireAuth
    // intercepts before Hono ever gets to look for a matching route.
    const { createApp } = await import('../src/server');
    const { app, db } = createApp();

    const sessionId = 'test-session-id';
    db.query(
      'INSERT INTO sessions (id, user_id, encrypted_token, created_at, expires_at) VALUES (?, ?, ?, ?, ?)'
    ).run(sessionId, 4242, encryptToken('ghu_fake'), Date.now(), Date.now() + 60_000);

    for (const path of ['/api/repos', '/api/ci', '/api/security', '/api/backlog', '/api/deployment']) {
      const res = await app.request(path, { headers: { cookie: `zenvora_session=${sessionId}` } });
      // a real GitHub call will fail without network access (500/502) or hang
      // on a real fetch attempt that this test doesn't mock — either way, the
      // one status that would mean "route not found" is 404, and that's the
      // one this test exists to rule out.
      expect(res.status).not.toBe(404);
    }
  });
});
```

- [ ] **Step 3: Run it, confirm it fails**

Run: `bun test tests/server.integration.test.ts`
Expected: FAIL — `createApp` doesn't wire `/api/*` yet.

- [ ] **Step 4: Replace `src/server.ts`**

```ts
import { Hono } from 'hono';
import { openDb } from './db/client';
import { createOAuthLoginRoute, createOAuthCallbackRoute, createRealGitHubOAuthClient } from './auth/oauth';
import { requireAuth, createLogoutRoute } from './auth/session';
import { createOctokit } from './github/client';
import { createReposRoute } from './github/repos';
import { createCiRoute } from './github/ci';
import { createSecurityRoute } from './github/security';
import { createBacklogRoute } from './github/backlog';
import { createDeploymentRoute } from './deployment/route';
import { fetchRepoSecurityStatus } from './github/security';
import { mountStaticFrontend } from './static';
import { startAlertScheduler, type AlertCheck } from './alerts/scheduler';
import type { AppVariables } from './types';
import type { PinnedContainer } from './deployment/drift';

const REPOS = [
  { owner: 'qclawchang', repo: 'homelab-infra' },
  { owner: 'ZenvoraAI', repo: 'day-and-you' },
  { owner: 'ZenvoraAI', repo: 'securevault-framework' },
  { owner: 'ZenvoraAI', repo: 'family-media' },
  { owner: 'ZenvoraAI', repo: 'aiqiuqi-memorial' },
  { owner: 'ZenvoraAI', repo: 'aws-infrastructure' },
];

const PINNED_CONTAINERS: PinnedContainer[] = [
  { name: 'homelab-family-api', ghcrPackage: 'family-media-api', pinnedTag: process.env.FAMILY_API_TAG ?? null },
  { name: 'homelab-securevault-api', ghcrPackage: 'securevault-api', pinnedTag: process.env.SECUREVAULT_API_TAG ?? null },
  // dayandyou-staging/-prod have no per-deploy pin variable in docker-compose.yml
  // at all (static :staging/:release tags) — pinnedTag: null means Task 13's
  // compareDrift can only ever report ok/upgrade_available for these two, never
  // a fault, since there is nothing pinned to have drifted from.
  { name: 'homelab-dayandyou-staging', ghcrPackage: 'dayandyou', pinnedTag: null },
  { name: 'homelab-dayandyou-prod', ghcrPackage: 'dayandyou', pinnedTag: null },
  { name: 'homelab-memorial-api', ghcrPackage: 'aiqiuqi-memorial-api', pinnedTag: process.env.MEMORIAL_API_TAG ?? null },
  { name: 'homelab-memorial-worker', ghcrPackage: 'aiqiuqi-memorial-worker', pinnedTag: process.env.MEMORIAL_WORKER_TAG ?? null },
];

export function createApp() {
  const db = openDb();
  const app = new Hono<{ Variables: AppVariables }>();

  app.get('/healthz', (c) => c.json({ status: 'ok' }));

  createOAuthLoginRoute(app, db);
  createOAuthCallbackRoute(app, db, createRealGitHubOAuthClient());
  createLogoutRoute(app, db);

  const api = new Hono<{ Variables: AppVariables }>();
  api.use('*', requireAuth(db));
  createReposRoute(api, db, createOctokit, REPOS);
  createCiRoute(api, db, createOctokit, REPOS);
  createSecurityRoute(api, db, createOctokit, REPOS);
  createBacklogRoute(api, db, createOctokit, 'qclawchang', REPOS);
  createDeploymentRoute(api, db, createOctokit, PINNED_CONTAINERS);
  app.route('/api', api);

  mountStaticFrontend(app);

  return { app, db };
}

if (import.meta.main) {
  const { app, db } = createApp();
  const port = Number(process.env.PORT ?? 3100);

  const ciChecks: AlertCheck[] = REPOS.map(({ owner, repo }) => ({
    key: `ci:${owner}/${repo}`,
    describe: `CI for ${owner}/${repo}`,
    evaluate: async () => {
      const octokit = createOctokit(process.env.ALERT_CHECK_TOKEN ?? '');
      const { data } = await octokit.actions.listWorkflowRunsForRepo({ owner, repo, per_page: 1 });
      return data.workflow_runs[0]?.conclusion === 'failure' ? 'bad' : 'ok';
    },
  }));

  // spec requires the proactive alert to cover security regressions too, not
  // just CI — reuses the same per-repo Dependabot lookup the security panel
  // itself uses (Task 10's fetchRepoSecurityStatus), so there's one source of
  // truth for "what counts as a security alert" rather than two.
  const securityChecks: AlertCheck[] = REPOS.map(({ owner, repo }) => ({
    key: `security:${owner}/${repo}`,
    describe: `Dependabot alerts for ${owner}/${repo}`,
    evaluate: async () => {
      const octokit = createOctokit(process.env.ALERT_CHECK_TOKEN ?? '');
      const status = await fetchRepoSecurityStatus(octokit, owner, repo);
      return status.openDependabotAlerts > 0 ? 'bad' : 'ok';
    },
  }));

  const alertChecks: AlertCheck[] = [...ciChecks, ...securityChecks];
  startAlertScheduler(db, alertChecks);

  Bun.serve({ fetch: app.fetch, port });
  console.log(`zenvora-admin listening on :${port}`);
}
```

Note: `ALERT_CHECK_TOKEN` is a new env var (already listed in Task 1's `.env.example`) — a fine-grained, read-only PAT the background alert job uses, since it runs independently of any active browser session and therefore has no session token to borrow.

- [ ] **Step 5: Run both server tests, confirm they pass**

Run: `bun test tests/server.test.ts tests/server.integration.test.ts`
Expected: PASS

- [ ] **Step 6: Run the full test suite**

Run: `bun test`
Expected: PASS — every test file from Tasks 1–19.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: wire all routes, static serving, and the alert scheduler into the app entrypoint"
```

---

### Task 20: CI/build workflow and Dockerfile

**Files:**
- Create: `zenvora-admin/.github/workflows/ci.yml`
- Create: `zenvora-admin/Dockerfile`

**Interfaces:**
- Produces: a GHCR image at `ghcr.io/zenvoraai/zenvora-admin:<commit-sha>` on every push to `main`, gated on `bun test` passing for both backend and frontend — this is the SHA that Task 21 pins in `homelab-infra`'s `docker-compose.yml`.

- [ ] **Step 1: Write `.github/workflows/ci.yml`**

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install
      - run: bun test
      - name: frontend install and test
        working-directory: frontend
        run: |
          bun install
          bun test

  build:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    # only this job pushes to GHCR — the test job (which also runs on pull_request)
    # has no business holding write access to packages, so the elevated
    # permission is scoped here rather than at the workflow level
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/zenvoraai/zenvora-admin:${{ github.sha }}
```

- [ ] **Step 2: Write `Dockerfile`**

```dockerfile
FROM oven/bun:1 AS frontend-build
WORKDIR /app/frontend
COPY frontend/package.json frontend/bun.lock* ./
RUN bun install --frozen-lockfile
COPY frontend/ ./
RUN bun run build

FROM oven/bun:1-slim
WORKDIR /app
COPY package.json bun.lock* ./
RUN bun install --frozen-lockfile --production
COPY src ./src
COPY --from=frontend-build /app/frontend/dist ./frontend/dist
ENV PORT=3100
EXPOSE 3100
CMD ["bun", "run", "src/server.ts"]
```

Note `bun.lock` (text), not `bun.lockb` — Bun ≥1.2 writes the text-format lockfile by default; the older binary `bun.lockb` name would silently match nothing here and `--frozen-lockfile` would then fail with nothing to freeze against.

- [ ] **Step 3: Verify the image builds locally**

Run: `docker build -t zenvora-admin:local .`
Expected: build succeeds; `docker run --rm -p 3100:3100 --env-file .env zenvora-admin:local` then `curl localhost:3100/healthz` returns `{"status":"ok"}`.

- [ ] **Step 4: Commit and push**

```bash
git add -A
git commit -m "ci: add test-gated GHCR build workflow and Dockerfile"
git push
```

Expected: the Actions run on GitHub goes green and produces `ghcr.io/zenvoraai/zenvora-admin:<sha>` — copy that SHA for Task 21.

---

### Task 21: Deploy into homelab-infra (compose, nginx, README)

**Files:**
- Modify: `homelab-infra/docker-compose.yml` (append `docker-socket-proxy` and `zenvora-admin` services)
- Create: `homelab-infra/nginx/conf.d/admin.valtou.com.conf`
- Modify: `homelab-infra/README.md` (services table, mem_limit budget note)

This task follows this repo's own "Onboarding a new service" checklist (`README.md`) exactly.

**Interfaces:** none — this is the terminal task, wiring the previous 20 tasks' output into the live host.

- [ ] **Step 1: Create the GitHub App** (manual, in the browser — must happen before the deploy steps below since `zenvora-admin` won't start without real client credentials)

At `https://github.com/settings/apps/new`:
- Callback URL: `https://admin.valtou.com/auth/callback`.
- Repository permissions (read-only): Contents, Metadata, Actions, Dependabot alerts, Secret-scanning alerts.
- Organization permissions (read-only): Administration (needed for 2FA status and the Actions-usage billing endpoint).
- **Leave "Expire user authorization tokens" OFF.** It's opt-in, not default. Turning it on would require refresh-token handling that nothing in this plan implements — deliberately out of scope for a single-operator tool with an already-short 12h session TTL and one-click re-login. This is a scope decision, not an oversight: don't "fix" it later by enabling expiration without also adding refresh logic.
- Install the App on **both** the personal account (for `homelab-infra`) and the `ZenvoraAI` organization — two separate installation flows.
- Copy the generated Client ID and Client Secret; they go into `/opt/secrets/zenvora-admin/.env` in Step 6.

**Smoke-test immediately after this step, before relying on it further**: confirm `GET /user` actually returns a populated `two_factor_authentication` field for this App's user-to-server token. This was flagged during review as uncertain for GitHub Apps specifically (GitHub's docs tie the field to classic OAuth's `user` scope, not confirmed for fine-grained App permissions). If it turns out to always be `null`, Task 5's `!== true` check already fails closed (safe — it just means nobody can log in, not that the wrong person can), but the check would need rework before the tool is usable at all; don't design that fallback now, just verify this first so it's not discovered only after implementing everything else.

- [ ] **Step 2: Append the two services to `docker-compose.yml`**

```yaml
  # CONTAINERS=1 permits the whole /containers/* GET surface in this proxy image
  # (list/inspect, but also logs/top/export/archive across all 7 containers on
  # the host), not just the /containers/json listing zenvora-admin's own code
  # calls — there is no finer-grained flag in tecnativa/docker-socket-proxy to
  # restrict this further, and a bespoke allow-list reverse-proxy in front of a
  # single internal sidecar is disproportionate complexity here. Accepted as a
  # residual risk given zenvora-admin's own attack surface is minimal (no file
  # uploads, no arbitrary command execution paths, one unauthenticated route
  # pair for the OAuth callback) — written down explicitly so it's a known,
  # chosen tradeoff rather than an oversight.
  docker-socket-proxy:
    image: tecnativa/docker-socket-proxy:0.3
    container_name: homelab-docker-socket-proxy
    restart: unless-stopped
    logging: *default-logging
    environment:
      - CONTAINERS=1
      - POST=0
      - EXEC=0
      - BUILD=0
      - COMMIT=0
      - ATTACH=0
      - IMAGES=0
      - INFO=0
      - NETWORKS=0
      - VOLUMES=0
    ports:
      - "127.0.0.1:2375:2375"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    mem_limit: 16m
  zenvora-admin:
    image: ghcr.io/${GHCR_OWNER:-zenvoraai}/zenvora-admin:${ZENVORA_ADMIN_TAG:-latest}
    container_name: homelab-zenvora-admin
    network_mode: host
    restart: unless-stopped
    logging: *default-logging
    depends_on:
      - docker-socket-proxy
    # Only the tag passthroughs live here, resolved from the same project
    # .env/shell source that already parameterizes the sibling services' own
    # image lines (${FAMILY_API_TAG:-latest} etc.) — so zenvora-admin's view of
    # "what's pinned" and the actual running deploy's pin are always the same
    # value, nothing to keep in sync by hand. Every secret (App credentials,
    # encryption key, SMTP, the alert-check PAT) lives ONLY in env_file below —
    # environment: wins over env_file: in Compose, so listing a secret in both
    # places risks it being silently blanked if the shell/project .env doesn't
    # also happen to export it.
    environment:
      - PORT
      - FAMILY_API_TAG
      - SECUREVAULT_API_TAG
      - MEMORIAL_API_TAG
      - MEMORIAL_WORKER_TAG
    env_file:
      - /opt/secrets/zenvora-admin/.env
    volumes:
      - /opt/data/zenvora-admin:/app/data
    mem_limit: 128m
```

- [ ] **Step 3: Create `nginx/conf.d/admin.valtou.com.conf` — port-80 only, no 443 block yet**

The 443 block must not be committed until the certificate actually exists (Step 5) — nginx refuses to start if a `server{}` block references a missing cert file, which would take down every site sharing this nginx, not just this one. Bootstrap in phases, matching how every other vhost here must have originally been bootstrapped:

```nginx
server {
    listen 80;
    server_name admin.valtou.com;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/certbot;
        try_files $uri =404;
    }

    location / { return 301 https://$host$request_uri; }
}
```

- [ ] **Step 4: Validate and deploy phase 1 (port 80 only)**

```bash
docker compose config
bash tests/verify-container-log-limits.sh
git add docker-compose.yml nginx/conf.d/admin.valtou.com.conf
git commit -m "feat: onboard zenvora-admin and its docker-socket-proxy sidecar (pre-TLS)"
```

On the host:

```bash
cd /opt/homelab-infra
git pull
docker compose exec nginx nginx -t   # syntax check against the running container's config, matching scripts/reload-nginx-after-cert-renewal.sh's own approach
docker compose exec nginx nginx -s reload
```

- [ ] **Step 5: Provision host-side secrets, then obtain the certificate**

```bash
# On the host, create the env file with real values (never commit these):
sudo mkdir -p /opt/secrets/zenvora-admin /opt/data/zenvora-admin
sudo vi /opt/secrets/zenvora-admin/.env   # fill in every var from .env.example, including the GitHub App credentials from Step 1

sudo certbot certonly --webroot -w /var/lib/homelab-acme -d admin.valtou.com
```

- [ ] **Step 6: Add the 443 block now that the cert exists, redeploy, verify TLS**

Append to `nginx/conf.d/admin.valtou.com.conf`:

```nginx
server {
    server_name admin.valtou.com;

    location / {
        proxy_pass http://127.0.0.1:3100;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/admin.valtou.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/admin.valtou.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}
```

```bash
git add nginx/conf.d/admin.valtou.com.conf
git commit -m "feat: add TLS server block for admin.valtou.com now the cert exists"
git push
# on the host:
git pull
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload
```

- [ ] **Step 7: Bring up the app services and verify the docker-socket-proxy networking assumption**

```bash
# on the host:
export ZENVORA_ADMIN_TAG=<the-sha-from-Task-20's-CI-run>
docker compose up -d docker-socket-proxy zenvora-admin
docker compose logs --since 5m zenvora-admin

# verify the loopback-published port is actually reachable from the host
# shell — this depends on Docker's userland-proxy, which this repo's own
# /etc/docker/daemon.json disables (for the log-limit configuration above),
# so don't assume it works without checking:
curl http://127.0.0.1:2375/containers/json
```

If that `curl` fails (connection refused), the userland-proxy path isn't available on this host as suspected — fall back to giving `docker-socket-proxy` `network_mode: host` too (dropping its `ports:` mapping), matching every other service's convention in this file. This accepts the same "firewalled from the public internet" baseline assumption this repo's README already states for every host-network app port — not a new risk category, just consistent with existing practice here.

- [ ] **Step 8: Update `README.md`'s services table, hostname/networking note, and memory-budget sentence**

Add two rows to the services table:

```markdown
| `docker-socket-proxy` | `homelab-docker-socket-proxy` | 2375 (loopback only, or host-network — see onboarding notes) | none | 16m |
| `zenvora-admin` | `homelab-zenvora-admin` | 3100 | `admin.valtou.com` | 128m |
```

Update the sum sentence: `2286 MiB` → `2430 MiB` (2286 + 16 + 128), keeping the rest of that paragraph's wording (measured host size, ceiling-not-reservation framing) unchanged — real headroom was independently verified during design (see `docs/superpowers/specs/2026-08-16-zenvora-admin-design.md`, "Open risks"). Also amend the "Every service uses `network_mode: host`" sentence near the top of the services section — `docker-socket-proxy` is the one exception (unless Step 7's fallback was needed, in which case it isn't an exception after all; update whichever is actually true post-deploy). Add `ZENVORA_ADMIN_TAG` to wherever the other `*_TAG` pinned-image variables are documented.

```bash
git add README.md
git commit -m "docs: document zenvora-admin and docker-socket-proxy in the services table"
```

- [ ] **Step 9: Manual verification of the one thing this plan can't test automatically** — the real docker-socket-proxy against the real socket, and the GHCR package-version tag shape (per Task 13's note that this wasn't independently confirmed) — both called out in the spec's Testing section as needing a manual check, not an automated end-to-end test.

Log into `https://admin.valtou.com`, complete the GitHub OAuth flow with 2FA enabled on the account, and confirm:
- The Deployment Drift panel shows all 7 real containers, with `fault`/`ok`/`upgrade_available` signals that make sense against what's actually pinned in `.env` (and never a `fault` for the two `dayandyou` containers).
- The CI Health panel's Actions-quota numbers look plausible against the real enhanced-billing API response (Task 9's note on this being unverified).
- The Security Posture panel's 2FA line matches the real account state.

---

## Self-review notes

- **Spec coverage:** every spec section maps to a task — Problem/Goals → Tasks 8–15 (the four panels + alert, now covering both CI and security per the alert-scope fix); Auth section → Tasks 4–6, plus Task 21 Step 1 for the actual GitHub App creation, which the first draft of this plan omitted entirely; Architecture (single process, docker-socket-proxy, SQLite) → Tasks 1–2, 12, 16, 19; Feature panels 1–6 → Tasks 8, 9, 10, 14, 11, 15 respectively; Data flow (SQLite cache, stale-on-error, now actually surfaced to the UI) → Task 7 + Task 18; Error handling → Task 7 (stale fallback), each panel's frontend error state, and the login-page rejection-reason rendering (Task 18); Testing section's explicit list → covered 1:1 by each task's test file, plus the strengthened Task 19 integration test that catches route-mount mismatches the original version's 401-only assertion couldn't; Open risks (host memory — resolved in the spec before this plan existed; GitHub App installation scope and the `two_factor_authentication` field's availability for App tokens — both now explicit verification steps in Task 21 Step 1, not left implicit; SMTP credential — Task 21 Step 5).
- **Placeholder scan:** no TBD/TODO/"add error handling" phrasing anywhere above; every step has real, runnable code. Two points are explicitly marked as needing empirical verification during implementation rather than treated as certain — the enhanced-billing API's response shape (Task 9) and the GHCR package-version tags array shape (Task 13) — both call out exactly what to check and where, rather than silently assuming either is correct.
- **Type consistency:** `AppVariables` (Task 1) is used consistently by every `Hono<{ Variables: AppVariables }>()` instantiation from Task 6 onward; `createApp()`'s signature change from `Hono` to `{ app, db }` (Task 19) is called out explicitly with the corresponding test update; `PinnedContainer`/`DriftSignal`/`GhcrVersion` (Task 13, now tag-based: `pinnedTag`/`runningTag`/`latestTag`, not digest-based) are the same shapes consumed in Task 14 and mirrored (as plain inline types, not a cross-package import) in the frontend's `DeploymentDrift.tsx` (Task 18). Every panel route (Tasks 8–11, 14) now registers at a path relative to its `/api` mount and returns `{ data, stale }`, consistently unwrapped by the frontend's `useApiData` hook (Task 18) — this replaces the first draft, where routes were registered at absolute `/api/...` paths that Task 19's `app.route('/api', api)` would have silently double-prefixed, undetected by that draft's test because `requireAuth` intercepted before Hono ever looked for a matching route.
- **This plan was revised once already**, after an independent 3-agent review (security / plan-coverage / library-API-correctness) found 2 CRITICAL bugs (the route double-prefix, and a digest-vs-tag mismatch that would have made every container in the deployment-drift panel report `fault` permanently) plus a cluster of HIGH/MEDIUM findings, all addressed inline above rather than in a separate errata section.
