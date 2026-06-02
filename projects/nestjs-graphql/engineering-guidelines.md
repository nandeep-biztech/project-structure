# Engineering Guidelines — NestJS Code-First GraphQL

> **Purpose of this file (read after orienting).**
> This is the **rulebook**, not the map. It answers *"how am I required to write, structure, and verify code here?"* — the conventions, the do's and don'ts, and the definition of done that every change (human or automated) must satisfy.
>
> It is **prescriptive**, not descriptive. For *what the system is and where things live*, see `[codebase-context.md](./codebase-context.md)`. Don't restate the architecture here; reference it and state the rule.
>
> Rule of thumb: every entry here should be phrased as something you **must / must not / should** do. If it's just a fact about the system, it belongs in the context file.

---

## 0. Golden rules (if you read nothing else)

1. **The TypeScript decorator is the schema.** Never hand-edit `schema.gql` — it is regenerated on startup. Change the `@ObjectType()`/`@Field()`, not the SDL.
2. **Respect the layers.** Resolver → Service → Repository, crossing the GQL/DB boundary only through a Mapper. A resolver must never touch a repository; a service must never return a TypeORM entity to GraphQL.
3. **Never expose secrets through `@Field()`.** `passwordHash` and similar live only on entities, are marked `select: false` and `@HideField()`, and are omitted by the mapper.
4. **Every relation field goes through a DataLoader**, never a direct service/repo call inside `@ResolveField`.
5. **Match the existing module shape exactly.** New domains mirror `users/` folder-for-folder. Consistency beats cleverness.
6. **A change is not done until its tests are written and green.** See §10.

---

## 1. Adding a new feature module — required checklist

When creating `src/modules/<name>s/`, produce **all** of these, mirroring `users/`. (`<name>` = the NestJS CLI `<name>` argument, singular e.g. `user`; `<name>s` = plural; `<Name>` = PascalCase class. Scaffold with `nest g resource <name>`. See [`codebase-context.md`](./codebase-context.md) §4 for the full tree and placeholder legend.)

- `models/<name>.type.ts` — `@ObjectType()` with described `@Field()`s; a `<name>-connection.type.ts` if it's listable; enums via `registerEnumType()`.
- `dto/create-<name>.input.ts` + `update-<name>.input.ts` (`extends PartialType(...)`) + `<name>s-filter.args.ts` (`@ArgsType()`).
- `entities/<name>.entity.ts` (`@Entity()` extending `BaseEntity`) + `<name>.repository.ts` (custom queries).
- `loaders/<name>.loader.ts` (`Scope.REQUEST`) if anything resolves this resource as a relation.
- `mappers/<name>.mapper.ts` — pure `toGql()` / `toConnection()`.
- `policies/manage-<name>.policy.ts` — CASL rules.
- `<name>.service.ts`, `<name>.resolver.ts`, `<name>.module.ts`, `index.ts`.
- `__tests__/` with a spec per class (§11).
- Register the module in `app.module.ts`.

Do not invent a new folder layout or collapse these into fewer files "because the module is small." The uniformity is load-bearing for automation.

---

## 2. GraphQL types & DTOs

- **Models (`models/`) carry GQL decorators only**, never `@Entity()`/`@Column()`. Entities (`entities/`) carry DB decorators only, never `@Field()`.
- Add a `description` to every `@ObjectType()` and `@Field()` — descriptions surface in the schema and the playground and are part of the API contract.
- Use `@nestjs/graphql` composition helpers (`PartialType`, `PickType`, `OmitType`, `IntersectionType`) instead of redeclaring fields. `UpdateXInput extends PartialType(CreateXInput)`.
- Nullability is explicit: mark optional fields `{ nullable: true }`; for list fields decide between `nullable: 'items'`, `'itemsAndList'` deliberately.
- Validate **all** input at the DTO with `class-validator` decorators (`@IsEmail`, `@MinLength`, `@Max`, `@IsOptional`, `@IsUrl`, …). The global `ValidationPipe` runs with `whitelist: true` + `forbidNonWhitelisted: true`, so undeclared fields are rejected — never rely on manual checks in the resolver for shape validation.
- class-validator covers **all shape/format validation** — including regex (`@Matches`), nested (`@ValidateNested` + `@Type`), conditional (`@ValidateIf`), and complex/cross-field rules via a custom `@ValidatorConstraint`/decorator. **must not** put **business rules** (uniqueness, "referenced id exists", stock, permissions, anything needing the DB or other records) in a validator — those belong in the **service** (§4). Rule: *format → DTO validator; rules needing state/auth → service.*
- Pagination args extend the shared pattern: `first` (with `@Min`/`@Max`), `after` (Relay cursor), `orderBy`, `direction`.
- Reuse the shared custom scalars in `common/scalars/` (`Date`, `JSON`, `Upload`) — don't reinvent a date/JSON scalar per module. File uploads always go through the `Upload` scalar (and the §12 upload validation rules).

