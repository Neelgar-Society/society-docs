# Frontend — `neelgar-society-react`

React 19 / TypeScript / Vite / Tailwind CSS. Feature-based folder structure,
not served by its own Node process in prod (see `Architecture.md` — static
build via Nginx).

## Folder structure

```
src/
├── assets/            # images, logo
├── components/         # shared, feature-agnostic components
│   ├── form/
│   ├── skeletons/      # loading skeletons
│   └── ExportButton/
├── config/             # env, endpoints config
├── constants/
├── context/             # React Context providers
│   ├── AuthContext.tsx
│   └── ThemeContext.tsx
├── features/            # one folder per domain feature (see below)
├── hooks/               # shared hooks
├── layouts/
│   ├── PrivateLayout.tsx   # authenticated shell — sidebar, TnC re-consent modal
│   └── PublicLayout.tsx    # public/unauthenticated shell
├── lib/                 # small standalone utilities (export client, filename utils)
├── pages/
│   ├── private/          # route-level pages behind auth, mirrors feature names
│   └── public/
├── routes/
│   └── AppRoutes.tsx     # single route table
├── services/
│   ├── apiClient.ts      # shared axios instance
│   ├── notifications.tsx
│   └── pendingCountsService.ts
├── types/
└── utils/
```

## Feature module pattern

Each feature under `features/<name>/` is self-contained:

```
features/events/
├── components/
├── services/
│   └── eventService.ts
├── event-types.ts
└── event.schema.ts        # form validation schema
```

New feature work should follow this pattern — don't add feature-specific
logic to shared `components/` or `services/`.

## API calls

All API calls go through the shared `services/apiClient.ts` axios instance
and `config/endpoints.ts` (centralized endpoint builder) — never hardcode
a URL string in a component or feature service. Real example
(`features/events/services/eventService.ts`):

```ts
import { api } from "@/services/apiClient";
import { ENDPOINTS } from "@/config/endpoints";

function unwrap<T>(res: any): T {
    return res.data?.data ?? res.data;
}

export async function listEvents(societyId: number): Promise<SocietyEvent[]> {
    const res = await api.get(ENDPOINTS.events.list(), { params: { societyId } });
    return unwrap<SocietyEvent[]>(res);
}
```

Note the `unwrap()` helper — backend responses are wrapped in
`SocApiResponse<T>` (see `Backend.md`), so every service function unwraps
`res.data?.data ?? res.data` rather than assuming a raw payload.

## Styling

Tailwind colors are never hardcoded as hex/rgb — they reference CSS
variables defined in `App.css` (`tailwind.config.cjs`):

```js
colors: {
  primary: "var(--color-primary)",
  secondary: "var(--color-secondary)",
  ...
}
```

To change a theme color, edit `App.css` only — Tailwind classes update
automatically. Don't add new hardcoded colors in components.

## Auth & layout

- `AuthContext.tsx` — holds auth/session state.
- `PrivateLayout.tsx` — wraps all authenticated routes; also owns the
  blocking Terms & Conditions re-consent modal for returning users on a
  ToC version bump (see `app.legal.tnc-version` in backend config).
- Role-based onboarding tours (`react-joyride`) are wired at the page
  level per role, not inside `PrivateLayout`.

## Path aliases

`@/` maps to `src/` (`vite.config.ts` + `tsconfig` paths) — always import
via `@/...`, not relative `../../../` chains.

## Linting & type-checking

- `npm run lint` — ESLint (flat config, `typescript-eslint` +
  `eslint-plugin-react-hooks` + `eslint-plugin-react-refresh`).
- `npm run build` runs `tsc -b` before `vite build` — type errors fail
  the build, not just lint.

## Known gap

No component or hook-level tests exist in this repo currently — testing
strategy for frontend is not yet defined. Flag if/when this should be
added to `CodingStandards.md`.