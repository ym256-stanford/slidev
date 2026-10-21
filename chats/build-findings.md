# Building, testing, linting, and running Slidev

All commands below run from the repository root. Install Node.js >= 22.12.0 first, then:

```bash
cd /Users/yuliiamurakami/Desktop/cs146s/slidev
npm install --global pnpm@12.4.2
pnpm install --frozen-lockfile
```

The root `package.json` specifies pnpm 12.4.2. If pnpm is already available at that version, skip the global installation. Another option is to enable Corepack's pnpm shim (`corepack enable`) and then use the same `pnpm` commands.

## Build

```bash
pnpm build                    # Build workspace packages
pnpm docs:build               # Build documentation and demo presentations
pnpm -C demo/starter build    # Build the starter slide deck
```

The root `build` script is `turbo run build --filter="./packages/*"`. It builds packages under `packages/` in dependency order (`turbo.json` declares `^build`); it does **not** build the documentation or demo presentations. The starter deck's `build` script runs `slidev build`. For any Slidev presentation, `slidev build` produces a static site in `dist/` by default.

## Test

```bash
pnpm build        # Required before running tests directly
pnpm test         # Run Vitest tests
pnpm typecheck    # Run vue-tsc --noEmit
pnpm verify       # Full local gate: build, typecheck, lint, test
```

The `test` script is `vitest test`; `vitest.config.ts` selects test projects under `packages/*` and `test`. Tests need the prior build because workspace packages resolve to their built `dist/`. `pnpm verify` performs that build first.

For Cypress end-to-end tests, after installing dependencies and building, keep the fixture server running in one terminal:

```bash
pnpm cy:fixture
```

In a second terminal, run either:

```bash
pnpm exec cypress run  # Headless end-to-end tests
pnpm cy                # Interactive Cypress UI (alternative)
```

`cy:fixture` starts the fixture on port 3041, matching `cypress.config.ts`'s `http://localhost:3041` base URL. Cypress tests are separate from `pnpm verify`.

## Lint

```bash
pnpm lint       # Run eslint . --cache
pnpm lint:fix   # Run the lint script with --fix
```

Linting includes formatting rules from `eslint.config.js`. The fix command modifies files.

## Run locally

```bash
pnpm build                  # First build the workspace packages
pnpm demo:dev               # Start the starter presentation
```

For development, use separate terminals:

```bash
pnpm dev                    # Terminal 1: watch/rebuild packages, not a web server
pnpm demo:dev               # Terminal 2: run the starter presentation
```

Other available development commands are `pnpm demo:composable-vue` (example presentation), `pnpm demo:vue-runner` (Vue Runner presentation), `pnpm docs` (VitePress documentation server), and `pnpm vscode:dev` (VS Code extension watch build). `pnpm play` aliases `pnpm demo:dev`. Demo scripts use `nodemon` to restart the Slidev server when the built CLI changes.

## Major directories and entry points

The root is a pnpm/Turborepo workspace, not a standalone web app. `pnpm-workspace.yaml` includes `packages/*`, `demo/*`, `cypress/fixtures/*`, and `docs`; root `package.json` and `turbo.json` coordinate builds and checks.