---

## 3. Resolvers — keep them thin

- A resolver method **delegates immediately** to a service and returns. No business logic, no DB access, no authorization branching beyond declarative guards/decorators.
- Authentication via `@UseGuards(GqlAuthGuard)` at the class level; mark genuinely public operations with `@Public()`.
- Authorization that is role-shaped uses `@Roles('admin', …)`; resource-shaped (own-record, ownership) authorization is enforced in the **service** via CASL (§5), not the resolver.
- Get the caller with `@CurrentUser()`; pass it into the service so the service can run the ability check.
- Annotate cost with `@Complexity(n)` and apply `@Throttle(...)` / `@UseInterceptors(CacheInterceptor)` where appropriate.
- Relation fields use `@ResolveField` → loader. Example: `getPosts(@Parent() user) { return this.postLoader.load(user.id); }`.

---

## 4. Services — the home of business logic

- Services depend on repositories, mappers, the `CaslAbilityFactory`, and event emitters — injected via the constructor. **Never inject the GraphQL request or use GQL types as logic primitives.**
- A service method that returns data to a resolver returns a **GQL model** (run the mapper), never a raw `Entity`.
- Mutations that change state should emit domain events (`this.events.emit('user.created', …)`) for decoupled side effects; don't inline cross-domain side effects.
- Hash secrets in the service (`bcrypt`, cost ≥ 12), never in the resolver or mapper.
- Throw Nest's semantic exceptions — `NotFoundException`, `ForbiddenException`, `UnauthorizedException` — not bare `Error`. The GQL exception filter maps these to proper GraphQL errors.

---

## 5. Authorization (CASL) — non-negotiable

- Every mutation and every sensitive read performs an ability check **in the service** before acting:
  ```typescript
  const ability = this.casl.createForUser(actor);
  if (ability.cannot(Action.Update, entity)) throw new ForbiddenException(...);
  ```
- Resource rules live in `policies/manage-<x>.policy.ts`. Encode "admin can manage everything; a user can read all but only update their own and never self-delete" style rules there — not scattered through services.
- Never trust a client-supplied id as proof of ownership; derive the actor from the JWT (`@CurrentUser()`) and check against the loaded entity.

---

## 6. Persistence (TypeORM)

