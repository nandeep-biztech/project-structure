# Engineering Guidelines — Admin Panel (React)

> **Purpose of this file (read after orienting).**
> This is the **rulebook** — *how you must write, structure, and verify code in the admin panel.* Every entry is a **must / must not / should**.
>
> It is **prescriptive**, not descriptive. For *what the system is and where things live*, see `[codebase-context.md](./codebase-context.md)`. The backend contract lives in `../nestjs-graphql/engineering-guidelines.md`.

---

## 0. Golden rules

1. **Pure backend consumer.** Never call a commerce platform, AI vendor, or S3 directly — only the backend GraphQL API. Catalog truth, sync execution, and authorization are server-side.
2. **Server state lives in Apollo.** Don't duplicate it into Zustand or hand-cache it; light Zustand is for UI chrome only.
3. **All GraphQL goes through generated typed hooks** — never hand-write untyped queries.
4. **RBAC in the UI is for UX, not security.** Hide/disable what a role can't do, but the **server enforces** authorization.
5. **Components presentational; logic in hooks.** Pages stay thin and compose features.
6. **Done = typecheck + lint + tests green and coverage holds** (§10).

---

## 1. Feature modules — checklist & naming

Adding `features/<feature>/`, mirror an existing one:

- `components/` — `<Feature>Table.tsx`, `<Feature>Form.tsx`, detail panes (+ co-located `.test.tsx`).
- `hooks/use<Feature>.ts` (+ `.test.ts`) — Apollo wiring + derived state.
- `graphql/operations.ts` — typed `graphql("query …")` documents (client-preset; **not** `.graphql` SDL files); `utils/` (pure, + tests); `types.ts`.

**Naming (required):** PascalCase components (`ProductTable.tsx`), **camelCase `use<Feature>.ts` hooks**, named GraphQL operations (`query Products`, `mutation TriggerSync`), tests `*.test.tsx` beside source. Routes are lazy + role-guarded.

### Module boundaries & dependencies (keep features decoupled)

