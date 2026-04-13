# project-structure

A collection of **reference project structures** for building large-scale, production-grade applications. Each project includes an interactive HTML viewer and a detailed markdown blueprint covering directory layout, file responsibilities, key implementations, and architectural reasoning.

The goal is not to prescribe *the one true structure*, but to provide a battle-tested starting point you can copy, adapt, and justify to your team.

---

## Quick Start

Open [`index.html`](./index.html) in a browser to browse all project structures via an interactive landing page.

---

## Contents

| Project | Stack | Interactive | Docs |
| --- | --- | --- | --- |
| **Print Designer** | React 19 · TypeScript · Vite 6 · Fabric.js 6 · Zustand · Apollo Client · GraphQL Codegen · Tailwind CSS 4 | [`View`](./projects/print-designer/index.html) | [`README`](./projects/print-designer/README.md) |
| **NestJS Code-First GraphQL** | NestJS · GraphQL (code-first) · Apollo Server · TypeORM · Redis · BullMQ | [`View`](./projects/nestjs-graphql/index.html) | [`README`](./projects/nestjs-graphql/README.md) |
| **Next.js Enterprise Ecommerce** | Next.js 16 · React 19 · TypeScript · Apollo Client · GraphQL Codegen · Auth.js v5 · Tailwind CSS 4 | [`View`](./projects/nextjs-enterprise/index.html) | [`README`](./projects/nextjs-enterprise/README.md) |

> More stacks will be added under `projects/`. Each will follow the same structure: **interactive HTML viewer + detailed markdown documentation**.

---

## Repository Structure

```
project-structure/
├── index.html                          # Landing page — links to all projects
├── README.md                           # This file
└── projects/
    ├── print-designer/
    │   ├── index.html                  # Interactive structure viewer
    │   └── README.md                   # Detailed markdown documentation
    ├── nestjs-graphql/
    │   ├── index.html                  # Interactive structure viewer
    │   └── README.md                   # Detailed markdown documentation
    └── nextjs-enterprise/
        ├── index.html                  # Interactive structure viewer
        └── README.md                   # Detailed markdown documentation
```

---

## How to use this repo

1. **Open `index.html`** in a browser to see the project catalog with quick stats and tech badges.
2. **Click a project card** to open its interactive structure viewer — searchable, filterable directory tree with descriptions for every file.
3. **Read the `README.md`** inside each project folder for the full markdown documentation: complete project tree, key file implementations, patterns, and testing strategy.

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

1. Create a new folder under `projects/` named `<framework>-<style>` (e.g. `projects/fastify-rest/`).
2. Add an `index.html` (interactive viewer) and `README.md` (detailed docs) inside it.
3. Follow the section order of existing documents: **Why this approach → Project tree → Module anatomy → Infrastructure → Key files → Patterns → Testing**.
4. Include runnable code snippets for the most important files, not pseudo-code.
5. Add a card to `index.html` (root) and a row to the **Contents** table above.

---

## License

MIT — copy, adapt, and ship.
