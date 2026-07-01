# Security Hardening (fork)

This fork of `omribenami/MyApi-Open` applies security fixes found in a pre-deployment audit
(2026-07-01) before trusting it with real credentials. Below: what changed, what you MUST set,
how to deploy on Proxmox behind Cloudflare, and the residual risks.

## What was fixed

| ID | Sev | Fix | File |
|----|-----|-----|------|
| C1 | CRITICAL | Removed `GET /api/v1/turso/export-sql` — it dumped the **entire DB** to any authenticated token (incl. scoped agent tokens). Now returns 410. | `src/index.js` |
| H2 | HIGH | Removed `POST /api/v1/turso/execute` — authenticated SSRF to a user-supplied host. Now 410. | `src/index.js` |
| H3 | HIGH | Startup validation now rejects placeholder secrets (substring match) and any secret < 32 chars, not just an exact-match list. `.env.docker` placeholders now fail fast. | `src/index.js` |
| — | HIGH | Removed public-constant fallbacks for `SESSION_SECRET`, `VAULT_KEY`, and PKCE/AFP/agent secrets. Dev uses ephemeral random; prod requires real env vars. | `src/index.js`, `src/database.js` |
| M1 | MED | AFP (remote file + **arbitrary shell exec** on registered devices) is now **OFF by default**. Enable only with `AFP_ENABLED=true`. | `src/index.js`, `docker-compose.prod.yml` |
| M4 | MED | Removed unauthenticated `DELETE /invitations/…?email=` bypass. | `src/index.js` |
| — | — | `docker-compose.prod.yml` now forces `NODE_ENV=production` so secret validation + secure cookies always engage regardless of `.env`. | `docker-compose.prod.yml` |

## Required before first boot

Generate unique strong secrets (the app now REFUSES to start in production without them):

```bash
cp .env.docker .env
# generate 4 strong secrets
for v in JWT_SECRET SESSION_SECRET ENCRYPTION_KEY VAULT_KEY; do
  echo "$v=$(openssl rand -hex 32)"
done
# paste the 4 lines into .env, replacing the placeholders
```

- `ENCRYPTION_KEY` / `VAULT_KEY` must be ≥ 32 chars. Losing them = losing all stored credentials.
- Keep `.env` out of git (already in `.gitignore` — verify).
- `AFP_ENABLED` stays `false` unless you accept: a master-token leak = full RCE on every registered device.

## Deploy on Proxmox behind Cloudflare

```bash
git clone https://github.com/emiliovos/MyApi-Open.git
cd MyApi-Open
# ... create .env with strong secrets as above ...
docker compose -f docker-compose.prod.yml up -d --build
# app listens on 127.0.0.1:4500 (loopback only — not exposed to LAN)
```

Front it with your existing Cloudflare Tunnel + **Cloudflare Access** (identity-gated), e.g.
`myapi.boxlab.cloud` → `http://localhost:4500`. Do NOT expose port 4500 to the LAN or WAN directly.

## Residual risks (accepted / not fixed here)

- **H1 — master token stored reversibly & returned at login.** A leak of the DB file (given any of
  `VAULT_KEY`/`ENCRYPTION_KEY`/`JWT_SECRET`) yields the full-access master token in recoverable form.
  Mitigation: strong unique keys, DB never leaves the host, back up `./data` encrypted, rotate the
  master token if you ever suspect exposure. Acceptable for single-user personal use.
- **M2 — SSRF guard on the services proxy is string-regex (no DNS resolution).** Custom services with
  an endpoint that *resolves* to a private IP could reach the LAN. Only add trusted custom services.
- **M3 — old deps** (`multer@1.x`, `pdf-parse@1.1.1`). Only exposed via authenticated KB upload.
  Consider `npm audit` / bumping before heavy use.
- Historical: any master token ever emailed by an OLD build is permanently in that mailbox. Fresh
  deploy of this fork is fine, but rotate if you ran an older version.

## Verify the hardening is active

```bash
# should return 410
curl -s http://localhost:4500/api/v1/turso/export-sql | head
# should refuse to start if you blank out VAULT_KEY in .env (test in a throwaway env)
```

Audit report: see the deploying operator's notes (audit-260701-myapi-open-security).
