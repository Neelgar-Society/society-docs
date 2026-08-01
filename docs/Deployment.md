# Deployment

All three repos deploy to a single machine, `server-H81` (Ubuntu 24.04,
self-hosted, behind CGNAT, exposed only via Cloudflare Tunnel). CI/CD runs
on a **self-hosted GitHub Actions runner** on that same machine — except
the Cloudflare Worker, which deliberately uses GitHub-hosted runners (see
`Architecture.md` for why).

## Backend — `neelgar-society-rest`

Trigger: push to `main`.

| Step | What happens |
|---|---|
| Build & test | `mvn test`, then `mvn clean package -DskipTests` |
| Env file | Writes `/etc/neelgar/app.env` from GitHub Secrets (see table below) |
| Backup | Copies current `/opt/neelgar/app.jar` → `app.jar.backup` |
| Stop | `systemctl stop neelgar` |
| Deploy | Copies new JAR to `/opt/neelgar/app.jar`, `chown neelgar:neelgar` |
| Start | `systemctl start neelgar` |
| Health check | Polls `http://localhost:8080/actuator/health` every 10s, up to 200s, accepts `200` or `401` |
| On failure | Dumps `journalctl -u neelgar -n 200`, stops service, restores `app.jar.backup`, restarts |

**Required GitHub Secrets** (written into `/etc/neelgar/app.env` on deploy):
`SPRING_PROFILES_ACTIVE`, `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`,
`APP_OAUTH2_ISSUER_URL`, `APP_RSA_PRIVATE_KEY`, `APP_RSA_PUBLIC_KEY`,
`CORS_ALLOWED_ORIGINS`, `MEMBER_UPLOAD_DIR`, `COOKIE_DOMAIN`,
`GEO_IMPORT_STAGING_DIR`, `MYSQLDUMP_PATH`, `DB_BACKUP_DIR`, `SMTP_HOST`,
`SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `APP_MAIL_ENC_KEY`,
`FRONTEND_BASE_URL`.

`APP_HTTP_SECURE_COOKIES` is hardcoded to `true` in the workflow itself
(not a secret) — this only applies to the prod deploy job.

Service runs under systemd unit `neelgar`, as OS user `neelgar`.

## Frontend — `neelgar-society-react`

Trigger: push to `main`.

| Step | What happens |
|---|---|
| Build & test | `npm ci`, `npx tsc --noEmit`, `npm run build`, verifies `dist/index.html` exists |
| Backup | Copies `/var/www/neelgar` → `/var/www/neelgar.backup` if non-empty |
| Deploy | Copies `dist/.` → `/var/www/neelgar/` |
| Permissions | `chown -R www-data:www-data /var/www/neelgar` |
| Verify | Confirms `/var/www/neelgar/index.html` exists |
| On failure | Restores from `/var/www/neelgar.backup` |

**Required GitHub Secrets** (baked into the build at build time — see
`Frontend.md`/`LocalSetup.md` for what each does): `VITE_API_BASE_URL`,
`VITE_OAUTH2_BASE_URL`, `VITE_OAUTH2_CLIENT_ID`,
`VITE_OAUTH2_CLIENT_SECRET` (unused by the public client, kept for
legacy/future use), `VITE_MAX_UPLOAD_MB`, `VITE_NOTIFICATION_TIMEOUT_MS`.

Served by Nginx directly from `/var/www/neelgar` — no Node process runs
in production.

## Edge layer — `neelgar-society-infra`

Trigger: push to `main` touching `worker/`, or manual dispatch.
Runs on a **GitHub-hosted runner**, not `server-H81` — so a Worker fix can
ship even during a full outage of the box.

| Step | What happens |
|---|---|
| Deploy | `wrangler deploy` via `cloudflare/wrangler-action`, using `CLOUDFLARE_API_TOKEN` secret |

Manual fallback if CI is unavailable:
```bash
cd worker
npx wrangler login
npx wrangler deploy
```

## Cloudflare Tunnel

`cloudflared` runs as a persistent systemd service on `server-H81`,
maintaining the outbound tunnel to Cloudflare — this is the only path
traffic takes into the box (no public IP, CGNAT). Nginx sits behind the
tunnel and either serves the static frontend or reverse-proxies `/api/*`
to the Spring Boot service on `localhost:8080`.

## Rollback summary

| Component | Rollback mechanism |
|---|---|
| Backend | Automatic — restores `app.jar.backup` on health-check failure |
| Frontend | Automatic — restores `/var/www/neelgar.backup` on deploy failure |
| Worker | Manual — re-run previous commit's workflow, or `wrangler rollback` |
| Nginx config | **None** — config lives only on the box, not version-controlled (see `Architecture.md` known gap) |

## Manual server access

For anything outside CI (e.g. rotating `APP_MAIL_ENC_KEY`, inspecting
Nginx config, debugging `cloudflared`), access is via direct SSH/console
to `server-H81` — not automated, so changes made this way won't show up
in any repo history. Document any manual change here or in
`Troubleshooting.md` when it happens.
