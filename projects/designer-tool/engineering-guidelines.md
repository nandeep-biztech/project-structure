# Engineering Guidelines — Designer Tool (React)

> **Purpose of this file (read after orienting).**
> This is the **rulebook** — _how you must write, structure, and verify code in the designer tool._ Every entry is a **must / must not / should**.
>
> It is **prescriptive**, not descriptive. For _what the system is and where things live_, see `[codebase-context.md](./codebase-context.md)`. The backend contract lives in `../nestjs-graphql/engineering-guidelines.md`.

---

## 0. Golden rules

1. **This app is a pure backend consumer.** Never call a commerce platform, AI vendor, or S3 directly — only the backend GraphQL API. Pricing, catalog truth, cart/checkout, and AI execution are server-side.
2. **Server state lives in Apollo; client/canvas state lives in Zustand.** Never copy server data into Zustand or hand-cache it.
3. **All GraphQL goes through generated typed documents** (client-preset `graphql()`), consumed via `useQuery(DOC)`/`useMutation(DOC)`. Never hand-write an untyped `gql` or use legacy per-operation hooks.
4. **Components are presentational; logic lives in hooks.** No data fetching, business rules, or Fabric.js calls inside a JSX component body.
5. **Feature-sliced + co-located tests.** Mirror an existing feature folder; every source file has a sibling test.
6. **A change is done only when typecheck + lint + tests pass and coverage holds** (§11).

---

## 1. Feature modules — checklist & naming

When adding `features/<feature>/`, mirror an existing feature:

- `components/` — presentational `.tsx` + co-located `.test.tsx`.
- `hooks/use<Feature>.ts` (+ `.test.ts`) — the feature's logic and store/Apollo wiring.
- `graphql/operations.ts` — typed `graphql("query …")` documents (client-preset; **not** `.graphql` SDL files).
- `utils/` (pure, + tests), `types.ts`, `constants.ts`.

**Naming (required):** **camelCase `useXxx.ts` hooks** (`useCanvasSync.ts`), **PascalCase components** (`DesignCanvas.tsx`), kebab-case for plain utils; tests are `*.test.ts(x)` beside source; GraphQL operations are named (`query Products`, `mutation AddToCart`). Decide CORE vs lazy: only `canvas/`, `multi-side/`, `history/` are CORE.

### Module boundaries & dependencies (keep features decoupled)

