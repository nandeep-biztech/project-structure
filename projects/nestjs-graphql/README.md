# NestJS Code-First GraphQL — Enterprise Project Structure

> A complete reference for structuring a large-scale NestJS application using the **code-first** GraphQL approach. Covers directory layout, feature module anatomy, infrastructure libraries, key file implementations, and architectural decisions.

---

## Table of Contents

1. [Why Code-First?](#why-code-first)
2. [Complete Project Tree](#complete-project-tree)
3. [Feature Module Structure](#feature-module-structure)
4. [Infrastructure Layer — `libs/](#infrastructure-layer--libs)
5. [Cross-Cutting Concerns](#cross-cutting-concerns)
6. [Key File Implementations](#key-file-implementations)

- [main.ts](#maints)
- [app.module.ts](#appmodulets)
- [graphql.config.ts](#graphqlconfigts)
- [models/user.type.ts](#modelsusertyepts)
- [dto/create-user.input.ts](#dtocreate-userinputts)
- [user.resolver.ts](#userresolverts)
- [user.service.ts](#userservicets)
- [user.module.ts](#usermodulets)
- [entities/user.entity.ts](#entitiesuserentityts)
- [entities/user.repository.ts](#entitiesuserrepositoryts)
- [loaders/user.loader.ts](#loadersuserloaderts)
- [mappers/user.mapper.ts](#mappersusermapperts)
- [policies/manage-user.policy.ts](#policiesmanage-userpolicyts)

7. [Apollo Plugin Pipeline](#apollo-plugin-pipeline)
2. [Enterprise Patterns & Decisions](#enterprise-patterns--decisions)
3. [Environment & Configuration](#environment--configuration)
4. [Testing Strategy](#testing-strategy)

---

## Why Code-First?

Nest offers **two ways** of building GraphQL applications — **code first** and **schema first**. Both are officially supported by `@nestjs/graphql`; the choice determines what you treat as the *source of truth* for your schema.

### Code-First — definition

> In the **code first** approach, you use decorators and TypeScript classes to generate the corresponding GraphQL schema. This approach is useful if you prefer to work exclusively with TypeScript and avoid context switching between language syntaxes.
> — [NestJS Docs](https://docs.nestjs.com/graphql/quick-start)

You define `@ObjectType()`, `@InputType()`, `@Field()`, `@Resolver()`, `@Query()`, `@Mutation()` on TypeScript classes, and Nest walks those decorators at bootstrap to **emit** the `schema.gql` file (or keep it in memory). TypeScript is the source of truth; SDL is a build artifact.

Enable it by pointing `autoSchemaFile` at a path (or `true` for in-memory):

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  sortSchema: true, // optional — sorts schema lexicographically
}),
```

Example of a code-first type + resolver:

```ts
// user.type.ts
@ObjectType()
export class User {
  @Field(() => ID) id: string;
  @Field() email: string;
  @Field({ nullable: true }) name?: string;
}

// user.resolver.ts
@Resolver(() => User)
export class UserResolver {
  constructor(private readonly users: UserService) {}

  @Query(() => User)
  user(@Args('id', { type: () => ID }) id: string): Promise<User> {
    return this.users.findById(id);
  }
}
```

### Schema-First — definition

> In the **schema first** approach, the source of truth is GraphQL SDL (Schema Definition Language) files. SDL is a language-agnostic way to share schema files between different platforms. Nest automatically generates your TypeScript definitions (using either classes or interfaces) based on the GraphQL schemas to reduce the need to write redundant boilerplate code.
> — [NestJS Docs](https://docs.nestjs.com/graphql/quick-start)

You hand-write `.graphql` SDL files, point Nest at them via `typePaths`, and let `@nestjs/graphql` generate TypeScript classes/interfaces from the AST so resolvers get typed. SDL is the source of truth; TypeScript types are the build artifact.

Enable it via `typePaths` + `definitions`:

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
    outputAs: 'class',
  },
}),
```

Example of a schema-first SDL file + resolver:

```graphql
# user.graphql
type User {
  id: ID!
  email: String!
  name: String
}

type Query {
  user(id: ID!): User!
}
```

```ts
// user.resolver.ts — types imported from generated src/graphql.ts
@Resolver('User')
export class UserResolver {
  constructor(private readonly users: UserService) {}

  @Query('user')
  user(@Args('id') id: string): Promise<User> {
    return this.users.findById(id);
  }
}
```

You can also generate typings on demand via the `GraphQLDefinitionsFactory` (with a `watch: true` option for live regeneration as `.graphql` files change).

### Code-First vs Schema-First — at a glance

| Concern         | Code-First                       | Schema-First                      |
| --------------- | -------------------------------- | --------------------------------- |
| Source of truth | TypeScript decorators            | Hand-written SDL files            |
| Build artifact  | `schema.gql` (auto-emitted)      | `graphql.ts` (auto-generated TS)  |
| Type safety     | Fully in-language, no drift      | Requires codegen step to sync     |
| Refactoring     | IDE rename = schema rename       | Must update SDL + TS manually     |
| Team collab     | SDL auto-generated after build   | SDL is explicit contract in git   |
| Boilerplate     | Less — one class per type        | More — SDL + generated TS + logic |
| Schema review   | Need to share built `schema.gql` | SDL files are reviewable directly |
| Context switch  | TypeScript only                  | TypeScript + SDL                  |

### Why code-first is the better default

1. **Single source of truth, zero drift.** The TypeScript class *is* the schema. There is no second file to keep in sync, so the classic schema-first failure mode — SDL says one thing, resolver signature says another — cannot happen.
2. **Refactor-safe.** An IDE rename on a `@Field()` property renames every usage *and* the generated SDL in one operation. Schema-first requires editing SDL, regenerating types, then fixing resolvers.
3. **No codegen step in the inner loop.** Schema-first needs `GraphQLDefinitionsFactory` (or `ts-node generate-typings`) in watch mode; code-first emits the schema as a side-effect of `nest start`. One less moving part in CI and local dev.
4. **First-class TypeScript ergonomics.** Generics, `PartialType`, `PickType`, `OmitType`, `IntersectionType` from `@nestjs/graphql` compose naturally — e.g. `UpdateUserInput extends PartialType(CreateUserInput)` — which is awkward to express in raw SDL.
5. **Decorator-driven cross-cutting concerns.** Guards, interceptors, directives, complexity, field-level authorization, and DataLoader wiring all attach to the same class that defines the schema. Schema-first forces this logic to live separately from the type it protects.
6. **Less boilerplate per feature.** One `@ObjectType()` class replaces `user.graphql` + the generated `User` interface + the manual resolver return-type annotation.
7. **Better tooling surface for large codebases.** Because types are real classes, you get go-to-definition, find-usages, and structural refactoring across the entire schema — not just text search across `.graphql` files.

**Use schema-first only when** frontend and backend teams want to *design* the API contract in SDL **before** any implementation exists, or when a non-TypeScript consumer (another backend, a codegen pipeline for mobile clients) needs the `.graphql` file to be the human-authored artifact. For everything else — and certainly for the enterprise structure described in the rest of this document — **code-first is the right default**.

---

## Complete Project Tree

The full tree below is the single source of truth — every folder and file in one view. Annotations on the right explain each entry's role.

```
my-enterprise-app/
│
├── src/
│   │
│   ├── modules/                                   # One folder per business domain
│   │   │
│   │   ├── users/
│   │   │   ├── user.resolver.ts                   # @Resolver(User) — queries, mutations, subscriptions
│   │   │   ├── user.service.ts                    # Business logic, no GQL concerns
│   │   │   ├── user.module.ts                     # NestJS module wiring
│   │   │   │
│   │   │   ├── models/                            # GQL types only — no DB decorators
│   │   │   │   ├── user.type.ts                   # @ObjectType() @Field()
│   │   │   │   ├── user-connection.type.ts        # Relay pagination connection
│   │   │   │   └── user-order.enum.ts             # registerEnumType() for sort fields
│   │   │   │
│   │   │   ├── dto/                               # GQL inputs and args
│   │   │   │   ├── create-user.input.ts           # @InputType()
│   │   │   │   ├── update-user.input.ts           # @InputType() + PartialType
│   │   │   │   └── users-filter.args.ts           # @ArgsType() — filter + cursor pagination
│   │   │   │
│   │   │   ├── entities/                          # DB layer — no GQL decorators
│   │   │   │   ├── user.entity.ts                 # @Entity() TypeORM columns
│   │   │   │   └── user.repository.ts             # Custom query methods
│   │   │   │
│   │   │   ├── loaders/                           # DataLoader — prevents N+1 queries
│   │   │   │   └── user.loader.ts                 # Scope.REQUEST, batches by ID
│   │   │   │
│   │   │   ├── mappers/                           # entity ↔ GQL type conversion
│   │   │   │   └── user.mapper.ts                 # toGql(), toConnection()
│   │   │   │
│   │   │   ├── subscribers/                       # GQL subscription payloads
│   │   │   │   └── user-created.event.ts          # PubSub event shape
│   │   │   │
│   │   │   ├── policies/                          # CASL authorization rules
│   │   │   │   └── manage-user.policy.ts          # who can read/write/delete
│   │   │   │
│   │   │   ├── __tests__/
│   │   │   │   ├── user.resolver.spec.ts
│   │   │   │   ├── user.service.spec.ts
│   │   │   │   ├── user.repository.spec.ts
│   │   │   │   ├── user.loader.spec.ts
│   │   │   │   └── user.mapper.spec.ts
│   │   │   │
│   │   │   └── index.ts                           # Barrel re-exports
│   │   │
│   │   ├── posts/                                 # Same structure as users/
│   │   │   ├── post.resolver.ts
│   │   │   ├── post.service.ts
│   │   │   ├── post.module.ts
│   │   │   ├── models/
│   │   │   │   ├── post.type.ts
│   │   │   │   ├── post-connection.type.ts
│   │   │   │   └── post-status.enum.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-post.input.ts
│   │   │   │   ├── update-post.input.ts
│   │   │   │   └── posts-filter.args.ts
│   │   │   ├── entities/
│   │   │   │   ├── post.entity.ts
│   │   │   │   └── post.repository.ts
│   │   │   ├── loaders/
│   │   │   │   └── post.loader.ts                 # Loads posts grouped by authorId
│   │   │   ├── mappers/
│   │   │   │   └── post.mapper.ts
│   │   │   ├── policies/
│   │   │   │   └── manage-post.policy.ts
│   │   │   ├── __tests__/
│   │   │   │   ├── post.resolver.spec.ts
│   │   │   │   ├── post.service.spec.ts
│   │   │   │   ├── post.repository.spec.ts
│   │   │   │   └── post.loader.spec.ts
│   │   │   └── index.ts
│   │   │
│   │   └── auth/                                  # Login / token refresh domain
│   │       ├── auth.resolver.ts                   # login(), refreshToken() mutations
│   │       ├── auth.service.ts                    # validateCredentials(), issueJwt()
│   │       ├── auth.module.ts
│   │       ├── dto/
│   │       │   ├── login.input.ts
│   │       │   └── auth-token.type.ts             # @ObjectType() accessToken payload
│   │       └── __tests__/
│   │           ├── auth.resolver.spec.ts
│   │           └── auth.service.spec.ts
│   │
│   ├── common/                                    # Shared cross-module utilities
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts          # @CurrentUser() from GQL context
│   │   │   ├── roles.decorator.ts                 # @Roles('admin', 'editor')
│   │   │   ├── public.decorator.ts                # @Public() — skips auth guard
│   │   │   └── complexity.decorator.ts            # @Complexity(n) — query cost hint
│   │   ├── guards/
│   │   │   ├── roles.guard.ts                     # Reads @Roles metadata via Reflector
│   │   │   └── throttler-gql.guard.ts             # GQL-aware rate limit guard
│   │   ├── filters/
│   │   │   ├── gql-exception.filter.ts            # Maps exceptions → GQL errors
│   │   │   └── http-exception.filter.ts           # REST health endpoint errors
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts             # Logs method + duration
│   │   │   └── timeout.interceptor.ts             # Kills requests > N ms
│   │   ├── pipes/
│   │   │   ├── parse-uuid.pipe.ts
│   │   │   └── zod-validation.pipe.ts
│   │   ├── scalars/
│   │   │   ├── date.scalar.ts                     # ISO 8601 ↔ Date
│   │   │   ├── json.scalar.ts                     # Arbitrary JSON fields
│   │   │   └── upload.scalar.ts                   # File upload scalar
│   │   ├── plugins/                               # Apollo server plugins
│   │   │   ├── complexity.plugin.ts               # Rejects over-budget queries
│   │   │   ├── depth-limit.plugin.ts              # Max nesting depth
│   │   │   ├── logging.plugin.ts                  # Logs every GQL operation
│   │   │   ├── tracing.plugin.ts                  # OpenTelemetry span per op
│   │   │   └── sentry.plugin.ts                   # Captures resolver errors
│   │   ├── directives/
│   │   │   └── auth.directive.ts
│   │   ├── enums/
│   │   │   ├── sort-direction.enum.ts             # ASC | DESC
│   │   │   └── action.enum.ts                     # CASL actions: Create/Read/Update/Delete
│   │   └── types/
│   │       ├── pagination.args.ts                 # first, after, last, before
│   │       ├── page-info.type.ts                  # @ObjectType() hasNextPage etc.
│   │       ├── connection.type.ts                 # Generic Connection<T>
│   │       └── edge.type.ts                       # Generic Edge<T>
│   │
│   ├── config/
│   │   ├── app.config.ts                          # port, env, allowedOrigins
│   │   ├── database.config.ts                     # host, port, name, ssl
│   │   ├── graphql.config.ts                      # depthLimit, maxComplexity
│   │   ├── redis.config.ts                        # host, port, ttl defaults
│   │   ├── queue.config.ts                        # concurrency, retry, DLQ
│   │   └── jwt.config.ts                          # secret, expiresIn
│   │
│   ├── health/
│   │   ├── health.controller.ts                   # GET /health
│   │   └── health.module.ts                       # Terminus: db + redis indicators
│   │
│   ├── app.module.ts                              # Root — imports all modules
│   └── main.ts                                    # Bootstrap, helmet, cors, pipes
│
├── libs/                                          # Internal shared libraries
│   │
│   ├── database/
│   │   ├── database.module.ts                     # TypeORM forRootAsync setup
│   │   ├── base.entity.ts                         # id (UUID), createdAt, updatedAt, deletedAt
│   │   ├── transaction.service.ts                 # Unit-of-work / transaction wrapper
│   │   ├── migrations/
│   │   │   └── 1700000000000-init.ts              # Timestamped migration files
│   │   └── seeds/
│   │       ├── user.seed.ts
│   │       └── post.seed.ts
│   │
│   ├── cache/
│   │   ├── cache.module.ts                        # Global Redis module (ioredis)
│   │   └── cache.service.ts                       # get<T>(), set(), del(), invalidatePattern()
│   │
│   ├── queue/
│   │   ├── queue.module.ts                        # BullMQ + Redis connection
│   │   ├── queue.constants.ts                     # Queue name constants
│   │   └── base.processor.ts                      # Retry logic, DLQ, tracing
│   │
│   ├── logger/
│   │   ├── logger.module.ts                       # Pino global registration
│   │   ├── logger.service.ts                      # Structured logging wrapper
│   │   └── gql-logger.plugin.ts                   # Apollo plugin: op name + latency
│   │
│   ├── auth/
│   │   ├── auth.module.ts                         # PassportJS + JWT global setup
│   │   ├── jwt.strategy.ts                        # Validates Bearer, attaches user
│   │   ├── gql-auth.guard.ts                      # Extracts GQL context (not HTTP)
│   │   ├── casl-ability.factory.ts                # Builds ability for a given user
│   │   └── roles.guard.ts                         # Checks @Roles() via Reflector
│   │
│   └── telemetry/
│       ├── tracing.module.ts                      # OpenTelemetry SDK + Jaeger exporter
│       ├── metrics.service.ts                     # Prometheus counters + histograms
│       └── tracing.plugin.ts                      # Apollo plugin: span per GQL op
│
├── test/
│   ├── e2e/
│   │   ├── users.e2e-spec.ts                      # Full CRUD + auth + pagination
│   │   ├── posts.e2e-spec.ts
│   │   ├── auth.e2e-spec.ts                       # Login, wrong password, me query
│   │   └── health.e2e-spec.ts                     # /health with db + redis status
│   ├── helpers/
│   │   ├── gql-request.helper.ts                  # Typed GQL wrapper over Supertest
│   │   └── auth.helper.ts                         # Gets JWT for a seeded account
│   ├── fixtures/
│   │   ├── user.fixture.ts                        # Static test data constants
│   │   └── post.fixture.ts
│   └── factories/
│       ├── user.factory.ts                        # Faker-based UserEntity builder
│       └── post.factory.ts                        # Faker-based PostEntity builder
│
├── scripts/
│   ├── seed.ts                                    # Runs all seeds in order
│   ├── migrate.ts                                 # TypeORM migration runner
│   └── codegen.ts                                 # GQL codegen for client consumers
│
├── docker/
│   ├── Dockerfile                                 # Multi-stage: build → production
│   ├── compose.yml                                # App + Postgres + Redis + Jaeger
│   └── nginx.conf                                 # Reverse proxy + TLS termination
│
├── schema.gql                                     # Auto-generated — do not edit
├── package.json
├── tsconfig.json
├── tsconfig.paths.json                            # @libs/*, @common/*, @modules/*
├── jest.config.ts
├── .eslintrc.js
├── .prettierrc
├── .env                                           # Local development
├── .env.test                                      # Test runner overrides
└── .env.prod                                      # Production (never commit)
```

---

## Feature Module Structure

Every feature module (`users/`, `posts/`, `auth/`) follows the identical layout shown in the full tree above. The pattern is:

- `models/` — GQL `@ObjectType()` classes only, no DB decorators
- `dto/` — `@InputType()` and `@ArgsType()` classes for mutations and queries
- `entities/` — TypeORM `@Entity()` classes only, no GQL decorators
- `loaders/` — `Scope.REQUEST` DataLoaders that batch DB reads per request
- `mappers/` — pure functions converting entity → GQL type
- `policies/` — CASL ability rules (who can do what to this resource)
- `subscribers/` — PubSub event payload types for `@Subscription`
- `__tests__/` — co-located unit tests for every class in the module

### Why separate `models/` from `entities/`?

The TypeORM entity carries only database columns. The `@ObjectType()` class in `models/` carries only GraphQL fields. A `mapper/` converts between the two.

- A schema change never requires a DB migration.
- A DB refactor never breaks the GQL contract.
- Sensitive DB columns (e.g. `passwordHash`) are never accidentally exposed via `@Field()`.

---

## Infrastructure Layer — `libs/`

All `libs/` modules are imported once in `AppModule` and provided globally — feature modules never set up their own DB connections, Redis clients, or auth guards directly. The full file breakdown is in the complete tree above. Key responsibilities:

| Lib          | Technology                 | What it provides                                                |
| ------------ | -------------------------- | --------------------------------------------------------------- |
| `database/`  | TypeORM + PostgreSQL       | Connection, base entity, migrations, seeds, transactions        |
| `cache/`     | Redis + ioredis            | `get<T>()`, `set()`, `del()`, `invalidatePattern()`             |
| `queue/`     | BullMQ + Redis             | Job queues, retry logic, dead-letter queue, base processor      |
| `logger/`    | Pino                       | Structured JSON logs, GQL operation logging plugin              |
| `auth/`      | PassportJS + JWT + CASL    | JWT strategy, GQL auth guard, ability factory, `@CurrentUser()` |
| `telemetry/` | OpenTelemetry + Prometheus | Distributed tracing, metrics histograms, Jaeger exporter        |

---

## Cross-Cutting Concerns

These decorators are applied on resolver methods and do not change the file structure.

```typescript
@Resolver(() => User)
@UseGuards(GqlAuthGuard)                        // JWT authentication
export class UserResolver {

  @Query(() => UserConnection)
  @Roles('admin', 'editor')                     // RBAC authorization
  @UseInterceptors(CacheInterceptor)            // Redis response caching
  @Throttle({ default: { limit: 30, ttl: 60000 } })  // Rate limiting
  @Complexity(5)                                // Query complexity budget
  findAll(@Args() args: UsersFilterArgs) { ... }

}
```

### Apollo Plugin Pipeline (registered in `app.module.ts`)

| Order | Plugin                 | Purpose                                               |
| ----- | ---------------------- | ----------------------------------------------------- |
| 1     | `ComplexityPlugin`     | Rejects queries exceeding max complexity score        |
| 2     | `DepthLimitPlugin`     | Rejects deeply nested queries (default: depth 7)      |
| 3     | `DataloaderPlugin`     | Injects per-request DataLoader instances into context |
| 4     | `LoggingPlugin`        | Logs operation name, duration, error status           |
| 5     | `TracingPlugin`        | Creates OpenTelemetry span per operation              |
| 6     | `SentryPlugin`         | Captures resolver errors with full context            |
| 7     | `PersistedQueryPlugin` | APQ — caches query documents by hash                  |

---

## Key File Implementations

### `main.ts`

```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import helmet from 'helmet';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Security headers
  app.use(helmet({
    contentSecurityPolicy: process.env.NODE_ENV === 'production',
  }));

  // CORS
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(','),
    credentials: true,
  });

  // Global validation pipe — strips unknown fields, auto-transforms types
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }));

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

