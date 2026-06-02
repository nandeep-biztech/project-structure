# Codebase Context — NestJS Code-First GraphQL

> **Purpose of this file (read first).**
> This is the **map** of the codebase, not the rulebook. It answers *"what is this system, where does everything live, and how does a request flow through it?"* — so an automated agent can orient itself **before** changing anything.
>
> It is **descriptive**, not prescriptive. For *how you are required to write code* (conventions, do's and don'ts, definition of done), see [`engineering-guidelines.md`](./engineering-guidelines.md).
>
> Rule of thumb: if a fact would still be true even if we never wrote another line of code, it belongs here. If it tells you *how* to write that next line, it belongs in the guidelines.

---

## 1. What this system is

A large-scale **NestJS** backend exposing a **GraphQL API** built with the **code-first** approach: TypeScript classes decorated with `@ObjectType()`, `@InputType()`, `@Resolver()`, etc. are the single source of truth, and `schema.gql` is **auto-generated** from them on startup.

- **API style:** GraphQL (Apollo Server via `@nestjs/apollo`), with a single REST endpoint (`GET /health`).
- **Schema authority:** TypeScript decorators → `schema.gql` is a build artifact, never hand-edited.
- **Persistence:** PostgreSQL via TypeORM.
- **Domains today:** `users`, `posts`, `auth`.

### Why code-first (the decision already made)
TypeScript is the source of truth; there is no SDL to keep in sync, so the schema cannot drift from the resolvers. IDE renames propagate to the schema, there is no codegen step in the inner loop, and `PartialType`/`PickType`/`OmitType` compose naturally. Schema-first would only be chosen if a non-TypeScript consumer needed the hand-authored `.graphql` file — that is not the case here.

---

## 2. Tech stack

| Concern | Technology |
| --- | --- |
| Framework | NestJS |
| API | GraphQL code-first, Apollo Server (`@nestjs/apollo`, `@nestjs/graphql`) |
| Database | PostgreSQL + TypeORM |
| Cache | Redis (`ioredis`) |
| Queues | BullMQ + Redis |
| Auth | PassportJS + JWT + CASL (RBAC + ability-based) |
| Validation | `class-validator` (via global `ValidationPipe`) |
| Logging | Pino (structured JSON) |
| Observability | OpenTelemetry + Prometheus + Jaeger; Sentry for errors |
| Testing | Jest (unit) + Supertest (e2e) |
| Security headers | Helmet, CORS allowlist |

---

## 3. Top-level layout

```
my-enterprise-app/
├── src/
│   ├── modules/        # One folder per business domain (users, posts, auth)
│   ├── common/         # Shared cross-module utilities (decorators, guards, filters, scalars, plugins…)
│   ├── config/         # Typed configuration (@nestjs/config registerAs)
│   ├── health/         # REST health endpoint (Terminus: db + redis)
│   ├── app.module.ts   # Root module — wires GraphQL, infra libs, feature modules, Apollo plugins
│   └── main.ts         # Bootstrap — helmet, CORS, global ValidationPipe, listen
├── libs/               # Internal shared infrastructure libraries (database, cache, queue, logger, auth, telemetry)
├── test/               # e2e specs, helpers, fixtures, factories
├── scripts/            # seed, migrate, codegen
├── docker/             # Dockerfile, compose.yml, nginx.conf
├── schema.gql          # AUTO-GENERATED — do not edit
└── tsconfig.paths.json # Path aliases: @libs/*, @common/*, @modules/*, @config/*
```

---

## 4. Anatomy of a feature module

Every domain under `src/modules/<domain>/` follows the **identical** layout. This consistency is the most important navigational fact in the repo — once you know one module, you know them all.

```
modules/users/
├── user.resolver.ts        # @Resolver(User) — queries, mutations, subscriptions, @ResolveField
├── user.service.ts         # Business logic + authorization. NO GraphQL concerns.
├── user.module.ts          # NestJS wiring (imports, providers, exports)
├── models/                 # GQL @ObjectType() classes ONLY — no DB decorators
│   ├── user.type.ts
│   ├── user-connection.type.ts   # Relay pagination connection
│   └── user-order.enum.ts        # registerEnumType() for sort fields
├── dto/                    # GQL @InputType() and @ArgsType() classes
│   ├── create-user.input.ts
│   ├── update-user.input.ts      # extends PartialType(CreateUserInput)
│   └── users-filter.args.ts      # filter + cursor pagination args
├── entities/               # TypeORM @Entity() classes ONLY — no GQL decorators
│   ├── user.entity.ts
│   └── user.repository.ts        # Custom query methods (findPaginated, etc.)
├── loaders/                # Scope.REQUEST DataLoaders — batch DB reads, prevent N+1
│   └── user.loader.ts
├── mappers/                # Pure entity ↔ GQL type conversion
│   └── user.mapper.ts            # toGql(), toConnection()
├── policies/               # CASL ability rules — who can read/write/delete
│   └── manage-user.policy.ts
├── subscribers/            # PubSub event payload shapes for @Subscription
├── __tests__/              # Co-located unit tests, one spec per class
└── index.ts                # Barrel re-exports
```