| Directory | Role and entry points |
| --- | --- |
| `packages/slidev/` | Node-side `@slidev/cli`. The executable is `bin/slidev.mjs`, which loads the compiled `dist/cli.mjs` from `node/cli.ts`. That file defines the default dev-server command and `build`, `export`, and other subcommands; `node/commands/serve.ts` creates the Vite server and `node/commands/build.ts` builds the site. `node/index.ts` is the programmatic API entry, exporting the server, parser, options, and Vite plugin. |
| `packages/client/` | Vue presentation app served by the CLI. The HTML template `index.html` receives the `main.ts` entry; `main.ts` creates and mounts `App.vue`. `index.ts` is its public API, separate from the browser startup file. Components and behavior live in `pages/`, `layouts/`, `builtin/`, `composables/`, `state/`, and `setup/`. |
| `packages/parser/` | Slide Markdown parsing. `src/index.ts` re-exports `src/core.ts`; `src/fs.ts` supplies file-aware parsing, and `tsdown.config.ts` builds separate `index`, `core`, `fs`, and `utils` entry points. The CLI imports its file-aware API through `node/parser.ts`. |
| `packages/types/` | Shared types for the CLI, parser, and client. Build entry `src/index.ts` gathers type modules; `index.d.ts` and `client.d.ts` provide declarations. |
| `packages/create-app/` and `packages/create-theme/` | Project and theme scaffolding packages. Each has an executable `index.mjs` and a `template/` directory, for `npm init slidev` and `npm init slidev-theme` respectively. |
| `packages/vscode/` | VS Code extension. `src/index.ts` exports `activate`/`deactivate`; `language-server/bin.ts` launches its language server. `tsdown.config.ts` bundles both entries. |
| `demo/` | Example presentations (`starter/`, `composable-vue/`, `vue-runner/`), each with a `slides.md` entry and its own package scripts. The starter deck is what `pnpm demo:dev` runs. |
| `docs/` | VitePress documentation site. `docs/.vitepress/config.ts` configures the site; `docs/package.json` provides its dev and build scripts. |
| `test/` and `cypress/` | Root Vitest tests and Cypress end-to-end specs/fixtures respectively. `vitest.config.ts` also includes tests under `packages/*`; `cypress.config.ts` configures the E2E runner. |
| `scripts/`, `.github/workflows/` | Repository automation (for example, demo build and package publishing) and CI workflows. |
| `assets/`, `patches/`, `skills/`, `plans/` | Shared branding assets, a pnpm dependency patch, Slidev skill materials, and implementation-plan notes. |

The main runtime path is `packages/slidev/bin/slidev.mjs` → `packages/slidev/node/cli.ts` → Vite serve/build commands → the `packages/client/index.html` template → `packages/client/main.ts` → `App.vue`. The CLI uses `@slidev/parser` to read slide Markdown and `@slidev/types` for shared contracts. `packages/slidev/node/setups/indexHtml.ts` connects the client's HTML template and browser entry to the generated page.

## Code and contribution conventions

### Stated in documentation or enforced by configuration

- **Use pnpm and the workspace dependency conventions.** `package.json` pins pnpm 12.4.2; `pnpm-workspace.yaml` defines the workspaces and version catalogs. Package manifests reference sibling packages as `workspace:*` and many shared external versions as `catalog:<group>` (for example, `catalog:prod` or `catalog:frontend`). Evidence: `package.json`, `pnpm-workspace.yaml`, `packages/slidev/package.json`.
- **Let ESLint handle formatting.** `eslint.config.js` uses `@antfu/eslint-config` with pnpm, Markdown, CSS, and Slidev-slide formatters. It excludes `skills/**/*.md` and `plans/**`, adjusts a few rules, and exempts scaffolding manifests and the VS Code manifest from one pnpm package rule. The contributor guide says installed Git hooks will format/fix code on commit. `package.json` configures a pre-commit `lint-staged` hook that applies `eslint --fix --cache` to staged JS, TS, Vue, and JSON files. Evidence: `CONTRIBUTING.md` § Code Style, `eslint.config.js`, `package.json`.
- **Keep types checked strictly.** Root `tsconfig.json` has `strict`, `strictNullChecks`, `noUnusedLocals`, `noEmit`, and package-name path mappings; TypeScript and Vue files are included across packages, demos, docs, and tests. The default `tsdown.config.ts` emits ESM and declarations for packages using it, while the VS Code extension has its own CommonJS build configuration. Evidence: `tsconfig.json`, `tsdown.config.ts`, `packages/vscode/tsdown.config.ts`.
- **Run the full gate before a PR.** `pnpm verify` runs build → typecheck → lint → test; build must precede tests that resolve package `dist/` files. Cypress is a separate E2E suite. Documentation PRs belong in this repository even though docs are synced elsewhere. Evidence: `CONTRIBUTING.md` §§ Documentation and Before you open a PR, `package.json`, `vitest.config.ts`, `cypress.config.ts`.

