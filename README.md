# project-structure

A collection of **reference project structures** for building large-scale, production-grade backend applications. Each document in this repository is a self-contained blueprint: directory layout, file responsibilities, key implementations, and the architectural reasoning behind every decision.

The goal is not to prescribe *the one true structure*, but to provide a battle-tested starting point you can copy, adapt, and justify to your team.

---

## Contents

| Document | Stack | Summary |
| --- | --- | --- |
| [`nestjs-graphql-enterprise.md`](./nestjs-graphql-enterprise.md) | NestJS · GraphQL (code-first) · Apollo · TypeORM | Enterprise project layout for a NestJS application using the **code-first** GraphQL approach. Covers the full directory tree, feature-module anatomy, infrastructure libraries (`libs/`), cross-cutting concerns, key file implementations, Apollo plugin pipeline, testing strategy, and the reasoning for choosing code-first over schema-first. |

> More stacks (Express, Fastify, tRPC, Go, Python) will be added as separate markdown files. Each will follow the same structure: **tree → module anatomy → key files → patterns → testing**.

---

## How to use this repo

1. **Pick the document that matches your stack.** Every file is standalone — you do not need to read the others.
2. **Start with the "Complete Project Tree" section.** It is the single source of truth for every folder and file, annotated with its role.
3. **Drill into "Key File Implementations"** when you need a concrete starting point for a resolver, service, module, entity, loader, mapper, or policy.
4. **Read the "Patterns & Decisions" section last.** It explains *why* things are structured this way so you can adapt the layout without breaking its invariants.

---

## Design principles shared across all documents

These principles apply to every structure in this repo, regardless of language or framework:

- **Feature modules over layered folders.** Code is grouped by business domain (`modules/users/`), not by technical role (`controllers/`, `services/`, `models/` at the root). A feature should be deletable in one `rm -rf`.
- **Clear boundaries between transport, domain, and persistence.** GraphQL/HTTP types live in `models/` or `dto/`, business logic lives in services, and database entities live in `entities/`. Mappers convert between them — no layer reaches across.
- **Infrastructure belongs in `libs/`.** Auth, config, logging, database, caching, pub/sub, and observability are shared libraries consumed by feature modules, never duplicated per feature.
- **Cross-cutting concerns are decorators, guards, interceptors, and filters** — not copy-pasted `try/catch` blocks inside resolvers or controllers.
- **Tests live next to the code they test** in `__tests__/` folders, with a separate top-level `test/` directory for end-to-end suites.
- **Configuration is validated at boot.** No `process.env.FOO` reads scattered through the codebase — config is loaded once, schema-validated, and injected.

---

## Contributing a new structure

If you want to add a reference structure for another stack:

1. Create a new top-level markdown file named `<framework>-<style>.md` (e.g. `fastify-rest-enterprise.md`).
2. Follow the section order of the existing documents: **Why this approach → Project tree → Module anatomy → Infrastructure → Key files → Patterns → Testing**.
3. Include runnable code snippets for the most important files, not pseudo-code.
4. Add a row to the **Contents** table above.

---

## License

MIT — copy, adapt, and ship.
