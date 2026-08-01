# Coding Standards

Branching, commits, and tags are covered in `Branching.md` /
`ReleaseProcess.md` — this doc covers code-level conventions only.

## Backend (Java / Spring Boot)

### Response envelope
Every controller returns `SocApiResponse<T>` — never a raw entity/DTO.

```java
public static <T> SocApiResponse<T> success(T data, String message) { ... }
public static <T> SocApiResponse<T> failure(String message, String error) { ... }
public static <T> SocApiResponse<T> failureWithCode(String message, String error, String errorCode) { ... }
```

Shape: `{ success, message, data, error, errorCode, timestamp }`, with
`@JsonInclude(NON_NULL)` — null fields are omitted from the JSON, not
sent as `null`.

### Controllers
- **Every** controller method must carry OpenAPI/Swagger annotations —
  `@Operation`, `@ApiResponses` at minimum. No exceptions, even for
  internal/admin-only endpoints. (See `Backend.md` for a full example.)
- Class-level `@Tag` (one per resource) and `@SecurityRequirement(name = "bearerAuth")`
  on any authenticated controller.
- Unauthenticated endpoints live in a separate `Public*Controller`, not
  mixed into an authenticated controller with conditional logic.
- Role-gated endpoints use `@PreAuthorize`, not manual role checks in the
  method body.

### Error handling
- Throw domain exceptions (e.g. `ResourceNotFoundException`,
  `BadRequestException`, `LastActiveMemberException`) — never construct
  an error `SocApiResponse` by hand inside a controller/service.
- All exception → HTTP status mapping happens centrally in
  `GlobalExceptionHandler` (`@RestControllerAdvice`). Adding a new
  exception type means adding a handler there, not scattering
  `try/catch` blocks across controllers.

### Persistence
- Schema changes only via Flyway migrations (`V<n>__<description>.sql`)
  — `ddl-auto: none`, no Hibernate auto-DDL, ever.
- Never edit a migration that has already shipped to `develop`/`main` —
  add a new one.
- Entity ↔ DTO conversion goes through a dedicated `mapper/` class
  (e.g. `FamilyMapper`, `RelationshipMapper`) — not inline `.builder()`
  calls scattered through services.

### Package structure
Package-by-layer (`controller/`, `service/`, `service/impl/`,
`repository/`, `entity/`, `dto/request|response/`, `mapper/`,
`exception/`) — see `Backend.md` for the full tree. Feature-specific
code that doesn't fit the layered model (e.g. `geoimport/`) gets its own
top-level package with its own `dto`/`exception` subpackages, rather
than being forced into the shared layers.

## Frontend (React / TypeScript)

### Feature modules
New domain features go under `src/features/<name>/` with their own
`components/`, `services/`, `<name>-types.ts`, `<name>.schema.ts` — not
added to shared `src/components/` or `src/services/`. See `Frontend.md`.

### API calls
- Always go through `services/apiClient.ts` + `config/endpoints.ts` —
  never a hardcoded URL string in a component.
- Every service function unwraps the backend's `SocApiResponse` envelope
  consistently:
  ```ts
  function unwrap<T>(res: any): T {
      return res.data?.data ?? res.data;
  }
  ```

### Styling
- No hardcoded hex/rgb colors in components — use the Tailwind color
  tokens (`primary`, `secondary`, `text.muted`, etc.) which map to CSS
  variables in `App.css`. Changing a color means editing `App.css`, not
  hunting through components.

### Imports
Use the `@/` path alias for anything under `src/` — no `../../../` chains.

### Type-checking & linting
- `npm run build` runs `tsc -b` before `vite build` — a type error fails
  the build, it's not just a lint warning.
- `npm run lint` (flat ESLint config: `typescript-eslint` +
  `react-hooks` + `react-refresh`) must pass before a PR is merged.

## Cross-cutting

- Every code change traces to a Jira ID in both the commit message and
  the PR (see `Branching.md`).
- Config/secrets are never hardcoded — always `${ENV_VAR}` on the
  backend, `import.meta.env.VITE_*` on the frontend.
- If a convention here is broken deliberately (rare exception), leave a
  comment explaining why — don't leave future readers guessing.
  