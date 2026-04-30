# AGENTS.md

Operational guide for AI coding agents working in this repository. Read this first.

> Companion docs: see [`architecture.md`](./architecture.md) for a deep dive into the system design and [`README.md`](./README.md) for end-user docs.

---

## 1. Project at a Glance

- **What it is:** An MCP (Model Context Protocol) server that exposes Sonarr, Radarr, Lidarr, Prowlarr, and TRaSH Guides as tools.
- **Language / runtime:** TypeScript, pure ESM, Node >=18.
- **Distribution:** npm package (`mcp-arr-server`, bin `mcp-arr`) and multi-arch Docker image (`ghcr.io/aplaceforallmystuff/mcp-arr`).
- **Transports:** `stdio` (default) and `http` (Streamable HTTP, stateless per-request).

---

## 2. Source Layout (Authoritative)

> ⚠️ `CLAUDE.md` describes a `src/tools/` folder and `src/types.ts`. **That layout does not exist.** Trust this file instead.

```
src/
├── index.ts          # Server entrypoint. ALL tool defs + dispatcher + transports (~2500 lines)
├── arr-client.ts     # *arr REST clients (Sonarr/Radarr/Lidarr/Prowlarr) + all *arr types (~860 lines)
└── trash-client.ts   # TRaSH Guides client + in-memory cache (~420 lines)

test/
└── http-transport.test.mjs   # Only test file (node:test, ESM)
```

Other relevant files:

| File | Purpose |
|---|---|
| `package.json` | Scripts, deps. `"type": "module"`, bin → `dist/index.js`. |
| `tsconfig.json` | `module: NodeNext`, `target: ES2022`, `strict: true`, out → `dist/`. |
| `server.json` | MCP registry manifest (version + env var declarations). |
| `tools.json` | Static catalog snapshot. **Not loaded at runtime.** Update when tools change. |
| `Dockerfile` | Multi-stage alpine build, runs as non-root `nodejs` user. |
| `docker-compose.yml` | Defaults to HTTP transport on port 3000. |
| `.github/workflows/` | `ci.yml` (build + typecheck on 18/20/22), `release.yml` (npm + GHCR + GH release on tag). |

---

## 3. Required Reading Before Editing

1. `src/index.ts` lines 85–890 — how `TOOLS[]` is conditionally built per configured service.
2. `src/index.ts` lines 1136–2422 — the giant `switch (name)` dispatcher you'll usually be touching.
3. `src/arr-client.ts` lines 382–531 — `ArrClient` base class and the shared `request<T>()` (line 398).
4. `src/trash-client.ts` lines 224–418 — `TrashClient` and its caching semantics.

---

## 4. Tool Conventions

- **Naming:** `{service}_{action}` snake_case. Examples: `sonarr_get_series`, `radarr_add_movie`, `lidarr_search_album`, `prowlarr_test_indexers`, `trash_list_profiles`.
- **Cross-service / generic:** `arr_status`, `arr_search_all`, `search`, `fetch`.
- **Schemas:** Inline JSON Schema (no Zod). Always `type: "object" as const`, with `properties` and `required`.
- **Conditional registration:** Wrap pushes to `TOOLS` in `if (clients.<service>)` so the tool list shrinks when env vars are missing.
- **TRaSH tools** are always registered (no *arr config needed) — except `trash_compare_*` which require the corresponding *arr client.

### Adding a new tool — checklist

1. Add the `Tool` definition to `TOOLS` in `src/index.ts` (inside the right `if (clients.X)` block).
2. Add a `case "<tool_name>":` in the `CallToolRequestSchema` dispatcher.
3. Cast args inline: `const a = args as { foo: string; bar?: number }`.
4. Guard config: `if (!clients.X) throw new Error("X not configured")`.
5. Wrap response with `jsonText(data)` (index.ts:918) for success or `textError(msg)` (index.ts:924) for handled errors.
6. Update `README.md` "Available Tools" section.
7. Update `tools.json` (registry catalog snapshot).
8. Add a CHANGELOG.md entry under `[Unreleased]`.

---

## 5. Service Client Conventions

- **HTTP:** Built-in global `fetch`. **Do not add axios, got, undici, etc.**
- **Auth:** `X-Api-Key` header injected by `ArrClient.request()` (arr-client.ts:402). Never set it elsewhere.
- **Base URL:** `${config.url}/api/${apiVersion}${endpoint}`. Trailing slashes stripped in the constructor.
- **API versions:** Sonarr/Radarr → `v3` (default). Lidarr → `v1` (set in `LidarrClient` constructor, line 701). Prowlarr → `v1` (line 819).
- **Errors:** `request()` throws `Error("${serviceName} API error: ${status} ${statusText} - ${text}")`. Never expose raw upstream payloads to the user.

---

## 6. Error Handling