---

### `app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { ThrottlerModule } from '@nestjs/throttler';
import { join } from 'path';
import appConfig from './config/app.config';
import graphqlConfig from './config/graphql.config';
import { DatabaseModule } from '../libs/database/database.module';
import { AuthModule } from '../libs/auth/auth.module';
import { LoggerModule } from '../libs/logger/logger.module';
import { TelemetryModule } from '../libs/telemetry/telemetry.module';
import { CacheModule } from '../libs/cache/cache.module';
import { HealthModule } from './health/health.module';
import { UsersModule } from './modules/users/user.module';
import { PostsModule } from './modules/posts/post.module';
import { ComplexityPlugin } from './common/plugins/complexity.plugin';
import { LoggingPlugin } from './common/plugins/logging.plugin';
import { TracingPlugin } from './common/plugins/tracing.plugin';
import { SentryPlugin } from './common/plugins/sentry.plugin';

@Module({
  imports: [
    // Config — loaded globally, available everywhere via ConfigService
    ConfigModule.forRoot({
      isGlobal: true,
      load: [appConfig, graphqlConfig],
      envFilePath: [`.env.${process.env.NODE_ENV}`, '.env'],
    }),

    // GraphQL — code-first, auto-generates schema.gql on startup
    GraphQLModule.forRootAsync<ApolloDriverConfig>({
      driver: ApolloDriver,
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        autoSchemaFile: join(process.cwd(), 'schema.gql'),
        sortSchema: true,
        playground: config.get('app.env') !== 'production',
        introspection: config.get('app.env') !== 'production',
        subscriptions: { 'graphql-ws': true },
        context: ({ req, res }) => ({ req, res }),
        plugins: [
          new ComplexityPlugin(config.get('graphql.maxComplexity')),
          new LoggingPlugin(),
          new TracingPlugin(),
          new SentryPlugin(),
        ],
        buildSchemaOptions: {
          dateScalarMode: 'timestamp',
          numberScalarMode: 'integer',
        },
        validationRules: [depthLimit(config.get('graphql.depthLimit'))],
        persistedQueries: { cache: new InMemoryLRUCache() },
      }),
    }),

    // Rate limiting
    ThrottlerModule.forRoot([{ ttl: 60000, limit: 100 }]),

    // Infrastructure
    DatabaseModule,
    AuthModule,
    CacheModule,
    LoggerModule,
    TelemetryModule,
    HealthModule,

    // Feature modules
    UsersModule,
    PostsModule,
  ],
})
export class AppModule {}
```

---

### `graphql.config.ts`

```typescript
import { registerAs } from '@nestjs/config';
import depthLimit from 'graphql-depth-limit';

