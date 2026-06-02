# Codebase Context — NestJS Code-First GraphQL

> **Purpose of this file (read first).**
> This is the **map** of the codebase, not the rulebook. It answers _"what is this system, where does everything live, and how does a request flow through it?"_ — so an automated agent can orient itself **before** changing anything.
>
> It is **descriptive**, not prescriptive. For _how you are required to write code_ (conventions, do's and don'ts, definition of done), see `[engineering-guidelines.md](./engineering-guidelines.md)`.
>
> Rule of thumb: if a fact would still be true even if we never wrote another line of code, it belongs here. If it tells you _how_ to write that next line, it belongs in the guidelines.

---

## 1. What this system is

A large-scale **NestJS** backend exposing a **GraphQL API** built with the **code-first** approach: TypeScript classes decorated with `@ObjectType()`, `@InputType()`, `@Resolver()`, etc. are the single source of truth, and `schema.gql` is **auto-generated** from them on startup.

- **API style:** GraphQL (Apollo Server via `@nestjs/apollo`) over HTTP `POST /graphql`, plus a WebSocket (`graphql-ws`) channel for subscriptions; one REST endpoint (`GET /health`) for infra probes.
- **Schema authority:** TypeScript decorators → `schema.gql` is a build artifact, never hand-edited.
- **Persistence:** PostgreSQL via TypeORM.
- **Domains today:** `users`, `posts`, `auth`.

### Endpoints exposed

The entire **domain API is GraphQL** (over HTTP + WebSocket); the only REST surface is the health probe.

| Endpoint               | Transport    | What it serves                         | Defined where                                                       |
| ---------------------- | ------------ | -------------------------------------- | ------------------------------------------------------------------- |
| `POST /graphql`        | HTTP         | All queries + mutations — the main API | code-first resolvers, mounted by `GraphQLModule` in `app.module.ts` |
| `/graphql` (WebSocket) | `graphql-ws` | GraphQL **subscriptions**              | `subscriptions: { 'graphql-ws': true }` in `app.module.ts`          |
| `GET /health`          | **REST**     | Liveness/readiness (db + redis status) | `health/health.controller.ts` (a normal `@Controller`, Terminus)    |

> Why REST for `/health`: load balancers and Kubernetes probes expect a plain HTTP `GET` returning 200/503 — they can't speak GraphQL. The two transports are also why `common/filters/` has both `gql-exception.filter.ts` and `http-exception.filter.ts`.

### Why code-first (the decision already made)

TypeScript is the source of truth; there is no SDL to keep in sync, so the schema cannot drift from the resolvers. IDE renames propagate to the schema, there is no codegen step in the inner loop, and `PartialType`/`PickType`/`OmitType` compose naturally. Schema-first would only be chosen if a non-TypeScript consumer needed the hand-authored `.graphql` file — that is not the case here.

---

## 2. Tech stack

| Concern          | Technology                                                              |
| ---------------- | ----------------------------------------------------------------------- |
| Runtime          | Node.js 24.16.0 "Krypton" (current active LTS)                          |
| Framework        | NestJS                                                                  |
| API              | GraphQL code-first, Apollo Server (`@nestjs/apollo`, `@nestjs/graphql`) |
| Database         | PostgreSQL + TypeORM                                                    |
| Cache            | Redis (`ioredis`)                                                       |
| Queues           | BullMQ + Redis                                                          |
| Auth             | PassportJS + JWT + CASL (RBAC + ability-based)                          |
| Validation       | `class-validator` (via global `ValidationPipe`)                         |
| Logging          | Pino (structured JSON)                                                  |
| Observability    | OpenTelemetry + Prometheus + Jaeger; Sentry for errors                  |
| Testing          | Jest (unit) + Supertest (e2e)                                           |
| Security headers | Helmet, CORS allowlist                                                  |

---

## 3. Top-level layout

`<app-name>` = the project root directory (whatever the repo is named).