- **must not** import from another feature's internals (`features/a/`** → `features/b/`**). Share **down** to `shared/`/`lib/` or compose **up** at the `pages`/`app` layer — never feature-to-feature.
- **CORE exception:** the CORE features `canvas/`, `multi-side/`, `history/` are shared infrastructure other features may import. CORE itself **must not** depend on any non-CORE feature.
- **Allowed import direction:** `app` → `pages` → `features` → (`shared`, `lib`, `store`, `gql`). Never upward — a feature importing a page or `app/` is a bug.
- `**lib/` vs `shared/` (don't mix):** `lib/` = app-level integrations/singletons (Apollo client, auth, i18n, config, telemetry); `shared/` = reusable **presentational\*\* UI + pure hooks/utils. No app singletons in `shared/`, no UI widgets in `lib/`.
- **should** enforce the above mechanically with `eslint-plugin-boundaries` (or `import/no-restricted-paths`) so a violation fails lint, not review.

---

## 2. Components

- **must** keep components presentational — props in, JSX out. Side effects, fetching, and Fabric.js manipulation go in hooks.
- **must not** put business logic (pricing, validation that the server owns, cart math) in a component or hook — display server-computed values.
- **must** handle the three async states for any server data: **loading, error, empty** — never render assuming data exists.
- **should** code-split at the feature/route boundary (`React.lazy`); keep CORE eagerly loaded.
- **must** be accessible: semantic elements, keyboard operability for toolbar/canvas actions, focus management in modals, `aria-*` on custom controls.

---

## 3. State

- **must** keep **server/async data in Apollo Client** (its cache is the source of truth). **must not** duplicate it into Zustand.
- **must** keep **canvas/UI state in Zustand slices** with Immer; mutate via actions, never reach into the store object directly from components.
- **must** sync Fabric.js ↔ store only through `useCanvasSync` (one place), debounced; never scatter ad-hoc Fabric→store writes.
- **should** use React local state for ephemeral UI (open/close, hover) — don't promote it to Zustand.
- **must** persist only recoverable draft state to IndexedDB (autosave/crash recovery), never tokens or server data; **key drafts per user id** and **clear them on logout** so one customer's design can't leak into the next session on a shared device.

---

## 4. GraphQL & data access (Apollo Client + codegen **client-preset**)

- **must** generate types with `**@graphql-codegen/client-preset`** (typed `graphql()` documents + fragment masking) into `src/gql/` — **not\*\* the legacy `typescript-react-apollo` per-operation hooks. `src/gql/` is generated; never hand-edit it.
- **must** run codegen against the **committed `schema.gql`** (synced from the backend) or a dev/staging endpoint — **never prod** (introspection is off there). Codegen runs in **CI and fails on schema drift**; map custom scalars (`DateTime`, `JSON`) and use `enumsAsTypes`. (No `Upload` scalar — uploads use presigned S3 `PUT` and bypass GraphQL, see §6.)
- **must** point codegen's `documents` at **source files** — `['src/**/*.{ts,tsx}', '!src/gql/**']` — never at `*.graphql` files (client-preset scans `graphql()` calls in TS/TSX).
- **must** write operations as typed `graphql("query …")` documents in the feature's `graphql/operations.ts`, consumed via Apollo `useQuery(DOC)` / `useMutation(DOC)` — never hand-write an untyped `gql`.
- **must** **colocate fragments** on the component that needs them (`graphql("fragment …")`) and read via `useFragment` (fragment masking): each component declares exactly the fields it uses — no over-fetching, no cross-feature field coupling.
- **must** use the backend's **Relay connection** pagination (`edges`/`pageInfo`/`first`/`after`) and configure `relayStylePagination` in the cache `typePolicies` (`lib/apollo/cache.ts`) for correct merges; never request unbounded lists.
- **must** centralize client wiring in `lib/apollo/`: a link chain **errorLink → retryLink → authLink (Bearer) → langLink (Accept-Language) → httpLink**, with a `split` routing subscriptions to `**graphql-ws`\*\* (JWT in `connectionParams`).
- **must** read errors from **extensions.code** (`UNAUTHENTICATED`, `FORBIDDEN`, `NOT_FOUND`, `VALIDATION_FAILED`, `CONFLICT`, `RATE_LIMITED`, `INTERNAL`) in the central **errorLink** → refresh on `UNAUTHENTICATED`, map the rest to UX (field errors, toast) — never parse message strings.
- **should** rely on the normalized cache for updates; use optimistic UI only where the server result is predictable.

> **Reference:** `@graphql-codegen/client-preset` — [https://www.npmjs.com/package/@graphql-codegen/client-preset](https://www.npmjs.com/package/@graphql-codegen/client-preset) (config, `graphql()` usage, fragment masking). Use it as the source of truth for the codegen setup over any paraphrase here.

---

## 5. Auth

- **must** authenticate via the backend login mutation; hold the access token **in memory** (refresh-token flow), attach it as `Authorization: Bearer` via an Apollo auth link, and refresh on `UNAUTHENTICATED`.
- **must not** store the access token in `localStorage`/`sessionStorage` in plaintext or log it; never put it in a `VITE_` var or the URL.
- **must** establish `graphql-ws` subscriptions with the JWT in `connectionParams`, and tear them down on logout.
- **must** handle **token expiry on long-lived subscriptions** (e.g. an AI-job subscription): refresh the JWT and **reconnect `graphql-ws` with the new `connectionParams`** — never let a job socket die silently or reconnect with a stale token.
- **must** treat the client as **untrusted for authorization** — hide actions the user can't perform for UX, but rely on the server to enforce (a hidden button is not security).

---

## 6. AI / image operations (backend-driven, async)

- **must** run bg-removal, vectorization, and generation by calling the **backend mutation** → then **poll `jobStatus` or subscribe**; never call an AI vendor or upload to S3 from the client.
- **must** load resulting assets from the **signed URL** the backend returns; don't assume a public bucket or construct S3 URLs.
- **must** show progress/cancel UI for long jobs and handle failure/timeout with a retry path (jobs can fail).
- **must** validate uploads client-side (type + `VITE_MAX_UPLOAD_SIZE_MB`) before sending — but treat server validation as authoritative.
- **must** upload images via the **presigned-URL flow**, never through the GraphQL `Upload` scalar: (1) request a presigned upload URL from the backend (mutation), (2) `PUT` the file bytes **directly to S3** via that URL, (3) reference the returned object key in a follow-up mutation. The client never holds S3 credentials and never builds bucket URLs itself.

---

## 7. Canvas (Fabric.js)

- **must** confine Fabric.js access to `features/canvas/` hooks/utils; other features manipulate the canvas only through canvas hooks/actions.
- **must** dispose Fabric instances and event listeners on unmount (no leaks); guard re-init on resize/DPI change.
- **must** route every mutating canvas action through the **history manager** so undo/redo and autosave stay correct.
- **should** keep heavy work (serialization, hit-testing, render) in `utils/`, debounced/offloaded; don't block the main thread on every event.

---

## 8. i18n (two layers — UI strings _and_ server content)

**Layer 1 — the strings we author (i18next):**

- **must not** hardcode _any_ user-facing string — every label, button, tooltip, toast, empty state, and client-side validation hint goes through `t('key')`. A literal in JSX is a bug.
- **must** keep translations in `src/lib/i18n/locales/<lang>/<namespace>.json`, **namespaced per feature** (`common`, `canvas`, `cart`, …); read via `useTranslation('<namespace>')`.
- **must** add every new key to **all** supported locales (at minimum the default); a missing key falls back to the default locale — **never** render a raw key or English to a non-default locale.
- **must** use **interpolation / ICU plurals** (`t('cartItems', { count })`) — **never** string-concatenate translated fragments (word order and pluralization differ per language).
- **should** lazy-load locale bundles per language; **should** format dates/numbers/currency with `Intl` using the active locale; **should** support RTL (`dir`, logical CSS) if any supported locale is RTL.

**Layer 2 — server content & the shared locale:**

- **must** keep **one** active locale in `src/lib/i18n`; the language switcher updates it, persists it, and the **same value** configures i18next **and** is sent as the `Accept-Language` header — the two layers must never diverge.
- **must** allowlist the locale against `VITE_SUPPORTED_LOCALES`, falling back to `VITE_DEFAULT_LOCALE`; don't send an unsupported `Accept-Language`.

---

## 9. Performance

- **must** lazy-load non-CORE features and route components.
- **should** memoize expensive renders/derived values; virtualize long lists (templates, clipart) and large grids.
- **must** lazy-load fonts/clipart/template thumbnails; never bundle large asset catalogs.
- **should** keep canvas event handlers cheap and debounced; offload print-resolution work to the server.

---

## 10. Security (client-side)

The backend is the enforcement boundary; these rules close the _frontend_ surface. (This is a customer-facing app, but the baseline hardening below is non-negotiable.)

### XSS & DOM safety

- **must not** use `dangerouslySetInnerHTML`/`eval`/`new Function`. If rendering HTML is ever unavoidable, sanitize with a vetted library (DOMPurify) and document why.
- **must** validate any URL before using it in `href`/`src` (allow `https:`/relative only) — reject `javascript:`/`data:`; add `rel="noopener noreferrer"` to `target="_blank"` links; never redirect to a user/query-supplied URL without allowlisting.
- **must** be careful with user-supplied content placed on the canvas/SVG (uploaded SVGs, text) — sanitize SVG and never inject raw markup.

### Token, secret & sensitive-data handling

- **must** keep the access token **in memory only** (§5) — never `localStorage`/`sessionStorage`, never logged, never in a `VITE_` var or URL.
- **must not** put any secret in the client; every `VITE_` value ships to the browser.
- **must** only persist **recoverable draft/canvas state** to IndexedDB (autosave) — **never** tokens, credentials, or server data. Key it per user id, and on logout clear the Apollo cache + Zustand + `graphql-ws` **and the user's IndexedDB drafts** (shared-device privacy).
- **must** validate `import.meta.env` **once at startup** in `lib/config` with a Zod schema and **fail fast** if a required var (e.g. `VITE_GRAPHQL_HTTP_URL`) is missing/malformed; the app reads typed config from `lib/config`, never `import.meta.env` directly.

### Transport & headers (`nginx.conf`)

- **must** ship a strict **CSP**: `default-src 'self'`; `script-src 'self'` (**no `unsafe-inline`, no `unsafe-eval`**); `style-src 'self'` — use a **nonce/hash** for any library that injects inline styles (Tailwind compiles to a static stylesheet; Fabric may set inline element styles) rather than blanket `unsafe-inline`; `connect-src` allowlisting the GraphQL/WS origins; `img-src`/`media-src` allowlisting the signed-URL/asset origin. Plus `frame-ancestors 'none'` (clickjacking), `X-Content-Type-Options: nosniff`, a sane `Referrer-Policy`, and **HSTS**; serve only over **HTTPS** (no mixed content).
- **must** load assets only via the backend's **signed URLs / allowed origins**.

### Telemetry & PII

- **must not** log tokens or PII; **scrub Sentry breadcrumbs/context** of sensitive fields (auth headers, customer data, upload contents) before sending.

### Supply chain

- **must** `npm ci` against a committed lockfile and run `**npm audit`\*\* in CI (fail on high/critical); keep deps current (Dependabot/Renovate).
- **must** minimize third-party scripts; any external script needs **SRI** + a CSP allowlist entry.

### Authorization & trust

- **must** rely on the **backend** for authorization and sanitization; UI gating is UX only.
- **must not** trust client-supplied locale/ids for anything security-relevant — the server re-checks.

---

## 11. Testing — definition of done

- **must** co-locate unit tests (`*.test.tsx`/`*.test.ts`) beside source; render via `__tests__/test-utils.tsx` (all providers); mock the API with **MSW**, not by stubbing fetch ad hoc.
- **must** test: component states (loading/error/empty), hook logic, store slices, and util pure functions. Canvas: init/dispose, sync, undo/redo.
- **must** cover cross-feature flows (design → AI → cart) with **Playwright** e2e.
- **must** verify **accessibility** (§2) in tests — `jest-axe`/`axe-core` asserting no critical violations on key screens, plus role/label queries and keyboard-operability checks for toolbar/canvas controls — so the §2 accessibility "must" is actually enforced.
- **"Done" is objective:** `tsc --noEmit` clean, ESLint/Prettier clean, unit + e2e green, coverage ≥ project threshold (default **80%**), and CI green — local green alone is not done.

---

## 12. Anti-patterns — do NOT do these

- ❌ Calling a commerce platform, AI vendor, or S3 directly from the client.
- ❌ Hand-writing untyped `gql`, editing `src/gql/` by hand, using the legacy `typescript-react-apollo` hooks, or pointing codegen at the prod endpoint.
- ❌ Copying server data into Zustand, or putting canvas/UI state in Apollo.
- ❌ Business logic (pricing, server-owned validation) in a component/hook.
- ❌ Storing the JWT in `localStorage`/logging it, or putting a secret in a `VITE_` var.
- ❌ Using `dangerouslySetInnerHTML`/`eval`, injecting unsanitized SVG/markup, or putting unvalidated input in `href`/`src`.
- ❌ Persisting tokens/credentials/server data to IndexedDB (only recoverable draft state belongs there); not clearing cache + session on logout.
- ❌ Shipping without a strict CSP + `frame-ancestors 'none'` + HSTS, serving over HTTP, or loading assets/scripts from non-allowlisted origins.
- ❌ Running an AI/image op synchronously or assuming a public asset URL instead of the backend's signed URL.
- ❌ Importing one feature's internals from another feature (share via `shared`/`lib` or compose at the page layer; only CORE may be imported).
- ❌ Fabric.js calls outside `features/canvas/`, or mutating the canvas without going through the history manager.
- ❌ Hardcoded user-facing strings instead of i18next keys.
- ❌ Rendering server data without loading/error/empty handling, or parsing error message text instead of `extensions.code`.
- ❌ Requesting unbounded lists instead of Relay pagination.
- ❌ Shipping a feature without co-located tests, or calling it done on red CI / below coverage.