- The dispatcher has an outer try/catch (index.ts:1139, 2415–2421) that converts thrown errors to MCP error responses. You usually just `throw new Error(...)`.
- For known/expected validation failures, return `textError("...")` instead — it sets `isError: true` without throwing.
- **Never log to stdout.** Stdout is reserved for the MCP stdio protocol. Use `console.error` only.

---

## 7. Response Conventions

- Wrap success payloads with `jsonText(data)` — pretty-printed JSON inside a `text` content block.
- **Pagination defaults:** library lists use `limit=25` (max `100`) + `offset`. Search results sliced to 10. Custom format lists capped at 50 with a `note`.
- Queue responses go through `getPaginatedQueue()` (index.ts:1083) → `{ total, returned, offset, limit, hasMore, nextOffset, items }`.

---

## 8. TRaSH Guides

- Data source: `raw.githubusercontent.com/TRaSH-Guides/Guides/master/docs/json` (NOT `trash-guides.info` despite what `CLAUDE.md` says).
- Discovery via GitHub Contents API (`api.github.com`).
- Cache: 1-hour TTL, in-memory `Map`s (trash-client.ts:91–176). Singleton `trashClient` exported.
- Custom format fetches are batched in groups of 20 to dodge GitHub rate limits.
- Naming key map: `plex` → `plex-imdb`, `emby` → `emby-imdb`, `jellyfin` → `jellyfin-imdb`, `standard` → `default`/`standard`.

---

## 9. Build, Run, Test

```bash
# Build (required before run/test)
npm run build

# Watch mode
npm run watch

# Tests (builds first, then runs node --test)
npm test

# Local stdio run (needs env vars set)
node dist/index.js

# Local HTTP run
$env:MCP_TRANSPORT="http"; node dist/index.js   # PowerShell
MCP_TRANSPORT=http node dist/index.js           # bash
```

CI runs on Node 18.x / 20.x / 22.x with `npm ci`, `npm run build`, `npx tsc --noEmit`. **Your changes must pass `tsc --noEmit`.**

### Test framework

- `node:test` + `node:assert/strict`. **No Jest, no Vitest.**
- Tests live as `.mjs` (raw ESM, no TS compilation needed for tests).
- Current coverage is intentionally minimal — only the HTTP transport regression in `test/http-transport.test.mjs`. When fixing transport bugs, extend that file.

---

## 10. Hard Rules

```yaml
rules:
  - id: esm-import-extensions
    rule: All relative imports MUST end in ".js" (NodeNext resolution), even from .ts files.
    example: import { ArrClient } from "./arr-client.js";

  - id: no-new-runtime-deps
    rule: Do NOT add runtime deps without explicit approval. The only runtime dep is @modelcontextprotocol/sdk.

  - id: no-stdout-logging
    rule: Never console.log. Stdio transport uses stdout for MCP frames. Use console.error.

  - id: env-only-config
    rule: Service URLs and API keys come from env vars only. Never hardcode.

  - id: keep-versions-in-sync
    rule: |
      When bumping version, update ALL of:
        - package.json "version"
        - server.json "version"
        - src/index.ts SERVER_VERSION constant (line ~34)
        - CHANGELOG.md

  - id: tool-naming
    rule: Tool names follow {service}_{action} snake_case. service ∈ {sonarr,radarr,lidarr,prowlarr,trash,arr}.

  - id: never-expose-raw-api-errors
    rule: Use the controlled error string from ArrClient.request(); do not pass upstream JSON through.

  - id: dispatcher-pattern
    rule: All tool calls go through the single switch in src/index.ts. Do not add a second registration site.

  - id: no-tools-folder
    rule: Do NOT create src/tools/*.ts unless the maintainer explicitly asks for a refactor. The flat layout is intentional.
```

---

## 11. Common Pitfalls (Observed)

- **Forgetting `.js` extensions** on imports → TS compiles fine but `node` fails at runtime.
- **Adding tools without the `if (clients.X)` guard** → tool advertised to clients that have no service configured.
- **Returning huge unbounded lists** → respect the `limit`/`offset` convention.
- **Editing `CLAUDE.md` to "fix" the architecture description** → it's stale; prefer updating `architecture.md` and `AGENTS.md`. Only touch `CLAUDE.md` if asked.
- **Forgetting to update `tools.json`** → it's the registry catalog snapshot.

---

## 12. Release Flow (For Reference Only — Don't Run Without Asking)

1. Update `CHANGELOG.md` (move `[Unreleased]` → version section).
2. Bump versions in `package.json`, `server.json`, `src/index.ts` (`SERVER_VERSION`).
3. `npm version patch|minor|major` (or manual + `git tag vX.Y.Z`).
4. `git push && git push --tags` → `release.yml` workflow handles npm + GHCR + GitHub Release.

---

## 13. When You're Stuck

- For exploration, prefer the Task tool over multiple Grep/Glob round-trips.
- The dispatcher in `src/index.ts` is the source of truth for behavior — read the matching `case` before assuming intent.
- If `CLAUDE.md` and `AGENTS.md` disagree, **`AGENTS.md` wins**.
