# Engineering Guidelines — NestJS Code-First GraphQL

> **Purpose of this file (read after orienting).**
> This is the **rulebook**, not the map. It answers *"how am I required to write, structure, and verify code here?"* — the conventions, the do's and don'ts, and the definition of done that every change (human or automated) must satisfy.
>
> It is **prescriptive**, not descriptive. For *what the system is and where things live*, see [`codebase-context.md`](./codebase-context.md). Don't restate the architecture here; reference it and state the rule.
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

When creating `src/modules/<domain>/`, produce **all** of these, mirroring `users/`:

- [ ] `models/<x>.type.ts` — `@ObjectType()` with described `@Field()`s; a `<x>-connection.type.ts` if it's listable; enums via `registerEnumType()`.
- [ ] `dto/create-<x>.input.ts` + `update-<x>.input.ts` (`extends PartialType(...)`) + `<x>s-filter.args.ts` (`@ArgsType()`).
- [ ] `entities/<x>.entity.ts` (`@Entity()` extending `BaseEntity`) + `<x>.repository.ts` (custom queries).
- [ ] `loaders/<x>.loader.ts` (`Scope.REQUEST`) if anything resolves this entity as a relation.
- [ ] `mappers/<x>.mapper.ts` — pure `toGql()` / `toConnection()`.
- [ ] `policies/manage-<x>.policy.ts` — CASL rules.
- [ ] `<x>.service.ts`, `<x>.resolver.ts`, `<x>.module.ts`, `index.ts`.
- [ ] `__tests__/` with a spec per class (§10).
- [ ] Register the module in `app.module.ts`.

Do not invent a new folder layout or collapse these into fewer files "because the module is small." The uniformity is load-bearing for automation.

---

## 2. GraphQL types & DTOs

- **Models (`models/`) carry GQL decorators only**, never `@Entity()`/`@Column()`. Entities (`entities/`) carry DB decorators only, never `@Field()`.
- Add a `description` to every `@ObjectType()` and `@Field()` — descriptions surface in the schema and the playground and are part of the API contract.
- Use `@nestjs/graphql` composition helpers (`PartialType`, `PickType`, `OmitType`, `IntersectionType`) instead of redeclaring fields. `UpdateXInput extends PartialType(CreateXInput)`.
- Nullability is explicit: mark optional fields `{ nullable: true }`; for list fields decide between `nullable: 'items'`, `'itemsAndList'` deliberately.
- Validate **all** input at the DTO with `class-validator` decorators (`@IsEmail`, `@MinLength`, `@Max`, `@IsOptional`, `@IsUrl`, …). The global `ValidationPipe` runs with `whitelist: true` + `forbidNonWhitelisted: true`, so undeclared fields are rejected — never rely on manual checks in the resolver for shape validation.
- Pagination args extend the shared pattern: `first` (with `@Min`/`@Max`), `after` (Relay cursor), `orderBy`, `direction`.

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

## 9. Configuration & secrets

- Read config only through `ConfigService` + a typed `registerAs` namespace in `src/config/`. Don't sprinkle `process.env.X` through business code.
- Never commit `.env.prod` or hardcode secrets. New config keys get added to the relevant `*.config.ts`, documented, and given safe local defaults in `.env`.

---

## 10. Testing — definition of done

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

## 11. Anti-patterns — do NOT do these

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
