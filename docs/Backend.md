# Backend — `neelgar-society-rest`

Spring Boot 3.5 / Java 17 REST API, MySQL persistence via Flyway-versioned
migrations, JWT-based OAuth2 auth.

## Package structure (package-by-layer)

```
com.neelgar.society/
├── aspect/          # cross-cutting concerns (e.g. logging/audit aspects)
├── config/
│   └── security/    # Spring Security, JWT, OAuth2 config
├── constants/
├── controller/       # REST endpoints
├── dto/
│   ├── request/
│   ├── response/
│   └── aladhan/     # Hijri calendar API DTOs
├── entity/          # JPA entities
├── enums/
├── exception/
│   ├── auth/
│   └── impl/        # GlobalExceptionHandler lives here
├── geoimport/        # geo data import feature (own dto/exception subpackages)
├── mapper/
├── repository/       # Spring Data JPA repositories
├── scheduler/        # cron-based jobs (see Scheduler section below)
├── service/
│   └── impl/
└── util/
```

## Controller conventions

Every controller method **must** include OpenAPI/Swagger annotations.
Real example (`EventController`):

```java
@Slf4j
@RestController
@RequestMapping("/api/v1/events")
@RequiredArgsConstructor
@Tag(name = "Events", description = "Manage society events (Vivah Samelan, meetings, camps, etc.)")
@SecurityRequirement(name = "bearerAuth")
public class EventController {

    @Operation(summary = "...")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "..."),
        @ApiResponse(responseCode = "404", description = "...")
    })
    @PreAuthorize("hasRole('...')")
    @GetMapping("/{id}")
    public ResponseEntity<SocApiResponse<EventResponse>> getEvent(@PathVariable Long id) {
        ...
    }
}
```

Standards:
- `@Tag` at class level — one tag per resource.
- `@SecurityRequirement(name = "bearerAuth")` on any authenticated controller.
- `@Operation` + `@ApiResponses` on every method — no exceptions.
- `@PreAuthorize` for role-gated endpoints.
- Responses wrapped in `SocApiResponse<T>` — a consistent envelope, not raw entities/DTOs.
- Public (unauthenticated) endpoints live under a `Public*Controller` naming
  pattern (e.g. `PublicMemberApplicationController`, `PublicLeadershipController`).

## Error handling

Centralized in `exception/impl/GlobalExceptionHandler.java` — controllers
should throw domain exceptions (see `exception/` and `exception/auth/`)
rather than building error responses manually.

## Database & migrations

- Flyway migrations in `src/main/resources/db/migration/`, strictly
  sequential (`V1__schema.sql`, `V2__seed_data.sql`, ... `V9__schema.sql`
  at time of writing).
- Naming: `V<n>__<description>.sql`. Never edit a migration that has
  already shipped — add a new `V<n+1>` instead.
- `ddl-auto: none` — schema changes only ever happen through Flyway,
  never Hibernate auto-DDL.

## Configuration profiles

- `application.yml` — shared defaults, all `${ENV_VAR}` placeholders.
- `application-dev.yml` — local dev overrides: enables Swagger UI at
  `/swagger-ui.html`, sets datasource from `DB_URL`/`DB_USERNAME`/`DB_PASSWORD`.
- `application-prod.yml` — production overrides.
- Swagger is **disabled by default** and only turned on under `dev`.

## Scheduled jobs

Defined in `application.yml` under `app.scheduler`, implemented in
`scheduler/`:

| Job | Schedule | Purpose |
|---|---|---|
| OAuth2 token cleanup | daily 2am | purge expired tokens |
| Audit log cleanup | monthly, 1st, 3am | enforce `audit-retention-days` |
| Account unlock | every 5 min | unlock accounts after lockout window |
| Photo cleanup | weekly, Sunday 4am | remove orphaned member photos |
| DB backup | daily 1:30am | `mysqldump` to `app.backup.dir` |
| Hijri sync | yearly, Jan 1, 5am | refresh Hijri calendar data from AlAdhan |

## Email system

Dynamic, database-driven (not hardcoded SMTP): `mail_account` table holds
AES-256-GCM encrypted credentials (key via `APP_MAIL_ENC_KEY`),
`email_template` table holds HTML bodies editable via a TipTap rich-text
editor in the admin UI. `JavaMailSender` instances are built dynamically
and cached in memory per mail account. On first startup, `EmailTemplateSeeder`
bootstraps a mail account from the legacy `spring.mail.*` env vars.

## Rate limiting & security headers

Configured under `security.ratelimit` (per-IP, per-user, per-endpoint
limits, looser for data API endpoints) and `security.headers`
(HSTS max-age) in `application.yml` — not annotation-driven, so check
there before assuming an endpoint is unprotected.
