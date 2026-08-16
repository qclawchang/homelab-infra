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
- Sessions live in SQLite; the cookie holds only an opaque session ID (httpOnly, secure, `SameSite=Lax`), never the GitHub token itself. Revocation is a row delete.
- Docker state is read only via the `docker-socket-proxy` sidecar's `GET /containers/*` — the backend never mounts `/var/run/docker.sock` directly.
- Deployment drift reports two distinct signals per container, never conflated: `fault` (running digest ≠ pinned digest) and `upgrade_available` (GHCR has something newer than the pin).
- Branch protection and secret scanning both render `unavailable_on_plan`, not a false negative — GitHub Free cannot enable either on private repos.
- GitHub API responses are cached in SQLite with a 60-second freshness window; a stale cached value is served (marked stale) if a live fetch fails, rather than the panel going blank.
- The proactive alert job fires an email only on an OK→bad transition, never repeatedly while a check stays bad.
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
        RepoGrid.tsx, CiHealth.tsx, SecurityPosture.tsx, DeploymentDrift.tsx, Backlog.tsx
        __tests__/panels.test.tsx

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

FAMILY_API_DIGEST=
SECUREVAULT_API_DIGEST=
DAYANDYOU_STAGING_DIGEST=
DAYANDYOU_PROD_DIGEST=
MEMORIAL_API_DIGEST=
MEMORIAL_WORKER_DIGEST=
```

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

  test('rejects a non-whitelisted user id', async () => {
    const db = openDb(':memory:');
    db.query('INSERT INTO oauth_states (state, created_at) VALUES (?, ?)').run('good-state', Date.now());
    const app = new Hono();
    createOAuthCallbackRoute(app, db, fakeClient({ id: 9999, login: 'someone-else', two_factor_authentication: true }));

    const res = await app.request('/auth/callback?code=abc&state=good-state');

    expect(res.status).toBe(403);
    expect(db.query('SELECT * FROM sessions').all()).toHaveLength(0);
    db.close();
  });

  test('rejects a whitelisted user without 2FA enabled', async () => {
    const db = openDb(':memory:');
    db.query('INSERT INTO oauth_states (state, created_at) VALUES (?, ?)').run('good-state', Date.now());
    const app = new Hono();
    createOAuthCallbackRoute(app, db, fakeClient({ id: 4242, login: 'qclawchang', two_factor_authentication: false }));

    const res = await app.request('/auth/callback?code=abc&state=good-state');

    expect(res.status).toBe(403);
    expect(db.query('SELECT * FROM sessions').all()).toHaveLength(0);
    db.close();
  });

  test('rejects an invalid or already-used state', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    createOAuthCallbackRoute(app, db, fakeClient({ id: 4242, login: 'qclawchang', two_factor_authentication: true }));

    const res = await app.request('/auth/callback?code=abc&state=never-issued');

    expect(res.status).toBe(400);
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

export function createOAuthCallbackRoute(app: Hono, db: Database, client: GitHubOAuthClient) {
  app.get('/auth/callback', async (c) => {
    const code = c.req.query('code');
    const state = c.req.query('state');
    if (!code || !state) return c.text('missing code or state', 400);

    pruneExpiredStates(db);
    const stateRow = db.query('SELECT state FROM oauth_states WHERE state = ?').get(state);
    if (!stateRow) return c.text('invalid or expired state', 400);
    db.query('DELETE FROM oauth_states WHERE state = ?').run(state);

    let accessToken: string;
    try {
      ({ accessToken } = await client.exchangeCode(code));
    } catch {
      return c.text('token exchange failed', 502);
    }

    const user = await client.fetchUser(accessToken);

    const whitelistedId = Number(process.env.WHITELISTED_GITHUB_USER_ID);
    if (user.id !== whitelistedId) {
      return c.text('this GitHub account is not authorized for zenvora-admin', 403);
    }
    if (user.two_factor_authentication !== true) {
      return c.text('two-factor authentication must be enabled on this GitHub account', 403);
    }

    const sessionId = randomBytes(24).toString('hex');
    const now = Date.now();
    const twelveHoursMs = 12 * 60 * 60 * 1000;
    db.query(
      'INSERT INTO sessions (id, user_id, encrypted_token, created_at, expires_at) VALUES (?, ?, ?, ?, ?)'
    ).run(sessionId, user.id, encryptToken(accessToken), now, now + twelveHoursMs);

    c.header(
      'Set-Cookie',
      `zenvora_session=${sessionId}; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=${twelveHoursMs / 1000}`
    );
    return c.redirect('/');
  });
}
```

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
    c.header('Set-Cookie', 'zenvora_session=; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=0');
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
- Produces: `RepoSummary`, `fetchRepoGrid(octokit): Promise<RepoSummary[]>`, `createReposRoute(app, db, octokitFor, repos)` — mounted at `/api/repos` by Task 19.

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
      listForAuthenticatedUser: async () => ({
        data: [{ name: 'homelab-infra', full_name: 'qclawchang/homelab-infra', private: false, pushed_at: '2026-08-16T10:47:14Z', language: 'Shell', description: null }],
      }),
      listForOrg: async () => ({
        data: [{ name: 'day-and-you', full_name: 'ZenvoraAI/day-and-you', private: true, pushed_at: '2026-08-16T10:48:16Z', language: 'TypeScript', description: null }],
      }),
    },
  };
}

