---
description: Audit this project for Bun-native commands, dependencies, and APIs
---
Audit the current project for Bun migration. Read `package.json`, its scripts, lockfiles, config files, and source imports before answering. Do not edit files.

Rules:
- Report only findings backed by this project. No generic Bun introduction.
- Default to removing the old tool and porting to Bun. Name an exception only when the project actually needs it.
- One line per finding. If the project is already Bun-native, say so and stop.
- Repo-relative paths, exact package names, and exact commands only.

## Command replacements

| Current | Bun |
|---|---|
| `npm install`, `yarn`, `pnpm install` | `bun install` |
| `npm ci` | `bun ci` |
| `npm add`, `yarn add`, `pnpm add` | `bun add` |
| `npm install -D`, `yarn add -D` | `bun add -d` |
| `npm remove`, `yarn remove`, `pnpm remove` | `bun remove` |
| `npm update`, `yarn upgrade`, `pnpm update` | `bun update` |
| `npm audit`, `npm outdated` | `bun audit`, `bun outdated` |
| `npm run <script>`, `yarn <script>`, `pnpm <script>` | `bun run <script>` |
| `npx <pkg>`, `yarn dlx <pkg>`, `pnpm dlx <pkg>` | `bunx <pkg>` |
| `node <file>`, `ts-node <file>`, `tsx <file>` | `bun <file>` |
| `nodemon <file>`, `tsx watch <file>`, `node --watch <file>` | `bun --watch <file>` |
| server reload with `nodemon` | `bun --hot <file>` |
| `tsc --noEmit` | `bun check` |
| `tsc -b` | `bun check -b` |
| `tsc --noEmit -p <dir>` | `bun check -p <dir>` |
| `tsc --noEmit && bun test` | `bun test --check` |
| `tsc --noEmit && bun build <entry>` | `bun build --check <entry>` |
| `tsc --emitDeclarationOnly` | `bun build --dts` |
| `jest`, `vitest` | `bun test` |
| `prettier --write <paths>` | `bun fmt <paths>` |
| `prettier --check <paths>` | `bun fmt --check <paths>` |
| `eslint <paths>` | `bun lint <paths>` — direct replacement, except when your Bun build lacks lint |
| `webpack`, `esbuild`, `rollup`, `tsup` | `bun build` or `Bun.build` |
| `concurrently` across workspace scripts | `bun --filter '<pattern>' <script>` — scripts only, not arbitrary commands |
| `dotenv -e <file> -- <cmd>` | `bun --env-file=<file> <file-or-script>` |
| `cross-env FOO=bar <cmd>` | `FOO=bar <cmd>` |
| `rimraf <path>` | `rm -rf <path>` |
| `http-server`, `serve <dir>` | `Bun.serve` with `Bun.file` or `routes` |

Notes:
- Bun loads `.env`, `.env.local`, and mode-specific `.env.*` files automatically.
- Package scripts run through Bun Shell on Windows, so `FOO=bar`, `rm -rf`, pipes, and redirects stay cross-platform there.
- `bun check` uses TypeScript 7 defaults: `strict` on, no implicit `types`, and removed `tsconfig` options are errors.
- `bun build --dts` writes `.d.ts` without `tsc`; it type checks first and fails on type errors. It cannot be combined with `--compile`.
- `bun test` is Jest-compatible. Import from `bun:test`; tests match `*.test.*`, `*.spec.*`, `*_test.*`, and `*_spec.*`. Safe port, except when tests rely on Vitest-only APIs (`vi`, workspaces) or custom Vitest config; rewrite those first.
- `bun build` is a direct replacement for bundling, except when you need Vite's plugin ecosystem or framework transforms.
- `bun lint` is a direct replacement, except when your Bun build lacks `bun lint`.

## Dependency replacements

| Package | Bun-native | Migration |
|---|---|---|
| `dotenv`, `dotenv-expand`, `dotenv-cli` | automatic `.env` loading | direct replacement; remove |
| `ts-node`, `tsx`, `ts-node-dev`, `nodemon` | `bun <file>`, `bun --watch`, `bun --hot` | remove after scripts switch |
| `jest`, `vitest`, `ts-jest`, `@types/jest` | `bun:test` | direct replacement; rewrite imports, except when tests use Vitest-only APIs |
| `prettier` | `bun fmt` | direct replacement; remove, except when using Prettier plugins `bun fmt` does not load |
| `eslint`, `oxlint` | `bun lint` | direct replacement, except when your Bun build lacks `bun lint` |
| `typescript` | `bun check`, `bun build --dts` | replace type checking and declaration emit; keep only if the editor needs project TypeScript |
| `uuid` | `crypto.randomUUID()`, `Bun.randomUUIDv7()` | v7 needs `Bun.randomUUIDv7()` |
| `better-sqlite3`, `sqlite3` | `bun:sqlite` | `better-sqlite3` API is close; rewrite driver calls |
| `pg`, `postgres`, `mysql2`, `mariadb` | `Bun.SQL` | rewrite to tagged-template SQL API |
| `ioredis`, `redis` | `Bun.redis` / `Bun.RedisClient` | direct replacement, except when you need Redis <7.2, Sentinel, or Cluster |
| `ws` | built-in `WebSocket`; `Bun.serve({ websocket })` | rewrite server/client handlers |
| `sharp` | `Bun.Image` | direct replacement for resize, rotate, flip, modulate, metadata, and JPEG/PNG/WebP/HEIC/AVIF output; keep only for SVG input, compositing, or formats outside those codecs |
| `node-fetch`, `cross-fetch`, `whatwg-fetch`, `isomorphic-fetch` | built-in `fetch`, `Request`, `Response`, `Headers`, `FormData` | direct replacement; remove the polyfill |
| `undici` | built-in `fetch` | direct replacement for fetch; keep only for `MockAgent`, `ProxyAgent`, or other non-fetch APIs |
| `execa`, `shelljs` | `Bun.$` | rewrite process calls |
| `bcrypt`, `bcryptjs`, `argon2` | `Bun.password` | verify existing hashes; bcrypt uses Modular Crypt Format |
| `js-yaml`, `yaml` | `Bun.YAML.parse` | direct replacement for parsing; keep for YAML stringify |
| `json5` | `Bun.JSON5.parse`, `Bun.JSON5.stringify` | direct API swap |
| `@iarna/toml`, `smol-toml`, `toml` | `Bun.TOML.parse`, `Bun.TOML.stringify` | direct API swap |
| `semver` | `Bun.semver.satisfies`, `Bun.semver.order` | direct replacement for those two functions; keep only for other semver helpers |
| `node-cron` | `Bun.cron(schedule, handler)` | in-process scheduler; rewrite registration |

## Exceptions

Default is removal. Port first; keep a package only when one of these is true:

- A framework build plugin does the work: Vite, Next.js, Nuxt, Astro, SvelteKit. Run them on Bun; `bun build` replaces the bundler, not the framework.
- An HTTP framework owns routing or middleware: `express`, `fastify`, `koa`, `hono`. Port the routes to `Bun.serve`, then remove it.
- Your Bun build lacks `--dts`, you need TypeScript project references, or the editor needs project TypeScript.
- A listed Bun API lacks the specific function you use.
- The package has no import or use in the scanned source.

## Output

Return a short report with these sections, omitting empty ones:

1. **Commands** — `script name`: `old` → `new`, with the reason if not obvious.
2. **Remove** — package, why, and the file or script that proves it.
3. **Replace** — package → Bun API, with the code change needed.
4. **Check** — only real exceptions, not generic warnings.
5. **Next** — exact commands to run, in order.