### Repeated patterns inferred from the source, not written as universal rules

- **Formatting and imports:** TS/JS samples generally use two-space indentation, single quotes, no semicolons, and no trailing blank padding. Type-only dependencies use `import type`. Code within a package uses relative imports, while cross-package imports use `@slidev/*` names; the root TS path mappings provide source resolution. These are observed patterns, not a substitute for running ESLint. Examples: `packages/slidev/node/options.ts`, `packages/parser/src/fs.ts`, `packages/client/composables/useNav.ts`.
- **Keep package roles distinct:** Markdown parsing stays in `packages/parser`, shared interfaces in `packages/types`, Node/Vite commands in `packages/slidev`, and presentation UI in `packages/client`. The parser even documents why it does not import the CLI package. Prefer an existing package boundary or public entry point to introducing a new cross-package shortcut. Evidence: `packages/parser/src/fs.ts`, `packages/slidev/node/index.ts`, `packages/client/index.ts`, `tsconfig.json`.
- **Use Vue's Composition API for the UI:** Vue components commonly use `<script setup lang="ts">`, composables named `use*`, and reactive `ref`/`computed` state. The app boots from `main.ts` and renders routes through `App.vue`. Examples: `packages/client/builtin/RenderWhen.vue`, `packages/client/pages/presenter.vue`, `packages/client/composables/useNav.ts`.
- **Place small behavior tests near the code; put broader tests at the root.** Files such as `packages/parser/src/utils.test.ts`, `packages/client/logic/slidePath.test.ts`, and `packages/vscode/src/utils/isSamePath.test.ts` exercise their neighboring modules with Vitest `describe`/`it`/`expect`. `test/parser.test.ts` uses Markdown fixtures and snapshots; `cypress/e2e/` holds browser flows. Add coverage at the appropriate level for the change. Evidence: those tests, `vitest.config.ts`, `cypress.config.ts`.
- **Favor explicit, typed boundaries and defensive cases.** Exports often give parameter/return types and separate type imports; path- and source-handling code checks invalid input and records actionable errors rather than trusting slide content. This is especially important where the server reads user files or the UI handles cross-platform paths. Examples: `packages/parser/src/fs.ts` (allowed roots, missing/circular imports), `packages/vscode/src/utils/isSamePath.test.ts`, `packages/slidev/node/options.ts`.
- **Comments tend to explain a non-obvious reason or compatibility constraint rather than narrate every step.** Examples include parser independence and backward compatibility in `packages/parser/src/fs.ts`, config-resolution rationale in `packages/slidev/node/options.ts`, and VS Code packaging exceptions in `eslint.config.js`. This is a tendency seen in sampled files, not a formal comment policy.
- **Commit subjects commonly use Conventional Commit-like prefixes and optional scopes.** Recent subjects include `fix: ...`, `feat: ...`, `test(e2e): ...`, `deps!: ...`, and `chore: ...`; the contributor guide does not prescribe a commit format. Evidence: recent `git log --format=%s`.

Sources: `package.json` (`packageManager`, `engines`, `scripts`, hooks), `pnpm-workspace.yaml`, `turbo.json`, `CONTRIBUTING.md`, `tsconfig.json`, `tsdown.config.ts`, `eslint.config.js`, `vitest.config.ts`, `cypress.config.ts`, `cypress/fixtures/basic/package.json`, `.github/workflows/test.yml`, `docs/package.json`, `docs/.vitepress/config.ts`, `scripts/demo.mjs`, `demo/starter/package.json`, `demo/composable-vue/package.json`, `demo/vue-runner/package.json`, `docs/guide/hosting.md`, and the package files and tests named above.

Environment at investigation time: Node.js v24.22.0 and Corepack 0.36.0 were available; `pnpm` was not on `PATH` and `node_modules` was absent. No install, build, test, lint, or server command was executed.