```
<app-name>/
├── src/
│   ├── modules/             # One folder per business domain (users, posts, auth) — full anatomy in §4
│   ├── common/             # Shared cross-module utilities
│   │   ├── decorators/     # @CurrentUser, @Roles, @Public, @Complexity
│   │   ├── guards/         # RolesGuard, throttler-gql guard
│   │   ├── filters/        # gql-exception, http-exception
│   │   ├── interceptors/   # logging, timeout
│   │   ├── pipes/          # parse-uuid (DTO shape validation uses class-validator, §2)
│   │   ├── scalars/        # Date, JSON, Upload          ← reused everywhere (guidelines §2)
│   │   ├── plugins/        # Apollo: complexity, depth-limit, logging, tracing, sentry
│   │   ├── directives/     # auth.directive
│   │   ├── enums/          # sort-direction, action
│   │   └── types/          # pagination.args, page-info, connection, edge
│   ├── config/             # Typed config (registerAs): app, database, graphql, redis, queue, jwt
│   ├── health/             # REST health endpoint (Terminus: db + redis)
│   ├── app.module.ts       # Root module — wires GraphQL, infra libs, feature modules, Apollo plugins
│   └── main.ts             # Bootstrap — helmet, CORS, global ValidationPipe, listen
├── libs/                   # Internal shared infrastructure libraries
│   ├── database/           # TypeORM: base.entity, transaction.service (unit-of-work), migrations/, seeds/
│   ├── cache/              # Redis (ioredis)
│   ├── queue/              # BullMQ + dead-letter queue
│   ├── logger/             # Pino (structured JSON)
│   ├── auth/               # Passport + JWT + CASL
│   ├── telemetry/          # OpenTelemetry + Prometheus
│   └── storage/            # AWS S3 (signed URLs)
├── integrations/           # Anti-corruption layer for external APIs: commerce + general capabilities — see §13
├── test/                   # e2e specs, helpers, fixtures, factories
├── scripts/                # seed, migrate, codegen
├── docker/                 # Dockerfile, compose.yml, nginx.conf
├── schema.gql              # AUTO-GENERATED — do not edit
└── tsconfig.paths.json     # Path aliases: @libs/*, @common/*, @modules/*, @config/*
```

---

## 4. Anatomy of a feature module

Every domain under `src/modules/<name>s/` follows the **identical** layout. This consistency is the most important navigational fact in the repo — once you know one module, you know them all. The bundle is what `nest g resource <name>` scaffolds.

> **Placeholder legend:** `<name>` = the NestJS CLI's `<name>` argument — the singular domain noun used in file/instance names (e.g. `user`, `post`); `<name>s` = its plural, used for the module folder and plural artifacts (e.g. `users/`, `users-filter.args.ts`); `<Name>` = PascalCase class/type name (e.g. `User`). Matches `nest generate <schematic> <name>`. The same placeholders are used in `[engineering-guidelines.md](./engineering-guidelines.md)` §1.

```
modules/<name>s/
├── <name>.resolver.ts        # @Resolver(<Name>) — queries, mutations, subscriptions, @ResolveField
├── <name>.service.ts         # Business logic + authorization. NO GraphQL concerns.
├── <name>.module.ts          # NestJS wiring (imports, providers, exports)
├── models/                   # GQL @ObjectType() classes ONLY — no DB decorators
│   ├── <name>.type.ts
│   ├── <name>-connection.type.ts   # Relay pagination connection
│   └── <name>-order.enum.ts        # registerEnumType() for sort fields
├── dto/                      # GQL @InputType() and @ArgsType() classes
│   ├── create-<name>.input.ts
│   ├── update-<name>.input.ts      # extends PartialType(Create<Name>Input)
│   └── <name>s-filter.args.ts      # filter + cursor pagination args
├── entities/                 # TypeORM @Entity() classes ONLY — no GQL decorators
│   ├── <name>.entity.ts
│   └── <name>.repository.ts        # Custom query methods (findPaginated, etc.)
├── loaders/                  # Scope.REQUEST DataLoaders — batch DB reads, prevent N+1
│   └── <name>.loader.ts
├── mappers/                  # Pure entity ↔ GQL type conversion
│   └── <name>.mapper.ts            # toGql(), toConnection()
├── policies/                 # CASL ability rules — who can read/write/delete
│   └── manage-<name>.policy.ts
├── subscribers/              # PubSub event payload shapes for @Subscription
├── __tests__/                # Co-located unit tests, one spec per class
└── index.ts                  # Barrel re-exports
```

