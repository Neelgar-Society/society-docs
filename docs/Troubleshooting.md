# Troubleshooting

Known issues and their fixes, plus common failure modes for CI/deploy.
Add new entries here as they're diagnosed — don't let tribal knowledge
stay tribal.

## Known issues (previously diagnosed)

### Cloudflare cache poisoning on logo asset
**Symptom:** Branded email logo (served via `app.frontend.base-url`)
intermittently failed to load or showed stale content.
**Cause:** Cloudflare had cached a stale response for the logo asset with
an incorrect `Content-Type` (HTML instead of image), and kept serving it.
**Fix:** Purge the specific asset from Cloudflare cache (Dashboard →
Caching → Configuration → Purge Cache → Custom Purge, or via API), and
verify the origin (Nginx) returns the correct `Content-Type` header for
static assets before re-caching.

### React Joyride v2 → v3 API changes
**Symptom:** Onboarding tour steps broke after upgrading `react-joyride`.
**Cause:** v3 changed the controlled-mode step API — callback signatures
and step-state handling differ from v2.
**Fix:** Tour step management was rewritten around v3's controlled-mode
API with a custom `TourTooltip.tsx`. If upgrading `react-joyride` again,
check the changelog for controlled-mode breaking changes before assuming
existing tour code still works.

### Hooks-before-return crash
**Symptom:** "Rendered fewer hooks than expected" React error.
**Cause:** A conditional early `return` was placed before a `useState`/
`useEffect` call in a component.
**Fix:** All hooks must run unconditionally at the top of the component,
before any early return. If a hook's behavior needs to be conditional,
put the condition inside the hook, not around it.

### Double-toggle checkbox bug
**Symptom:** Checkbox required two clicks to change state visually.
**Cause:** Both the checkbox's own `onChange` and a parent click handler
were toggling the same state, double-firing the update.
**Fix:** Ensure only one handler owns the toggle — either the input's
`onChange` or a wrapping `onClick`, not both.

### Missing `@PostMapping`
**Symptom:** New endpoint returned 405 Method Not Allowed.
**Cause:** Controller method was missing the `@PostMapping` annotation
entirely (copy-paste from a `@GetMapping` method without updating it).
**Fix:** Double-check the HTTP verb annotation matches the intended verb
whenever copying an existing controller method as a template.

### Mobile sidebar DOM-copy targeting
**Symptom:** Onboarding tour highlighting broke specifically on mobile,
where the sidebar is rendered as a DOM copy (for the slide-out drawer)
rather than the same node used on desktop.
**Fix:** Tour step targets must account for the mobile sidebar being a
separate DOM node, not the same element reference as desktop — target
selectors need a mobile-specific path or a shared `data-tour-id`
attribute that exists on both copies.

## Common CI/deploy failures

| Symptom | Likely cause | Where to look |
|---|---|---|
| Backend health check fails after deploy | New env var missing from `/etc/neelgar/app.env` | GitHub Secrets list in `Deployment.md` |
| Backend deploy rolls back automatically | Health check didn't return 200/401 within 200s | `sudo journalctl -u neelgar -n 200` (also dumped automatically on failure) |
| Frontend shows old content after deploy | Cloudflare edge cache, not a deploy failure | Purge cache for the affected path |
| Frontend build fails on `tsc -b` | Type error introduced, not a lint warning | Run `npx tsc --noEmit` locally before pushing |
| Cloudflare Worker not intercepting errors | Route mismatch or `FALLBACK_CODES` doesn't include the status seen | `worker/wrangler.toml` routes, `worker/src/index.js` |
| Site fully down, no branded maintenance page shown | `cloudflared` itself down, or DNS/route misconfigured | `sudo systemctl status cloudflared` on `server-H81` |

## Open gaps (not yet fixed — tracked here until resolved)

- **Nginx config is not version-controlled.** It lives only on
  `server-H81`. A box rebuild would require manually recreating it from
  memory/backup. Candidate fix: commit it into `neelgar-society-infra`.
- **No `.env.example` in either `neelgar-society-rest` or
  `neelgar-society-react`.** Required variables currently have to be
  reconstructed from `application.yml` / `import.meta.env` usage (see
  `LocalSetup.md`).
- **No frontend tests exist.** Testing strategy for
  `neelgar-society-react` is undefined.
- **Migration file numbering discrepancy.** Docs/memory referenced
  `V9`/`V10` Flyway migrations for the T&C consent system, but the
  snapshot reviewed while writing these docs only contained up to
  `V9__schema.sql`. Confirm `V10` is actually merged to `develop` before
  relying on this.

## Pending bookmarked fix (not yet applied)

In `updateMember`'s deceased/DOD branch:
(a) auto-deactivate the family if the deceased was the last active
member, and
(b) prompt head-reassignment if the deceased was head and other active
members remain.
Combine both into one change when picked up.