### The two-model split (a defining characteristic)
`models/` (GraphQL types) and `entities/` (DB tables) are **deliberately separate**, bridged by a `mapper/`:
- A schema change never forces a DB migration, and a DB refactor never breaks the GQL contract.
- Sensitive columns (e.g. `passwordHash`) live on the entity but are **never** exposed as a `@Field()`. The mapper omits them; `@HideField()` guards them at the type level.

---

## 5. Infrastructure layer — `libs/`

`libs/` modules are imported **once** in `AppModule` and provided globally. Feature modules never create their own DB connections, Redis clients, or auth guards — they consume these.

| Lib | Tech | Provides |
| --- | --- | --- |
| `libs/database/` | TypeORM + PostgreSQL | Connection, `BaseEntity` (uuid id, createdAt, updatedAt, deletedAt), migrations, seeds, transactions |
| `libs/cache/` | Redis + ioredis | `get<T>()`, `set()`, `del()`, `invalidatePattern()` |
| `libs/queue/` | BullMQ + Redis | Job queues, retry logic, dead-letter queue, base processor |
| `libs/logger/` | Pino | Structured JSON logs + Apollo op-logging plugin |
| `libs/auth/` | Passport + JWT + CASL | `JwtStrategy`, `GqlAuthGuard`, `CaslAbilityFactory`, `@CurrentUser()`, `RolesGuard` |
| `libs/telemetry/` | OpenTelemetry + Prometheus | Tracing spans, metrics histograms, Jaeger exporter |

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

| Order | Plugin | Purpose |
| --- | --- | --- |
| 1 | `ComplexityPlugin` | Rejects queries over the max complexity score (default 200) |
| 2 | `DepthLimitPlugin` | Rejects queries nested deeper than the limit (default 7) |
| 3 | `DataloaderPlugin` | Injects per-request DataLoader instances into context |
| 4 | `LoggingPlugin` | Logs operation name, duration, error status |
| 5 | `TracingPlugin` | OpenTelemetry span per operation |
| 6 | `SentryPlugin` | Captures resolver errors with context |
| 7 | `PersistedQueryPlugin` | APQ — caches query documents by hash |

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
- **Path aliases.** `@libs/*`, `@common/*`, `@modules/*`, `@config/*` (configured in `tsconfig.paths.json` and mirrored in `jest.config.ts`) — no `../../../` chains.
- **Typed config.** Each `src/config/*.config.ts` uses `registerAs` and reads from env; consumed via `ConfigService`.

---

## 10. Environment & configuration

Config is loaded globally in `app.module.ts` from `.env.${NODE_ENV}` then `.env`. Per-environment files: `.env` (local), `.env.test` (test runner), `.env.prod` (production — never committed).

Notable env vars: `PORT`, `DB_*`, `REDIS_*`, `JWT_SECRET`/`JWT_EXPIRES_IN`, `GQL_DEPTH_LIMIT` (7), `GQL_MAX_COMPLEXITY` (200), `ALLOWED_ORIGINS`, `OTEL_EXPORTER_JAEGER_ENDPOINT`, `SENTRY_DSN`.

---

## 11. Testing topology (where tests live — *how* to test is in the guidelines)

- **Unit tests** are co-located in each module's `__tests__/`, one spec per class (resolver, service, repository, loader, mapper).
- **e2e tests** live in `test/e2e/`, one spec per domain plus `health`.
- **Shared test infra** in `test/`: `helpers/` (`gqlRequest`, `getAuthToken`), `fixtures/` (static data), `factories/` (Faker-based entity builders like `buildUserEntity`).
- Boundary rule: **unit tests mock everything at the boundary; e2e tests mock nothing except external services** (Stripe, SendGrid, …). Repository unit tests use in-memory SQLite.

---

## 12. Layer responsibility summary

| Layer | Responsibility | Key tech |
| --- | --- | --- |
| `src/modules/` | Feature business logic | NestJS, `@nestjs/graphql` |
| `src/common/` | Decorators, guards, filters, pipes, scalars, plugins | class-validator, CASL |
| `src/config/` | Typed configuration | `@nestjs/config` |
| `libs/database/` | Persistence | TypeORM, PostgreSQL |
| `libs/cache/` | Caching | Redis, ioredis |
| `libs/queue/` | Async jobs | BullMQ, Redis |
| `libs/auth/` | AuthN + AuthZ | Passport, JWT, CASL |
| `libs/logger/` | Structured logging | Pino |
| `libs/telemetry/` | Observability | OpenTelemetry, Prometheus |
| `test/` | Quality assurance | Jest, Supertest |