### The two-model split (a defining characteristic)

`models/` (GraphQL types) and `entities/` (DB tables) are **deliberately separate**, bridged by a `mapper/`:

- A schema change never forces a DB migration, and a DB refactor never breaks the GQL contract.
- Sensitive columns (e.g. `passwordHash`) live on the entity but are **never** exposed as a `@Field()`. The mapper omits them; `@HideField()` guards them at the type level.

---

## 5. Infrastructure layer — `libs/`

`libs/` modules are imported **once** in `AppModule` and provided globally. Feature modules never create their own DB connections, Redis clients, or auth guards — they consume these.

| Lib               | Tech                       | Provides                                                                                             |
| ----------------- | -------------------------- | ---------------------------------------------------------------------------------------------------- |
| `libs/database/`  | TypeORM + PostgreSQL       | Connection, `BaseEntity` (uuid id, createdAt, updatedAt, deletedAt), migrations, seeds, transactions |
| `libs/cache/`     | Redis + ioredis            | `get<T>()`, `set()`, `del()`, `invalidatePattern()`                                                  |
| `libs/queue/`     | BullMQ + Redis             | Job queues, retry logic, dead-letter queue, base processor                                           |
| `libs/logger/`    | Pino                       | Structured JSON logs + Apollo op-logging plugin                                                      |
| `libs/auth/`      | Passport + JWT + CASL      | `JwtStrategy`, `GqlAuthGuard`, `CaslAbilityFactory`, `@CurrentUser()`, `RolesGuard`                  |
| `libs/telemetry/` | OpenTelemetry + Prometheus | Tracing spans, metrics histograms, Jaeger exporter                                                   |
| `libs/storage/`   | AWS S3                     | `upload()`, `getSignedUrl()`, `delete()` — object storage behind a thin interface (no public buckets) |

---

## 6. How a GraphQL request flows

```
HTTP POST /graphql
  → Apollo plugin pipeline (see §7)
  → GqlAuthGuard           (JWT → attaches user to GQL context)
  → RolesGuard             (@Roles metadata via Reflector)
  → ValidationPipe         (validates @InputType / @ArgsType via class-validator)
  → Resolver method        (thin — delegates immediately)
      → Service            (business logic + CASL authorization check)
          → Repository     (TypeORM query; soft-delete aware: WHERE deletedAt IS NULL)
      → Mapper             (entity → GQL type, strips sensitive fields)
  → @ResolveField for relations
      → DataLoader (Scope.REQUEST)   (batches N lookups into one WHERE id IN (...))
  → Response
```

Key invariant: **resolvers are thin, services hold logic, repositories hold queries, mappers cross the boundary, loaders batch relations.** No layer reaches across another.

---

## 7. Apollo plugin pipeline (registered in `app.module.ts`)

| Order | Plugin                 | Purpose                                                     |
| ----- | ---------------------- | ----------------------------------------------------------- |
| 1     | `ComplexityPlugin`     | Rejects queries over the max complexity score (default 200) |
| 2     | `DepthLimitPlugin`     | Rejects queries nested deeper than the limit (default 7)    |
| 3     | `DataloaderPlugin`     | Injects per-request DataLoader instances into context       |
| 4     | `LoggingPlugin`        | Logs operation name, duration, error status                 |
| 5     | `TracingPlugin`        | OpenTelemetry span per operation                            |
| 6     | `SentryPlugin`         | Captures resolver errors with context                       |
| 7     | `PersistedQueryPlugin` | APQ — caches query documents by hash                        |