export default registerAs('graphql', () => ({
  depthLimit: parseInt(process.env.GQL_DEPTH_LIMIT ?? '7', 10),
  maxComplexity: parseInt(process.env.GQL_MAX_COMPLEXITY ?? '200', 10),
}));
```

---

### `models/user.type.ts`

```typescript
import { ObjectType, Field, ID, HideField } from '@nestjs/graphql';
import { Post } from '../posts/models/post.type';

@ObjectType({ description: 'A registered user account' })
export class User {
  @Field(() => ID, { description: 'UUID primary key' })
  id: string;

  @Field({ description: 'Unique email address' })
  email: string;

  @Field({ description: 'Display name' })
  name: string;

  @Field(() => String, { nullable: true, description: 'Avatar URL' })
  avatarUrl?: string;

  @Field({ description: 'Account creation timestamp' })
  createdAt: Date;

  @Field({ description: 'Last update timestamp' })
  updatedAt: Date;

  @HideField()  // Never exposed in GraphQL — only used internally
  passwordHash: string;

  @Field(() => [Post], { nullable: 'itemsAndList' })
  posts?: Post[];  // Resolved via PostLoader (DataLoader)
}
```

---

### `dto/create-user.input.ts`

```typescript
import { InputType, Field } from '@nestjs/graphql';
import { IsEmail, MinLength, IsOptional, IsUrl, MaxLength } from 'class-validator';

@InputType({ description: 'Input for creating a new user' })
export class CreateUserInput {
  @Field({ description: 'Must be a valid, unique email address' })
  @IsEmail()
  email: string;

  @Field({ description: 'Display name (2–80 characters)' })
  @MinLength(2)
  @MaxLength(80)
  name: string;

  @Field(() => String, { nullable: true, description: 'Public avatar URL' })
  @IsOptional()
  @IsUrl()
  avatarUrl?: string;
}
```

---

### `dto/users-filter.args.ts`

```typescript
import { ArgsType, Field, Int } from '@nestjs/graphql';
import { IsOptional, Min, Max } from 'class-validator';
import { UserOrderField } from '../models/user-order.enum';
import { SortDirection } from '@common/enums/sort-direction.enum';

@ArgsType()
export class UsersFilterArgs {
  @Field(() => String, { nullable: true })
  @IsOptional()
  search?: string;

  @Field(() => String, { nullable: true, description: 'Relay cursor' })
  @IsOptional()
  after?: string;

  @Field(() => Int, { defaultValue: 20 })
  @Min(1)
  @Max(100)
  first: number = 20;

  @Field(() => UserOrderField, { defaultValue: UserOrderField.CREATED_AT })
  orderBy: UserOrderField = UserOrderField.CREATED_AT;

  @Field(() => SortDirection, { defaultValue: SortDirection.DESC })
  direction: SortDirection = SortDirection.DESC;
}
```

---

### `user.resolver.ts`

```typescript
import {
  Resolver, Query, Mutation, Args, ResolveField,
  Parent, ID, Subscription,
} from '@nestjs/graphql';
import { UseGuards, UseInterceptors } from '@nestjs/common';
import { CacheInterceptor } from '@nestjs/cache-manager';
import { Throttle } from '@nestjs/throttler';
import { GqlAuthGuard } from '@libs/auth/gql-auth.guard';
import { CurrentUser } from '@libs/auth/current-user.decorator';
import { Roles } from '@common/decorators/roles.decorator';
import { Complexity } from '@common/decorators/complexity.decorator';
import { User } from './models/user.type';
import { UserConnection } from './models/user-connection.type';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';
import { UsersFilterArgs } from './dto/users-filter.args';
import { UsersService } from './user.service';
import { PostLoader } from '../posts/loaders/post.loader';
import { Post } from '../posts/models/post.type';
import { PubSub } from 'graphql-subscriptions';

const pubSub = new PubSub();

