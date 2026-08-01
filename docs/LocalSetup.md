# Local Setup

## Prerequisites

| Tool | Version |
|---|---|
| Java | 17 |
| Maven | bundled wrapper (`./mvnw`) or 3.9+ |
| Node.js | 20+ (matches CI) |
| MySQL | 8.x |
| Git | any recent version |

## 1. Clone repos

```bash
git clone https://github.com/Neelgar-Society/neelgar-society-rest.git
git clone https://github.com/Neelgar-Society/neelgar-society-react.git
```

## 2. Database

Create a local MySQL database. Flyway will handle schema migrations on
startup (`baseline-on-migrate: true`) — do not run manual DDL.

```sql
CREATE DATABASE neelgar_society_dev;
```

## 3. Backend — `neelgar-society-rest`

No `.env.example` currently exists in this repo — the variables below are
pulled directly from `application.yml` / `application-dev.yml`. Set them as
actual environment variables, or in your IDE's run configuration.

| Variable | Purpose | Dev suggestion |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | Active profile | `dev` |
| `DB_URL` | MySQL JDBC URL | `jdbc:mysql://localhost:3306/neelgar_society_dev` |
| `DB_USERNAME` | MySQL user | — |
| `DB_PASSWORD` | MySQL password | — |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USERNAME` / `SMTP_PASSWORD` | Bootstrap mail account seed (only used once, on first startup, to seed `mail_account`) | Zoho or any test SMTP |
| `FRONTEND_BASE_URL` | Used to build links in emails (e.g. logo URL) | `http://localhost:5173` |
| `APP_OAUTH2_ISSUER_URL` | OAuth2 issuer | `http://localhost:8080` |
| `APP_RSA_PRIVATE_KEY` / `APP_RSA_PUBLIC_KEY` | JWT signing keypair | generate a local dev keypair, never reuse prod keys |
| `APP_MAIL_ENC_KEY` | AES-256-GCM key encrypting `mail_account` credentials | any 32-byte base64 key for dev |
| `CORS_ALLOWED_ORIGINS` | Allowed frontend origins | `http://localhost:5173` |
| `DB_BACKUP_DIR` / `MYSQLDUMP_PATH` | Scheduled DB backup job | any local path / `mysqldump` binary path |
| `MEMBER_UPLOAD_DIR` | Member photo uploads | any local path |
| `GEO_IMPORT_STAGING_DIR` | Geo data import staging | any local path |
| `APP_HTTP_SECURE_COOKIES` | Defaults to `false` — leave unset locally | — |
| `COOKIE_DOMAIN` | Defaults to empty — leave unset locally | — |

Swagger UI is **enabled only under the `dev` profile**, at `/swagger-ui.html`.

```bash
cd neelgar-society-rest
./mvnw spring-boot:run
```

Runs on `http://localhost:8080`.

## 4. Frontend — `neelgar-society-react`

No `.env.example` exists here either — variables are consumed via
`import.meta.env` and injected at build time by Vite.

| Variable | Purpose |
|---|---|
| `VITE_API_BASE_URL` | Backend API base URL |
| `VITE_OAUTH2_BASE_URL` | OAuth2 base URL |
| `VITE_OAUTH2_CLIENT_ID` | OAuth2 client id (`neelgar-web`, public client — no secret required despite the var existing in CI secrets) |
| `VITE_MAX_UPLOAD_MB` | Client-side upload size guard |
| `VITE_NOTIFICATION_TIMEOUT_MS` | Toast/notification duration |

Create `neelgar-society-react/.env`:
```
VITE_API_BASE_URL=http://localhost:8080
VITE_OAUTH2_BASE_URL=http://localhost:8080
VITE_OAUTH2_CLIENT_ID=neelgar-web
VITE_MAX_UPLOAD_MB=5
VITE_NOTIFICATION_TIMEOUT_MS=4000
```

```bash
cd neelgar-society-react
npm ci
npm run dev
```

Runs on `http://localhost:5173`. `vite.config.ts` already proxies `/api` and
`/oauth2` to `http://localhost:8080`, so no CORS workaround is needed in dev.

## 5. Verify

- Backend health: `http://localhost:8080/actuator/health`
- Swagger: `http://localhost:8080/swagger-ui.html` (dev profile only)
- Frontend: `http://localhost:5173`

## Known gaps (tracked, not yet fixed)

- Neither repo has a committed `.env.example` — new devs currently have to
  reconstruct required variables from `application.yml` / grep for
  `import.meta.env`, as done above. Worth adding one to each repo.
  