---

## 8. Cross-cutting decorators (applied on resolver methods)

```typescript
@Resolver(() => User)
@UseGuards(GqlAuthGuard)                              // JWT authentication
export class UserResolver {
  @Query(() => UserConnection)
  @Roles('admin', 'editor')                           // RBAC authorization
  @UseInterceptors(CacheInterceptor)                  // Redis response caching
  @Throttle({ default: { limit: 30, ttl: 60000 } })   // Rate limiting
  @Complexity(5)                                       // Query cost hint
  findAll(@Args() args: UsersFilterArgs) { ... }
}
```

These live in `src/common/` (`decorators/`, `guards/`, `interceptors/`) and `libs/auth/`.

---

## 9. Established patterns (the "why" behind the structure)

- **Relay-style cursor pagination.** All list queries return a `Connection` type (`edges`, `pageInfo`, `totalCount`), never a bare array. Generic `Connection<T>`/`Edge<T>` live in `src/common/types/`.
- **N+1 prevention via DataLoader.** Every `@ResolveField` that loads a relation goes through a `Scope.REQUEST` loader, never directly through a service. `Scope.REQUEST` guarantees a fresh per-request cache so data can't leak between requests.
- **Soft deletes.** `BaseEntity` carries `deletedAt`; repositories filter `WHERE deletedAt IS NULL` by default. Records are never physically removed (audit trail + recovery).
- **Path aliases.** `@libs/`_, `@common/_`, `@modules/_`, `@config/_`(configured in`tsconfig.paths.json`and mirrored in`jest.config.ts`) — no `../../../` chains.
- **Typed config.** Each `src/config/*.config.ts` uses `registerAs` and reads from env; consumed via `ConfigService`.

---

## 10. Security model — controls already in place

This maps _where each defense lives and what it protects_. It is descriptive; the enforceable security rules ("you must …") live in `[engineering-guidelines.md](./engineering-guidelines.md)` — primarily §12 (Security — required checks & gates), with the layer-specific rules in §2–§6.

| Layer                    | Threat addressed                         | Control & where it lives                                                                                                                  |
| ------------------------ | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Transport                | MITM, header attacks, cross-origin abuse | Helmet security headers + CORS allowlist (`ALLOWED_ORIGINS`) in `main.ts`; TLS termination in `docker/nginx.conf`                         |
| Authentication           | Forged/absent identity                   | JWT via Passport — `libs/auth/jwt.strategy.ts`, `gql-auth.guard.ts`; caller exposed as `@CurrentUser()`                                   |
| Authorization (RBAC)     | Wrong role acting                        | `@Roles()` + `RolesGuard` (`Reflector` metadata) in `src/common/` / `libs/auth/`                                                          |
| Authorization (resource) | Acting on records you don't own          | CASL ability check **in the service** before mutating; rules in `policies/manage-<x>.policy.ts`                                           |
| Input validation         | Malformed / over-posted input            | Global `ValidationPipe` (`whitelist` + `forbidNonWhitelisted` + `transform`) in `main.ts`, driven by `class-validator` decorators on DTOs |
| Query abuse (DoS)        | Deep / expensive / flooding queries      | `DepthLimitPlugin` (default 7), `ComplexityPlugin` (default 200), `@Throttle` rate limiting — `src/common/plugins/` + `ThrottlerModule`   |
| Sensitive-data exposure  | Leaking secrets via the API              | `passwordHash` is `@HideField()` on the model, `{ select: false }` on the entity, and omitted by the mapper — never reaches the schema    |
| Injection                | SQL injection                            | Parameterized TypeORM query builder (`:param`), never string-concatenated user input; repository-owned queries                            |
| Credential storage       | Password theft                           | `bcrypt` hashing (cost ≥ 12) in the service layer only                                                                                    |
| Data lifecycle           | Hard-delete data loss / no audit trail   | Soft deletes (`deletedAt`); repositories filter `WHERE deletedAt IS NULL`                                                                 |
| Introspection leakage    | Schema disclosure in prod                | `playground` + `introspection` disabled when `NODE_ENV=production` (`app.module.ts`)                                                      |
| Error leakage            | Stack traces / internals to clients      | `gql-exception.filter.ts` maps exceptions to safe GraphQL errors; `SentryPlugin` captures full detail server-side only                    |
| Secret management        | Committed credentials                    | Secrets via env (`ConfigService`); `.env.prod` never committed                                                                            |
| CSRF                     | Cross-site request forgery               | Stateless **Bearer JWT** auth (token in `Authorization` header, no ambient session cookie) — no CSRF surface to exploit                    |