- All entities extend `BaseEntity` (uuid `id`, `createdAt`, `updatedAt`, `deletedAt`).
- **Soft delete only** — use `repository.softDelete(...)`; never `delete()`/`remove()`. Custom queries must filter `WHERE deletedAt IS NULL` (or use TypeORM's soft-delete-aware methods).
- Put non-trivial queries in the custom `*.repository.ts` (e.g. `findPaginated`), not inline in the service. Build them with the query builder and **parameterized** values (`:search`) — never string-concatenate user input.
- Mark sensitive columns `{ select: false }` (e.g. `password_hash`) so they're excluded from default selects.
- Use snake_case column names via `{ name: 'avatar_url' }`; index columns you filter/lookup on (`@Index()`).
- Schema changes ship as timestamped migrations in `libs/database/migrations/`. Never rely on `synchronize: true` outside tests.
- **must** wrap any operation that performs **more than one write** (multiple `save()`s, a write plus a related-record write, the sync upserts) in a single transaction via `libs/database/transaction.service` — never leave multi-step writes non-atomic, or a mid-operation failure leaves partial data.

---

## 7. DataLoaders & N+1

- One loader per relation, `@Injectable({ scope: Scope.REQUEST })` — this is mandatory, not optional. A singleton loader leaks cache across requests.
- The batch function must return results **in the same order as the input ids**, returning an `Error` (not throwing) for missing ids.
- Resolve loaders in tests with `module.resolve(...)` (not `module.get(...)`) because of REQUEST scope.

---

## 8. Cross-cutting concerns

- Shared decorators/guards/interceptors/pipes/scalars live in `src/common/`; shared infra in `libs/`. Don't duplicate a guard or scalar inside a feature module.
- Add new Apollo plugins to the pipeline in `app.module.ts` in a deliberate order (complexity/depth limits run **before** expensive work).
- All logging goes through the Pino logger service (structured JSON) — no `console.log`.
- Respect the configured `GQL_DEPTH_LIMIT` and `GQL_MAX_COMPLEXITY`; don't disable them to make a heavy query pass — fix the query or paginate.

---

## 9. Runtime & toolchain

- Target the **current Node.js active LTS: `24.16.0` ("Krypton")**. Pin it explicitly so local, CI, and Docker all agree:
  - `.nvmrc` → `24.16.0`
  - `package.json` → `"engines": { "node": ">=24.16.0 <25" }`
  - `docker/Dockerfile` base image → `node:24.16.0-alpine` (or the slim variant).
- Stay on the LTS line — bump the patch when a new `24.x` LTS ships; do **not** jump to a non-LTS major (odd majors / `Current`) for production.
- Don't use APIs newer than the pinned LTS guarantees, and don't downgrade below the `engines` floor to make a dependency install.

## 10. Configuration & secrets

- Read config only through `ConfigService` + a typed `registerAs` namespace in `src/config/`. Don't sprinkle `process.env.X` through business code.
- Never commit `.env.prod` or hardcode secrets. New config keys get added to the relevant `*.config.ts`, documented, and given safe local defaults in `.env`.

---

## 11. Testing — definition of done

A change is complete only when the appropriate tests exist and pass.

- **Unit tests** (co-located in `__tests__/`): mock everything at the boundary with `@golevelup/ts-jest`'s `createMock<T>()`. Build test data with the Faker factories in `test/factories/` (`buildUserEntity(overrides)`), never ad-hoc literals scattered across specs.
  - Resolver spec: assert it delegates to the service with the right args.
  - Service spec: assert business logic, authorization (allowed **and** forbidden paths), and event emission.
  - Repository spec: run against in-memory SQLite; assert ordering, filtering, and that soft-deleted rows are excluded.
  - Loader spec: assert N concurrent `load()` calls produce **one** DB call (batching).
  - Mapper spec: assert field mapping **and** that `passwordHash` (and other secrets) are not exposed.
- **e2e tests** (`test/e2e/`): mock nothing except true external services. Drive through the real schema with the `gqlRequest`/`getAuthToken` helpers. Cover the happy path, auth failure (401/unauthorized), authorization failure (forbidden), and validation/duplicate errors.
- Every authorization rule must have both an **allowed** and a **denied** test. Every "secret not exposed" guarantee must have an explicit test.

---

## 12. Security — required checks & gates

The defenses already in the architecture (JWT auth, CASL, `ValidationPipe`, depth/complexity limits, secret hiding, parameterized queries) are mapped in [`codebase-context.md`](./codebase-context.md) §10. This section is what you **must do** to keep them intact and to close known gaps. Treat the **must** items as merge-blocking gates.

### Per-change (every PR)
- **must** keep the auth → authz → validate chain intact: a new query/mutation is behind `GqlAuthGuard` unless explicitly `@Public()`, role-gated with `@Roles()` where relevant, and resource-authorized via CASL in the service. A deny-path test is required (see §11).
- **must** put `@Complexity(n)` on every new resolver field and keep total query cost within `GQL_MAX_COMPLEXITY`; never raise the depth/complexity limits to force a heavy query through.
- **must not** add a `@Field()` that exposes a secret/PII column; new sensitive columns are `{ select: false }` + `@HideField()`, and the mapper omits them. Add a "not exposed" test.
- **must** build all DB access with parameterized query-builder values — no string interpolation of user input.
- **must not** introduce a new secret as a literal; route it through `ConfigService`/env and document it. Run secret scanning (below) before pushing.

### Transport & error exposure
- **must** keep Helmet enabled and restrict CORS to an explicit `ALLOWED_ORIGINS` allowlist — never `origin: '*'` (especially with `credentials: true`). Don't widen CORS to make a client work; add its origin to the allowlist.
- **must** serve all non-local traffic over TLS (terminated at the proxy, `docker/nginx.conf`); never expose the app port directly in production.
- **must not** leak stack traces or internal messages to clients: throw Nest's semantic exceptions, let `gql-exception.filter.ts` map them to safe client-facing errors, and keep full detail server-side only (logs + `SentryPlugin`).

### Abuse & flooding (defense in depth)
- **must** keep `@Throttle` / `ThrottlerModule` as the application-layer rate limit — but treat it as the *inner* layer only. **Volumetric/network DDoS must be absorbed at the edge** (WAF/CDN + nginx connection & request-rate limits in `docker/nginx.conf`); don't rely on app throttling alone.
- **must** rate-limit and **lock out** authentication attempts per account **and** per IP with exponential backoff — global throttling is not enough to stop credential stuffing / brute-force on `login`.
- **must** validate every file upload behind `upload.scalar.ts`: enforce an allowlisted MIME type and a max size, store outside the web root, and **should** virus-scan untrusted uploads before processing.
- **must not** switch from Bearer-token auth to cookie/session auth without adding CSRF protection (double-submit token or SameSite) — the current CSRF safety depends on there being no ambient credential (context §10).

### CI gates (add to `.github/workflows/`)
- **must** `npm ci` against a committed lockfile (never `npm install` in CI) and run **`npm audit --audit-level=high`** — high/critical advisories fail the build.
- **must** run **secret scanning** (e.g. `gitleaks`) on every push; a detected secret fails the build and the secret is rotated, not just deleted.
- **should** run **SAST** (CodeQL or Semgrep with a NestJS/GraphQL ruleset) on PRs.
- **should** keep dependencies current via Dependabot/Renovate and review the diffs (supply-chain).

### GraphQL hardening (beyond depth + complexity)
- **should** enforce **alias-count, directive-count, and token-count** limits (e.g. GraphQL Armor) — depth/complexity alone don't stop alias/batch amplification.
- **should** cap **request batching** (max array-batched operations per HTTP request).
- **should**, in production, prefer a **persisted-query allowlist** over open APQ, and keep `introspection`/`playground` disabled (already wired to `NODE_ENV`).

### Auth & token hardening
- **should** issue short-lived access tokens with refresh-token rotation, validate JWT `aud`/`iss`, and support revocation/denylist on logout.
- **must** hash passwords with `bcrypt` (cost ≥ 12) in the service layer only.

### Container & runtime
- **must** run the container as a **non-root user**, install with `npm ci --omit=dev`, and pin the base image to the LTS digest (`node:24.16.0-...`, see §9).
- **should** scan the built image (Trivy/Grype) in CI and fail on high/critical OS-package CVEs.

### Auditability
- **should** emit an audit log entry for privileged mutations (actor, action, target id) distinct from operational logs.

---

## 13. External integrations

The descriptive map — the `integrations/` anti-corruption layer, its two categories (commerce + general capabilities), the two mapper boundaries, and the flows — is in [`codebase-context.md`](./codebase-context.md) §13. These are the rules that keep it clean.

### Commerce platforms (`integrations/commerce/`, per-tenant)
- **must** route every external-platform call through a `CommercePlatformProvider`. Business code (services, resolvers) **must not** import a platform SDK, hit a Magento/Shopify URL, or branch on `platform === '…'` — only the adapter under `integrations/commerce/<platform>/` may.
- **must** return **canonical DTOs** from providers, never raw platform payloads. The platform → canonical conversion happens in the adapter's mapper; nothing platform-shaped crosses into `modules/`.
- **must** keep the read path pure Postgres: catalog queries serving the admin/designer **must not** call the platform live (per the Postgres-after-sync decision). Freshness comes from sync, not request-time fetches.
- **must** run sync as a **BullMQ job**, never inside a GraphQL request. `syncNow` enqueues and returns a `syncRunId`; progress is tracked in the `sync-run` entity.
- **must** make sync **idempotent** — upsert keyed on `(tenantId, platform, externalId)`; reconcile platform-removed items as soft-deletes (never duplicate on re-sync).
- **must** scope every catalog/sync read and write by `tenantId` (from the JWT), enforced in the repository layer, not per query. Cross-tenant data access is a security bug, not a feature gap.
- **must** store platform credentials encrypted in `tenant-platform-config` and never expose them via a `@Field()` (see §12 and context §10).
- **must** wrap platform calls in `<platform>.client.ts` with timeout + retry/backoff + circuit-breaker so a platform outage can't cascade into our API; cache read-through in Redis where it helps.
- **must** guard against **SSRF** on outbound calls: a tenant-configured `baseUrl` must match an allowlisted scheme/host pattern (HTTPS only), and requests to loopback, link-local, and private/internal IP ranges must be rejected — including across redirects. Never let tenant config point the server at an arbitrary address.
- **must not** persist platform-owned cart contents — `addToCart` proxies to the platform (which owns cart & checkout) and returns a canonical `CartDTO`; keep at most a thin session ↔ platform-cart reference.
- **should** add a new platform by creating `integrations/commerce/<platform>/` that implements the interface plus a registry case — with **zero** change to `modules/`.

### General capability providers (bg-removal, vectorization, `ai/`, global config)
- **must** put every external image/AI vendor behind a **capability interface** under `integrations/<capability>/` (e.g. `background-removal/`, `vectorization/`) — `ai/` is reserved for **inherently-generative** capabilities only. Business code calls the interface, never the vendor SDK/URL directly.
- **must** return **canonical DTOs** from these providers too; a vendor swap must not ripple into `modules/`.
- **must** run any long-running/costly op (bg-removal, vectorization, image generation) as a **BullMQ job**, never inside a GraphQL request. The mutation persists the asset as `PENDING`, enqueues, and returns an id; the worker calls the provider and flips it to `READY`.
- **must** store outputs in S3 via `libs/storage/` and serve them through **short-lived signed URLs** — never a public bucket; asset access is `tenantId`-scoped.
- **must** keep these vendor keys (`REMOVE_BG_API_KEY`, `STABILITY_API_KEY`, …) in `src/config/` read via `ConfigService` — **global** product keys, not the per-tenant `tenant-platform-config`. (Only move a key to tenant config if a client supplies their own.)
- **must** validate source uploads (MIME/size, per §12) before sending them to a vendor, and **must** quota/rate-limit AI calls **per tenant** — generation costs real money, so treat unbounded calls as an abuse vector.
- **must** wrap each provider's client with timeout + retry/backoff + circuit-breaker (and SSRF-safe URL handling if a base URL is ever configurable), same as commerce clients.
- **should** give every *capability* its own folder (`background-removal/`, `vectorization/`, `ai/image-generation/`), but keep the *vendor* as a filename prefix (`removebg.*`, `stability.*`) — add a vendor sub-folder only once a 2nd vendor exists. Group a capability under `ai/` only when it's genuinely generative; otherwise put it flat at `integrations/<capability>/`.

---

## 14. Anti-patterns — do NOT do these

- ❌ Editing `schema.gql` by hand.
- ❌ Returning a TypeORM entity from a service to a resolver (skipping the mapper).
- ❌ Putting `@Field()` on an entity or `@Column()` on a model.
- ❌ Business logic or DB queries inside a resolver method.
- ❌ Calling a service/repository directly from `@ResolveField` instead of a loader.
- ❌ A singleton (non-`Scope.REQUEST`) DataLoader.
- ❌ Hard deletes, or custom queries that forget the soft-delete filter.
- ❌ String-concatenated SQL / unparameterized user input.
- ❌ Trusting client-supplied ids for ownership instead of checking CASL against the loaded entity.
- ❌ `console.log`, bare `throw new Error(...)`, or `process.env` reads in business code.
- ❌ Shipping a feature without its co-located unit tests and the deny-path authorization tests.
- ❌ Calling a platform SDK/URL, or branching on the platform name, anywhere outside `integrations/commerce/<platform>/`.
- ❌ Reading product/category/attribute data live from the platform in a request that serves the admin/designer (bypassing the synced Postgres copy).
- ❌ Running a full catalog sync synchronously inside a GraphQL resolver instead of a BullMQ job.
- ❌ Querying catalog/sync data without a `tenantId` scope, or letting one tenant read another's data.
- ❌ Returning raw platform payloads (or persisting platform cart contents) instead of canonical DTOs.
- ❌ Making an outbound request to a tenant-supplied `baseUrl` without allowlist + private-IP-range validation (SSRF).
- ❌ Relying on app-level `@Throttle` alone for DDoS, or on global throttling alone to stop login brute-force.
- ❌ Calling an image/AI vendor SDK or URL directly from a service instead of through a capability interface in `integrations/`.
- ❌ Running background-removal / vectorization / image-generation synchronously in a GraphQL request instead of a BullMQ job.
- ❌ Serving generated assets from a public S3 bucket instead of short-lived signed URLs, or skipping tenant scoping on asset access.
- ❌ Putting a global vendor key (`REMOVE_BG_API_KEY`, …) in the per-tenant config, or leaving AI calls unquota'd per tenant.
- ❌ Filing a not-inherently-AI capability (e.g. vectorization) under `ai/`, or pre-creating a *vendor* sub-folder before a 2nd vendor exists (capability folders are always fine).

