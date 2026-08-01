# Architecture

## Overview

Neelgar Society is a self-hosted, full-stack society management application
running entirely on a single Ubuntu 24.04 machine (`server-H81`), exposed
publicly through Cloudflare rather than a traditional cloud host.

## Repositories

| Repo | Role |
|---|---|
| `neelgar-society-react` | Frontend — Vite/React/TypeScript/Tailwind, built to a static bundle |
| `neelgar-society-rest` | Backend — Spring Boot 3.5 / Java 17 REST API, MySQL persistence, Flyway migrations |
| `neelgar-society-infra` | Edge layer — Cloudflare Worker for branded outage handling |
| `society-docs` | This handbook |

## Request Flow

```
Browser
  │
  ▼
Cloudflare (DNS + edge)
  │
  ▼
Cloudflare Worker (neelgar-society-infra)
  │  passes through normal traffic
  │  intercepts 502/521/522/523/525/526/530 → branded maintenance page
  ▼
Cloudflare Tunnel (cloudflared, systemd service on server-H81)
  │
  ▼
Nginx (server-H81)
  ├── static files → /var/www/neelgar (React build output)
  └── reverse proxy /api/* → Spring Boot (port 8080, local only)
                                   │
                                   ▼
                                 MySQL
```

`server-H81` sits behind CGNAT with no public IP — `cloudflared` is the only
way traffic reaches it, so there is no direct inbound port exposure.

## Why the Cloudflare Worker exists

If `server-H81`, `cloudflared`, or the backend goes down, Cloudflare has
nothing to proxy to and would otherwise show its own raw error page,
exposing terms like "origin" and "Bad gateway". The Worker intercepts these
failure codes and returns a clean branded "we'll be right back" page
instead. It deploys via GitHub-hosted runners (not the `server-H81`
self-hosted runner), so a fix can ship even if the box itself is down.

## Frontend deployment model

The React app is **not** served by its own Node process in production. CI
runs `npm run build`, producing a static `dist/` bundle, which is copied to
`/var/www/neelgar` and served directly by Nginx. Environment-specific values
(API base URL, OAuth2 client config, upload limits) are baked in at build
time via `VITE_*` GitHub Actions secrets — there is no runtime env
injection, so a config change requires a rebuild and redeploy.

## Backend

Spring Boot REST API on Java 17, MySQL for persistence, Flyway for
versioned schema migrations. Runs as a service on `server-H81`, reachable
only via Nginx's reverse proxy — not exposed directly.

## CI/CD

All three application repos use GitHub Actions with a **self-hosted
runner** on `server-H81` (except the Cloudflare Worker deploy, which
deliberately uses GitHub-hosted runners for outage resilience). Each
deploy workflow includes a build/verify step and an automatic rollback
on failure using a `.backup` copy of the previous deployed artifact.

## Known gap

The Nginx configuration itself lives only on `server-H81` and is not
version-controlled in any repository. If the box needs to be rebuilt,
this config would need to be manually recreated. See
`Troubleshooting.md` for mitigation notes.