- **must not** import from another feature's internals (`features/a/`** → `features/b/**`). Share **down** to `shared/`/`lib/` or compose **up** at the `pages`/`app` layer — never feature-to-feature. (No "CORE" features here — all features are peers.)
- **Allowed import direction:** `app` → `pages` → `features` → (`shared`, `lib`, `store`, `gql`). Never upward — a feature importing a page or `app/` is a bug.
- `**lib/` vs `shared/` (don't mix):** `lib/` = app-level integrations/singletons (Apollo client, auth, rbac, i18n, config, telemetry); `shared/` = reusable **presentational** UI kit (`Table`, `Form`, `Modal`, `DataState`) + pure hooks/utils. No app singletons in `shared/`, no UI widgets in `lib/`.
- **should** enforce the above mechanically with `eslint-plugin-boundaries` (or `import/no-restricted-paths`) so a violation fails lint, not review.

---

## 2. Components & forms

- **must** keep components presentational; fetching/derivation lives in hooks.
- **must** handle **loading / error / empty** for every server-backed view (use the shared `DataState` wrapper) — never assume data exists.
- **must** build tables on **TanStack Table** with server-side **Relay pagination** (`first`/`after`) and put sort/filter/page state in the **URL** (shareable, back-button safe) — never fetch unbounded lists.
- **must** build forms with **React Hook Form + Zod**; client validation mirrors the backend DTO rules **for UX**, but the **server is authoritative** (surface its `VALIDATION_FAILED` field errors).
- **must** be accessible: labelled inputs, keyboard-navigable tables/menus, focus-trapped modals, `aria-*` on custom controls.

---

## 3. State

- **must** keep **server data in Apollo** (source of truth); **must not** mirror it into Zustand.
- **should** keep table/filter state in URL params, not global state.
- **must** limit Zustand to app UI chrome (sidebar, theme, layout); keep form state in RHF and ephemeral state in `useState`.

---

## 4. GraphQL & data access (Apollo Client + codegen **client-preset**)

- **must** generate types with **`@graphql-codegen/client-preset`** (typed `graphql()` documents + fragment masking) into `src/gql/` — **not** the legacy `typescript-react-apollo` per-operation hooks. `src/gql/` is generated; never hand-edit it.
- **must** run codegen against the **committed `schema.gql`** (synced from the backend) or a dev/staging endpoint — **never prod** (introspection is off there). Codegen runs in **CI and fails on schema drift**; map custom scalars (`DateTime`, `JSON`) and use `enumsAsTypes`. (No `Upload` mapping — the admin panel doesn't upload files; assets are read via signed URLs. Add `Upload` only if/when an admin upload flow exists.)
- **must** write operations as typed `graphql("query …")` documents in the feature's `graphql/operations.ts`, consumed via Apollo `useQuery(DOC)` / `useMutation(DOC)` — never hand-write an untyped `gql`.
- **must** **colocate fragments** on the component that needs them (`graphql("fragment …")`) and read via `useFragment` (fragment masking) — no over-fetching, no cross-feature field coupling.
- **must** use **Relay connection** pagination for all lists and configure `relayStylePagination` in the cache `typePolicies` (`lib/apollo/cache.ts`); never fetch unbounded lists.
- **must** centralize client wiring in `lib/apollo/`: a link chain **errorLink → retryLink → authLink (Bearer) → langLink (Accept-Language) → httpLink**, with a `split` routing subscriptions (sync progress) to **`graphql-ws`** (JWT in `connectionParams`).
- **must** read errors from **extensions.code** (`UNAUTHENTICATED`, `FORBIDDEN`, `NOT_FOUND`, `VALIDATION_FAILED`, `CONFLICT`, `RATE_LIMITED`, `INTERNAL`) in the central **errorLink** → refresh on `UNAUTHENTICATED`, map the rest to UX — never parse message strings.
- **should** update the Apollo cache on mutations (or refetch the affected query) so tables stay consistent without a full reload.

> **Reference:** `@graphql-codegen/client-preset` — https://www.npmjs.com/package/@graphql-codegen/client-preset (config, `graphql()` usage, fragment masking). Use it as the source of truth for the codegen setup over any paraphrase here.

---

## 5. Auth & RBAC

- **must** authenticate via the login mutation; access token **in memory** (refresh flow), attached as `Authorization: Bearer`; refresh on `UNAUTHENTICATED`.
- **must not** store the token in `localStorage`/`sessionStorage` plaintext, log it, or put it in a `VITE_` var/URL.
- **must** drive route guards and action visibility from the **current user's roles**, but **must not** treat UI gating as enforcement — every action is re-authorized server-side (a hidden button is not security).
- **must** route `graphql-ws` (sync progress) with the JWT in `connectionParams`; tear down on logout.
- **must** handle **token expiry on long-lived subscriptions**: when the JWT expires mid-subscription (e.g. a long sync run), refresh it and **reconnect `graphql-ws` with the new `connectionParams`** — never let a sync-progress socket die silently or reconnect with a stale token.

---

## 6. Sync & long-running jobs

- **must** trigger sync via the backend **mutation** (which enqueues a BullMQ job) and then **subscribe or poll** `sync-run` for progress/errors — never block the UI waiting, and never run sync logic client-side.
- **must** show run status, counts, and surfaced errors; allow retry; disable a duplicate "Sync Now" while a run is active.
- **must not** fetch catalog data live from a platform — the catalog is read from the backend's synced copy.

---

## 7. Settings & secrets

- **must** treat platform **credentials** as **write-only** in the UI — submit to the backend, never request/display them back; show only a "configured" indicator.
- **must not** place any secret in the client or a `VITE_` var (all `VITE_` ships to the browser).
- **must** edit the store-view↔locale map and platform base URL through the backend's per-tenant config endpoints; the backend validates them (SSRF/allowlist) — the UI just collects input.
- **must** validate `import.meta.env` **once at startup** in `lib/config` with a Zod schema and **fail fast** if a required var (e.g. `VITE_GRAPHQL_HTTP_URL`) is missing/malformed; the rest of the app reads typed config from `lib/config`, never `import.meta.env` directly.
- **should** read feature flags only through that typed `lib/config` (never raw `import.meta.env` in components), so gating is centralized and testable.

---

## 8. i18n (two layers — UI strings *and* server content)

**Layer 1 — the strings we author (i18next):**

- **must not** hardcode *any* user-facing string — every label, button, table header, tooltip, toast, empty state, and client-side validation hint goes through `t('key')`. A literal in JSX is a bug.
- **must** keep translations in `src/lib/i18n/locales/<lang>/<namespace>.json`, **namespaced per feature** (`common`, `catalog`, `sync`, `users`, `settings`, …); read via `useTranslation('<namespace>')`.
- **must** add every new key to **all** supported locales (at minimum the default); a missing key falls back to the default locale — **never** render a raw key.
- **must** use **interpolation / ICU plurals** (`t('rowsSelected', { count })`) — **never** concatenate translated fragments.
- **should** lazy-load locale bundles; **should** format dates/numbers/currency with `Intl` using the active locale; **should** support RTL if any supported locale is RTL.

**Layer 2 — server content & the shared locale:**

- **must** keep **one** active locale in `src/lib/i18n`; the language switcher updates+persists it, and the **same value** configures i18next **and** is sent as `Accept-Language` — the two layers must never diverge.
- **must** allowlist the locale against `VITE_SUPPORTED_LOCALES`, falling back to `VITE_DEFAULT_LOCALE`.

---

## 9. Performance

- **must** lazy-load routes/feature modules; virtualize large tables.
- **should** memoize expensive derived rows/columns; debounce filter inputs; keep table sort/filter server-side via Relay pagination.

---

## 10. Security (client-side)

This is a **sensitive back-office** (staff, RBAC, platform-credential management), so client-side security is first-class. The backend remains the enforcement boundary; these rules close the *frontend* surface.

### XSS & DOM safety
- **must not** use `dangerouslySetInnerHTML`. If rendering HTML is ever unavoidable, sanitize with a vetted library (DOMPurify) and document why — never inject server/user strings into the DOM raw.
- **must** validate any URL before using it in `href`/`src` (allow `https:` / relative only) — reject `javascript:`/`data:` schemes; never build links from unsanitized input.
- **must not** use `eval`, `new Function`, or render untrusted strings as markup/templates.
- **should** add `rel="noopener noreferrer"` to every `target="_blank"` link, and never perform a redirect to a URL taken from query/user input without allowlisting (open-redirect).

### Token, secret & sensitive-data handling
- **must** keep the access token **in memory only** (§5) — never `localStorage`/`sessionStorage`, never logged, never in a `VITE_` var or URL.
- **must not** put any secret in the client; all `VITE_` values ship to the browser (§7).
- **must not** persist server data, tokens, or credentials to `localStorage`/`IndexedDB` — admin data is sensitive; keep it in the Apollo cache (memory).
- **must** fully clear session state on logout / `UNAUTHENTICATED`: reset the Apollo cache, Zustand UI state, and tear down `graphql-ws` connections.
- **must** treat platform credentials as **write-only** (§7) — never request or display them back.

### Transport & headers (`nginx.conf`)
- **must** ship a strict **Content-Security-Policy** (`default-src 'self'`; no `unsafe-inline`/`unsafe-eval` for scripts; explicit allowlist for the GraphQL/WS origins and any asset/CDN origin).
- **must** set `frame-ancestors 'none'` (anti-clickjacking), `X-Content-Type-Options: nosniff`, a sane `Referrer-Policy`, and **HSTS**; serve only over **HTTPS** (no mixed content).
- **must** load assets only via backend **signed URLs / allowed origins**.

### Supply chain
- **must** `npm ci` against a committed lockfile and run **`npm audit`** in CI (fail on high/critical); keep deps current (Dependabot/Renovate).
- **must** minimize third-party scripts; any external script needs **SRI** + a CSP allowlist entry — no arbitrary analytics/tag-manager injection.

### Session & telemetry
- **should** auto-logout on inactivity and on token expiry (sensitive back-office); handle `UNAUTHENTICATED` by clearing session and routing to login.
- **must not** log tokens or PII; scrub Sentry breadcrumbs/context of sensitive fields before sending.

### Authorization (reminder)
- **must** rely on the **server** to enforce every action; UI role-gating (§5) is UX only — a hidden/disabled control is not a security control.

---

## 11. Testing — definition of done

- **must** co-locate unit tests beside source; render via `__tests__/test-utils.tsx`; mock the API with **MSW**.
- **must** test: loading/error/empty states, table sort/filter/pagination, form validation + submit, RBAC gating (allowed **and** denied), and error-code → UX mapping.
- **must** verify **accessibility** (§2) in tests — `jest-axe`/`axe-core` asserting no critical violations on key screens, plus role/label-based queries in component tests — so the accessibility "must" is actually enforced, not aspirational.
- **must** cover key flows with **Playwright** e2e: login + RBAC, catalog browse/filter, sync trigger→complete, settings save.
- **"Done" is objective:** `tsc --noEmit` clean, ESLint/Prettier clean, unit + e2e green, coverage ≥ project threshold (default **80%**), CI green — local green alone is not done.

---

## 12. Anti-patterns — do NOT do these

- ❌ Calling a commerce platform, AI vendor, or S3 directly from the client.
- ❌ Hand-writing untyped `gql`, editing `src/gql/` by hand, using the legacy `typescript-react-apollo` hooks, or pointing codegen at the prod endpoint.
- ❌ Duplicating server data into Zustand, or putting server state anywhere but Apollo.
- ❌ Importing one feature's internals from another feature (share via `shared`/`lib` or compose at the page layer).
- ❌ Treating UI role-gating as security instead of relying on server enforcement.
- ❌ Storing/logging the JWT, or putting a secret/credential in a `VITE_` var.
- ❌ Reading back or displaying platform credentials (they're write-only).
- ❌ Using `dangerouslySetInnerHTML`/`eval`, or putting unvalidated input into `href`/`src` (XSS, `javascript:` URLs).
- ❌ Persisting server data, tokens, or credentials to `localStorage`/`IndexedDB`, or not clearing the Apollo cache + session on logout.
- ❌ Shipping without a strict CSP + `frame-ancestors 'none'` + HSTS, serving over HTTP, or loading assets/scripts from non-allowlisted origins.
- ❌ Fetching catalog data live from a platform, or running sync logic client-side.
- ❌ Fetching unbounded lists instead of Relay pagination; ignoring loading/error/empty states.
- ❌ Parsing error message text instead of `extensions.code`.
- ❌ Hardcoded user-facing strings instead of i18next keys.
- ❌ Shipping without co-located tests, or calling it done on red CI / below coverage.