@Resolver(() => User)
@UseGuards(GqlAuthGuard)
export class UserResolver {
  constructor(
    private readonly usersService: UsersService,
    private readonly postLoader: PostLoader,
  ) {}

  // ─── Queries ───────────────────────────────────────────────────────────────

  @Query(() => UserConnection, { name: 'users', description: 'Paginated user list' })
  @UseInterceptors(CacheInterceptor)
  @Throttle({ default: { limit: 30, ttl: 60000 } })
  @Complexity(3)
  findAll(@Args() args: UsersFilterArgs): Promise<UserConnection> {
    return this.usersService.findAll(args);
  }

  @Query(() => User, { nullable: true, name: 'user' })
  @Complexity(1)
  findOne(
    @Args('id', { type: () => ID }) id: string,
  ): Promise<User | null> {
    return this.usersService.findOne(id);
  }

  // ─── Mutations ─────────────────────────────────────────────────────────────

  @Mutation(() => User, { description: 'Create a new user (admin only)' })
  @Roles('admin')
  createUser(
    @Args('input') input: CreateUserInput,
    @CurrentUser() actor: User,
  ): Promise<User> {
    return this.usersService.create(input, actor);
  }

  @Mutation(() => User)
  updateUser(
    @Args('id', { type: () => ID }) id: string,
    @Args('input') input: UpdateUserInput,
    @CurrentUser() actor: User,
  ): Promise<User> {
    return this.usersService.update(id, input, actor);
  }

  @Mutation(() => Boolean)
  @Roles('admin')
  deleteUser(
    @Args('id', { type: () => ID }) id: string,
    @CurrentUser() actor: User,
  ): Promise<boolean> {
    return this.usersService.softDelete(id, actor);
  }

  // ─── Resolved Fields ───────────────────────────────────────────────────────

  @ResolveField('posts', () => [Post])
  getPosts(@Parent() user: User): Promise<Post[]> {
    // DataLoader batches ALL post lookups in a single DB query per request
    return this.postLoader.load(user.id);
  }

  // ─── Subscriptions ─────────────────────────────────────────────────────────

  @Subscription(() => User, {
    filter: (payload, variables) =>
      payload.userCreated.id !== variables.excludeId,
  })
  userCreated(
    @Args('excludeId', { type: () => ID, nullable: true }) _excludeId?: string,
  ) {
    return pubSub.asyncIterator('USER_CREATED');
  }
}
```

---

### `user.service.ts`

```typescript
import { Injectable, NotFoundException, ForbiddenException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { UserRepository } from './entities/user.repository';
import { UserMapper } from './mappers/user.mapper';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';
import { UsersFilterArgs } from './dto/users-filter.args';
import { User } from './models/user.type';
import { UserConnection } from './models/user-connection.type';
import { CaslAbilityFactory } from '@libs/auth/casl-ability.factory';
import { Action } from '@common/enums/action.enum';

@Injectable()
export class UsersService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly mapper: UserMapper,
    private readonly casl: CaslAbilityFactory,
    private readonly events: EventEmitter2,
  ) {}

  async findAll(args: UsersFilterArgs): Promise<UserConnection> {
    const [entities, total] = await this.userRepo.findPaginated(args);
    return this.mapper.toConnection(entities, total, args);
  }

  async findOne(id: string): Promise<User | null> {
    const entity = await this.userRepo.findOneBy({ id });
    return entity ? this.mapper.toGql(entity) : null;
  }

  async create(input: CreateUserInput, actor: User): Promise<User> {
    const ability = this.casl.createForUser(actor);
    if (ability.cannot(Action.Create, 'User')) {
      throw new ForbiddenException('Insufficient permissions');
    }

    const entity = this.userRepo.create({
      ...input,
      passwordHash: await this.hashPassword(input.password),
    });

    const saved = await this.userRepo.save(entity);
    this.events.emit('user.created', { user: saved });

    return this.mapper.toGql(saved);
  }

  async update(id: string, input: UpdateUserInput, actor: User): Promise<User> {
    const entity = await this.userRepo.findOneOrFail({ where: { id } });
    const ability = this.casl.createForUser(actor);

    if (ability.cannot(Action.Update, entity)) {
      throw new ForbiddenException('Cannot update this user');
    }

    Object.assign(entity, input);
    const saved = await this.userRepo.save(entity);
    return this.mapper.toGql(saved);
  }

  async softDelete(id: string, actor: User): Promise<boolean> {
    const entity = await this.userRepo.findOneOrFail({ where: { id } });
    const ability = this.casl.createForUser(actor);

    if (ability.cannot(Action.Delete, entity)) {
      throw new ForbiddenException('Cannot delete this user');
    }

    await this.userRepo.softDelete(id);
    return true;
  }

  private async hashPassword(password: string): Promise<string> {
    const bcrypt = await import('bcrypt');
    return bcrypt.hash(password, 12);
  }
}
```

---

### `user.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserEntity } from './entities/user.entity';
import { UserRepository } from './entities/user.repository';
import { UserResolver } from './user.resolver';
import { UsersService } from './user.service';
import { UserMapper } from './mappers/user.mapper';
import { UserLoader } from './loaders/user.loader';
import { PostsModule } from '../posts/post.module';

@Module({
  imports: [
    TypeOrmModule.forFeature([UserEntity]),
    PostsModule,  // exposes PostLoader
  ],
  providers: [
    UserResolver,
    UsersService,
    UserRepository,
    UserMapper,
    UserLoader,
  ],
  exports: [UsersService, UserLoader],
})
export class UsersModule {}
```

---

### `entities/user.entity.ts`

```typescript
import { Entity, Column, Index } from 'typeorm';
import { BaseEntity } from '@libs/database/base.entity';

@Entity('users')
export class UserEntity extends BaseEntity {
  @Column({ unique: true })
  @Index()
  email: string;

  @Column()
  name: string;

  @Column({ name: 'avatar_url', nullable: true })
  avatarUrl?: string;

  @Column({ name: 'password_hash', select: false })
  passwordHash: string;

  @Column({ default: 'user' })
  role: string;
}
```

---

### `entities/user.repository.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { DataSource, Repository } from 'typeorm';
import { UserEntity } from './user.entity';
import { UsersFilterArgs } from '../dto/users-filter.args';

@Injectable()
export class UserRepository extends Repository<UserEntity> {
  constructor(private dataSource: DataSource) {
    super(UserEntity, dataSource.createEntityManager());
  }

  async findPaginated(args: UsersFilterArgs): Promise<[UserEntity[], number]> {
    const qb = this.createQueryBuilder('user')
      .where('user.deletedAt IS NULL')
      .orderBy(`user.${args.orderBy}`, args.direction)
      .take(args.first);

    if (args.search) {
      qb.andWhere(
        '(user.name ILIKE :search OR user.email ILIKE :search)',
        { search: `%${args.search}%` },
      );
    }

    if (args.after) {
      const cursor = Buffer.from(args.after, 'base64').toString('utf8');
      qb.andWhere('user.createdAt < :cursor', { cursor });
    }

    return qb.getManyAndCount();
  }
}
```

---

### `loaders/user.loader.ts`

```typescript
import { Injectable, Scope } from '@nestjs/common';
import DataLoader from 'dataloader';
import { UserRepository } from '../entities/user.repository';
import { UserMapper } from '../mappers/user.mapper';
import { User } from '../models/user.type';

@Injectable({ scope: Scope.REQUEST })  // Critical: new instance per GQL request
export class UserLoader {
  private readonly loader: DataLoader<string, User>;

  constructor(
    private readonly userRepo: UserRepository,
    private readonly mapper: UserMapper,
  ) {
    this.loader = new DataLoader<string, User>(async (ids) => {
      const users = await this.userRepo.findByIds([...ids]);
      const userMap = new Map(users.map(u => [u.id, u]));

      return ids.map(id => {
        const entity = userMap.get(id);
        if (!entity) return new Error(`User with id ${id} not found`);
        return this.mapper.toGql(entity);
      });
    });
  }

  load(id: string): Promise<User> {
    return this.loader.load(id);
  }

  loadMany(ids: string[]): Promise<(User | Error)[]> {
    return this.loader.loadMany(ids);
  }

  clear(id: string): this {
    this.loader.clear(id);
    return this;
  }
}
```

> **Why `Scope.REQUEST`?** DataLoader caches results within a single request batch. If the loader were a singleton, stale data from one request could leak into another. `Scope.REQUEST` guarantees a fresh loader — and a fresh cache — for every incoming GraphQL operation.

---

### `mappers/user.mapper.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { UserEntity } from '../entities/user.entity';
import { User } from '../models/user.type';
import { UserConnection } from '../models/user-connection.type';
import { UsersFilterArgs } from '../dto/users-filter.args';

@Injectable()
export class UserMapper {
  toGql(entity: UserEntity): User {
    const user = new User();
    user.id = entity.id;
    user.email = entity.email;
    user.name = entity.name;
    user.avatarUrl = entity.avatarUrl;
    user.createdAt = entity.createdAt;
    user.updatedAt = entity.updatedAt;
    // passwordHash intentionally omitted
    return user;
  }