> **Known gaps & the checks that close them** (dependency/secret/SAST scanning, GraphQL Armor, batching caps, container hardening, token rotation, persisted-query allowlist, audit logging, **edge/volumetric DDoS mitigation, SSRF allowlisting on outbound platform calls, login brute-force lockout, file-upload validation**) are not described here because they are _actions you must take_, not facts about the system — they live as enforceable rules in `[engineering-guidelines.md](./engineering-guidelines.md)` §12 (and §13 for the integration layer).

---

## 11. Environment & configuration

Config is loaded globally in `app.module.ts` from `.env.${NODE_ENV}` then `.env`. Per-environment files: `.env` (local), `.env.test` (test runner), `.env.prod` (production — never committed).

Notable env vars: `PORT`, `DB_`_, `REDIS\__`, `JWT_SECRET`/`JWT_EXPIRES_IN`, `GQL_DEPTH_LIMIT`(7),`GQL_MAX_COMPLEXITY`(200),`ALLOWED_ORIGINS`, `OTEL_EXPORTER_JAEGER_ENDPOINT`, `SENTRY_DSN`; storage: `AWS_S3_BUCKET`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`; **global provider keys** (general capabilities, §13b): `REMOVE_BG_API_KEY`, `STABILITY_API_KEY`.

---

## 12. Testing topology (where tests live — _how_ to test is in the guidelines)

- **Unit tests** are co-located in each module's `__tests__/`, one spec per class (resolver, service, repository, loader, mapper).
- **e2e tests** live in `test/e2e/`, one spec per domain plus `health`.
- **Shared test infra** in `test/`: `helpers/` (`gqlRequest`, `getAuthToken`), `fixtures/` (static data), `factories/` (Faker-based entity builders like `buildUserEntity`).
- Boundary rule: **unit tests mock everything at the boundary; e2e tests mock nothing except external services** (Stripe, SendGrid, …). Repository unit tests use in-memory SQLite.

---

## 13. External integrations (`integrations/`)

`integrations/` is a sibling to `libs/` and is **the only place that knows how to talk to an external third-party API.** It holds two categories of provider, which differ in how they're selected:

| Category | Examples | Platform-specific? | Provider resolved by |
| --- | --- | --- | --- |
| **Commerce platforms** | `commerce/magento/` | Yes — *is* the platform | **per-tenant** (the client's platform) |
| **General capabilities** | `background-removal/`, `vectorization/`, `ai/` | No — work for any client | **global config** (your API keys) |

> Rule for what lands here: *"if this vendor shut down tomorrow, do I swap an adapter or rewrite infrastructure?"* Swap an adapter → `integrations/`. Rewrite infra → `libs/` (e.g. S3 lives in `libs/storage/`, §5).

### 13a. Commerce platforms — `commerce/`

This product is a **design solution layered on top of clients' existing commerce platforms** (Magento today; Shopify and others later). A client's products, categories, and attributes already live in their platform — we read them in, store a copy, and let the admin/designer work against that copy. The React **admin panel** and **designer tool** consume only our GraphQL API; they never talk to the platform directly.

### Where the platform is touched

Only two flows reach the external platform; every other read is pure Postgres.

| Flow            | Direction     | What happens                                                                                                                                               |
| --------------- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sync**        | platform → us | Admin triggers `syncNow`; a **BullMQ** job pulls products/categories/attributes and upserts them into Postgres — the single source of truth for all reads. |
| **Add to cart** | us → platform | Designer sends `addToCart` (GraphQL in); we proxy it to the platform's cart API. **The platform owns the cart & checkout.**                                |

### The anti-corruption layer — `integrations/commerce/`

`integrations/` is a sibling to `libs/` and is **the only place that knows whether a platform speaks REST or GraphQL.** Business code talks to one platform-agnostic interface.

> **Illustrative, not final.** The file/folder layout below is a _proposed_ example of how the anti-corruption layer is organized — the _principle_ (one platform-agnostic provider interface; per-platform adapters as the only place that knows REST vs GraphQL) is the firm recommendation, but exact names and file breakdown are not decided yet.

```
integrations/commerce/
├── commerce-provider.interface.ts   # contract: fetchProducts/Categories/Attributes, addToCart → canonical DTOs
├── commerce-provider.registry.ts    # forTenant(tenant) → the right adapter + that client's credentials
├── dto/                             # canonical ProductDTO, CategoryDTO, AttributeDTO, CartDTO (platform-agnostic)
└── magento/
    ├── magento.provider.ts          # implements the interface over REST
    ├── magento.client.ts            # low-level REST + auth token, timeout, retry, circuit-breaker
    └── magento.mapper.ts            # Magento JSON → canonical DTO  (the anti-corruption boundary)
