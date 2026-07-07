# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Headplane is a feature-complete web UI for [Headscale](https://headscale.net) (self-hosted Tailscale control server). It manages nodes, ACLs, DNS, users, and Headscale config. Also see `AGENTS.md` for project tenets (simple starts, no breaking changes, docs-first).

## Hybrid project: TypeScript app + Go binaries

This repo is **two stacks in one**:

- **React Router 7 (framework mode) app** under `app/` — the web UI and its SSR server. This is where ~all feature work happens.
- **Go binaries** under `cmd/` + `internal/`, sharing the root `go.mod`:
  - `hp_agent` — the Headplane Agent. Runs alongside Headplane, joins the Tailnet via `tsnet`, and pulls node details (versions, etc.) not exposed by the Headscale API.
  - `hp_ssh` — **WASM** WebSSH shim. Compiled to `public/hp_ssh.wasm` (`GOOS=js GOARCH=wasm`); runs in the browser to SSH into Tailnet nodes ephemerally.
  - `hp_healthcheck` — Docker healthcheck binary (probes the URL written to `HEADPLANE_LISTEN_FILE`).
  - `fake_sh` — fake shell binary for the Docker image.

## Commands

Tooling versions are pinned in `mise.toml` (Go 1.25.1, pnpm 10.4.0, Node 24.2) and enforced by `package.json` `engines`. **Use pnpm, never npm/yarn** (`preinstall` enforces this).

```sh
pnpm dev               # run app + docs dev servers together (parallel /^dev:/)
pnpm dev:app           # app only — uses config.example.yaml, pino-pretty logs
pnpm dev:docker        # docker compose up (Headscale + Caddy dev stack)
pnpm build             # react-router build (TS app only)
pnpm typecheck         # react-router typegen && tsc  (uses TypeScript-Go / tsc 7 rc)
pnpm lint              # oxlint   (pass --fix to autofix)
pnpm format            # oxfmt
pnpm test:unit         # vitest unit project
pnpm test:integration  # vitest integration:* projects (testcontainers — needs Docker)
pnpm docs:dev          # VitePress docs site (docs/)
```

Run a single test with vitest's filters, e.g. `pnpm test:unit -t "name"` or `vitest run --project unit path/to/file.test.ts`.

Build the Go pieces via `./build.sh` (see `./build.sh --help`): `--wasm`, `--agent`, `--healthcheck`, `--fake-shell`. Default (no flags) builds wasm + app + agent + healthcheck. WASM build vendors deps and applies `patches/tailscale-derp-port.patch` (Tailscale's WASM ignores non-443 DERP ports).

`docker exec headscale headscale <command>` runs Headscale CLI commands against the dev stack.

## App architecture (`app/`)

**Routing** is config-based in `app/routes.ts` (not file-system). Most authenticated pages live under the `layout/app.tsx` layout. Feature routes are grouped by domain folders: `routes/machines`, `routes/acls`, `routes/dns`, `routes/users`, `routes/settings`, `routes/auth`, `routes/ssh`. API/utility endpoints under `/api/*`, `/events/*`, `/healthz`.

**Server code lives in `app/server/` and runs only in Node** (never the browser). Read `app/server/README.md` first — it is authoritative. Key structure:

- **Two SSR entries**, selected by *which file loads*, not by runtime `if (PROD)` branches:
  - `app.ts` — the application module (loads config → builds context → exports React Router `RequestListener` as default). Used by Vite dev (`runtime/vite-plugin.ts`) and bundled by prod.
  - `main.ts` — production bootstrap. Wraps the listener with `runtime/http.ts` (`composeListener` / `startHttpServer`): static assets from `build/client/`, basename redirect. Run as `node build/server/index.js`.
- **`context.ts` → `createAppContext(config)`** constructs all process-lifetime services: `db` (Drizzle/SQLite at `<data_path>/hp_persist.db`), `headscale` (REST client), `agents` (agent manager), `auth`, `oidc`, `hsLive` (live store), `hs` (parsed Headscale config), `integration`. Route handlers consume these through **named React Router contexts** (`authContext`, `headscaleContext`, etc.), e.g. `const auth = context.get(authContext)` in a loader/action. When adding a process-lifetime service, construct it in `context.ts`, expose a context, and seed it in `app.ts`'s `getLoadContext`.
- `app/server/headscale/api/` — Headscale REST client; per-domain resources in `resources/` (nodes, users, policy, api-keys, pre-auth-keys). `capabilities.ts` gates behavior on Headscale version.
- `app/server/config/` — YAML config loading + schema (arktype/valibot). **Env vars override the file**: any `HEADPLANE_*` var, with `__` for nesting (e.g. `HEADPLANE_SERVER__PORT=8080`). `config.example.yaml` is the reference; `__PREFIX__` (basename, default `/admin`) is a build-time define.
- `app/server/config/integration/` — Headplane can manage the Headscale process to apply config/restart. Exactly one integration may be enabled: `docker`, `kubernetes`, or `proc`.
- `app/server/web/` — auth service, identity, RBAC roles. Supports OIDC and proxy-auth (header-based).

**Client-side**: `app/components` (shared UI), `app/routes/**/components` & `dialogs` (feature-local), `app/hooks`, `app/utils`. Styling is Tailwind v4 (`app/tailwind.css`). Imports use the `~` alias → `app/`.

## Tests (`tests/`)

Vitest projects (see `vitest.config.ts`): `unit` (`tests/unit/`), and integration projects `integration:api`, `integration:platform`, `integration:oidc` (`tests/integration/`). Integration tests use **testcontainers** and need a working Docker daemon; timeouts are 60s.

## Conventions

- Commit hooks (lefthook): staged JS/TS gets `oxlint --fix`, everything gets `oxfmt`, Go files get `gofmt -w`. Don't bypass hooks.
- Docs are first-class — update `docs/` (VitePress) when changing staple features. NixOS module docs regenerate via `mise run nixos-docs`.
- Semantic versioning; pre-release builds publish under the `next` tag.