  toConnection(
    entities: UserEntity[],
    total: number,
    args: UsersFilterArgs,
  ): UserConnection {
    const edges = entities.map(entity => ({
      cursor: Buffer.from(entity.createdAt.toISOString()).toString('base64'),
      node: this.toGql(entity),
    }));

    return {
      edges,
      totalCount: total,
      pageInfo: {
        hasNextPage: entities.length === args.first,
        hasPreviousPage: !!args.after,
        startCursor: edges[0]?.cursor ?? null,
        endCursor: edges[edges.length - 1]?.cursor ?? null,
      },
    };
  }
}
```

---

### `policies/manage-user.policy.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { AbilityBuilder, createMongoAbility } from '@casl/ability';
import { UserEntity } from '../entities/user.entity';
import { Action } from '@common/enums/action.enum';

@Injectable()
export class ManageUserPolicy {
  defineFor(actor: UserEntity) {
    const { can, cannot, build } = new AbilityBuilder(createMongoAbility);

    if (actor.role === 'admin') {
      can(Action.Manage, 'User');  // admin can do everything
    } else {
      can(Action.Read, 'User');                         // anyone can read
      can(Action.Update, 'User', { id: actor.id });     // own record only
      cannot(Action.Delete, 'User');                    // never self-delete
    }

    return build();
  }
}
```

---

## Apollo Plugin Pipeline

```typescript
// src/common/plugins/complexity.plugin.ts
import { Plugin } from '@nestjs/apollo';
import { GraphQLSchemaHost } from '@nestjs/graphql';
import { ApolloServerPlugin, GraphQLRequestListener } from '@apollo/server';
import { fieldExtensionsEstimator, simpleEstimator, getComplexity } from 'graphql-query-complexity';

@Plugin()
export class ComplexityPlugin implements ApolloServerPlugin {
  constructor(
    private readonly gqlSchemaHost: GraphQLSchemaHost,
    private readonly maxComplexity: number = 200,
  ) {}

  async requestDidStart(): Promise<GraphQLRequestListener<any>> {
    const { schema } = this.gqlSchemaHost;
    const maxComplexity = this.maxComplexity;

    return {
      async didResolveOperation({ request, document }) {
        const complexity = getComplexity({
          schema,
          operationName: request.operationName,
          query: document,
          variables: request.variables,
          estimators: [
            fieldExtensionsEstimator(),
            simpleEstimator({ defaultComplexity: 1 }),
          ],
        });

        if (complexity > maxComplexity) {
          throw new Error(
            `Query complexity ${complexity} exceeds maximum allowed ${maxComplexity}`,
          );
        }
      },
    };
  }
}
```

---

## Enterprise Patterns & Decisions

### Relay-style cursor pagination

All list queries return a `Connection` type rather than a simple array. This enables infinite-scroll clients and is the GraphQL community standard for large datasets.

```typescript
@ObjectType()
export class UserConnection {
  @Field(() => [UserEdge])
  edges: UserEdge[];

  @Field(() => PageInfo)
  pageInfo: PageInfo;

  @Field(() => Int)
  totalCount: number;
}

@ObjectType()
export class UserEdge {
  @Field(() => String)
  cursor: string;

  @Field(() => User)
  node: User;
}
```

### N+1 prevention with DataLoader

Without DataLoader, resolving `posts` on a list of 100 users fires 100 individual DB queries. With DataLoader, all 100 IDs are batched into a single `WHERE id IN (...)` query per request.

Every `@ResolveField` that loads a related entity must go through a loader, never call a service directly.

### Soft deletes

`BaseEntity` includes a `deletedAt` column. All repositories add `.where('deletedAt IS NULL')` by default. Records are never physically removed — this preserves audit trails and enables recovery.

```typescript
// libs/database/base.entity.ts
import { PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn, DeleteDateColumn } from 'typeorm';

export abstract class BaseEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @DeleteDateColumn()
  deletedAt?: Date;
}
```

### Path aliases

Configure `tsconfig.paths.json` to avoid `../../../` chains:

```json
{
  "compilerOptions": {
    "paths": {
      "@libs/*": ["libs/*"],
      "@common/*": ["src/common/*"],
      "@modules/*": ["src/modules/*"],
      "@config/*": ["src/config/*"]
    }
  }
}
```

And in `jest.config.ts`:

```typescript
moduleNameMapper: {
  '^@libs/(.*)$': '<rootDir>/libs/$1',
  '^@common/(.*)$': '<rootDir>/src/common/$1',
  '^@modules/(.*)$': '<rootDir>/src/modules/$1',
}
```

---

## Environment & Configuration

```bash
# .env
NODE_ENV=development
PORT=3000

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
DB_USER=postgres
DB_PASS=secret

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Auth
JWT_SECRET=change-me-in-production
JWT_EXPIRES_IN=7d

# GraphQL
GQL_DEPTH_LIMIT=7
GQL_MAX_COMPLEXITY=200

# CORS
ALLOWED_ORIGINS=http://localhost:4200,https://myapp.com

# Telemetry
OTEL_EXPORTER_JAEGER_ENDPOINT=http://localhost:14268/api/traces
SENTRY_DSN=https://xxx@sentry.io/yyy
```

---

## Testing Strategy

Each feature module owns its unit tests inside `__tests__/`. Shared E2E tests live in `test/e2e/`. The rule is simple: **unit tests mock everything at the boundary, E2E tests mock nothing except external services** (Stripe, SendGrid, etc.).

### Test file layout (full)

```
src/modules/users/__tests__/
├── user.resolver.spec.ts       # Resolver unit test — mocks UsersService
├── user.service.spec.ts        # Service unit test — mocks UserRepository
├── user.repository.spec.ts     # Repository unit test — uses in-memory SQLite
├── user.loader.spec.ts         # DataLoader unit test — verifies batching
└── user.mapper.spec.ts         # Pure function test — entity → GQL type

src/modules/posts/__tests__/
├── post.resolver.spec.ts
├── post.service.spec.ts
├── post.repository.spec.ts
├── post.loader.spec.ts
└── post.mapper.spec.ts

src/modules/auth/__tests__/
├── auth.resolver.spec.ts
├── auth.service.spec.ts
└── jwt.strategy.spec.ts

src/libs/cache/__tests__/
└── cache.service.spec.ts

src/libs/auth/__tests__/
├── gql-auth.guard.spec.ts
├── roles.guard.spec.ts
└── casl-ability.factory.spec.ts

test/e2e/
├── users.e2e-spec.ts
├── posts.e2e-spec.ts
├── auth.e2e-spec.ts
└── health.e2e-spec.ts

test/
├── helpers/
│   ├── gql-request.helper.ts   # Typed GQL request wrapper
│   └── auth.helper.ts          # Gets a JWT for a given role
├── fixtures/
│   ├── user.fixture.ts
│   └── post.fixture.ts
└── factories/
    ├── user.factory.ts          # Faker-based entity builder
    └── post.factory.ts
```

---

### Shared test factory (`test/factories/user.factory.ts`)

```typescript
import { faker } from '@faker-js/faker';
import { UserEntity } from '@modules/users/entities/user.entity';

export function buildUserEntity(overrides: Partial<UserEntity> = {}): UserEntity {
  const entity = new UserEntity();
  entity.id = faker.string.uuid();
  entity.email = faker.internet.email();
  entity.name = faker.person.fullName();
  entity.avatarUrl = faker.image.avatar();
  entity.role = 'user';
  entity.passwordHash = '$2b$12$hashedpassword';
  entity.createdAt = faker.date.past();
  entity.updatedAt = new Date();
  return Object.assign(entity, overrides);
}

export function buildUserEntities(count: number, overrides: Partial<UserEntity> = {}): UserEntity[] {
  return Array.from({ length: count }, () => buildUserEntity(overrides));
}
```

---

### Shared GQL helper (`test/helpers/gql-request.helper.ts`)

```typescript
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';

export async function gqlRequest(
  app: INestApplication,
  query: string,
  variables: Record<string, unknown> = {},
  token?: string,
) {
  const req = request(app.getHttpServer())
    .post('/graphql')
    .send({ query, variables });

  if (token) req.set('Authorization', `Bearer ${token}`);

  const res = await req.expect(200);
  return res.body as { data?: Record<string, unknown>; errors?: { message: string }[] };
}
```

---

### Auth helper (`test/helpers/auth.helper.ts`)

```typescript
import { INestApplication } from '@nestjs/common';
import { gqlRequest } from './gql-request.helper';

export async function getAuthToken(
  app: INestApplication,
  email: string,
  password: string,
): Promise<string> {
  const { data } = await gqlRequest(app, `
    mutation Login($email: String!, $password: String!) {
      login(input: { email: $email, password: $password }) {
        accessToken
      }
    }
  `, { email, password });

  return data!.login.accessToken as string;
}
```

---

### Module: Users

#### `user.resolver.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { createMock } from '@golevelup/ts-jest';
import { UserResolver } from '../user.resolver';
import { UsersService } from '../user.service';
import { PostLoader } from '../../posts/loaders/post.loader';
import { buildUserEntity } from '@test/factories/user.factory';
import { UserMapper } from '../mappers/user.mapper';

