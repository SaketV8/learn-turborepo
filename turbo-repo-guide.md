# The Turborepo Monorepo Guide (pnpm + Turborepo + TypeScript)

> A complete, self-contained reference for building and working in **any** Turborepo + pnpm + TypeScript monorepo.
> Read it top to bottom once, then use the [Table of Contents](#table-of-contents) and the [Cheat Sheet](#22-cheat-sheet) as a reference.
>
> **About the examples:** Throughout this guide we use one imaginary monorepo called **`acme`**. Its shared packages use the `@acme/` scope. Replace `acme` / `@acme` with your own project name and scope.

---

## Table of Contents

1. [What is a monorepo, and why Turborepo?](#1-what-is-a-monorepo-and-why-turborepo)
2. [The mental model: three tools, three jobs](#2-the-mental-model-three-tools-three-jobs)
3. [Basic setup from scratch](#3-basic-setup-from-scratch)
4. [Anatomy of a typical monorepo](#4-anatomy-of-a-typical-monorepo)
5. [Workspaces (pnpm) in depth](#5-workspaces-pnpm-in-depth)
6. [Catalogs (pnpm) in depth](#6-catalogs-pnpm-in-depth)
7. [Dependencies in a monorepo](#7-dependencies-in-a-monorepo)
8. [Internal packages: how packages share code](#8-internal-packages-how-packages-share-code)
9. [TypeScript in a monorepo](#9-typescript-in-a-monorepo)
10. [Turborepo: `turbo.json` in depth](#10-turborepo-turbojson-in-depth)
11. [Running tasks: `turbo run`, filters, and passing arguments](#11-running-tasks-turbo-run-filters-and-passing-arguments)
12. [Caching: local, remote, and debugging cache misses](#12-caching-local-remote-and-debugging-cache-misses)
13. [Development workflow](#13-development-workflow)
14. [Linting, formatting, and testing](#14-linting-formatting-and-testing)
15. [Adding a new app: a Hono API (two ways)](#15-adding-a-new-app-a-hono-api-two-ways)
16. [Adding a new shared package](#16-adding-a-new-shared-package)
17. [Adding any other kind of app: the general recipe](#17-adding-any-other-kind-of-app-the-general-recipe)
18. [CI pipelines](#18-ci-pipelines)
19. [Docker and deployment](#19-docker-and-deployment)
20. [Publishing packages with Changesets](#20-publishing-packages-with-changesets)
21. [Keeping a growing monorepo healthy](#21-keeping-a-growing-monorepo-healthy)
22. [Cheat sheet](#22-cheat-sheet)
23. [Troubleshooting and gotchas](#23-troubleshooting-and-gotchas)
24. [Further reading](#24-further-reading)

---

## 1. What is a monorepo, and why Turborepo?

A **monorepo** is one Git repository that holds several projects: **apps** (things you deploy or run) and **packages** (code those apps share).

```
acme/
├── apps/web          ← a website
├── apps/api          ← an API server
├── packages/db       ← database schema + client, used by the API
├── packages/ui       ← React components, used by the website
└── packages/shared   ← types and validation schemas, used by both
```

**What you gain:**

- **Shared code with no publishing.** `apps/web` imports `@acme/ui` straight from `packages/ui`. You don't publish to npm or bump versions.
- **Atomic changes.** You can change an API contract and every app that uses it in one commit and one PR.
- **One set of tools.** One TypeScript config, one linter config, one formatter, one lockfile, one CI pipeline.
- **Easy refactoring.** "Find all usages" works across every app.

**What it costs:**

- When you run `build` across 10 packages, you need to know the **order** (build `shared` before `web`), you want things to run **in parallel** when they can, and you don't want to **redo** work that hasn't changed.
- CI can get slow if it always builds and tests everything.
- Tooling (TypeScript paths, Docker builds, deploys) needs a bit more thought.

That's the problem **Turborepo** solves. It is a **task runner** that:

1. Reads your package dependency graph (who depends on whom).
2. Runs scripts (`build`, `dev`, `test`, `check-types`, …) in the right order, in parallel where it can.
3. **Caches** the results. If nothing changed, it replays the previous output in milliseconds, locally and (optionally) across your whole team and CI.

> Turborepo does **not** install packages, link packages, or compile TypeScript. Section 2 shows which tool does what.

### When a monorepo is *not* a good fit

- Projects with **completely unrelated** code, teams, and release cycles gain little.
- Projects that need **different access permissions** per team (Git permissions are per repository).
- Very large organizations may need heavier tooling (Nx, Bazel, Rush). Turborepo deliberately stays small and simple: it orchestrates your existing `package.json` scripts rather than replacing them.

---

## 2. The mental model: three tools, three jobs

This is the most important section. Most monorepo confusion comes from mixing up these three layers:

| Layer | Tool | Its job | Config files |
|---|---|---|---|
| **Package manager** | **pnpm** | Installs npm packages, **links workspace packages to each other**, manages versions (catalogs) | `pnpm-workspace.yaml`, every `package.json`, `pnpm-lock.yaml` |
| **Task runner** | **Turborepo** | Runs `package.json` scripts across packages in dependency order, in parallel, with caching | `turbo.json` |
| **Compiler / bundler** | **TypeScript**, **Vite**, **tsdown**, **Next.js**, **Wrangler**, … | Type-checks and builds the actual code | `tsconfig.json`, `vite.config.ts`, `next.config.ts`, … |

How they connect:

```
pnpm install
  └─ reads pnpm-workspace.yaml → finds apps/* and packages/*
  └─ resolves "workspace:*" → symlinks packages/ui into apps/web/node_modules/@acme/ui
  └─ resolves "catalog:"    → swaps in the version defined in pnpm-workspace.yaml
  └─ writes ONE pnpm-lock.yaml for the whole repo

pnpm build  (root script → "turbo run build")
  └─ turbo reads turbo.json + the package graph that pnpm defined
  └─ runs `build` in each package that HAS a "build" script, dependencies first
       └─ that script calls vite / tsc / tsdown / next / whatever
```

**Rule of thumb:**

- "Package X can't find package Y" → it's a **pnpm** problem (workspaces or dependencies).
- "Tasks run in the wrong order", "it's not caching", or "it's not running at all" → it's a **Turborepo** problem.
- "Type error" or "build error" → it's a **TypeScript or bundler** problem.

---

## 3. Basic setup from scratch

### 3.1 Option A: the official generator

```bash
pnpm dlx create-turbo@latest acme
# choose pnpm when asked
```

This gives you a working repo with two Next.js apps, a `ui` package, and shared ESLint and TypeScript config packages. Other scaffolders (for example Better-T-Stack, `create-t3-turbo`) generate opinionated full-stack starters on top of the same building blocks.

### 3.2 Option B: by hand (recommended once, for understanding)

**Step 1: create the root**

```bash
mkdir acme && cd acme
git init
pnpm init
mkdir -p apps packages
```

**Step 2: tell pnpm where the packages live** with `pnpm-workspace.yaml`:

```yaml
packages:
  - apps/*
  - packages/*
```

**Step 3: root `package.json`**

```json
{
  "name": "acme",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "test": "turbo run test",
    "check-types": "turbo run check-types"
  },
  "devDependencies": {
    "turbo": "^2",
    "typescript": "catalog:"
  },
  "packageManager": "pnpm@10.18.0",
  "engines": { "node": ">=22" }
}
```

- `"private": true` stops the root from being published to npm by accident.
- `"packageManager"` pins the pnpm version (use the output of `pnpm -v`). Corepack and Turborepo both read it, and Turborepo **requires** this field to detect your package manager.
- `"engines"` documents (and with `engine-strict`, enforces) the Node version. Also add a `.nvmrc` (e.g. `22`) so `nvm`, `fnm`, and CI can pick the same Node version.

**Step 4: `turbo.json`**

```json
{
  "$schema": "https://turborepo.com/schema.json",
  "tasks": {
    "build":       { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "lint":        { "dependsOn": ["^lint"] },
    "test":        { "dependsOn": ["^build"], "outputs": ["coverage/**"] },
    "check-types": { "dependsOn": ["^check-types"] },
    "dev":         { "cache": false, "persistent": true }
  }
}
```

**Step 5: a shared TypeScript config package** at `packages/typescript-config/package.json`:

```json
{ "name": "@acme/typescript-config", "version": "0.0.0", "private": true }
```

Next to it, add `packages/typescript-config/base.json` with your shared compiler options (see section 9).

**Step 6: an app** at `apps/web/package.json`:

```json
{
  "name": "web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "check-types": "tsc --noEmit"
  },
  "devDependencies": {
    "@acme/typescript-config": "workspace:*",
    "typescript": "catalog:",
    "vite": "catalog:"
  }
}
```

**Step 7: add the catalog** to `pnpm-workspace.yaml` (section 6 explains this):

```yaml
packages:
  - apps/*
  - packages/*

catalog:
  typescript: ^5.9.0
  vite: ^7.0.0
```

**Step 8: install and run**

```bash
pnpm install
pnpm build
pnpm build      # second time: ">>> FULL TURBO", everything came from cache
```

**Step 9: ignore generated folders** in `.gitignore`:

```
node_modules
dist
coverage
.turbo
.env*
!.env.example
```

That's a complete Turborepo. Everything else in this guide builds on these pieces.

---

## 4. Anatomy of a typical monorepo

Here is the example repo used for the rest of this guide:

```
acme/
├── package.json              ← root: scripts that call turbo, repo-wide dev tools
├── pnpm-workspace.yaml       ← workspace globs + catalogs + pnpm settings
├── pnpm-lock.yaml            ← ONE lockfile for the whole repo
├── turbo.json                ← task pipeline definition
├── .nvmrc                    ← Node version
├── .gitignore
│
├── apps/
│   ├── web/                  ← "web": Vite + React single-page app
│   └── api/                  ← "api": Hono API server
│
└── packages/
    ├── ui/                   ← "@acme/ui": shared React components
    ├── db/                   ← "@acme/db": database schema + client (e.g. Drizzle)
    ├── shared/               ← "@acme/shared": Zod schemas & types for API contracts
    ├── typescript-config/    ← "@acme/typescript-config": shared tsconfig files
    └── eslint-config/        ← "@acme/eslint-config": shared lint rules
```

### The dependency graph

`workspace:*` entries in each `package.json` define this graph:

```
   web ──dependencies──►  @acme/ui
   web ──dependencies──►  @acme/shared
   api ──dependencies──►  @acme/db
   api ──dependencies──►  @acme/shared

   every package ──devDependencies──► @acme/typescript-config
   every package ──devDependencies──► @acme/eslint-config
```

Turborepo turns this **package graph** into a **task graph**. With `"build": { "dependsOn": ["^build"] }`, `turbo run build` runs `shared#build`, `ui#build`, and `db#build` first (in parallel, they don't depend on each other), then `web#build` and `api#build` (also in parallel).

Check it yourself:

```bash
pnpm turbo run build --graph          # prints a Graphviz graph (or --graph=graph.html)
pnpm turbo ls                         # list the packages turbo sees
pnpm ls -r --depth -1                 # list every workspace package (pnpm's view)
pnpm why @acme/shared -r              # who depends on shared?
```

### Apps vs packages

There's no technical difference. Both are just folders with a `package.json`. It's a naming convention:

- **`apps/`**: things you **deploy or run** (websites, APIs, CLIs, workers). Nothing else in the repo imports them at runtime.
- **`packages/`**: things that are **imported** by apps or other packages, or that hold shared tooling (configs, generators, infrastructure-as-code).

A useful exception: an app may expose **types** to another app (for example an API exporting its route types for a typed client). That's fine as long as it's a type-only import (see section 16).

### Root files you'll commonly see

| File | Purpose |
|---|---|
| `pnpm-workspace.yaml` | Workspace globs, catalogs, and (since pnpm 10) most pnpm settings |
| `.npmrc` | Registry and auth settings (e.g. a private registry). Never commit tokens; use `${NPM_TOKEN}` env references |
| `turbo.json` | Task rules |
| `.nvmrc` / `.node-version` | Node version for local tools and CI |
| `tsconfig.json` (optional) | Only if you want root-level scripts type-checked; most monorepos don't need one |
| `.github/CODEOWNERS` | Who reviews which folder (very handy in monorepos) |
| `lefthook.yml` / `.husky/` | Git hooks (lint/format on commit) |

---

## 5. Workspaces (pnpm) in depth

### 5.1 What a "workspace" is

A **workspace** is any folder matched by the globs in `pnpm-workspace.yaml` that contains a `package.json`. Each one is a "workspace package", identified by its **`name`** field, **not** its folder name.

```yaml
# pnpm-workspace.yaml
packages:
  - apps/*        # every direct child of apps/ is a workspace
  - packages/*    # every direct child of packages/ is a workspace
  # - "packages/**"      would also match nested folders
  # - "!**/test/**"      excludes folders
```

| Folder | `name` in package.json | How you refer to it |
|---|---|---|
| `apps/web` | `web` | `turbo run build -F web` |
| `packages/ui` | `@acme/ui` | `import { Button } from "@acme/ui/button"` |

> **Tip:** Scope shared packages (`@acme/...`) so they can never clash with a real npm package. Apps can have short names (`web`, `api`) because nothing imports them.

> **Note:** You may see a `"workspaces": [...]` field in a root `package.json`. That field is used by **npm, Yarn, and Bun**. **pnpm ignores it** and only reads `pnpm-workspace.yaml`. If you only use pnpm, you don't need it; if you keep it, keep both in sync.

### 5.2 The `workspace:` protocol

```json
"@acme/ui": "workspace:*"
```

This says: **"Use the copy of this package in this repo. Never download it from npm."** If no local package has that name, `pnpm install` fails, which protects you from accidentally installing a public package with the same name.

What pnpm does with it is create a **symlink**:

```
apps/web/node_modules/@acme/ui  →  ../../../packages/ui
```

So when you edit `packages/ui/src/button.tsx`, `apps/web` sees the change **right away**. You don't need to reinstall or publish. (For compiled packages you still need to rebuild; see section 8.)

The different forms:

| Specifier | Meaning locally | What it becomes if you `pnpm publish` (with ui at `1.4.2`) |
|---|---|---|
| `workspace:*` | Any version, always link the local one | `1.4.2` (exact) |
| `workspace:^` | Link the local one | `^1.4.2` |
| `workspace:~` | Link the local one | `~1.4.2` |
| `workspace:^1.0.0` | Link it, and fail if the local version doesn't satisfy `^1.0.0` | `^1.0.0` |

For a private monorepo of apps, **always use `workspace:*`**. The other forms only matter if you publish packages to npm.

Add a workspace dependency from the CLI:

```bash
pnpm add @acme/shared --workspace -F api
# or, written out:
pnpm add "@acme/shared@workspace:*" -F api
```

### 5.3 pnpm's strict `node_modules` (no phantom dependencies)

pnpm does **not** flatten every package into one big `node_modules`. A package can only `import` what **its own** `package.json` declares.

```ts
// apps/api/src/index.ts
import { drizzle } from "drizzle-orm";   // ❌ fails unless api lists drizzle-orm itself,
                                         //    even though packages/db depends on it
```

This feels annoying at first, but it's a feature. Every package's dependencies are explicit, so `turbo prune`, Docker builds, deploys, and caching all work reliably.

**Fix:** Add the dependency to the package that imports it (`pnpm add drizzle-orm -F api`). Better still, re-export what you need from `@acme/db` so consumers don't depend on the ORM directly.

> Some older tools assume a flat `node_modules`. As an escape hatch, pnpm supports `publicHoistPattern` (hoist specific packages to the root) or, as a last resort, `nodeLinker: hoisted`. Prefer fixing the dependency declaration.

### 5.4 Root vs package dependencies

| Put it in the **root** `package.json` | Put it in the **package's** `package.json` |
|---|---|
| Repo-wide tools that run from the root: `turbo`, `prettier`, `lefthook`, `@changesets/cli` | Anything that package's code or scripts import or run |
| | `vite` in `apps/web`, `drizzle-kit` in `packages/db`, `vitest` in each tested package |

```bash
pnpm add -D some-tool -w        # -w / --workspace-root = install at the root
pnpm add hono -F api            # install into one package
```

Turborepo's advice is to install dependencies **where they're used**, not at the root. That keeps each package's cache hash, `turbo prune`, and Docker images accurate.

### 5.5 pnpm settings in `pnpm-workspace.yaml`

Since pnpm 10, most settings that used to live in `.npmrc` can live in `pnpm-workspace.yaml`:

```yaml
packages:
  - apps/*
  - packages/*

catalog:
  zod: ^4.0.0

# Force a version of a transitive dependency (e.g. a security patch)
overrides:
  "cookie@<0.7.0": ^0.7.0

# Supply-chain protection: don't install versions published less than 1 day ago (minutes)
minimumReleaseAge: 1440

# Fail install when a package's "engines" doesn't match your Node version
engineStrict: true
```

### 5.6 Install scripts (`onlyBuiltDependencies` / `allowBuilds`)

Since pnpm 10, npm packages' `postinstall` and other lifecycle scripts **don't run by default**. It's a supply-chain security measure. You explicitly allow the packages that need them, typically ones that download a native binary (`esbuild`, `sharp`, `@swc/core`, `prisma`, `workerd`):

```yaml
# pnpm 10
onlyBuiltDependencies:
  - esbuild
  - sharp

# newer pnpm versions use a map instead
allowBuilds:
  esbuild: true
  sharp: true
```

`pnpm approve-builds` lets you pick interactively. If a package fails mysteriously after install ("binary not found"), it probably needs to be on this list. pnpm warns about ignored build scripts during `pnpm install`.

📖 More: [pnpm Workspaces](https://pnpm.io/workspaces) · [`pnpm-workspace.yaml` settings](https://pnpm.io/settings)

---

## 6. Catalogs (pnpm) in depth

### 6.1 The problem

Libraries like `zod`, `typescript`, and `react` are used by **many** packages. Without catalogs, each `package.json` has its own version string:

```json
// apps/web          "zod": "^4.1.0"
// packages/shared   "zod": "^3.23.0"     ← drifted. Now you have 2 copies of zod.
```

Two copies of a library means a bigger bundle, `instanceof` checks that fail, broken React hooks ("Invalid hook call"), and type errors like *"Type 'ZodString' is not assignable to type 'ZodString'"*.

### 6.2 The solution: one version, defined once

```yaml
# pnpm-workspace.yaml
catalog:
  zod: ^4.1.0
  typescript: ^5.9.0
  vite: ^7.0.0
  vitest: ^3.2.0
  tailwindcss: ^4.1.0
```

```json
// any package.json
"zod": "catalog:",
"typescript": "catalog:"
```

`"catalog:"` means **"look up this package's version in the default catalog in `pnpm-workspace.yaml`."**

To **upgrade zod across the whole repo**, change one line in `pnpm-workspace.yaml` and run `pnpm install`. That's it. It also makes upgrade PRs (Renovate, Dependabot) much smaller.

### 6.3 Named catalogs

Sometimes you need groups of versions that move together, or two versions of something side by side during a migration:

```yaml
catalog:                 # the default catalog → "catalog:" or "catalog:default"
  zod: ^4.1.0

catalogs:                # named catalogs → "catalog:<name>"
  react19:
    react: ^19.2.0
    react-dom: ^19.2.0
    "@types/react": ^19.2.0
    "@types/react-dom": ^19.2.0
  react18:               # e.g. an older app that hasn't migrated yet
    react: ^18.3.1
    react-dom: ^18.3.1
```

```json
"react": "catalog:react19"
```

### 6.4 Useful catalog settings and behaviors

- **When you publish**, `catalog:` is replaced with the real version range, just like `workspace:`.
- **`catalogMode`** (in `pnpm-workspace.yaml`) controls what `pnpm add` does:
  - `manual` (default): `pnpm add zod` writes a normal version and ignores the catalog.
  - `prefer`: uses the catalog version if one exists, and falls back to a normal version if not.
  - `strict`: only allows catalog versions.
- **`cleanupUnusedCatalogs: true`** removes catalog entries that no package uses any more.
- **`pnpm add zod@catalog: -F api`** adds a dependency that uses the catalog entry.

### 6.5 When *not* to use the catalog

Use the catalog when a dependency is used by **two or more packages** and they **must** agree on a version. A dependency used by exactly one package (like `drizzle-kit` in `packages/db`) can just have its own version. Moving it into the catalog doesn't hurt, but it doesn't help either. Some teams put *everything* in the catalog for consistency; that's a valid choice too.

📖 More: [pnpm Catalogs](https://pnpm.io/catalogs)

---

## 7. Dependencies in a monorepo

### 7.1 The four kinds

| Field | Meaning | In a monorepo of private apps |
|---|---|---|
| `dependencies` | Needed **at runtime** by the code | Libraries your code imports (`react`, `hono`, `drizzle-orm`, `zod`) and internal packages it imports (`@acme/shared`) |
| `devDependencies` | Needed only to **develop, build, test, or deploy** | Compilers, bundlers, test runners, type packages, shared configs (`typescript`, `vite`, `vitest`, `@types/*`, `@acme/typescript-config`) |
| `peerDependencies` | "**The consumer** must provide this, and we share its copy" | `react` in a component library like `@acme/ui` |
| `optionalDependencies` / `peerDependenciesMeta.optional` | Nice to have; install doesn't fail without it | Platform-specific binaries, optional integrations |

**Does the distinction matter if nothing is published?** It matters in three places:

1. **`pnpm install --prod`**, **`pnpm deploy --prod`**, and **Docker** images skip `devDependencies`.
2. **`turbo prune <app>`** uses the graph to build a minimal sub-repo for one app.
3. **Bundlers** such as Vite bundle what you import no matter which field it's in. So for a bundled frontend app, the split is mostly documentation. For a Node server you run without bundling everything, it **does** matter: missing runtime dependencies crash in production.

A good habit: if code under `src/` imports it at runtime, it goes in `dependencies`. Everything else goes in `devDependencies`.

### 7.2 Case study: a shared React component package

`packages/ui/package.json`, line by line:

```json
{
  "name": "@acme/ui",
  "private": true,
  "type": "module",
  "exports": {
    "./*": "./src/components/*.tsx",
    "./styles.css": "./src/styles.css"
  },
  "scripts": {
    "lint": "eslint .",
    "check-types": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "clsx": "^2.1.1"
  },
  "peerDependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@acme/eslint-config": "workspace:*",
    "@acme/typescript-config": "workspace:*",
    "@types/react": "catalog:react19",
    "react": "catalog:react19",
    "react-dom": "catalog:react19",
    "typescript": "catalog:",
    "vitest": "catalog:"
  }
}
```

- **`clsx` in `dependencies`**: the components import it at runtime, and it doesn't matter if two copies exist.
- **`react` in `peerDependencies`**: React **must** exist exactly once in the final app. Hooks break with two copies. As a peer, `ui` says "use the React of whoever imports me".
- **`react` also in `devDependencies`**: so `ui` itself can type-check and run its tests without an app around it. The `catalog:` reference guarantees it's the same version the apps use.
- **`@acme/*-config` in `devDependencies`**: only used by tooling (`tsconfig.json` `extends`, `eslint.config.js` `import`).
- **`exports`**: the only paths consumers may import (section 8.4).

### 7.3 Pinning tightly coupled dependencies

Some dependencies must move **together**, at **exactly** matching versions:

- A framework and its plugins that declare exact peer versions (e.g. a core package `x@2.0.0-beta.12` and `x-plugin@2.0.0-beta.12`).
- **Pre-release (beta, rc) versions**, which have no SemVer compatibility promise: `^2.0.0-beta.12` can accept `2.0.0-beta.20` with breaking changes.
- Libraries that must exist as a **single copy** because they rely on object identity: React, GraphQL, Effect, Zod schemas shared across packages, ORMs whose types are compared structurally across packages.

For these, use **exact versions** (no `^`) and put them in a **named catalog** so they're bumped in one place, in one commit:

```yaml
catalogs:
  framework-x:
    x-core: 2.0.0-beta.12
    x-plugin-a: 2.0.0-beta.12
    x-plugin-b: 2.0.0-beta.12
```

Then verify there's only one copy after an upgrade:

```bash
pnpm why x-core -r      # should show one version
pnpm dedupe             # collapse duplicates in the lockfile where ranges allow it
```

### 7.4 Injected dependencies (advanced)

By default, `workspace:*` is a **symlink**. That means the linked package resolves *its* peer dependencies from its own folder, not from the consumer. Occasionally this produces two copies of a peer (e.g. React) or breaks tools that refuse to follow symlinks. pnpm can **copy** (inject) the package instead:

```json
// apps/web/package.json
"dependenciesMeta": {
  "@acme/ui": { "injected": true }
}
```

Or globally with `injectWorkspacePackages: true` in `pnpm-workspace.yaml`. Trade-off: injected packages are hard-linked copies, so they need re-syncing after a rebuild. Reach for this only when you actually hit the problem.

---

## 8. Internal packages: how packages share code

Turborepo calls packages that live in your repo and are consumed through `workspace:*` **Internal Packages**. There are three ways to build them.

### 8.1 Just-in-Time (JIT) packages

The package exports **raw TypeScript source**. The *consuming* app's bundler (Vite, Next.js, Wrangler, tsx, Bun) compiles it.

```json
// packages/shared/package.json
"exports": {
  ".":   "./src/index.ts",     // import ... from "@acme/shared"
  "./*": "./src/*.ts"          // import ... from "@acme/shared/user"
}
```

- ✅ No build step, so there's nothing to cache and no `dist/` to keep in sync. Instant hot reload.
- ✅ Simplest setup. Works great when every consumer has a bundler or runtime that understands TS.
- ❌ It can't be consumed by plain `node` without a TS loader, and it can't be published to npm as is.
- ❌ TypeScript type-checks the package's source again inside every consumer, and Turborepo can't cache a build of it.
- ❌ The consumer's bundler config applies (e.g. path aliases like `@/` inside the package won't resolve). Use relative imports inside JIT packages.

### 8.2 Compiled packages

The package has its own `build` script (for example `tsc` or [`tsdown`](https://tsdown.dev)) that writes `dist/`, and `exports` points at `dist/`:

```json
{
  "name": "@acme/shared",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsdown src/index.ts --dts",
    "dev": "tsdown src/index.ts --dts --watch"
  }
}
```

Now the `"dependsOn": ["^build"]` setting in `turbo.json` matters. Turbo builds this package **before** any app that depends on it, and caches `dist/**`.

Use this when a consumer runs plain Node without a bundler, when the package is published, or when the package is large and you want its build cached.

> `"types"` must come **first** inside an `exports` condition object. Node and TypeScript pick the first matching condition.

### 8.3 Publishable packages

Compiled packages that are also published to a registry (npm, GitHub Packages, Azure Artifacts). They need versioning and changelogs; see section 20.

### 8.4 The `exports` field (and what happens without it)

- **With `exports`**, consumers can **only** import the paths you list. That's good encapsulation: you can refactor internals freely.
- **Without `exports`** (only `main`, or nothing), any file is importable by its path, e.g. `@acme/shared/src/internal/helpers`. That's convenient but leaks internals.

Prefer **explicit entry points** (`"./button": "./src/button.tsx"`) or a **wildcard** (`"./*": "./src/*.tsx"`) over one giant `index.ts` barrel file. Barrels make bundlers and TypeScript do extra work and can pull in code you don't use.

### 8.5 Config packages

You can share **any** config through a package: `tsconfig` files, ESLint configs, a Tailwind preset, a Vitest config, Prettier config, and so on. Consumers add `"@acme/<name>-config": "workspace:*"` to `devDependencies` and then `extends` or `import` from it.

```
packages/typescript-config/
├── package.json        { "name": "@acme/typescript-config", "private": true }
├── base.json
├── react-library.json
└── node.json
```

📖 More: [Turborepo: Internal Packages](https://turborepo.com/docs/core-concepts/internal-packages) · [Node.js `exports`](https://nodejs.org/api/packages.html#package-entry-points)

---

## 9. TypeScript in a monorepo

### 9.1 One base config, one `tsconfig.json` per package

Every package has its **own** `tsconfig.json` that extends a shared base. Editors use the nearest `tsconfig.json`, and each package's `check-types` task becomes independently cacheable.

`packages/typescript-config/base.json`:

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "moduleDetection": "force",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "noEmit": true
  }
}
```

`packages/typescript-config/react-library.json`:

```json
{
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx"
  }
}
```

Consumer, e.g. `packages/ui/tsconfig.json`:

```json
{
  "extends": "@acme/typescript-config/react-library.json",
  "include": ["src"]
}
```

> That `extends` only resolves because `@acme/typescript-config` is a devDependency of `ui`, which makes pnpm symlink it into `packages/ui/node_modules`. **Every package that extends a shared config must list the config package.**

### 9.2 Choosing `moduleResolution`

| Setting | Use it when | Notes |
|---|---|---|
| `"Bundler"` (+ `"module": "ESNext"` or `"Preserve"`) | The code is bundled (Vite, Next.js, Wrangler, tsdown) or run with tsx/Bun | Extensionless relative imports allowed. Respects `exports`. The easiest choice for JIT packages |
| `"NodeNext"` (+ `"module": "NodeNext"`) | Code is compiled by `tsc` and run directly by Node, or published as a library | Requires `.js` extensions on relative imports. Most accurate for real Node behavior |

Both respect the `exports` field of your internal packages. The legacy `"Node"`/`"Node10"` setting ignores `exports`; avoid it.

### 9.3 `check-types` vs `build`

- Bundlers like Vite and esbuild **strip types without checking them**. A successful `build` doesn't mean the code type-checks.
- So every package gets a separate `"check-types": "tsc --noEmit"` script, and CI runs `turbo run check-types`.
- Compiled packages that emit `.d.ts` must run `build` before dependents can type-check. See the "transit node" pattern in section 10.3 to keep `check-types` parallel and still correct.

### 9.4 Path aliases

`paths` (e.g. `"@/*": ["./src/*"]`) are fine **inside apps**, as long as the bundler is configured with the same alias (Vite: `vite-tsconfig-paths` or `resolve.alias`).

For sharing code **between** packages, don't use `paths` pointing into other packages' folders. Use real workspace packages with `exports`. That's what keeps the dependency graph (and therefore Turborepo's ordering and caching) correct.

Inside **JIT packages**, prefer relative imports or Node [subpath imports](https://nodejs.org/api/packages.html#subpath-imports) (`"imports": { "#*": "./src/*" }` in package.json), because the consuming app's alias config doesn't apply to them.

### 9.5 Project references?

TypeScript [project references](https://www.typescriptlang.org/docs/handbook/project-references.html) (`composite: true`, `tsc -b`) let `tsc` build packages incrementally in order. In a Turborepo they mostly duplicate what Turborepo already does (ordering + caching), and they add config to keep in sync. Turborepo's docs suggest you usually don't need them. Consider them only for very large repos where editor performance suffers.

---

## 10. Turborepo: `turbo.json` in depth

### 10.1 The key idea: tasks ↔ scripts

A **task** in `turbo.json` is linked to a **script** in `package.json` **by name**.

```
turbo run build
  → for EVERY workspace package:
       does its package.json have a "build" script?
         yes → it's a task; run it (respecting dependsOn)
         no  → skip that package silently
```

So `turbo.json` doesn't contain commands. It contains **rules** about scripts: ordering, caching, inputs, outputs, and env vars.

> A task that is in `turbo.json` but has no matching script anywhere does nothing. A script with no matching task in `turbo.json` can't be run with `turbo run` (you'll get "could not find task").

### 10.2 A realistic `turbo.json`, line by line

```jsonc
{
  "$schema": "https://turborepo.com/schema.json",   // editor autocomplete + validation
  "ui": "tui",                                       // interactive terminal UI (vs "stream")

  "tasks": {
    "build": {
      "dependsOn": ["^build"],                // build my workspace DEPENDENCIES first
      "inputs": ["$TURBO_DEFAULT$", ".env*"], // hash = default files + .env files
      "outputs": ["dist/**"]                  // cache & restore these files
    },
    "lint":        { "dependsOn": ["^lint"] },
    "check-types": { "dependsOn": ["^check-types"] },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },

    "dev": {
      "cache": false,        // never cache a dev server
      "persistent": true     // long-running; never "finishes"
    },
    "start": {
      "dependsOn": ["build"],   // NO caret → this package's OWN build first
      "cache": false,
      "persistent": true
    },

    "db:generate": { "cache": false, "interactive": true },
    "db:migrate":  { "cache": false },
    "deploy":      { "dependsOn": ["build"], "cache": false }
  },

  "globalEnv": ["NODE_ENV", "CI"],
  "globalDependencies": [".env"]
}
```

### 10.3 Every task option explained

#### `dependsOn`: ordering

| Value | Meaning | Example |
|---|---|---|
| `"^build"` | Run `build` in all my **dependencies** (from package.json) first | `web` waits for `ui#build` and `shared#build` |
| `"build"` | Run `build` in **this same package** first | `start` needs its own `build` |
| `"api#build"` | Run the `build` task of the specific package `api` first | A dependency not expressed in package.json (e.g. e2e tests need the API built) |
| `[]` | No dependencies, run immediately | |

The `^` (caret) means "upstream", i.e. my dependencies. It's the most important character in Turborepo.

**The "transit node" pattern.** `"check-types": { "dependsOn": ["^check-types"] }` makes type-checking run in dependency order, but it doesn't really need to: each package type-checks on its own. What it *does* need is for its dependencies' **source changes** to invalidate its cache. A transit task gives you that, while keeping everything parallel:

```json
{
  "tasks": {
    "transit": { "dependsOn": ["^transit"] },
    "check-types": { "dependsOn": ["transit"] },
    "lint": { "dependsOn": ["transit"] }
  }
}
```

No package has a `transit` script, so nothing runs for it, but its hash chain still links each package to its dependencies.

#### `inputs`: what affects the cache hash

- Default: **all files in the package that Git tracks** (or doesn't ignore).
- `$TURBO_DEFAULT$` means "the default set". Add extras next to it (e.g. `.env*` files, which are usually git-ignored and so not in the default set).
- Narrow it to get more cache hits, e.g. `"inputs": ["src/**", "tsconfig.json"]`. Then editing a README won't invalidate `check-types`.
- Exclude with `!`, e.g. `["$TURBO_DEFAULT$", "!**/*.md", "!**/*.test.ts"]`.
- `$TURBO_ROOT$` refers to the repo root, e.g. `"$TURBO_ROOT$/tsconfig.base.json"` for a file outside the package.

#### `outputs`: what gets cached

- Files the task produces. Turbo saves them into the cache and **restores** them on a cache hit.
- `"dist/**"` covers Vite, tsc, and tsdown output. Next.js needs `[".next/**", "!.next/cache/**"]`. Test coverage: `["coverage/**"]`.
- If you leave it out, only the **logs** are cached. The task "replays" but no files are restored, which breaks downstream tasks that need `dist/`.

#### `cache`

- `true` (default). Set it to `false` for anything that has side effects or never finishes: dev servers, deploys, migrations, publishing.

#### `persistent`

- Marks long-running processes (`dev`, `start`, watchers). Turbo won't let other tasks `dependsOn` a persistent task, because it would wait forever.

#### `with`: run a sibling task alongside

- Starts other (usually persistent) tasks **at the same time**, without waiting for them.

```jsonc
// apps/web/turbo.json
{
  "extends": ["//"],
  "tasks": {
    "dev": { "with": ["api#dev"], "persistent": true, "cache": false }
  }
}
```

Now `turbo dev -F web` also starts the API's dev server.

#### `interactive`

- Lets the task read **stdin** (prompts like "Are you sure? y/N"). Requires `cache: false`. In the TUI, you select the task's pane to type into it. Migration generators and deploy CLIs often prompt.

#### `env` and `passThroughEnv` (per task)

- `env: ["DATABASE_URL"]`: these variables **are part of the hash** **and** are available to the task. Change the value, and you get a cache miss. Use it for anything that changes the build output, like `VITE_API_URL`.
- `passThroughEnv: ["AWS_*"]`: available to the task but **not** hashed. Use it for credentials and other things that don't change the output.
- Wildcards (`*`, `VITE_*`) and exclusions (`!VITE_SECRET`) are allowed.

#### Global keys

- `globalEnv`: like `env`, but for every task. If a `globalEnv` value changes, **every** cache entry misses. Keep this list short.
- `globalPassThroughEnv`: like `passThroughEnv`, but global.
- `globalDependencies`: files that, when changed, invalidate **every** task's hash.

### 10.4 Environment variables (why your env var is "undefined")

Turborepo 2 runs in **Strict Mode** by default. A task **only sees** env vars that are listed in `env`, `globalEnv`, `passThroughEnv`, or `globalPassThroughEnv`, plus a few system variables like `PATH` and `HOME`.

If a script works with `pnpm -F web build` but an env var is missing under `turbo run build`, add the variable to `env` (or `passThroughEnv`).

- `turbo run build --env-mode=loose` lets everything through. It's handy for debugging, not for everyday use.
- **Framework inference:** for known frameworks, prefixed variables like `VITE_*` and `NEXT_PUBLIC_*` are automatically included in the hash.
- **Turborepo does not load `.env` files.** Your framework does (Vite, Next.js), or a tool like `dotenv-cli` does (`"dev": "dotenv -e .env -- tsx watch src/index.ts"`). Add `.env*` to `inputs` so changes bust the cache.
- Never put secrets in variables that end up in client bundles (`VITE_*`, `NEXT_PUBLIC_*`): they ship to every browser.

### 10.5 Per-package `turbo.json` (Package Configurations)

You can override task settings for **one package** by putting a `turbo.json` inside it:

```jsonc
// apps/docs/turbo.json  (a Next.js app)
{
  "extends": ["//"],              // "//" = the root turbo.json
  "tasks": {
    "build": {
      "outputs": [".next/**", "!.next/cache/**"],
      "env": ["NEXT_PUBLIC_*"]
    }
  }
}
```

This keeps the root file generic while each app declares its own outputs and env vars. Array fields **replace** the root's values; to keep the root's values and add to them, start the array with `"$TURBO_EXTENDS$"`.

### 10.6 Root tasks

To run a script defined in the **root** `package.json` through turbo (so it gets cached), register it with the `//#` prefix:

```json
"tasks": {
  "//#format:check": {},
  "//#format": { "cache": false }
}
```

Then `turbo run format:check`. Simple root-only scripts can also just be run with `pnpm <script>`.

📖 More: [Configuring tasks](https://turborepo.com/docs/crafting-your-repository/configuring-tasks) · [turbo.json reference](https://turborepo.com/docs/reference/configuration) · [Using environment variables](https://turborepo.com/docs/crafting-your-repository/using-environment-variables) · [Package Configurations](https://turborepo.com/docs/reference/package-configurations)

---

## 11. Running tasks: `turbo run`, filters, and passing arguments

### 11.1 Ways to invoke turbo

```bash
pnpm build                     # root script → "turbo run build"
pnpm turbo run build           # call turbo directly (uses the repo's local turbo)
pnpm turbo build               # "run" is optional
pnpm turbo build check-types   # several tasks at once
```

Installing turbo globally (`pnpm add -g turbo`) lets you type `turbo build` anywhere. It defers to the repo's local version.

### 11.2 Filters (`--filter` / `-F`): pick which packages

| Filter | Selects |
|---|---|
| `-F web` | Only the package named `web` |
| `-F @acme/ui` | Only `ui` |
| `-F "@acme/*"` | Every package matching the name glob |
| `-F ./apps/*` | Every package in a **directory** (note the `./`) |
| `-F web...` | `web` **and all its dependencies** |
| `-F web^...` | Only `web`'s dependencies (not `web` itself) |
| `-F ...@acme/shared` | `shared` **and everything that depends on it** |
| `-F ...^@acme/shared` | Only the dependents of `shared` |
| `-F "[main]"` | Packages changed since the `main` branch (Git) |
| `-F "...[HEAD^1]"` | Changed in the last commit, plus their dependents |
| `-F "!web"` | Everything except `web` (combine with others) |
| `--affected` | Packages changed vs. the default branch, plus dependents. Great for CI |

```bash
pnpm turbo check-types -F ...@acme/shared   # "I changed shared, check everything that uses it"
pnpm turbo build --affected                 # CI: only build what this PR touched
```

> Even with a filter, `dependsOn` still applies. `turbo build -F web` **also** builds `web`'s dependencies first if they have `build` scripts. Use `--only` to skip dependency tasks.

### 11.3 Passing arguments to the underlying script: `--`

Anything after `--` is passed through to the script:

```bash
pnpm turbo run test -F @acme/ui -- --reporter=verbose
# runs:  vitest run --reporter=verbose
```

A handy pattern is root scripts that end in `--` so callers can append arguments:

```json
"scripts": {
  "db:generate": "turbo run db:generate -F @acme/db --",
  "test:ui":     "turbo run test -F @acme/ui --"
}
```

`pnpm db:generate --name add_users` becomes `turbo run db:generate -F @acme/db -- --name add_users`.

### 11.4 Other useful flags

| Flag | What it does |
|---|---|
| `--dry` / `--dry=json` | Show what *would* run and the hashes, without running anything |
| `--graph` / `--graph=graph.html` | Visualize the task graph |
| `--force` | Ignore the cache and re-run everything |
| `--summarize` | Write a JSON run summary to `.turbo/runs/` (great for "why a cache miss?") |
| `--continue` | Keep going when a task fails (`--continue=dependencies-successful` only skips dependents of failed tasks) |
| `--concurrency=4` / `--concurrency=50%` | Limit parallelism |
| `--output-logs=errors-only` | Quieter logs |
| `--ui=stream` | Plain interleaved logs instead of the TUI (useful in CI and for copy/paste) |
| `--log-order=grouped` | Group logs per task |
| `--only` | Run only the named tasks, not their `dependsOn` tasks |

### 11.5 Other turbo commands

| Command | What it does |
|---|---|
| `turbo ls` | List the packages turbo sees (`turbo ls web` shows one package's details) |
| `turbo watch dev` | Re-run tasks when files change, following the dependency graph |
| `turbo prune web --docker` | Create `out/` with only `web` + its deps + a pruned lockfile, for small Docker images |
| `turbo gen workspace` | Scaffold a new package (interactive) |
| `turbo gen` | Run your own custom generators (`turbo/generators/`) |
| `turbo login` / `turbo link` | Connect to Vercel Remote Cache |
| `turbo boundaries` | Check for imports that break package boundaries (section 21) |
| `turbo daemon status` | Inspect the background daemon (file watching) |

### 11.6 pnpm's own recursive commands (no caching)

Sometimes you just want to run a command, without turbo's caching or ordering:

```bash
pnpm -r exec rm -rf dist              # run a shell command in every package
pnpm -r --parallel run lint           # run a script everywhere (no dependency ordering)
pnpm -F web exec vite --version       # run a binary inside one package
pnpm -F @acme/db db:generate          # run one package's script without turbo
```

> **Watch out for pnpm built-in command names.** `pnpm deploy`, `pnpm publish`, `pnpm install`, `pnpm update`, `pnpm link`, etc. are **pnpm commands**, so they run instead of a root script with the same name. Use `pnpm run deploy` to run *your* script, or name the script something else (e.g. `release`, `ship`).

📖 More: [Running tasks](https://turborepo.com/docs/crafting-your-repository/running-tasks) · [`turbo run` reference](https://turborepo.com/docs/reference/run) · [pnpm filtering](https://pnpm.io/filtering)

---

## 12. Caching: local, remote, and debugging cache misses

### 12.1 How the cache works

For each task in each package, turbo computes a **hash** from:

- the files matched by `inputs` (default: the package's git-tracked files),
- the **hashes of the tasks it `dependsOn`** (so a change in `shared` changes `web`'s hash),
- the task's config in `turbo.json`,
- the values of the `env` and `globalEnv` variables,
- the `globalDependencies` files,
- the lockfile entries for that package's external dependencies.

```
hash seen before?  ── yes → FULL TURBO: replay logs, restore `outputs`  (ms)
                   └─ no  → run the script, then store logs + `outputs` under that hash
```

The local cache lives in `.turbo/cache` at the repo root. Each package also gets a `.turbo/` folder with task logs. Both are safe to delete at any time.

**Rule:** only cache tasks that are **deterministic**: same inputs → same outputs. Anything with side effects (deploying, migrating a database, publishing) must have `cache: false`.

### 12.2 Remote cache: share cache with teammates and CI

With a remote cache, a build that ran on CI (or on a teammate's machine) is a cache hit for you. This is where most of Turborepo's CI time savings come from.

**Vercel Remote Cache (free tier available):**

```bash
pnpm turbo login
pnpm turbo link
```

**Self-hosted:** the remote cache is an open HTTP API. Community servers such as [ducktors/turborepo-remote-cache](https://github.com/ducktors/turborepo-remote-cache) store artifacts in S3, Azure Blob Storage, or GCS. Point turbo at one with environment variables:

```bash
TURBO_API=https://your-cache-server
TURBO_TEAM=your-team
TURBO_TOKEN=<from your secrets manager>
```

> Store `TURBO_TOKEN` in a secret store (AWS Secrets Manager, Azure Key Vault, or your CI's secret variables). Never commit it.

**Signed artifacts.** To protect against cache poisoning, enable signature verification so turbo rejects artifacts that weren't signed with your key:

```json
{ "remoteCache": { "signature": true } }
```

and provide the key via `TURBO_REMOTE_CACHE_SIGNATURE_KEY` (again, from your secret store).

### 12.3 Debugging "why didn't it cache?"

```bash
pnpm turbo build --summarize     # writes .turbo/runs/<id>.json
pnpm turbo build --dry=json      # shows each task's hash + inputs, without running
```

Run it twice and diff the two summaries. The field that changed is the culprit. Common causes:

- A generated file that isn't git-ignored (e.g. a router's `routeTree.gen.ts`, a GraphQL codegen output) gets rewritten on every build, so the hash keeps changing. Ignore it in Git, or exclude it in `inputs`.
- An env var that changes every run (a timestamp, the CI build number, a commit SHA) is listed in `env` or `globalEnv`.
- `globalDependencies` includes a file that changes often.
- `outputs` is missing, so the cache "hits" but nothing is restored.
- The lockfile changed (any dependency bump changes the hashes of packages that use it).

📖 More: [Caching](https://turborepo.com/docs/crafting-your-repository/caching) · [Remote Caching](https://turborepo.com/docs/core-concepts/remote-caching)

---

## 13. Development workflow

### 13.1 What `pnpm dev` does

```
pnpm dev
  └─ root script: "turbo run dev"
       └─ which packages have a "dev" script?
            apps/web          → YES: "vite"                    (:5173)
            apps/api          → YES: "tsx watch src/index.ts"  (:3000)
            packages/ui       → NO  (JIT package: nothing to run)
            packages/shared   → NO
       └─ runs both in parallel (cache:false, persistent:true)
```

Because `ui` and `shared` are **JIT** packages, Vite and tsx pick up edits to them immediately. No watcher needed.

### 13.2 If you have compiled packages

A compiled package needs `dist/` to exist before the app starts. Two common setups:

1. **Build once, then start the dev servers:**

   ```json
   "dev": { "dependsOn": ["^build"], "cache": false, "persistent": true }
   ```

   Turbo builds (or restores from cache) every dependency, then starts the dev servers. Edits to a compiled package need a rebuild.

2. **Give the compiled package its own `dev` script** (`tsdown --watch`, `tsc --watch`). `turbo run dev` starts it alongside the apps, and it rebuilds `dist/` on every change.

3. **`turbo watch`**: `turbo watch dev` (or `turbo watch build check-types`) re-runs tasks when their inputs change, following the dependency graph, so non-persistent tasks get re-run in order.

### 13.3 Running just part of the stack

```bash
pnpm turbo dev -F web           # only the website
pnpm turbo dev -F web -F api    # website + API
pnpm turbo dev -F web...        # web + dev scripts of everything it depends on
```

Use `with` (section 10.3) when one app should **always** start with another.

### 13.4 Ports and environment

- Give each app a fixed, distinct port (`vite --port 5173`, `PORT=3000`), so they don't fight.
- Keep a committed `.env.example` per app listing the variables it needs, with placeholder values. Real `.env` files stay git-ignored.
- Proxy API calls from the frontend dev server (`server.proxy` in `vite.config.ts`) to avoid CORS during development.

### 13.5 Using the TUI

With `"ui": "tui"`, turbo shows one pane per task. Use the arrow keys to select a task, `Enter`/`i` to interact with it (type into prompts), and `/` to search logs. `--ui=stream` gives plain interleaved output instead.

---

## 14. Linting, formatting, and testing

### 14.1 Linting: per package (cacheable)

Give each package a `lint` script and a config that imports the shared one. This makes `turbo run lint` cache per package, so only changed packages are linted.

`packages/eslint-config/package.json`:

```json
{
  "name": "@acme/eslint-config",
  "private": true,
  "type": "module",
  "exports": {
    "./base": "./base.js",
    "./react": "./react.js"
  },
  "dependencies": {
    "@eslint/js": "catalog:",
    "typescript-eslint": "catalog:",
    "eslint-plugin-react-hooks": "catalog:"
  }
}
```

`packages/eslint-config/base.js`:

```js
import js from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  { ignores: ["dist/**", "coverage/**"] },
  js.configs.recommended,
  ...tseslint.configs.recommended,
);
```

Consumer `apps/api/eslint.config.js`:

```js
import base from "@acme/eslint-config/base";
export default [...base];
```

(The config package's lint plugins are regular `dependencies` of the config package, so consumers only need `@acme/eslint-config` and `eslint`.)

**Alternative: one root-level linter.** Fast linters like [Biome](https://biomejs.dev) or [Oxlint](https://oxc.rs) are often run once from the root (`"lint": "biome check ."`). This is simpler, but you lose per-package caching. With very fast linters that's usually fine.

### 14.2 Formatting: at the root

Formatting is repo-wide and cheap, so it's normally a **root** script, not a turbo task:

```json
"scripts": {
  "format": "prettier --write .",
  "format:check": "prettier --check ."
}
```

Pair it with a Git hook (lefthook, husky + lint-staged) that formats only staged files on commit.

### 14.3 Testing with Vitest

Give each tested package its own `test` script, so turbo caches tests per package:

```json
// packages/shared/package.json
"scripts": {
  "test": "vitest run",
  "test:watch": "vitest"
}
```

```json
// turbo.json
"test":       { "dependsOn": ["^build"], "outputs": ["coverage/**"] },
"test:watch": { "cache": false, "persistent": true }
```

- `dependsOn: ["^build"]` is needed only if tests import **compiled** packages. With JIT packages, `"dependsOn": ["transit"]` (section 10.3) is enough.
- Share common Vitest settings via a config package (`@acme/vitest-config`) and `mergeConfig` it in each package's `vitest.config.ts`.
- For a single command that runs **all** packages' tests in one Vitest process (nice for local watch mode and one combined coverage report), Vitest supports a root config with `test.projects: ["apps/*", "packages/*"]`. Use it for local DX, and keep per-package `turbo run test` for cached CI runs.

### 14.4 End-to-end tests

E2E tests (Playwright, Cypress) usually live in their own package (e.g. `apps/e2e`) and need the apps built:

```jsonc
// apps/e2e/turbo.json
{
  "extends": ["//"],
  "tasks": {
    "test:e2e": { "dependsOn": ["web#build", "api#build"], "cache": false }
  }
}
```

---

## 15. Adding a new app: a Hono API (two ways)

[Hono](https://hono.dev) is a small, fast web framework that runs everywhere: Node, Bun, Deno, Cloudflare Workers, AWS Lambda. Here are two ways to add it, depending on where you'll deploy.

### Option A: Hono on Node.js (or Bun), as a standalone server

Use this when deploying to a VM or a container (ECS, Azure Container Apps, Kubernetes, Cloud Run). It has its own `dev`, `build`, and `start` scripts, so it shows the Turborepo task concepts in action.

#### Step 1: create the package

```bash
mkdir -p apps/api/src
```

`apps/api/package.json`:

```json
{
  "name": "api",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsdown",
    "start": "node dist/index.js",
    "lint": "eslint .",
    "check-types": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "@acme/db": "workspace:*",
    "@acme/shared": "workspace:*",
    "@hono/node-server": "catalog:",
    "hono": "catalog:",
    "zod": "catalog:"
  },
  "devDependencies": {
    "@acme/eslint-config": "workspace:*",
    "@acme/typescript-config": "workspace:*",
    "@types/node": "catalog:",
    "tsdown": "catalog:",
    "tsx": "catalog:",
    "typescript": "catalog:",
    "vitest": "catalog:"
  }
}
```

`apps/api/tsconfig.json`:

```json
{
  "extends": "@acme/typescript-config/base.json",
  "compilerOptions": { "types": ["node"] },
  "include": ["src"]
}
```

#### Step 2: write the API

`apps/api/src/app.ts` (the app, separate from the server so it's easy to test):

```ts
import { Hono } from "hono";
import { cors } from "hono/cors";
import { logger } from "hono/logger";
import { zValidator } from "@hono/zod-validator";
import { CreateUserInput } from "@acme/shared";

export const app = new Hono()
  .use("*", logger())
  .use("/api/*", cors({ origin: ["http://localhost:5173"] }))
  .get("/health", (c) => c.json({ ok: true }))
  .get("/api/hello/:name", (c) => c.json({ message: `Hello, ${c.req.param("name")}!` }))
  .post("/api/users", zValidator("json", CreateUserInput), async (c) => {
    const input = c.req.valid("json");
    // const user = await db.insert(users).values(input).returning();
    return c.json({ created: input.email }, 201);
  });

export type AppType = typeof app;   // used for a typed client (section 16)
```

`apps/api/src/index.ts`:

```ts
import { serve } from "@hono/node-server";
import { app } from "./app";

const port = Number(process.env.PORT ?? 3000);
serve({ fetch: app.fetch, port }, () => console.log(`api on :${port}`));
```

(Add `@hono/zod-validator` to the catalog and to `api`'s dependencies.)

#### Step 3: bundle workspace code for production

`apps/api/tsdown.config.ts`:

```ts
import { defineConfig } from "tsdown";

export default defineConfig({
  entry: ["src/index.ts"],
  format: "esm",
  platform: "node",
  noExternal: [/^@acme\//],   // bundle internal JIT packages into dist
});
```

> **Why this matters:** plain `node` can't run the `.ts` files that JIT packages export. In `dev`, `tsx` handles TS, so it works. For `start`, the internal packages must be **bundled** into `dist` (as above) or be **compiled** packages (section 8.2). External npm packages (`hono`, `zod`, …) stay as imports and must be installed at runtime, which is what `pnpm deploy` in section 19 takes care of.

#### Step 4: how it interacts with `turbo.json` (no changes needed)

| You run | What turbo does for `api` |
|---|---|
| `pnpm dev` | Runs `tsx watch` **in parallel** with the other dev servers |
| `pnpm build` | Runs `tsdown` after its dependencies' `build` tasks, caches `dist/**` |
| `pnpm build` again | `FULL TURBO`: restored from cache instantly |
| `pnpm turbo start -F api` | `dependsOn: ["build"]` → builds (or restores from cache), then runs `node dist/index.js` |

If an env var **changes the build output**, declare it in `apps/api/turbo.json`:

```jsonc
{ "extends": ["//"], "tasks": { "build": { "env": ["API_VERSION"] } } }
```

Runtime-only variables (database URL, secrets) go in `passThroughEnv` of `start`, or simply come from the deployment environment.

### Option B: Hono on Cloudflare Workers (edge / serverless)

Use this to deploy to Cloudflare's edge with Wrangler. Wrangler bundles your code with esbuild, so JIT internal packages work without any extra config.

`apps/worker/package.json`:

```json
{
  "name": "worker",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "wrangler dev --port 8787",
    "deploy": "wrangler deploy",
    "cf-typegen": "wrangler types",
    "check-types": "tsc --noEmit"
  },
  "dependencies": {
    "@acme/shared": "workspace:*",
    "hono": "catalog:"
  },
  "devDependencies": {
    "@acme/typescript-config": "workspace:*",
    "typescript": "catalog:",
    "wrangler": "catalog:"
  }
}
```

`apps/worker/wrangler.jsonc`:

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "acme-worker",
  "main": "src/index.ts",
  "compatibility_date": "2026-11-01",
  "compatibility_flags": ["nodejs_compat"],
  "d1_databases": [
    { "binding": "DB", "database_name": "acme-db", "database_id": "<your-d1-database-id>" }
  ]
}
```

`apps/worker/src/index.ts`:

```ts
import { Hono } from "hono";

// `Env` is generated by `pnpm cf-typegen` (wrangler types) from wrangler.jsonc
const app = new Hono<{ Bindings: Env }>();

app.get("/health", (c) => c.json({ ok: true }));

app.get("/api/users/count", async (c) => {
  const row = await c.env.DB.prepare("SELECT COUNT(*) AS n FROM users").first<{ n: number }>();
  return c.json({ count: row?.n ?? 0 });
});

export default app; // a Hono app *is* a valid Workers `fetch` handler
```

Turbo notes:

- There's no `build` script: `wrangler deploy` bundles at deploy time. Add `"build": "wrangler deploy --dry-run --outdir dist"` if you want CI to verify the bundle and cache it.
- `deploy` is already in `turbo.json` with `cache: false`. Run it with `pnpm turbo run deploy -F worker` (not `pnpm deploy`, which is a pnpm built-in).
- Deploy credentials (`CLOUDFLARE_API_TOKEN`) belong in CI secrets, passed through with `passThroughEnv`.

### Install and run (both options)

```bash
pnpm install                   # links the new workspace, installs hono
pnpm dev                       # starts api (or worker) alongside the other dev servers
curl localhost:3000/health
pnpm check-types               # turbo includes the new app automatically
```

You didn't touch the root `turbo.json`. Turbo found the new app through the `apps/*` glob and its script names.

📖 More: [Hono on Node.js](https://hono.dev/docs/getting-started/nodejs) · [Hono on Cloudflare Workers](https://hono.dev/docs/getting-started/cloudflare-workers) · [Wrangler configuration](https://developers.cloudflare.com/workers/wrangler/configuration/)

---

## 16. Adding a new shared package

Example: shared Zod schemas and types used by both `web` and `api`. This gives you one source of truth for API contracts.

```bash
mkdir -p packages/shared/src
```

`packages/shared/package.json` (JIT style):

```json
{
  "name": "@acme/shared",
  "private": true,
  "type": "module",
  "exports": {
    ".": "./src/index.ts",
    "./*": "./src/*.ts"
  },
  "scripts": {
    "lint": "eslint .",
    "check-types": "tsc --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "zod": "catalog:"
  },
  "devDependencies": {
    "@acme/eslint-config": "workspace:*",
    "@acme/typescript-config": "workspace:*",
    "typescript": "catalog:",
    "vitest": "catalog:"
  }
}
```

`packages/shared/tsconfig.json`:

```json
{ "extends": "@acme/typescript-config/base.json", "include": ["src"] }
```

`packages/shared/src/index.ts`:

```ts
import { z } from "zod";

export const CreateUserInput = z.object({
  email: z.email(),
  name: z.string().min(1),
});
export type CreateUserInput = z.infer<typeof CreateUserInput>;
```

Use it in the apps:

```bash
pnpm add @acme/shared --workspace -F api -F web
```

```ts
import { CreateUserInput } from "@acme/shared";
```

**Scaffolding shortcut:** `pnpm turbo gen workspace` asks you questions and generates the folder and `package.json` for you. `pnpm turbo gen workspace --copy` copies an existing package as a template.

### Tip: end-to-end type safety with Hono RPC

Hono's RPC client gives the frontend fully typed API calls with no codegen:

1. In `apps/api`, export the type (`export type AppType = typeof app`, already done above) and expose it:

   ```json
   // apps/api/package.json
   "exports": { "./client": "./src/app.ts" }
   ```

2. Add `"api": "workspace:*"` to `web`'s **devDependencies** (types only, no runtime code).
3. In `web`:

   ```ts
   import { hc } from "hono/client";
   import type { AppType } from "api/client";

   const client = hc<AppType>("/");
   const res = await client.api.hello[":name"].$get({ param: { name: "Ada" } });
   const data = await res.json();   // typed: { message: string }
   ```

Because it's an `import type`, no server code ends up in the website bundle. See [Hono RPC](https://hono.dev/docs/guides/rpc). (tRPC and oRPC are alternatives built on the same idea.)

---

## 17. Adding any other kind of app: the general recipe

Whatever the app is (Next.js, Astro, a docs site, an Expo mobile app, a CLI, a queue worker), the checklist is the same:

1. **Create the folder** under `apps/` (or `packages/` if other code imports it). Most framework generators work from inside the folder:
   ```bash
   cd apps && pnpm create vite@latest admin      # or: pnpm create next-app docs, pnpm create astro ...
   ```
   Then delete any `pnpm-lock.yaml`, `package-lock.json`, `.git`, or `node_modules` the generator made inside the folder. There's only **one** lockfile, at the root.
2. **Set a unique `name`** in its `package.json`. Add `"private": true`.
3. **Use the shared stuff:**
   - `"@acme/typescript-config": "workspace:*"` + `tsconfig.json` `extends`
   - `"@acme/eslint-config": "workspace:*"` + `eslint.config.js`
   - switch shared libraries to `"catalog:"` (`typescript`, `react`, `zod`, `tailwindcss`, …)
   - add `workspace:*` deps on `ui`, `shared`, … as needed
4. **Name scripts after the turbo tasks** so they join the pipeline automatically: `dev`, `build`, `lint`, `check-types`, `test`, `start`.
5. **Declare outputs** if they aren't `dist/**`. Add an `apps/<app>/turbo.json` with `"extends": ["//"]` (Next.js: `[".next/**", "!.next/cache/**"]`; Astro: `["dist/**"]`; Expo/React Native: usually none).
6. **Declare env vars** that affect the build in `env` (e.g. `"NEXT_PUBLIC_*"`), and runtime-only ones in `passThroughEnv`.
7. **Consuming JIT packages:** Next.js needs `transpilePackages: ["@acme/ui"]` in `next.config.ts`. Vite, Astro, Wrangler, tsx, and Bun handle TS from workspace packages out of the box. React Native/Expo needs Metro configured for monorepos (recent Expo versions detect it automatically).
8. **Tailwind across packages:** if the app uses components from `@acme/ui`, Tailwind must scan that package's files too (Tailwind v4: `@source "../../packages/ui/src";` in the app's CSS).
9. **Deploy target:** add your platform's config (Dockerfile + `turbo prune`, Vercel/Netlify project with root directory `apps/<app>`, `wrangler.jsonc`, an IaC definition, etc.).
10. **Run `pnpm install`**, then `pnpm turbo run build --dry` to confirm turbo sees it.
11. **Commit** the new folder **and** the updated `pnpm-lock.yaml`.

---

## 18. CI pipelines

The goal: **only do work for what changed**, and **reuse cache from previous runs**.

### 18.1 The ingredients

1. Install with the **exact** lockfile: `pnpm install --frozen-lockfile` (the default in CI).
2. Run all checks in one turbo call so they share the graph and run in parallel:
   `pnpm turbo run lint check-types test build`
3. Add `--affected` so PRs only run tasks for changed packages and their dependents. It needs Git history, so fetch enough of it.
4. Connect a **remote cache** (section 12.2) so CI reuses results from earlier runs and from developers' machines.

### 18.2 GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    env:
      TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}   # remote cache (optional)
      TURBO_TEAM: ${{ vars.TURBO_TEAM }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0              # full history so --affected can compare to main

      - uses: pnpm/action-setup@v4    # reads "packageManager" from package.json

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: pnpm

      - run: pnpm install --frozen-lockfile

      - run: pnpm turbo run lint check-types test build --affected
```

On `pull_request` events, `--affected` compares against the PR's base branch automatically. Without a remote cache, you can still cache `.turbo/cache` between runs with `actions/cache`.

### 18.3 Azure Pipelines

```yaml
# azure-pipelines.yml
trigger:
  branches: { include: [main] }
pr:
  branches: { include: [main] }

pool:
  vmImage: ubuntu-latest

variables:
  - group: turbo-remote-cache        # variable group (can be linked to Azure Key Vault)
  - name: TURBO_SCM_BASE
    value: origin/main               # what --affected compares against

steps:
  - checkout: self
    fetchDepth: 0                    # full history for --affected

  - task: UseNode@1
    inputs:
      version: "22.x"

  - script: |
      corepack enable
      pnpm install --frozen-lockfile
    displayName: Install

  - script: pnpm turbo run lint check-types test build --affected --ui=stream
    displayName: Lint, type-check, test, build
    env:
      TURBO_TOKEN: $(TURBO_TOKEN)    # secret variables must be mapped into env explicitly
      TURBO_TEAM: $(TURBO_TEAM)
```

### 18.4 Deploying from CI

- Deploy jobs typically run **after** CI passes, only on `main`, and only for affected apps:
  `pnpm turbo run deploy --affected` (or `-F web` per job).
- Deploy tasks have `cache: false`, and their credentials come from CI secrets via `passThroughEnv`.
- `turbo-ignore` (`npx turbo-ignore web`) exits early when an app is unaffected by a commit. Hosting platforms (Vercel, Netlify) use it as an "ignored build step" to skip unnecessary deploys.

📖 More: [Constructing CI](https://turborepo.com/docs/crafting-your-repository/constructing-ci) · [GitHub Actions guide](https://turborepo.com/docs/guides/ci-vendors/github-actions)

---

## 19. Docker and deployment

### 19.1 `turbo prune`: a minimal sub-repo per app

Copying the whole monorepo into a Docker image is slow and invalidates the layer cache on every change. `turbo prune api --docker` creates `out/` containing **only** `api`, its internal dependencies, and a pruned lockfile:

```
out/
├── json/   ← only package.json files + lockfile + pnpm-workspace.yaml  (for the install layer)
└── full/   ← full source of the needed packages                        (for the build layer)
```

Installing from `out/json` first means the dependency layer is cached until a `package.json` or the lockfile changes.

### 19.2 Dockerfile for the Node API (Option A in section 15)

```dockerfile
FROM node:22-alpine AS base
RUN npm install -g corepack@latest && corepack enable
WORKDIR /app

# 1. Prune the monorepo down to what "api" needs
FROM base AS pruner
COPY . .
RUN pnpm dlx turbo prune api --docker

# 2. Install deps (cached until package.json / lockfile change), then build
FROM base AS builder
COPY --from=pruner /app/out/json/ .
RUN pnpm install --frozen-lockfile
COPY --from=pruner /app/out/full/ .
RUN pnpm turbo run build -F api
# Create a self-contained folder with dist/ + production node_modules
RUN pnpm deploy --filter=api --prod /prod/api

# 3. Small runtime image
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -S app && adduser -S app -G app
USER app
COPY --from=builder /prod/api .
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

Notes:

- **`pnpm deploy`** copies one workspace package plus its production dependencies into a standalone folder. In pnpm 10 it requires `injectWorkspacePackages: true` in `pnpm-workspace.yaml` (or the `--legacy` flag).
- Make sure the package's `files` field (or the absence of one) includes `dist/` so `pnpm deploy` copies it.
- Add a `.dockerignore` with `node_modules`, `.turbo`, `dist`, `.git`, and `.env*` so local artifacts and secrets never enter the build context.
- To use the remote cache inside `docker build`, pass `TURBO_TEAM` and `TURBO_TOKEN` as **build secrets** (`--secret`), not as `ARG`s, which end up in the image history.

### 19.3 Other deployment targets

| Target | How it fits a monorepo |
|---|---|
| Vercel / Netlify | Set the project's **root directory** to `apps/<app>`. They detect pnpm workspaces and Turborepo, and run `turbo build -F <app>` |
| Cloudflare Workers / Pages | `wrangler deploy` from the app folder (section 15, Option B) |
| Containers (ECS, Azure Container Apps, Kubernetes, Cloud Run) | `turbo prune` + Dockerfile, one image per app |
| Static hosting (S3 + CloudFront, Azure Static Web Apps) | `turbo build -F web`, upload `apps/web/dist` |
| Infrastructure-as-code (Pulumi, AWS CDK, Terraform CDK, SST, Alchemy) | Put it in its own package (e.g. `packages/infra`) with a `deploy` script, `cache: false`, `dependsOn` the apps' builds. All its tooling is `devDependencies`: it runs in CI, never in production |

📖 More: [Turborepo + Docker](https://turborepo.com/docs/guides/tools/docker) · [`pnpm deploy`](https://pnpm.io/cli/deploy)

---

## 20. Publishing packages with Changesets

Only needed if some packages are published to a registry (npm, GitHub Packages, Azure Artifacts). Private app monorepos can skip this.

[Changesets](https://github.com/changesets/changesets) handles versioning and changelogs across many packages:

```bash
pnpm add -Dw @changesets/cli
pnpm changeset init
```

The workflow:

1. With each PR that changes a publishable package, run **`pnpm changeset`**. It asks which packages changed and whether it's a patch, minor, or major change, and writes a small markdown file in `.changeset/`.
2. When you're ready to release, **`pnpm changeset version`** bumps versions, updates dependents' ranges, and writes `CHANGELOG.md` files.
3. **`pnpm turbo run build --filter="./packages/*" && pnpm changeset publish`** builds and publishes everything with a new version. `workspace:*` and `catalog:` are rewritten to real versions on publish.

The [`changesets/action`](https://github.com/changesets/action) GitHub Action automates steps 2 and 3 with a "Version Packages" PR.

Publishable packages should be **compiled** (section 8.2), have `"private": false` (or no `private` field), a `"files": ["dist"]` field, and correct `exports` with `types`. Tools like [`publint`](https://publint.dev) and [Are the Types Wrong?](https://arethetypeswrong.github.io) catch packaging mistakes before you publish.

---

## 21. Keeping a growing monorepo healthy

### 21.1 Package boundaries

Rules that keep the graph clean:

- A package only imports **other packages by name** (`@acme/ui`), never by relative path into another package's folder (`../../packages/ui/src/...`). Relative cross-package imports bypass the dependency graph, so turbo's ordering and caching become wrong.
- Every import must be declared in that package's `package.json` (pnpm enforces this at runtime; lint for it too).
- **No cycles.** If `a` depends on `b` and `b` on `a`, extract the shared part into a third package.

`turbo boundaries` checks the first two rules automatically. You can also add **tags** to restrict which packages may depend on which:

```jsonc
// packages/db/turbo.json
{ "extends": ["//"], "tags": ["server-only"] }
```

```jsonc
// root turbo.json
{
  "boundaries": {
    "tags": {
      "server-only": { "dependents": { "deny": ["frontend"] } }
    }
  }
}
```

Now a package tagged `frontend` (e.g. `web`) fails the check if it depends on `@acme/db`.

### 21.2 Code generators

As the repo grows, new packages should look the same. `turbo gen` supports custom generators (built on [Plop](https://plopjs.com)) in `turbo/generators/config.ts`, e.g. "new React library" or "new API route", so nobody has to copy-paste `package.json`, `tsconfig.json`, and `eslint.config.js` by hand.

### 21.3 Dependency hygiene

- Keep shared versions in **catalogs** (section 6).
- Run `pnpm outdated -r` and `pnpm up -r --interactive --latest` regularly, or let Renovate / Dependabot do it (both understand pnpm catalogs).
- Run `pnpm audit` in CI, and use `overrides` for urgent transitive security fixes.
- After upgrades, `pnpm dedupe` and `pnpm why <pkg> -r` to make sure singletons (React, Zod, …) still resolve to one copy.
- Tools like [knip](https://knip.dev) find unused files, exports, and dependencies across all workspaces.

### 21.4 Ownership

Add a `CODEOWNERS` file so changes to each folder request the right reviewers:

```
/apps/web/        @acme/frontend-team
/apps/api/        @acme/backend-team
/packages/db/     @acme/backend-team
/turbo.json       @acme/platform-team
/pnpm-workspace.yaml @acme/platform-team
```

### 21.5 Documentation

Give every package a short `README.md`: what it is, who uses it, its scripts, and its env vars. In a monorepo, people often land in a package they've never seen.

---

## 22. Cheat sheet

```bash
# ── install / dependencies ──────────────────────────────────────────
pnpm install                                  # install everything, link workspaces
pnpm install --frozen-lockfile                # CI: fail if the lockfile is out of date
pnpm add <pkg> -F <workspace>                 # add a dep to one package
pnpm add -D <pkg> -F <workspace>              # add a devDep to one package
pnpm add -D <pkg> -w                          # add to the root
pnpm add <pkg>@catalog: -F <workspace>        # add using the catalog version
pnpm add @acme/shared --workspace -F api      # link a workspace package
pnpm remove <pkg> -F <workspace>
pnpm up -r --interactive --latest             # interactive upgrades across the repo
pnpm outdated -r                              # what's outdated, everywhere
pnpm why <pkg> -r                             # why is this installed / which versions
pnpm ls -r --depth -1                         # list workspace packages
pnpm dedupe                                   # collapse duplicate versions in the lockfile
pnpm approve-builds                           # allow install scripts for selected packages

# ── turbo ───────────────────────────────────────────────────────────
pnpm turbo build                              # build everything (cached)
pnpm turbo build -F web                       # one package (+ its deps' build)
pnpm turbo build -F web...                    # web and everything it depends on
pnpm turbo check-types -F ...@acme/shared     # shared and everything that uses it
pnpm turbo build --affected                   # only what changed vs main
pnpm turbo lint check-types test build        # several tasks, one graph
pnpm turbo build --force                      # bypass cache
pnpm turbo build --dry                        # what would run?
pnpm turbo build --graph=graph.html           # visualize
pnpm turbo build --summarize                  # debug cache misses
pnpm turbo run <task> -F <pkg> -- --flag      # pass args to the script
pnpm turbo watch dev                          # re-run tasks on change
pnpm turbo ls                                 # packages turbo sees
pnpm turbo gen workspace                      # scaffold a package
pnpm turbo prune <app> --docker               # minimal subset for Docker
pnpm turbo boundaries                         # check import boundaries

# ── pnpm without turbo ──────────────────────────────────────────────
pnpm -F <pkg> <script>                        # run one package's script directly
pnpm -F <pkg> exec <bin>                      # run a binary in one package
pnpm -r exec <cmd>                            # run a command in every package
pnpm deploy --filter=<app> --prod <dir>       # standalone folder for one app

# ── cleanup ─────────────────────────────────────────────────────────
rm -rf .turbo                                 # clear the local turbo cache
pnpm -r exec rm -rf node_modules dist .turbo && rm -rf node_modules && pnpm install   # nuclear reset
```

### turbo.json quick reference

| Key | One-liner |
|---|---|
| `dependsOn: ["^x"]` | Run `x` in my dependencies first |
| `dependsOn: ["x"]` | Run my own `x` first |
| `dependsOn: ["pkg#x"]` | Run `x` in a specific package first |
| `with: ["pkg#x"]` | Start `x` alongside this task |
| `inputs` | Files that affect the hash (`$TURBO_DEFAULT$` = default) |
| `outputs` | Files to cache and restore |
| `cache: false` | Never cache |
| `persistent: true` | Long-running (dev servers) |
| `interactive: true` | Accepts stdin (needs `cache: false`) |
| `env` | Env vars that are hashed and available |
| `passThroughEnv` | Env vars that are available but not hashed |
| `globalEnv` / `globalDependencies` | Same idea, for all tasks |
| `extends: ["//"]` | Per-package turbo.json inherits from the root |
| `"//#task"` | A task defined by a root `package.json` script |

### Where does it go?

| I want to… | File |
|---|---|
| Add a package folder pattern | `pnpm-workspace.yaml` → `packages` |
| Share a library version | `pnpm-workspace.yaml` → `catalog` |
| Allow a native install script | `pnpm-workspace.yaml` → `onlyBuiltDependencies` / `allowBuilds` |
| Change task order, caching, env | `turbo.json` (root or per package) |
| Change how code compiles | `tsconfig.json`, bundler config |
| Use package A in package B | B's `package.json` → `"A": "workspace:*"` |

---

## 23. Troubleshooting and gotchas

| Symptom | Likely cause | Fix |
|---|---|---|
| `Cannot find module '@acme/xyz'` | Not declared in the consumer's package.json, or no `pnpm install` since adding it | `pnpm add @acme/xyz --workspace -F <consumer>` |
| Import works in one package and fails in another | pnpm strict node_modules (phantom dependency) | Add the dep to the package that imports it |
| `File '@acme/typescript-config/...' not found` in tsconfig | Package doesn't depend on the config package | Add `"@acme/typescript-config": "workspace:*"` to devDependencies |
| Types from an internal package are `any` / not found | `exports` missing a `types` condition, or `moduleResolution` is legacy `Node` | Add `types` first in `exports`; use `Bundler` or `NodeNext` |
| `turbo run X` does nothing for a package | That package has no script named `X` | Add the script, or check its spelling |
| `turbo run X` errors with "could not find task" | `X` isn't in `turbo.json` `tasks` | Add it |
| "Could not resolve workspaces" / "missing packageManager" | No `packageManager` field in the root package.json | Add `"packageManager": "pnpm@<version>"` |
| Env var is `undefined` only under turbo | Strict env mode | Add it to `env` / `passThroughEnv` / `globalEnv` |
| Always a cache miss | Generated files are tracked, or a changing env var is hashed | `--summarize` and diff two runs (section 12.3) |
| Cache "hit", but `dist/` is missing | `outputs` not declared | Add `outputs` |
| Downstream app breaks after a cache hit | Upstream compiled package's `outputs` incomplete | Include every generated file (`dist/**`, `*.d.ts`, …) |
| Two versions of zod/react in the bundle | Different ranges in different packages | Move them to the catalog. Run `pnpm why <pkg> -r` and `pnpm dedupe` |
| "Invalid hook call" from a shared UI package | Two copies of React | Make `react` a `peerDependency` of the UI package; same catalog version everywhere; consider `injected` (section 7.4) |
| Native module error after install (`esbuild`, `sharp`, `prisma`) | Its install script was blocked | Allow it (`pnpm approve-builds`), then reinstall |
| `pnpm deploy` runs pnpm's deploy instead of my script | Script name collides with a pnpm built-in command | `pnpm run deploy`, or rename the script |
| A dev server is waited on forever | Something `dependsOn` a `persistent` task | Persistent tasks can't be dependencies. Use `with` instead |
| A prompt hangs | Task isn't `interactive`, or the TUI pane isn't selected | `interactive: true` + `cache: false`, then select the pane in the TUI and type |
| `node dist/index.js` fails with `.ts` import errors | JIT package not bundled into a Node app | Bundle with `noExternal`, or make the package compiled (section 15) |
| Next.js fails to compile a workspace package | JIT package not transpiled | `transpilePackages: ["@acme/ui"]` |
| Tailwind classes from `@acme/ui` missing in the app | Tailwind doesn't scan the package | Add an `@source` (v4) / `content` (v3) entry for the package |
| Generator created a nested lockfile | Framework CLI ran its own install | Delete it, then run `pnpm install` from the root |
| CI: `ERR_PNPM_OUTDATED_LOCKFILE` | `package.json` changed without updating the lockfile | Run `pnpm install` locally and commit `pnpm-lock.yaml` |
| `--affected` runs everything in CI | Shallow clone, or base branch not fetched | `fetch-depth: 0` / `fetchDepth: 0`; set `TURBO_SCM_BASE` if needed |
| `corepack: command not found` (Node 25+) | Node no longer bundles Corepack | `npm i -g corepack`, or install pnpm directly (`npm i -g pnpm`) |
| `node_modules` weirdness after big changes | Stale links | `rm -rf node_modules **/node_modules && pnpm install` |

---

## 24. Further reading

**Turborepo**
- Docs home: https://turborepo.com/docs
- Crafting your repository (best end-to-end tutorial): https://turborepo.com/docs/crafting-your-repository
- `turbo.json` reference: https://turborepo.com/docs/reference/configuration
- `turbo run` flags & filter syntax: https://turborepo.com/docs/reference/run
- Internal packages: https://turborepo.com/docs/core-concepts/internal-packages
- Caching & remote caching: https://turborepo.com/docs/crafting-your-repository/caching
- Environment variables: https://turborepo.com/docs/crafting-your-repository/using-environment-variables
- Developing applications (dev tasks, `with`, `watch`): https://turborepo.com/docs/crafting-your-repository/developing-applications
- Constructing CI: https://turborepo.com/docs/crafting-your-repository/constructing-ci
- Docker guide (`turbo prune`): https://turborepo.com/docs/guides/tools/docker
- TypeScript guide: https://turborepo.com/docs/guides/tools/typescript
- Testing guide (Vitest, Jest, Playwright): https://turborepo.com/docs/guides/tools
- Boundaries: https://turborepo.com/docs/reference/boundaries
- Generators (`turbo gen`): https://turborepo.com/docs/guides/generating-code
- Example repos: https://github.com/vercel/turborepo/tree/main/examples

**pnpm**
- Workspaces: https://pnpm.io/workspaces
- Catalogs: https://pnpm.io/catalogs
- Filtering: https://pnpm.io/filtering
- `pnpm-workspace.yaml` settings: https://pnpm.io/settings
- `pnpm deploy`: https://pnpm.io/cli/deploy
- Why pnpm's node_modules is strict: https://pnpm.io/symlinked-node-modules-structure

**TypeScript / Node module resolution**
- `package.json` `exports`: https://nodejs.org/api/packages.html#package-entry-points
- TS modules reference (`moduleResolution`): https://www.typescriptlang.org/docs/handbook/modules/reference.html
- TS project references: https://www.typescriptlang.org/docs/handbook/project-references.html
- TSConfig bases: https://github.com/tsconfig/bases

**Tools mentioned in this guide**
- Hono: https://hono.dev
- tsdown (library bundler): https://tsdown.dev
- tsx: https://tsx.is
- Vite: https://vite.dev
- Vitest: https://vitest.dev
- ESLint (flat config): https://eslint.org/docs/latest/use/configure/
- Biome: https://biomejs.dev · Oxc (Oxlint/Oxfmt): https://oxc.rs
- Drizzle ORM: https://orm.drizzle.team
- Zod: https://zod.dev
- Wrangler / Cloudflare Workers: https://developers.cloudflare.com/workers
- Lefthook (git hooks): https://lefthook.dev
- knip: https://knip.dev

**Publishing packages**
- Changesets: https://github.com/changesets/changesets
- publint: https://publint.dev
- Are the Types Wrong?: https://arethetypeswrong.github.io