```

Adding a platform = one new folder implementing the interface + a registry case — **no change to `modules/`**.

### Two mapping boundaries (deliberately separate — do not merge)

```
Magento REST JSON
  └(1) magento.mapper.ts       platform shape → canonical DTO     [integrations/]
ProductDTO (canonical)
  └(2) sync.service upsert     canonical DTO → TypeORM entity      [modules/sync/]
ProductEntity (Postgres)  ◄── single source of truth for reads
  └(3) catalog mapper.toGql    entity → GraphQL type               [modules/catalog/]  (the ordinary §4 mapper)
GraphQL Product → React admin / designer
```

Mapper (1) is the anti-corruption boundary (stops platform shapes leaking inward); mapper (3) is the entity↔GQL mapper described in §4.

### Supporting feature modules

> **Illustrative, not final.** The modules below are a _proposed_ breakdown to show how the integration concerns map onto the §4 feature-module anatomy. Exact names and boundaries are not decided yet — treat them as an example of the shape, not the committed structure.

- `modules/catalog/` — products/categories/attributes as **our** domain; reads Postgres only (mirrors the §4 anatomy). Entities carry `tenantId`, `platform`, `externalId`, `syncedAt`.
- `modules/sync/` — `syncNow` mutation + BullMQ processor + a `sync-run` entity tracking status, counts, errors, and timings.
- `modules/cart/` — `addToCart` proxy to the resolved provider.
- `modules/tenancy/` — per-client `tenant-platform-config` (platform + **encrypted** credentials) and current-tenant resolution from the JWT.

### Multi-tenancy & isolation

**One platform per client.** Every catalog/sync row carries `tenantId`; all reads are scoped by the current tenant (from the JWT), enforced in the repository layer. Platform credentials are encrypted at rest and never exposed through GraphQL.

> The enforceable rules for this layer (never call a platform outside its adapter, canonical DTOs only, sync via BullMQ, tenant scoping, credential encryption, resilience) live in [`engineering-guidelines.md`](./engineering-guidelines.md) §13.

### 13b. General capability providers (platform-agnostic)

Image/asset operations the designer needs — **background removal, vectorization, AI image generation** — are independent of any commerce platform. They're external vendors behind the same adapter pattern, but resolved from **global config** (your keys), not per tenant.

> **Illustrative, not final.** Shape and vendor names below are a proposed example; the firm parts are the principle (vendor behind a capability interface) and the placement rules.

```
integrations/
├── commerce/                 # platform-specific (see §13a)
│   └── magento/ …
├── background-removal/       # general · flat (vendor named in the file, not a folder)
│   ├── background-removal.provider.interface.ts
│   └── removebg.provider.ts / removebg.client.ts / removebg.mapper.ts   # → api.remove.bg, X-Api-Key
├── vectorization/            # general · flat (vendor may be algorithmic OR ML — hidden by the adapter)
│   └── …
└── ai/                       # umbrella · INHERENTLY-generative AI only (grows: image-gen, text-gen, upscaling…)
    └── image-generation/     # capability folder (parallel to background-removal/); vendor stays a filename
        ├── image-generation.provider.interface.ts
        └── stability.provider.ts / stability.client.ts / stability.mapper.ts