describe('UserResolver', () => {
  let resolver: UserResolver;
  let usersService: jest.Mocked<UsersService>;
  let postLoader: jest.Mocked<PostLoader>;
  const mapper = new UserMapper();

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserResolver,
        { provide: UsersService, useValue: createMock<UsersService>() },
        { provide: PostLoader, useValue: createMock<PostLoader>() },
      ],
    }).compile();

    resolver = module.get(UserResolver);
    usersService = module.get(UsersService);
    postLoader = module.get(PostLoader);
  });

  describe('findOne', () => {
    it('returns a user when found', async () => {
      const user = mapper.toGql(buildUserEntity());
      usersService.findOne.mockResolvedValue(user);

      const result = await resolver.findOne(user.id);
      expect(result).toEqual(user);
      expect(usersService.findOne).toHaveBeenCalledWith(user.id);
    });

    it('returns null when user does not exist', async () => {
      usersService.findOne.mockResolvedValue(null);
      const result = await resolver.findOne('non-existent-id');
      expect(result).toBeNull();
    });
  });

  describe('createUser', () => {
    it('delegates to usersService.create with actor', async () => {
      const actor = mapper.toGql(buildUserEntity({ role: 'admin' }));
      const input = { email: 'new@test.com', name: 'New User' };
      const created = mapper.toGql(buildUserEntity({ email: input.email }));

      usersService.create.mockResolvedValue(created);

      const result = await resolver.createUser(input as any, actor);
      expect(result).toEqual(created);
      expect(usersService.create).toHaveBeenCalledWith(input, actor);
    });
  });

  describe('getPosts (ResolveField)', () => {
    it('calls postLoader with user id', async () => {
      const user = mapper.toGql(buildUserEntity());
      postLoader.load.mockResolvedValue([]);

      await resolver.getPosts(user);
      expect(postLoader.load).toHaveBeenCalledWith(user.id);
    });
  });
});
```

---

#### `user.service.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { ForbiddenException, NotFoundException } from '@nestjs/common';
import { createMock } from '@golevelup/ts-jest';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { UsersService } from '../user.service';
import { UserRepository } from '../entities/user.repository';
import { UserMapper } from '../mappers/user.mapper';
import { CaslAbilityFactory } from '@libs/auth/casl-ability.factory';
import { buildUserEntity } from '@test/factories/user.factory';

describe('UsersService', () => {
  let service: UsersService;
  let userRepo: jest.Mocked<UserRepository>;
  let casl: jest.Mocked<CaslAbilityFactory>;
  let events: jest.Mocked<EventEmitter2>;
  const mapper = new UserMapper();

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: UserRepository, useValue: createMock<UserRepository>() },
        { provide: UserMapper, useValue: mapper },
        { provide: CaslAbilityFactory, useValue: createMock<CaslAbilityFactory>() },
        { provide: EventEmitter2, useValue: createMock<EventEmitter2>() },
      ],
    }).compile();

    service = module.get(UsersService);
    userRepo = module.get(UserRepository);
    casl = module.get(CaslAbilityFactory);
    events = module.get(EventEmitter2);
  });

  describe('findOne', () => {
    it('maps entity to GQL type when found', async () => {
      const entity = buildUserEntity();
      userRepo.findOneBy.mockResolvedValue(entity);

      const result = await service.findOne(entity.id);
      expect(result?.id).toBe(entity.id);
      expect(result?.email).toBe(entity.email);
    });

    it('returns null when entity not found', async () => {
      userRepo.findOneBy.mockResolvedValue(null);
      const result = await service.findOne('missing-id');
      expect(result).toBeNull();
    });
  });

  describe('create', () => {
    const input = { email: 'a@b.com', name: 'Alice', password: 'secret123' };

    it('creates user and emits event when actor is admin', async () => {
      const adminEntity = buildUserEntity({ role: 'admin' });
      const actor = mapper.toGql(adminEntity);
      const savedEntity = buildUserEntity({ email: input.email });

      casl.createForUser.mockReturnValue({ cannot: () => false } as any);
      userRepo.create.mockReturnValue(savedEntity);
      userRepo.save.mockResolvedValue(savedEntity);

      const result = await service.create(input as any, actor);

      expect(result.email).toBe(savedEntity.email);
      expect(events.emit).toHaveBeenCalledWith('user.created', { user: savedEntity });
    });

    it('throws ForbiddenException when actor lacks permission', async () => {
      const actor = mapper.toGql(buildUserEntity({ role: 'user' }));
      casl.createForUser.mockReturnValue({ cannot: () => true } as any);

      await expect(service.create(input as any, actor)).rejects.toThrow(ForbiddenException);
      expect(userRepo.save).not.toHaveBeenCalled();
    });
  });

  describe('softDelete', () => {
    it('soft-deletes and returns true for admin', async () => {
      const entity = buildUserEntity();
      const actor = mapper.toGql(buildUserEntity({ role: 'admin' }));

      userRepo.findOneOrFail.mockResolvedValue(entity);
      casl.createForUser.mockReturnValue({ cannot: () => false } as any);
      userRepo.softDelete.mockResolvedValue(undefined as any);

      const result = await service.softDelete(entity.id, actor);
      expect(result).toBe(true);
      expect(userRepo.softDelete).toHaveBeenCalledWith(entity.id);
    });

    it('throws ForbiddenException when actor cannot delete', async () => {
      const entity = buildUserEntity();
      const actor = mapper.toGql(buildUserEntity({ role: 'user' }));

      userRepo.findOneOrFail.mockResolvedValue(entity);
      casl.createForUser.mockReturnValue({ cannot: () => true } as any);

      await expect(service.softDelete(entity.id, actor)).rejects.toThrow(ForbiddenException);
    });
  });
});
```

---

#### `user.repository.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { TypeOrmModule, getRepositoryToken } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';
import { UserRepository } from '../entities/user.repository';
import { UserEntity } from '../entities/user.entity';
import { buildUserEntity } from '@test/factories/user.factory';

describe('UserRepository', () => {
  let repo: UserRepository;
  let dataSource: DataSource;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [
        TypeOrmModule.forRoot({
          type: 'sqlite',
          database: ':memory:',
          entities: [UserEntity],
          synchronize: true,
        }),
        TypeOrmModule.forFeature([UserEntity]),
      ],
      providers: [UserRepository],
    }).compile();

    repo = module.get(UserRepository);
    dataSource = module.get(DataSource);
  });

  afterEach(async () => {
    await dataSource.getRepository(UserEntity).clear();
  });

  afterAll(async () => {
    await dataSource.destroy();
  });

  it('findPaginated returns results ordered by createdAt DESC', async () => {
    const entities = buildUserEntity({ name: 'Alice' });
    await repo.save(entities);

    const [results, count] = await repo.findPaginated({
      first: 10,
      orderBy: 'createdAt',
      direction: 'DESC',
    } as any);

    expect(count).toBeGreaterThanOrEqual(1);
    expect(results[0].name).toBe('Alice');
  });

  it('findPaginated filters by search term', async () => {
    await repo.save(buildUserEntity({ name: 'Searchable Bob', email: 'bob@test.com' }));
    await repo.save(buildUserEntity({ name: 'Other Person', email: 'other@test.com' }));

    const [results] = await repo.findPaginated({
      search: 'Searchable',
      first: 10,
      orderBy: 'createdAt',
      direction: 'DESC',
    } as any);

    expect(results).toHaveLength(1);
    expect(results[0].name).toBe('Searchable Bob');
  });

  it('does not return soft-deleted records', async () => {
    const entity = await repo.save(buildUserEntity());
    await repo.softDelete(entity.id);

    const [results] = await repo.findPaginated({
      first: 10,
      orderBy: 'createdAt',
      direction: 'DESC',
    } as any);

    expect(results.find(u => u.id === entity.id)).toBeUndefined();
  });
});
```

---

#### `user.loader.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { createMock } from '@golevelup/ts-jest';
import { UserLoader } from '../loaders/user.loader';
import { UserRepository } from '../entities/user.repository';
import { UserMapper } from '../mappers/user.mapper';
import { buildUserEntity } from '@test/factories/user.factory';

describe('UserLoader', () => {
  let loader: UserLoader;
  let userRepo: jest.Mocked<UserRepository>;
  const mapper = new UserMapper();

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UserLoader,
        { provide: UserRepository, useValue: createMock<UserRepository>() },
        { provide: UserMapper, useValue: mapper },
      ],
    }).compile();

    loader = await module.resolve(UserLoader);  // resolve() for REQUEST scope
    userRepo = module.get(UserRepository);
  });

  it('batches multiple load() calls into a single DB query', async () => {
    const entities = [buildUserEntity(), buildUserEntity()];
    userRepo.findByIds.mockResolvedValue(entities);

    // Two concurrent loads — should result in ONE DB call
    const [result1, result2] = await Promise.all([
      loader.load(entities[0].id),
      loader.load(entities[1].id),
    ]);

    expect(userRepo.findByIds).toHaveBeenCalledTimes(1);
    expect(result1.id).toBe(entities[0].id);
    expect(result2.id).toBe(entities[1].id);
  });

  it('returns an Error object for missing IDs', async () => {
    userRepo.findByIds.mockResolvedValue([]);
    const results = await loader.loadMany(['missing-id']);
    expect(results[0]).toBeInstanceOf(Error);
  });
});
```

---

#### `user.mapper.spec.ts`

```typescript
import { UserMapper } from '../mappers/user.mapper';
import { buildUserEntity } from '@test/factories/user.factory';