describe('GET /api/repos', () => {
  test('returns the combined personal + org repo grid', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); c.set('userId', 4242); await next(); });
    createReposRoute(app, db, () => fakeOctokit() as any, []);

    const res = await app.request('/api/repos');
    const body = await res.json();

    expect(res.status).toBe(200);
    expect(body.map((r: any) => r.fullName)).toEqual([
      'qclawchang/homelab-infra',
      'ZenvoraAI/day-and-you',
    ]);
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

export async function fetchRepoGrid(octokit: Octokit): Promise<RepoSummary[]> {
  const [{ data: userRepos }, { data: orgRepos }] = await Promise.all([
    octokit.repos.listForAuthenticatedUser({ affiliation: 'owner', per_page: 100 }),
    octokit.repos.listForOrg({ org: 'ZenvoraAI', per_page: 100 }),
  ]);

  return [...userRepos, ...orgRepos].map((repo: any) => ({
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
  _repos: { owner: string; repo: string }[]
) {
  app.get('/api/repos', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, `repos:${c.get('userId')}`, () => fetchRepoGrid(octokit));
    return c.json(result.data);
  });
}
```

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
- Produces: `RepoCiStatus`, `CiHealth`, `fetchCiHealth(octokit, repos)`, `createCiRoute(app, db, octokitFor, repos)` — mounted at `/api/ci` by Task 19.

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
    billing: {
      getGithubActionsBillingOrg: async () => ({ data: { total_minutes_used: 1700, included_minutes: 2000 } }),
    },
  };
}

describe('GET /api/ci', () => {
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

    const res = await app.request('/api/ci');
    const body = await res.json();

    expect(body.repos.find((r: any) => r.repo === 'ZenvoraAI/day-and-you').conclusion).toBe('failure');
    expect(body.quotaWarning).toBe(true);
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

  const { data: billing } = await octokit.billing.getGithubActionsBillingOrg({ org: 'ZenvoraAI' });

  return {
    repos: runs,
    actionsMinutesUsed: billing.total_minutes_used,
    actionsMinutesIncluded: billing.included_minutes,
    quotaWarning: billing.total_minutes_used >= billing.included_minutes * QUOTA_WARNING_RATIO,
  };
}

export function createCiRoute(
  app: Hono,
  db: Database,
  octokitFor: (token: string) => Octokit,
  repos: { owner: string; repo: string }[]
) {
  app.get('/api/ci', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, 'ci-health', () => fetchCiHealth(octokit, repos));
    return c.json(result.data);
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
- Produces: `RepoSecurityStatus`, `SecurityPosture`, `fetchSecurityPosture(octokit, repos)`, `createSecurityRoute(app, db, octokitFor, repos)` — mounted at `/api/security` by Task 19.

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

describe('GET /api/security', () => {
  test('reports 2FA status, alert counts, and marks unsupported checks unavailable-on-plan', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); await next(); });
    createSecurityRoute(app, db, () => fakeOctokit() as any, [{ owner: 'ZenvoraAI', repo: 'day-and-you' }]);

    const res = await app.request('/api/security');
    const body = await res.json();

    expect(body.twoFactorEnabled).toBe(true);
    expect(body.repos[0]).toEqual({
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

export async function fetchSecurityPosture(
  octokit: Octokit,
  repos: { owner: string; repo: string }[]
): Promise<SecurityPosture> {
  const { data: user } = await octokit.users.getAuthenticated();

  const repoStatuses = await Promise.all(
    repos.map(async ({ owner, repo }) => {
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
    })
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
  app.get('/api/security', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, 'security-posture', () => fetchSecurityPosture(octokit, repos));
    return c.json(result.data);
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
- Produces: `BacklogItem`, `Backlog`, `fetchBacklog(octokit, username, repos)`, `createBacklogRoute(app, db, octokitFor, username, repos)` — mounted at `/api/backlog` by Task 19.

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

describe('GET /api/backlog', () => {
  test('aggregates review-requested PRs, assigned issues, and stale branches', async () => {
    const db = openDb(':memory:');
    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); await next(); });
    createBacklogRoute(app, db, () => fakeOctokit() as any, 'qclawchang', [{ owner: 'ZenvoraAI', repo: 'day-and-you' }]);

    const res = await app.request('/api/backlog');
    const body = await res.json();

    expect(body.reviewRequestedPrs).toHaveLength(1);
    expect(body.assignedIssues).toHaveLength(1);
    expect(body.staleBranches).toEqual([{ repo: 'ZenvoraAI/day-and-you', branch: 'old-feature', lastCommitAt: expect.any(String) }]);
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
  const staleBranches: Backlog['staleBranches'] = [];

  for (const { owner, repo } of repos) {
    const { data: branches } = await octokit.repos.listBranches({ owner, repo, per_page: 100 });
    for (const branch of branches) {
      const { data: commit } = await octokit.repos.getCommit({ owner, repo, ref: branch.commit.sha });
      const commitDate = commit.commit.committer?.date;
      if (commitDate && new Date(commitDate).getTime() < staleCutoff) {
        staleBranches.push({ repo: `${owner}/${repo}`, branch: branch.name, lastCommitAt: commitDate });
      }
    }
  }

  return {
    reviewRequestedPrs: prSearch.data.items.map(toItem),
    assignedIssues: issueSearch.data.items.map(toItem),
    staleBranches,
  };
}

export function createBacklogRoute(
  app: Hono,
  db: Database,
  octokitFor: (token: string) => Octokit,
  username: string,
  repos: { owner: string; repo: string }[]
) {
  app.get('/api/backlog', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, 'backlog', () => fetchBacklog(octokit, username, repos));
    return c.json(result.data);
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
- Produces: `RunningContainer { name, image, imageDigest }`, `listRunningContainers(proxyBaseUrl?): Promise<RunningContainer[]>` — consumed by Task 14.

- [ ] **Step 1: Write the failing test**

```ts
// tests/deployment/dockerProxy.test.ts
import { describe, test, expect, mock, afterEach } from 'bun:test';
import { listRunningContainers } from '../../src/deployment/dockerProxy';

const originalFetch = globalThis.fetch;
afterEach(() => { globalThis.fetch = originalFetch; });

describe('listRunningContainers', () => {
  test('maps the docker-socket-proxy response into RunningContainer records', async () => {
    globalThis.fetch = mock(async () =>
      new Response(
        JSON.stringify([
          { Names: ['/homelab-family-api'], Image: 'ghcr.io/zenvoraai/family-media-api:abc123', ImageID: 'sha256:deadbeef' },
        ]),
        { status: 200 }
      )
    ) as any;

    const result = await listRunningContainers('http://fake-proxy:2375');

    expect(result).toEqual([
      { name: 'homelab-family-api', image: 'ghcr.io/zenvoraai/family-media-api:abc123', imageDigest: 'sha256:deadbeef' },
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
  imageDigest: string | null;
}

export async function listRunningContainers(
  proxyBaseUrl: string = process.env.DOCKER_PROXY_URL ?? 'http://127.0.0.1:2375'
): Promise<RunningContainer[]> {
  const res = await fetch(`${proxyBaseUrl}/containers/json`);
  if (!res.ok) throw new Error(`docker-socket-proxy returned ${res.status}`);

  const containers = (await res.json()) as Array<{ Names: string[]; Image: string; ImageID: string }>;

  return containers.map((c) => ({
    name: c.Names[0]?.replace(/^\//, '') ?? 'unknown',
    image: c.Image,
    imageDigest: c.ImageID.startsWith('sha256:') ? c.ImageID : null,
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

### Task 13: GHCR digest comparison (drift signals)

**Files:**
- Create: `src/deployment/drift.ts`
- Test: `tests/deployment/drift.test.ts`

**Interfaces:**
- Produces: `PinnedContainer { name, ghcrPackage, pinnedDigest }`, `GhcrVersion { digest, createdAt }`, `DriftSignal` (discriminated union: `ok` | `fault` | `upgrade_available`), `compareDrift(pinned, runningDigest, latestGhcrVersion): DriftSignal`, `fetchLatestGhcrVersion(octokit, packageName): Promise<GhcrVersion | null>` — both consumed by Task 14.

- [ ] **Step 1: Write the failing test**

```ts
// tests/deployment/drift.test.ts
import { describe, test, expect } from 'bun:test';
import { compareDrift, fetchLatestGhcrVersion } from '../../src/deployment/drift';

describe('compareDrift', () => {
  const pinned = { name: 'family-api', ghcrPackage: 'family-media-api', pinnedDigest: 'sha256:aaa' };

  test('flags a fault when the running digest does not match the pin', () => {
    const signal = compareDrift(pinned, 'sha256:bbb', { digest: 'sha256:aaa', createdAt: '2026-08-16T00:00:00Z' });
    expect(signal).toEqual({ kind: 'fault', runningDigest: 'sha256:bbb', pinnedDigest: 'sha256:aaa' });
  });

  test('flags upgrade-available when the pin is correct but GHCR has something newer', () => {
    const signal = compareDrift(pinned, 'sha256:aaa', { digest: 'sha256:ccc', createdAt: '2026-08-16T00:00:00Z' });
    expect(signal).toEqual({ kind: 'upgrade_available', pinnedDigest: 'sha256:aaa', latestDigest: 'sha256:ccc' });
  });

  test('reports ok when the running digest matches the pin and nothing newer exists', () => {
    const signal = compareDrift(pinned, 'sha256:aaa', { digest: 'sha256:aaa', createdAt: '2026-08-16T00:00:00Z' });
    expect(signal).toEqual({ kind: 'ok' });
  });
});

describe('fetchLatestGhcrVersion', () => {
  test('returns the most recently created version', async () => {
    const octokit = { packages: { getAllPackageVersionsForPackageOwnedByOrg: async () => ({ data: [{ name: 'sha256:aaa', created_at: '2026-08-16T00:00:00Z' }] }) } };
    expect(await fetchLatestGhcrVersion(octokit as any, 'family-media-api')).toEqual({ digest: 'sha256:aaa', createdAt: '2026-08-16T00:00:00Z' });
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
  pinnedDigest: string;
}

export interface GhcrVersion {
  digest: string;
  createdAt: string;
}

export type DriftSignal =
  | { kind: 'ok' }
  | { kind: 'fault'; runningDigest: string | null; pinnedDigest: string }
  | { kind: 'upgrade_available'; pinnedDigest: string; latestDigest: string };

export function compareDrift(
  pinned: PinnedContainer,
  runningDigest: string | null,
  latestGhcrVersion: GhcrVersion | null
): DriftSignal {
  if (runningDigest !== pinned.pinnedDigest) {
    return { kind: 'fault', runningDigest, pinnedDigest: pinned.pinnedDigest };
  }
  if (latestGhcrVersion && latestGhcrVersion.digest !== pinned.pinnedDigest) {
    return { kind: 'upgrade_available', pinnedDigest: pinned.pinnedDigest, latestDigest: latestGhcrVersion.digest };
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

  const latest = versions[0];
  if (!latest) return null;
  return { digest: latest.name, createdAt: latest.created_at };
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
- Produces: `fetchDeploymentDrift(octokit, pinned)`, `createDeploymentRoute(app, db, octokitFor, pinned)` — mounted at `/api/deployment` by Task 19.

- [ ] **Step 1: Write the failing test**

```ts
// tests/deployment/route.test.ts
import { describe, test, expect, mock } from 'bun:test';

mock.module('../../src/deployment/dockerProxy', () => ({
  listRunningContainers: async () => [
    { name: 'homelab-family-api', image: 'ghcr.io/zenvoraai/family-media-api:abc', imageDigest: 'sha256:aaa' },
  ],
}));

const { Hono } = await import('hono');
const { openDb } = await import('../../src/db/client');
const { createDeploymentRoute } = await import('../../src/deployment/route');

describe('GET /api/deployment', () => {
  test('reports a container matching its pin as ok', async () => {
    const db = openDb(':memory:');
    const octokit = {
      packages: {
        getAllPackageVersionsForPackageOwnedByOrg: async () => ({ data: [{ name: 'sha256:aaa', created_at: '2026-08-16T00:00:00Z' }] }),
      },
    };

    const app = new Hono();
    app.use('*', async (c, next) => { c.set('githubToken', 'ghu_x'); await next(); });
    createDeploymentRoute(app, db, () => octokit as any, [
      { name: 'homelab-family-api', ghcrPackage: 'family-media-api', pinnedDigest: 'sha256:aaa' },
    ]);

    const res = await app.request('/api/deployment');
    const body = await res.json();

    expect(body).toEqual([{ container: 'homelab-family-api', signal: { kind: 'ok' } }]);
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
      const signal = compareDrift(p, runningContainer?.imageDigest ?? null, latest);
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
  app.get('/api/deployment', async (c) => {
    const octokit = octokitFor(c.get('githubToken') as string);
    const result = await cachedFetch(db, 'deployment-drift', () => fetchDeploymentDrift(octokit, pinned));
    return c.json(result.data);
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
    const status = await check.evaluate();
    const transition = recordCheckAndGetTransition(db, check.key, status);
    if (transition && transition.to === 'bad') {
      await notify({
        subject: `zenvora-admin alert: ${check.describe}`,
        body: `${check.describe} went from ${transition.from} to bad at ${new Date().toISOString()}.`,
      });
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
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { Hono } from 'hono';
import { mountStaticFrontend } from '../src/static';

describe('mountStaticFrontend', () => {
  let dir: string;

  beforeAll(() => {
    dir = mkdtempSync(join(tmpdir(), 'zenvora-static-'));
    writeFileSync(join(dir, 'index.html'), '<html><body>zenvora-admin</body></html>');
  });

  afterAll(() => { rmSync(dir, { recursive: true, force: true }); });

  test('serves index.html at the root path', async () => {
    const app = new Hono();
    mountStaticFrontend(app, dir);

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
- Create: `frontend/src/components/RepoGrid.tsx`, `CiHealth.tsx`, `SecurityPosture.tsx`, `DeploymentDrift.tsx`, `Backlog.tsx`
- Modify: `frontend/src/App.tsx` (render all 5 panels)
- Test: `frontend/src/components/__tests__/panels.test.tsx`

**Interfaces:**
- Consumes: `apiGet`, `ApiError` (Task 17); response shapes from `RepoSummary` (Task 8), `CiHealth` (Task 9), `SecurityPosture` (Task 10), deployment drift rows (Task 14), `Backlog` (Task 11) — kept as inline TS interfaces here rather than importing across the frontend/backend boundary.

- [ ] **Step 1: Write `frontend/src/hooks/useApiData.ts`**

```tsx
import { useEffect, useState } from 'react';
import { apiGet, ApiError } from '../api';

export type ApiDataState<T> =
  | { status: 'loading' }
  | { status: 'error'; message: string }
  | { status: 'success'; data: T };

export function useApiData<T>(path: string): ApiDataState<T> {
  const [state, setState] = useState<ApiDataState<T>>({ status: 'loading' });

  useEffect(() => {
    let cancelled = false;
    apiGet<T>(path)
      .then((data) => { if (!cancelled) setState({ status: 'success', data }); })
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
    globalThis.fetch = mock(async () => new Response(JSON.stringify([
      { name: 'homelab-infra', fullName: 'qclawchang/homelab-infra', private: false, pushedAt: null, language: 'Shell', description: null },
    ]), { status: 200 })) as any;

    render(<RepoGrid />);
    await waitFor(() => expect(screen.getByText('homelab-infra')).toBeDefined());
  });

  test("shows a retry message on failure", async () => {
    globalThis.fetch = mock(async () => new Response('error', { status: 500 })) as any;
    render(<RepoGrid />);
    await waitFor(() => expect(screen.getByText(/couldn't load repos/)).toBeDefined());
  });
});

describe('DeploymentDrift', () => {
  test('visually distinguishes a fault from an upgrade-available signal', async () => {
    globalThis.fetch = mock(async () => new Response(JSON.stringify([
      { container: 'homelab-family-api', signal: { kind: 'fault', runningDigest: 'sha256:bbb', pinnedDigest: 'sha256:aaa' } },
      { container: 'homelab-securevault-api', signal: { kind: 'upgrade_available', pinnedDigest: 'sha256:aaa', latestDigest: 'sha256:ccc' } },
    ]), { status: 200 })) as any;

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
  | { kind: 'fault'; runningDigest: string | null; pinnedDigest: string }
  | { kind: 'upgrade_available'; pinnedDigest: string; latestDigest: string };

interface DeploymentRow { container: string; signal: DriftSignal; }

export function DeploymentDrift() {
  const state = useApiData<DeploymentRow[]>('/api/deployment');

  if (state.status === 'loading') return <section className="card">loading deployment status…</section>;
  if (state.status === 'error') return <section className="card status-fault">couldn't load deployment status, retry</section>;

  return (
    <section>
      <h2>Deployment drift</h2>
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

- [ ] **Step 9: Update `frontend/src/App.tsx` to render all five panels**

```tsx
import { RepoGrid } from './components/RepoGrid';
import { CiHealth } from './components/CiHealth';
import { SecurityPosture } from './components/SecurityPosture';
import { DeploymentDrift } from './components/DeploymentDrift';
import { Backlog } from './components/Backlog';

export function App() {
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

- [ ] **Step 10: Run it, confirm it passes**

Run: `cd frontend && bun test src/components/__tests__/panels.test.tsx`
Expected: PASS

- [ ] **Step 11: Run `bun run dev` and check the golden path in a browser** — per this project's own standard, don't claim a UI change works without seeing it render. Log in via a real GitHub App test installation if credentials are available yet; otherwise confirm each panel's loading/error states render correctly with the dev server up and `/api/*` returning 401.

- [ ] **Step 12: Commit**

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
  { name: 'homelab-family-api', ghcrPackage: 'family-media-api', pinnedDigest: process.env.FAMILY_API_DIGEST ?? '' },
  { name: 'homelab-securevault-api', ghcrPackage: 'securevault-api', pinnedDigest: process.env.SECUREVAULT_API_DIGEST ?? '' },
  { name: 'homelab-dayandyou-staging', ghcrPackage: 'dayandyou', pinnedDigest: process.env.DAYANDYOU_STAGING_DIGEST ?? '' },
  { name: 'homelab-dayandyou-prod', ghcrPackage: 'dayandyou', pinnedDigest: process.env.DAYANDYOU_PROD_DIGEST ?? '' },
  { name: 'homelab-memorial-api', ghcrPackage: 'aiqiuqi-memorial-api', pinnedDigest: process.env.MEMORIAL_API_DIGEST ?? '' },
  { name: 'homelab-memorial-worker', ghcrPackage: 'aiqiuqi-memorial-worker', pinnedDigest: process.env.MEMORIAL_WORKER_DIGEST ?? '' },
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

  const alertChecks: AlertCheck[] = REPOS.map(({ owner, repo }) => ({
    key: `ci:${owner}/${repo}`,
    describe: `CI for ${owner}/${repo}`,
    evaluate: async () => {
      const octokit = createOctokit(process.env.ALERT_CHECK_TOKEN ?? '');
      const { data } = await octokit.actions.listWorkflowRunsForRepo({ owner, repo, per_page: 1 });
      return data.workflow_runs[0]?.conclusion === 'failure' ? 'bad' : 'ok';
    },
  }));
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
  packages: write

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
COPY frontend/package.json frontend/bun.lockb* ./
RUN bun install --frozen-lockfile
COPY frontend/ ./
RUN bun run build

FROM oven/bun:1-slim
WORKDIR /app
COPY package.json bun.lockb* ./
RUN bun install --frozen-lockfile --production
COPY src ./src
COPY --from=frontend-build /app/frontend/dist ./frontend/dist
ENV PORT=3100
EXPOSE 3100
CMD ["bun", "run", "src/server.ts"]
```

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

- [ ] **Step 1: Append the two services to `docker-compose.yml`** — `docker-socket-proxy` binds only to loopback (`127.0.0.1:2375`), not `network_mode: host`, so its API is reachable from `zenvora-admin` (which is host-networked, and can therefore reach any loopback-published port) without being exposed on any non-loopback interface

```yaml
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
    environment:
      - PORT
      - GITHUB_APP_CLIENT_ID
      - GITHUB_APP_CLIENT_SECRET
      - GITHUB_APP_REDIRECT_URI
      - WHITELISTED_GITHUB_USER_ID
      - TOKEN_ENCRYPTION_KEY
      - ALERT_CHECK_TOKEN
      - DOCKER_PROXY_URL
      - SMTP_HOST
      - SMTP_PORT
      - SMTP_SECURE
      - SMTP_USER
      - SMTP_PASS
      - ALERT_EMAIL_FROM
      - ALERT_EMAIL_TO
      - FAMILY_API_DIGEST
      - SECUREVAULT_API_DIGEST
      - DAYANDYOU_STAGING_DIGEST
      - DAYANDYOU_PROD_DIGEST
      - MEMORIAL_API_DIGEST
      - MEMORIAL_WORKER_DIGEST
    env_file:
      - /opt/secrets/zenvora-admin/.env
    volumes:
      - /opt/data/zenvora-admin:/app/data
    mem_limit: 128m
```

- [ ] **Step 2: Create `nginx/conf.d/admin.valtou.com.conf`** — same shape as the other vhosts in this directory

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

- [ ] **Step 3: Update `README.md`'s services table and memory-budget note**

Add two rows to the services table:

```markdown
| `docker-socket-proxy` | `homelab-docker-socket-proxy` | 2375 (loopback only) | none | 16m |
| `zenvora-admin` | `homelab-zenvora-admin` | 3100 | `admin.valtou.com` | 128m |
```

Update the sum sentence: `2286 MiB` → `2430 MiB` (2286 + 16 + 128), keeping the rest of that paragraph's wording (measured host size, ceiling-not-reservation framing) unchanged — real headroom was independently verified during design (see `docs/superpowers/specs/2026-08-16-zenvora-admin-design.md`, "Open risks").

- [ ] **Step 4: Validate locally**

```bash
docker compose config
bash tests/verify-container-log-limits.sh
sudo nginx -t -c "$(pwd)/nginx/nginx.conf" 2>&1 || echo "run nginx syntax check on the host instead if not installed locally"
```

Expected: `docker compose config` prints a valid merged config; `verify-container-log-limits.sh` exits 0 (both new services carry `logging: *default-logging`).

- [ ] **Step 5: Commit**

```bash
git add docker-compose.yml nginx/conf.d/admin.valtou.com.conf README.md
git commit -m "feat: onboard zenvora-admin and its docker-socket-proxy sidecar"
```

- [ ] **Step 6: Provision host-side secrets and deploy** (manual, on the Lightsail host — not automatable from here)

```bash
# On the host, create the env file with real values (never commit these):
sudo mkdir -p /opt/secrets/zenvora-admin /opt/data/zenvora-admin
sudo vi /opt/secrets/zenvora-admin/.env   # fill in every var from .env.example

cd /opt/homelab-infra
git pull
export ZENVORA_ADMIN_TAG=<the-sha-from-Task-20's-CI-run>
docker compose up -d docker-socket-proxy zenvora-admin
docker compose logs --since 5m zenvora-admin
```

- [ ] **Step 7: Register the certbot webroot hostname and verify TLS** (per README's "Onboarding a new service" step 5, and its Certbot section)

```bash
# On the host:
sudo certbot certonly --webroot -w /var/lib/homelab-acme -d admin.valtou.com
docker compose restart nginx
curl -I https://admin.valtou.com/healthz
```

Expected: `200` from the health check over HTTPS, confirming nginx → `zenvora-admin` routing and the certificate both work.

- [ ] **Step 8: Manual verification of the one thing this plan can't test automatically** — the real docker-socket-proxy against the real socket (per spec's Testing section, this was called out as needing a manual check, not an automated end-to-end test)

Log into `https://admin.valtou.com`, complete the GitHub OAuth flow with 2FA enabled on the account, and confirm the Deployment Drift panel shows all 7 real containers with `ok` signals (assuming the digests in `.env` are current) rather than an error.

---

## Self-review notes

- **Spec coverage:** every spec section maps to a task — Problem/Goals → Tasks 8–15 (the four panels + alert); Auth section → Tasks 4–6; Architecture (single process, docker-socket-proxy, SQLite) → Tasks 1–2, 12, 16, 19; Feature panels 1–6 → Tasks 8, 9, 10, 14, 11, 15 respectively; Data flow (SQLite cache, stale-on-error) → Task 7; Error handling → Task 7 (stale fallback) and each panel's frontend error state (Task 18); Testing section's explicit list → covered 1:1 by each task's test file; Open risks (host memory, GitHub App installation scope, SMTP credential) → memory was resolved in the spec itself before this plan was written; App installation scope and SMTP credential remain manual provisioning steps in Task 21 Step 6, which is the correct place for them since they're host-specific secrets, not code.
- **Placeholder scan:** no TBD/TODO/"add error handling" phrasing anywhere above; every step has real, runnable code.
- **Type consistency:** `AppVariables` (Task 1) is used consistently by every `Hono<{ Variables: AppVariables }>()` instantiation from Task 6 onward; `createApp()`'s signature change from `Hono` to `{ app, db }` (Task 19) is called out explicitly with the corresponding test update, so it isn't a silent break; `PinnedContainer`/`DriftSignal`/`GhcrVersion` (Task 13) are the same shapes consumed in Task 14 and mirrored (as plain inline types, not a cross-package import) in the frontend's `DeploymentDrift.tsx` (Task 18).