```

**Why these placements (the rules behind the tree):**
- **Group by *capability*, not by technique.** `background-removal/` and `vectorization/` stay flat even though their vendors may use ML — that's an implementation detail the adapter hides. Only **inherently**-generative capabilities (no non-AI equivalent, e.g. text→image) go under `ai/`.
- **Every *capability* gets a folder; only the *vendor* stays a filename until there's a 2nd vendor.** So `background-removal/`, `vectorization/`, and `ai/image-generation/` are all capability folders, but there's no `removebg/` or `stability/` vendor sub-folder (the filename carries the vendor). `ai/` is the umbrella grouping the generative capabilities.
- **`ai/` groups real shared concerns** — token/credit cost tracking, prompt inputs, model/version config, content moderation, higher latency.

**Supporting infra & workflow:**
- **S3 → `libs/storage/`** (an infra primitive we operate, §5), not `integrations/` — accessed via signed URLs, never a public bucket.
- **Keys → `src/config/`** (e.g. `REMOVE_BG_API_KEY`, `STABILITY_API_KEY`), read through `ConfigService`. These are *global* product keys, so they do **not** go in the per-tenant `tenant-platform-config` (§13a).
- **Workflow → a feature module** (illustrative: `modules/media/`). These ops are slow, costly, and can fail, so they run as **BullMQ jobs**, never inside a GraphQL request:

```
designer → generateImage / removeBackground mutation (GraphQL)
  → media.service: validate + authorize + tenant-scope, persist asset (PENDING), enqueue job, return assetId
  → media.processor (BullMQ): provider.run() → libs/storage.upload() → asset READY + s3Key
  → designer gets result via subscription or jobStatus poll
```

> Enforceable rules for general providers (vendor behind an interface, AI/image ops async via BullMQ, S3 via signed URLs, per-tenant AI quotas, global keys in config) are in [`engineering-guidelines.md`](./engineering-guidelines.md) §13.

---

## 14. Layer responsibility summary

| Layer             | Responsibility                                       | Key tech                  |
| ----------------- | ---------------------------------------------------- | ------------------------- |
| `src/modules/`    | Feature business logic                               | NestJS, `@nestjs/graphql` |
| `src/common/`     | Decorators, guards, filters, pipes, scalars, plugins | class-validator, CASL     |
| `src/config/`     | Typed configuration                                  | `@nestjs/config`          |
| `integrations/`   | External API adapters (anti-corruption layer): commerce + general capabilities | Magento REST, remove.bg, Stability (+ future) |
| `libs/database/`  | Persistence                                          | TypeORM, PostgreSQL       |
| `libs/cache/`     | Caching                                              | Redis, ioredis            |
| `libs/queue/`     | Async jobs                                           | BullMQ, Redis             |
| `libs/storage/`   | Object storage                                       | AWS S3                    |
| `libs/auth/`      | AuthN + AuthZ                                        | Passport, JWT, CASL       |
| `libs/logger/`    | Structured logging                                   | Pino                      |
| `libs/telemetry/` | Observability                                        | OpenTelemetry, Prometheus |
| `test/`           | Quality assurance                                    | Jest, Supertest           |
