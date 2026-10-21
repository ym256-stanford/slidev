# Slidev repository guide

Slidev turns Markdown into interactive, Vue-powered presentations for developers. This repository contains the CLI, browser client, Markdown parser, shared types, scaffolding tools, VS Code extension, docs, and example decks; the root is a pnpm/Turborepo workspace, not a standalone slide deck.

## Architecture

- `packages/slidev/` is the Node-side `@slidev/cli`: `bin/slidev.mjs` loads the build of `node/cli.ts`; `node/commands/serve.ts` starts Vite and `node/commands/build.ts` builds static slides. `node/index.ts` is the library API.
- `packages/client/` is the Vue presentation UI: the CLI supplies `index.html`, which loads `main.ts` and mounts `App.vue`. `index.ts` is the public client API, **not** the browser entry.
- `packages/parser/src/core.ts` parses slide Markdown; `src/fs.ts` loads files. `packages/types/src/index.ts` collects shared contracts. Keep Node/Vite code in `slidev`, UI in `client`, parsing in `parser`.
- `packages/create-app/` and `packages/create-theme/` provide templates and `index.mjs` scaffolding CLIs; `packages/vscode/src/index.ts` activates the extension, with `language-server/bin.ts` as its other build entry.
- `demo/starter/slides.md` is the default local deck. `docs/` is the VitePress site; `test/` holds broader Vitest tests, and `cypress/e2e/` holds browser tests.

## Commands

Run from the repository root with Node.js >= 22.12.0. `package.json` pins pnpm 12.4.2:

```bash
npm install --global pnpm@12.4.2  # Skip if this version is already available
pnpm install --frozen-lockfile
pnpm build                         # Build packages/* in dependency order
pnpm test                          # Vitest; requires pnpm build first
pnpm typecheck
pnpm lint
pnpm verify                        # build -> typecheck -> lint -> test
pnpm demo:dev                      # Start the starter deck after a build
pnpm docs:build                    # Separately build docs and demo decks
```

Alternatively, avoid a global pnpm install with `corepack enable`; it exposes the version pinned by `packageManager`. If that shim location is not writable, use this isolated setup in the same shell before the `pnpm` commands above:

```bash
PNPM_SHIM_DIR=$(mktemp -d)
corepack enable --install-directory "$PNPM_SHIM_DIR"
export PATH="$PNPM_SHIM_DIR:$PATH"
```

For live package changes, run `pnpm dev` (watch builds) and `pnpm demo:dev` (presentation server) in separate terminals. For Cypress, build first, run `pnpm cy:fixture` in one terminal (port 3041), then `pnpm exec cypress run` in another. Cypress is **not** part of `pnpm verify`.

## Conventions

- `eslint.config.js` uses Antfu ESLint with Markdown, CSS, and Slidev formatters. Match nearby TS/JS: two-space indentation, single quotes, no semicolons, and `import type` for type-only imports; run `pnpm lint` rather than relying on the pre-commit `lint-staged` fixes.
- Use relative imports within a package and `@slidev/*` across packages. Manifests use `workspace:*` for sibling packages and `catalog:<group>` for shared dependency versions in `pnpm-workspace.yaml`.
- Vue UI generally uses `<script setup lang="ts">`, `ref`/`computed`, and `use*` composables. Keep small Vitest `*.test.ts` files beside their implementation; use `test/` for broader fixture/snapshot tests and `cypress/e2e/` for browser flows.

## Guardrails

- `pnpm dev` only watches package builds; it does **not** start the starter presentation. Running `pnpm test` before `pnpm build` fails when tests resolve workspace packages through `dist/`. Root `pnpm build` does **not** build docs or demos.
- Do not hand-edit generated/ignored outputs: `dist/`, `node_modules/`, `docs/.vitepress/dist/`, `packages/vscode/syntaxes/codeblock-patch.json`, or copied files under `packages/create-app/template/{pages,snippets}` and its `slides.md`. Edit `demo/starter/` for the copied template content.
- Regenerate tracked outputs from their sources: `packages/types/src/builtin-layouts.ts` via `scripts/gen-layouts.ts`; `packages/vscode/schema/*.json` via its `prepare` script from `packages/types/src/frontmatter.ts`; `packages/client/.generated/unocss-tokens.ts` via `packages/client/scripts/unocss-scan.ts`. Update `pnpm-lock.yaml` through pnpm and test snapshots through their tests, not by manual edits.
- `packages/slidev/README.md` and `packages/slidev/skills/` are publish-time copies of root `README.md` and `skills/`. `skills/slidev/` itself is manually maintained from `docs/` per `skills/GENERATION.md`, not an automatically regenerated directory.