describe('UserMapper', () => {
  const mapper = new UserMapper();

  describe('toGql', () => {
    it('maps all expected fields', () => {
      const entity = buildUserEntity();
      const gql = mapper.toGql(entity);

      expect(gql.id).toBe(entity.id);
      expect(gql.email).toBe(entity.email);
      expect(gql.name).toBe(entity.name);
      expect(gql.avatarUrl).toBe(entity.avatarUrl);
      expect(gql.createdAt).toBe(entity.createdAt);
    });

    it('does NOT expose passwordHash', () => {
      const entity = buildUserEntity({ passwordHash: 'secret-hash' });
      const gql = mapper.toGql(entity);
      expect((gql as any).passwordHash).toBeUndefined();
    });
  });

  describe('toConnection', () => {
    it('produces correct pageInfo for first page', () => {
      const entities = [buildUserEntity(), buildUserEntity()];
      const connection = mapper.toConnection(entities, 50, { first: 2 } as any);

      expect(connection.totalCount).toBe(50);
      expect(connection.edges).toHaveLength(2);
      expect(connection.pageInfo.hasNextPage).toBe(true);
      expect(connection.pageInfo.hasPreviousPage).toBe(false);
    });

    it('sets hasNextPage false when results < first', () => {
      const entities = [buildUserEntity()];
      const connection = mapper.toConnection(entities, 1, { first: 20 } as any);
      expect(connection.pageInfo.hasNextPage).toBe(false);
    });
  });
});
```

---

### Module: Posts

#### `post.resolver.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { createMock } from '@golevelup/ts-jest';
import { PostResolver } from '../post.resolver';
import { PostsService } from '../post.service';
import { UserLoader } from '../../users/loaders/user.loader';
import { buildPostEntity } from '@test/factories/post.factory';
import { PostMapper } from '../mappers/post.mapper';

describe('PostResolver', () => {
  let resolver: PostResolver;
  let postsService: jest.Mocked<PostsService>;
  let userLoader: jest.Mocked<UserLoader>;
  const mapper = new PostMapper();

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        PostResolver,
        { provide: PostsService, useValue: createMock<PostsService>() },
        { provide: UserLoader, useValue: createMock<UserLoader>() },
      ],
    }).compile();

    resolver = module.get(PostResolver);
    postsService = module.get(PostsService);
    userLoader = module.get(UserLoader);
  });

  it('findOne returns null when post does not exist', async () => {
    postsService.findOne.mockResolvedValue(null);
    expect(await resolver.findOne('missing')).toBeNull();
  });

  it('getAuthor delegates to UserLoader with authorId', async () => {
    const entity = buildPostEntity();
    const post = mapper.toGql(entity);
    userLoader.load.mockResolvedValue({ id: entity.authorId } as any);

    await resolver.getAuthor(post);
    expect(userLoader.load).toHaveBeenCalledWith(entity.authorId);
  });

  it('publishPost calls postsService.publish', async () => {
    const post = mapper.toGql(buildPostEntity());
    postsService.publish.mockResolvedValue(post);

    const result = await resolver.publishPost(post.id, post as any);
    expect(postsService.publish).toHaveBeenCalledWith(post.id, post);
    expect(result).toEqual(post);
  });
});
```

---

#### `post.service.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { ForbiddenException } from '@nestjs/common';
import { createMock } from '@golevelup/ts-jest';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { PostsService } from '../post.service';
import { PostRepository } from '../entities/post.repository';
import { PostMapper } from '../mappers/post.mapper';
import { CaslAbilityFactory } from '@libs/auth/casl-ability.factory';
import { buildPostEntity } from '@test/factories/post.factory';
import { buildUserEntity } from '@test/factories/user.factory';

describe('PostsService', () => {
  let service: PostsService;
  let postRepo: jest.Mocked<PostRepository>;
  let casl: jest.Mocked<CaslAbilityFactory>;
  const mapper = new PostMapper();
  const userMapper = new (require('../../users/mappers/user.mapper').UserMapper)();

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        PostsService,
        { provide: PostRepository, useValue: createMock<PostRepository>() },
        { provide: PostMapper, useValue: mapper },
        { provide: CaslAbilityFactory, useValue: createMock<CaslAbilityFactory>() },
        { provide: EventEmitter2, useValue: createMock<EventEmitter2>() },
      ],
    }).compile();

    service = module.get(PostsService);
    postRepo = module.get(PostRepository);
    casl = module.get(CaslAbilityFactory);
  });

  it('publish sets publishedAt and saves', async () => {
    const entity = buildPostEntity({ publishedAt: undefined });
    const actor = userMapper.toGql(buildUserEntity({ role: 'admin' }));

    postRepo.findOneOrFail.mockResolvedValue(entity);
    casl.createForUser.mockReturnValue({ cannot: () => false } as any);
    postRepo.save.mockResolvedValue({ ...entity, publishedAt: new Date() });

    const result = await service.publish(entity.id, actor);
    expect(result.publishedAt).toBeDefined();
  });

  it('throws ForbiddenException when actor is not author or admin', async () => {
    const entity = buildPostEntity({ authorId: 'different-user-id' });
    const actor = userMapper.toGql(buildUserEntity({ role: 'user' }));

    postRepo.findOneOrFail.mockResolvedValue(entity);
    casl.createForUser.mockReturnValue({ cannot: () => true } as any);

    await expect(service.publish(entity.id, actor)).rejects.toThrow(ForbiddenException);
  });
});
```

---

#### `post.loader.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { createMock } from '@golevelup/ts-jest';
import { PostLoader } from '../loaders/post.loader';
import { PostRepository } from '../entities/post.repository';
import { PostMapper } from '../mappers/post.mapper';
import { buildPostEntity } from '@test/factories/post.factory';

describe('PostLoader', () => {
  let loader: PostLoader;
  let postRepo: jest.Mocked<PostRepository>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        PostLoader,
        { provide: PostRepository, useValue: createMock<PostRepository>() },
        { provide: PostMapper, useValue: new PostMapper() },
      ],
    }).compile();

    loader = await module.resolve(PostLoader);
    postRepo = module.get(PostRepository);
  });

  it('groups posts by authorId in a single batch call', async () => {
    const [uid1, uid2] = ['user-1', 'user-2'];
    const posts = [
      buildPostEntity({ authorId: uid1 }),
      buildPostEntity({ authorId: uid1 }),
      buildPostEntity({ authorId: uid2 }),
    ];
    postRepo.findByAuthorIds.mockResolvedValue(posts);

    const [p1, p2] = await Promise.all([
      loader.load(uid1),
      loader.load(uid2),
    ]);

    expect(postRepo.findByAuthorIds).toHaveBeenCalledTimes(1);
    expect(p1).toHaveLength(2);
    expect(p2).toHaveLength(1);
  });
});
```

---

### Module: Auth

#### `auth.service.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { createMock } from '@golevelup/ts-jest';
import { AuthService } from '../auth.service';
import { UserRepository } from '../../users/entities/user.repository';
import { buildUserEntity } from '@test/factories/user.factory';
import * as bcrypt from 'bcrypt';

describe('AuthService', () => {
  let service: AuthService;
  let userRepo: jest.Mocked<UserRepository>;
  let jwtService: jest.Mocked<JwtService>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        AuthService,
        { provide: UserRepository, useValue: createMock<UserRepository>() },
        { provide: JwtService, useValue: createMock<JwtService>() },
      ],
    }).compile();

    service = module.get(AuthService);
    userRepo = module.get(UserRepository);
    jwtService = module.get(JwtService);
  });

  describe('login', () => {
    it('returns accessToken on valid credentials', async () => {
      const entity = buildUserEntity();
      entity.passwordHash = await bcrypt.hash('correct-password', 10);
      userRepo.findOne.mockResolvedValue(entity);
      jwtService.sign.mockReturnValue('signed-token');

      const result = await service.login(entity.email, 'correct-password');
      expect(result.accessToken).toBe('signed-token');
    });

    it('throws UnauthorizedException on wrong password', async () => {
      const entity = buildUserEntity();
      entity.passwordHash = await bcrypt.hash('real-password', 10);
      userRepo.findOne.mockResolvedValue(entity);

      await expect(service.login(entity.email, 'wrong-password'))
        .rejects.toThrow(UnauthorizedException);
    });

    it('throws UnauthorizedException when user not found', async () => {
      userRepo.findOne.mockResolvedValue(null);
      await expect(service.login('nobody@test.com', 'any')).rejects.toThrow(UnauthorizedException);
    });
  });

  describe('validateToken', () => {
    it('returns user payload on valid JWT', async () => {
      const payload = { sub: 'user-id', email: 'a@b.com', role: 'user' };
      jwtService.verify.mockReturnValue(payload);
      const entity = buildUserEntity({ id: payload.sub });
      userRepo.findOneBy.mockResolvedValue(entity);

      const result = await service.validateToken('valid.jwt.token');
      expect(result?.id).toBe(payload.sub);
    });
  });
});
```

---

#### `jwt.strategy.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { ConfigService } from '@nestjs/config';
import { createMock } from '@golevelup/ts-jest';
import { JwtStrategy } from '../jwt.strategy';
import { UserRepository } from '../../users/entities/user.repository';
import { buildUserEntity } from '@test/factories/user.factory';

