# CommerceOS Admin

This is an example e-commerce admin platform built for the [Frontend Architecture: Monoliths to Micro-Frontends](https://github.com/Charca/fem-frontend-architecture) workshop on Frontend Masters.

This repo will serve as a foundation for the exercises in the workshop, as well as providing a concrete implementation when discussing architectural concepts.

_Note:_ There is no real data persistance or authentication in this repo. Depending on which repo you're looking at, the data is stored either in memory using MSW, or in a local `db.json` file.

## Stack

- React
- TypeScript
- Vite
- npm
- TanStack Router
- TanStack Query
- Tailwind CSS
- shadcn-style UI primitives
- lucide-react

## Run

```bash
npm install
npm run dev
```

## Scripts

```bash
npm run dev
npm run build
npm run lint
npm run typecheck
```

## Architecture

This branch organizes the app as a **modular monolith**. A conventional React project organizes by technology — a top-level `api/`, `components/`, `hooks/`, `screens/` — which lets any file import any other and tends to erode into a big ball of mud. Organizing by module instead, with enforced boundaries between modules, keeps that erosion in check.

It isn't a permanent solution. As the project grows, the shared layer can become its own ball of mud, and at that point the options worth considering are a monorepo or micro frontends.

### The three-tier layout

```
src/
├── app/          ← composition root: knows every module, no module knows it
│   ├── providers/    AppProviders, theme-provider
│   ├── router/       router.tsx — wires screens to URLs
│   └── shell/        app-shell, sidebar-nav, command-menu
│
├── modules/      ← the domain, one folder per bounded context
│   ├── analytics/  authentication/  catalog/  customers/  dashboard/
│   └── discounts/  inventory/  orders/  settings/  users/
│
└── shared/       ← generic, domain-agnostic, depends on nothing but itself
    ├── ui/           button, card, dialog, table (design system primitives)
    ├── components/   page-header, stat-card, empty-state (composed, still generic)
    ├── api/          http client
    └── lib/          utils
```

Dependencies flow one way:

```
app  ──▶  modules  ──▶  shared
```

`app/` may import everything — it's the assembly point where modules get composed into an application. Nothing in `modules/` may import from `app/`. That is what keeps a module extractable later: if `catalog/` reached up into the router or the shell, it could never be lifted out into a package or a micro frontend.

### Anatomy of one module

A module is a vertical slice. The same `api/`, `components/`, `hooks/`, `screens/` folders exist as in a technology-first layout — they just move down one level, inside the module:

```
modules/catalog/
├── domain/      catalog.types.ts        ← the module's vocabulary
├── api/         products.api.ts         ← how it talks to the server
├── hooks/       use-catalog-filters.ts  ← its state logic
├── components/  product-image-field.tsx ← its private building blocks
├── features/    products-table/         ← larger self-contained chunks
│                search-filters/
└── screens/     catalog.index.tsx       ← what the router mounts
    ...          catalog.detail.tsx
```

Everything catalog needs in order to exist lives in one folder. Delete the folder, delete the feature — that's the test.

### Core, supporting, and generic sub-domains

Splitting the domain into three categories predicts how much to invest in each module, and which way dependencies should flow:

| Category | In this repo | Treatment |
| --- | --- | --- |
| **Core** | `catalog`, `orders`, `customers`, `inventory` | The competitive advantage. Richest domain models, strictest boundaries, most tests. |
| **Supporting** | `discounts`, `analytics`, `dashboard`, `settings`, `users` | Necessary but not differentiating. Pragmatic implementations, more tolerance for "good enough". |
| **Generic** | `authentication`, all of `shared/ui` | Solved problems. Buy or use a library (Auth0, Clerk, shadcn/ui) rather than hand-rolling. |

Generic modules are the ones everything else is allowed to depend on. That's why the ESLint config carves `authentication` out as its own element type that any module may import, while a core module like `orders` stays off-limits to its peers. The dependency rules encode the category.

### Enforcing boundaries

Boundaries are enforced by [`eslint-plugin-boundaries`](https://www.jsboundaries.dev/) in [`eslint.config.js`](./eslint.config.js), so violations fail `npm run lint` instead of relying on review discipline.

Two details in that config are worth knowing:

**Element order matters.** Each file is assigned the type of the *first* pattern it matches, so the specific `authentication` pattern must be listed before the generic `src/modules/*/**/*` pattern. Reversed, every file under `modules/authentication/` is typed as an ordinary module and the `authentication` type never matches anything.

**Type-only imports are allowed across boundaries.** The `{ allow: { dependency: { kind: "type" } } }` rule exempts `import type`, since types are erased at compile time and create no runtime coupling. This is pragmatic and common, but it is a real trade-off: a shared `Order` type can spread everywhere unnoticed, and shape changes still ripple across module lines. The stricter alternative is to promote genuinely cross-cutting types into `shared/domain/` (as this repo does for `audit-log.types.ts`) so the sharing is deliberate and visible — which is also how `shared/` starts growing into a ball of mud.

### Not yet implemented: module entry points

Modules currently import into each other's internals:

```ts
import { RequireAuth } from "../../authentication/providers/auth-provider";
```

The next step is an `index.ts` per module acting as its published contract, exporting only the screens and types other modules may use and keeping the rest private. Consumers then write `import { Order } from "@/modules/orders"`, and the `boundaries/entry-point` rule rejects deep paths even when the module-to-module dependency itself is legal.

The payoff is that anything inside `orders/` can be restructured freely — folders renamed, files split, data layer swapped — as long as `index.ts` keeps its shape. Without an entry point, every internal file path is a de-facto public API and refactoring gets expensive again.