describe('JwtStrategy', () => {
  let strategy: JwtStrategy;
  let userRepo: jest.Mocked<UserRepository>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        JwtStrategy,
        { provide: UserRepository, useValue: createMock<UserRepository>() },
        { provide: ConfigService, useValue: { get: () => 'test-secret' } },
      ],
    }).compile();

    strategy = module.get(JwtStrategy);
    userRepo = module.get(UserRepository);
  });

  it('validate() returns user from DB by sub claim', async () => {
    const entity = buildUserEntity();
    userRepo.findOneBy.mockResolvedValue(entity);

    const result = await strategy.validate({ sub: entity.id, email: entity.email });
    expect(result?.id).toBe(entity.id);
  });

  it('returns null when user no longer exists', async () => {
    userRepo.findOneBy.mockResolvedValue(null);
    const result = await strategy.validate({ sub: 'gone', email: 'gone@test.com' });
    expect(result).toBeNull();
  });
});
```

---

#### `gql-auth.guard.spec.ts`

```typescript
import { ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';
import { GqlAuthGuard } from '../gql-auth.guard';
import { AuthGuard } from '@nestjs/passport';

describe('GqlAuthGuard', () => {
  let guard: GqlAuthGuard;

  beforeEach(() => {
    guard = new GqlAuthGuard();
  });

  it('extracts request from GQL context', () => {
    const mockReq = { headers: { authorization: 'Bearer token' } };
    const mockCtx = {
      getContext: () => ({ req: mockReq }),
    } as any;

    jest.spyOn(GqlExecutionContext, 'create').mockReturnValue(mockCtx);

    const result = guard.getRequest({} as ExecutionContext);
    expect(result).toBe(mockReq);
  });
});
```

---

### Lib: Cache

#### `cache.service.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { CacheService } from '../cache.service';

describe('CacheService', () => {
  let service: CacheService;
  let cacheManager: { get: jest.Mock; set: jest.Mock; del: jest.Mock };

  beforeEach(async () => {
    cacheManager = { get: jest.fn(), set: jest.fn(), del: jest.fn() };

    const module = await Test.createTestingModule({
      providers: [
        CacheService,
        { provide: CACHE_MANAGER, useValue: cacheManager },
      ],
    }).compile();

    service = module.get(CacheService);
  });

  it('get() returns parsed value from Redis', async () => {
    cacheManager.get.mockResolvedValue(JSON.stringify({ name: 'Alice' }));
    const result = await service.get<{ name: string }>('key');
    expect(result?.name).toBe('Alice');
  });

  it('get() returns null on cache miss', async () => {
    cacheManager.get.mockResolvedValue(null);
    const result = await service.get('missing-key');
    expect(result).toBeNull();
  });

  it('set() serializes value before storing', async () => {
    await service.set('key', { name: 'Alice' }, 60);
    expect(cacheManager.set).toHaveBeenCalledWith('key', JSON.stringify({ name: 'Alice' }), 60);
  });

  it('del() calls cache manager delete', async () => {
    await service.del('key');
    expect(cacheManager.del).toHaveBeenCalledWith('key');
  });
});
```

---

### E2E Tests

#### `test/e2e/users.e2e-spec.ts`

```typescript
import { INestApplication } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import { AppModule } from '@src/app.module';
import { DataSource } from 'typeorm';
import { gqlRequest } from '../helpers/gql-request.helper';
import { getAuthToken } from '../helpers/auth.helper';

describe('Users (e2e)', () => {
  let app: INestApplication;
  let db: DataSource;
  let adminToken: string;
  let userToken: string;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    await app.init();
    db = module.get(DataSource);

    adminToken = await getAuthToken(app, 'admin@seed.com', 'password');
    userToken = await getAuthToken(app, 'user@seed.com', 'password');
  });

  afterAll(async () => {
    await db.query('DELETE FROM users WHERE email LIKE \'e2e-%\'');
    await app.close();
  });

  describe('Query: users', () => {
    it('returns paginated user list for authenticated user', async () => {
      const { data, errors } = await gqlRequest(app, `
        query { users(first: 5) { edges { node { id email } } totalCount } }
      `, {}, userToken);

      expect(errors).toBeUndefined();
      expect(data?.users.totalCount).toBeGreaterThan(0);
    });

    it('returns 401 when not authenticated', async () => {
      const { errors } = await gqlRequest(app, `
        query { users(first: 5) { totalCount } }
      `);
      expect(errors?.[0]?.message).toMatch(/unauthorized/i);
    });
  });

  describe('Query: user', () => {
    it('finds user by id', async () => {
      const { data } = await gqlRequest(app, `
        query GetUser($id: ID!) { user(id: $id) { id email name } }
      `, { id: 'seed-user-id' }, userToken);

      expect(data?.user?.id).toBe('seed-user-id');
    });

    it('returns null for non-existent id', async () => {
      const { data } = await gqlRequest(app, `
        query { user(id: "00000000-0000-0000-0000-000000000000") { id } }
      `, {}, userToken);
      expect(data?.user).toBeNull();
    });
  });

  describe('Mutation: createUser', () => {
    it('admin can create a user', async () => {
      const { data, errors } = await gqlRequest(app, `
        mutation CreateUser($input: CreateUserInput!) {
          createUser(input: $input) { id email name }
        }
      `, { input: { email: 'e2e-new@test.com', name: 'E2E User' } }, adminToken);

      expect(errors).toBeUndefined();
      expect(data?.createUser.email).toBe('e2e-new@test.com');
    });

    it('regular user cannot create a user', async () => {
      const { errors } = await gqlRequest(app, `
        mutation { createUser(input: { email: "e2e-deny@test.com", name: "Denied" }) { id } }
      `, {}, userToken);
      expect(errors?.[0]?.message).toMatch(/forbidden/i);
    });

    it('rejects duplicate email', async () => {
      await gqlRequest(app, `
        mutation { createUser(input: { email: "e2e-dup@test.com", name: "First" }) { id } }
      `, {}, adminToken);

      const { errors } = await gqlRequest(app, `
        mutation { createUser(input: { email: "e2e-dup@test.com", name: "Second" }) { id } }
      `, {}, adminToken);

      expect(errors).toBeDefined();
    });
  });

  describe('Mutation: updateUser', () => {
    it('user can update their own profile', async () => {
      const { data } = await gqlRequest(app, `
        mutation UpdateUser($id: ID!, $input: UpdateUserInput!) {
          updateUser(id: $id, input: $input) { id name }
        }
      `, { id: 'seed-user-id', input: { name: 'Updated Name' } }, userToken);

      expect(data?.updateUser.name).toBe('Updated Name');
    });
  });

  describe('Mutation: deleteUser', () => {
    it('admin can soft-delete a user', async () => {
      const { data: created } = await gqlRequest(app, `
        mutation { createUser(input: { email: "e2e-delete@test.com", name: "To Delete" }) { id } }
      `, {}, adminToken);

      const { data } = await gqlRequest(app, `
        mutation DeleteUser($id: ID!) { deleteUser(id: $id) }
      `, { id: created?.createUser.id }, adminToken);

      expect(data?.deleteUser).toBe(true);
    });
  });
});
```

---

#### `test/e2e/auth.e2e-spec.ts`

```typescript
import { INestApplication } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import { AppModule } from '@src/app.module';
import { gqlRequest } from '../helpers/gql-request.helper';

describe('Auth (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const module = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = module.createNestApplication();
    await app.init();
  });

  afterAll(() => app.close());

  it('login returns accessToken for valid credentials', async () => {
    const { data, errors } = await gqlRequest(app, `
      mutation { login(input: { email: "admin@seed.com", password: "password" }) { accessToken } }
    `);
    expect(errors).toBeUndefined();
    expect(data?.login.accessToken).toBeDefined();
  });

  it('login returns error for wrong password', async () => {
    const { errors } = await gqlRequest(app, `
      mutation { login(input: { email: "admin@seed.com", password: "wrong" }) { accessToken } }
    `);
    expect(errors?.[0]?.message).toMatch(/unauthorized/i);
  });

  it('me query returns current user when authenticated', async () => {
    const { data: loginData } = await gqlRequest(app, `
      mutation { login(input: { email: "user@seed.com", password: "password" }) { accessToken } }
    `);
    const token = loginData?.login.accessToken as string;

    const { data } = await gqlRequest(app, `query { me { id email } }`, {}, token);
    expect(data?.me?.email).toBe('user@seed.com');
  });
});
```

---

#### `test/e2e/health.e2e-spec.ts`

```typescript
import { INestApplication } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import * as request from 'supertest';
import { AppModule } from '@src/app.module';

describe('Health (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const module = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = module.createNestApplication();
    await app.init();
  });

  afterAll(() => app.close());

  it('GET /health returns 200 with db and redis status', async () => {
    const { body } = await request(app.getHttpServer())
      .get('/health')
      .expect(200);

    expect(body.status).toBe('ok');
    expect(body.info.database.status).toBe('up');
    expect(body.info.redis.status).toBe('up');
  });
});
```

---

## Summary

| Layer             | Responsibility                     | Key Technologies          |
| ----------------- | ---------------------------------- | ------------------------- |
| `src/modules/`    | Feature business logic             | NestJS, `@nestjs/graphql` |
| `src/common/`     | Decorators, guards, filters, pipes | Class-validator, CASL     |
| `src/config/`     | Typed configuration                | `@nestjs/config`, Joi     |
| `libs/database/`  | Data persistence                   | TypeORM, PostgreSQL       |
| `libs/cache/`     | Response + data caching            | Redis, ioredis            |
| `libs/queue/`     | Async job processing               | BullMQ, Redis             |
| `libs/auth/`      | Authentication + authorization     | PassportJS, JWT, CASL     |
| `libs/logger/`    | Structured logging                 | Pino, Apollo plugin       |
| `libs/telemetry/` | Observability                      | OpenTelemetry, Prometheus |
| `test/`           | Quality assurance                  | Jest, Supertest           |
