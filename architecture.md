# Architecture

Deep dive into the design of `mcp-arr-server`. For the operational/agent guide see [`AGENTS.md`](./AGENTS.md). For end-user setup see [`README.md`](./README.md).

---

## 1. High-Level View

`mcp-arr` is a single-process Node.js MCP server that proxies five upstream data sources behind a unified tool interface:

```
┌─────────────────────┐
│   MCP Client        │  (Claude Desktop, IDE, custom)
│   (stdio or HTTP)   │
└─────────┬───────────┘
          │  JSON-RPC 2.0  (MCP protocol)
          ▼
┌──────────────────────────────────────────────────────────┐
│  mcp-arr-server (src/index.ts)                           │
│                                                          │
│   ┌────────────────────────────────────────────────┐     │
│   │  Transport layer                               │     │
│   │   • StdioServerTransport                       │     │
│   │   • StreamableHTTPServerTransport (stateless)  │     │
│   └────────────────────────────────────────────────┘     │
│                       │                                  │
│   ┌────────────────────────────────────────────────┐     │
│   │  Tool registry (TOOLS: Tool[])                 │     │
│   │   built conditionally from env vars            │     │
│   └────────────────────────────────────────────────┘     │
│                       │                                  │
│   ┌────────────────────────────────────────────────┐     │
│   │  Single dispatcher: switch(name) { ... }       │     │
│   └────────┬─────────────────┬──────────────┬──────┘     │
│            │                 │              │            │
│            ▼                 ▼              ▼            │
│   ┌──────────────┐  ┌─────────────┐  ┌────────────┐      │
│   │ ArrClient    │  │ TrashClient │  │ helpers    │      │
│   │ + subclasses │  │ + cache     │  │ jsonText/  │      │
│   │              │  │             │  │ pagination │      │
│   └──────┬───────┘  └──────┬──────┘  └────────────┘      │
└──────────┼─────────────────┼─────────────────────────────┘
           │ X-Api-Key       │ no auth
           ▼                 ▼
   ┌────────────────┐  ┌──────────────────────────────┐
   │ Sonarr/Radarr/ │  │ raw.githubusercontent.com /  │
   │ Lidarr/        │  │ api.github.com               │
   │ Prowlarr APIs  │  │ (TRaSH-Guides/Guides repo)   │
   └────────────────┘  └──────────────────────────────┘
```

---

## 2. Module Responsibilities

The codebase is intentionally **flat** — three source files, no subdirectories.

### `src/index.ts` (~2500 lines)

The single entrypoint and the only place where MCP behavior is wired up.

- **Configuration discovery (lines 41–82).** Reads env vars for each of the four *arr services into a `services` array. A service is "configured" iff both `<NAME>_URL` and `<NAME>_API_KEY` are set. Only configured services get an instance in the `clients` registry (`SonarrClient` / `RadarrClient` / `LidarrClient` / `ProwlarrClient`).
- **Tool catalog (lines 85–890).** Builds a `TOOLS: Tool[]` array. Always-on tools (`arr_status`, `search`, `fetch`, `arr_search_all`, `trash_*`) are pushed unconditionally; service-specific tools are inside `if (clients.X) { ... }` blocks. This means the visible tool surface mirrors the configuration — a Sonarr-only deployment never advertises Radarr tools.
- **Server creation (lines 893–903).** A single `Server` instance from `@modelcontextprotocol/sdk` with `capabilities: { tools: {} }`.
- **Helpers (lines 905–928).** `jsonText(data)` and `textError(msg)` shape MCP responses uniformly. `SearchEntry` is the type used by the unified `search`/`fetch` pair.
- **Unified search/fetch (lines 931–1081).** `runUnifiedSearch(query)` aggregates hits from TRaSH profiles + each configured *arr library; `fetchSearchEntry(id)` parses opaque IDs (`trash-profile:radarr:<name>`, `arr:sonarr:series:<tvdbId>`, etc.) and returns the full record.
- **Pagination helper (lines 1083–1128).** `getPaginatedQueue()` is the only piece of shared business logic between *arr clients.
- **Dispatcher (lines 1136–2422).** One `switch (name)` for every tool. Outer try/catch converts thrown errors into `{ content, isError: true }` MCP responses.
- **Transports (lines 2433–2500).** `main()` selects between `StdioServerTransport` and the HTTP server based on `MCP_TRANSPORT`.

### `src/arr-client.ts` (~860 lines)

Houses both the *arr REST client classes and all the *arr-related TypeScript interfaces. Co-locating types with the code that produces them avoids circular import patterns.

- **`ArrClient` base (line 382).** Constructor stores `config`, `serviceName`, `apiVersion` (default `'v3'`), strips the trailing slash from the URL. The protected `request<T>(endpoint, options?)` method (line 398) is the **only** point of HTTP I/O — it injects `X-Api-Key`, sets `Content-Type: application/json`, builds `${url}/api/${apiVersion}${endpoint}`, and converts non-2xx into a controlled `Error("${serviceName} API error: ${status} ${statusText} - ${text}")`. Shared methods on the base class cover endpoints common to v3 *arr APIs: status, queue, calendar, root folders, quality profiles, quality definitions, download clients, naming, media management, health, tags, indexers.
- **`SonarrClient extends ArrClient` (line 535).** TV-series specific endpoints: series CRUD, episode list/search, lookup, missing search, refresh.
- **`RadarrClient extends ArrClient` (line 629).** Movie-equivalent endpoints.
- **`LidarrClient extends ArrClient` (line 698).** Overrides `apiVersion = 'v1'`. Adds artist/album endpoints and `getMetadataProfiles`. Overrides `getCalendar` because Lidarr's response shape differs.
- **`ProwlarrClient extends ArrClient` (line 816).** `apiVersion = 'v1'`. Indexer-centric endpoints: list/test indexers, stats, cross-indexer search.

### `src/trash-client.ts` (~420 lines)

Caching reader for the [TRaSH-Guides/Guides](https://github.com/TRaSH-Guides/Guides) GitHub repo. **Despite being labelled "TRaSH Guides", the actual data source is GitHub raw + the GitHub Contents API**, not `trash-guides.info`.

- **`TrashCache` (line 91).** Seven typed `Map<string, CacheEntry<T>>` instances (profiles, profile lists, custom formats, CF lists, CF groups, quality sizes, naming) with a 1-hour TTL. `clearCache()` invalidates everything; `isValid()` short-circuits getters when an entry is fresh.
- **Discovery (line 212).** `listGitHubDir(path)` hits `api.github.com/repos/TRaSH-Guides/Guides/contents/${path}` and filters `.json` children. This is how the client knows what profiles/CFs/groups exist without hardcoding lists.
- **Fetching (line 204).** `fetchJSON<T>(url)` is a thin `fetch` wrapper; everything else is per-resource methods on `TrashClient`.
- **Categorization (line 181).** `CF_CATEGORIES` maps regexes onto buckets (hdr/audio/resolution/source/streaming/anime/unwanted/release/language). `categorizeCustomFormat(name)` returns the matching tags. This drives the `category` field surfaced by `trash_list_custom_formats`.
- **Concurrency (line 290).** When fully resolving the CF catalog, requests are batched in groups of 20 with `Promise.all` to stay under GitHub's anonymous rate limit.
- **Singleton (line 421).** `export const trashClient = new TrashClient()` so the cache is shared across the whole process.

---

## 3. Request Lifecycle

For a tool call like `sonarr_get_series`:

1. Client sends `tools/call` JSON-RPC request over the active transport.
2. The MCP SDK delivers `{ name, arguments }` to the `CallToolRequestSchema` handler in `src/index.ts:1136`.
3. The outer try wraps the entire dispatcher.
4. `switch (name)` matches `case "sonarr_get_series"`.
5. Args are cast inline (`const a = args as { limit?: number; offset?: number }`).
6. Configuration guard: `if (!clients.sonarr) throw new Error("sonarr not configured")`.
7. `clients.sonarr.getSeries()` invokes `ArrClient.request<Series[]>('/series')` which builds the URL, attaches `X-Api-Key`, and `await`s `fetch`.
8. On HTTP failure, `request()` throws; the outer try/catch converts that into `{ content: [{ type: "text", text: "Error: ..." }], isError: true }`.
9. On success, the dispatcher slices/paginates as needed and returns `jsonText(payload)`.

---

## 4. Transport Layer

Two transports, mutually exclusive at runtime.

### Stdio (default)

- Selected when `MCP_TRANSPORT` is unset or `stdio`.
- `StdioServerTransport` reads JSON-RPC frames from stdin and writes responses to stdout.
- **Implication:** any `console.log` in source code corrupts the protocol. Logs must go to `console.error` (which is stderr).

### HTTP (`MCP_TRANSPORT=http`)

- Implemented in `startHttpServer()` (`src/index.ts:2433`).
- Plain `node:http` server with two endpoints:
  - `GET /health` → `{ status: "ok", version, transport: "http" }`. Used by `docker-compose` healthchecks and the test runner's readiness probe.
  - `<MCP_PATH>` (default `/mcp`) → handled by `StreamableHTTPServerTransport` from the MCP SDK.
- **Statelessness:** `sessionIdGenerator: undefined` in the transport options. A fresh transport is constructed per request; the previous one is closed after the response completes (`src/index.ts:2462–2470`). This avoids the "Stateless transport cannot be reused" bug fixed in 1.6.3 and is what `test/http-transport.test.mjs` regression-tests.
- Configurable via `HOST` (default `0.0.0.0` in compose, `127.0.0.1` otherwise), `PORT` (default `3000`), `MCP_PATH` (default `/mcp`).

---

## 5. Configuration Surface

All configuration is environment-variable driven; nothing is read from disk at runtime besides `package.json`/SDK internals.

| Var | Required? | Used by | Notes |
|---|---|---|---|
| `SONARR_URL` + `SONARR_API_KEY` | optional pair | `SonarrClient` | Either both or neither. |
| `RADARR_URL` + `RADARR_API_KEY` | optional pair | `RadarrClient` | |
| `LIDARR_URL` + `LIDARR_API_KEY` | optional pair | `LidarrClient` | |
| `PROWLARR_URL` + `PROWLARR_API_KEY` | optional pair | `ProwlarrClient` | |
| `MCP_TRANSPORT` | optional | transport selector | `stdio` (default) or `http`. |
| `HOST` | optional | HTTP transport | Bind address. |
| `PORT` | optional | HTTP transport | Default `3000`. |
| `MCP_PATH` | optional | HTTP transport | Default `/mcp`. |

`server.json` mirrors this list for the MCP registry; keep them in sync.

The TRaSH client requires no configuration. With **zero** *arr services configured the server still starts and exposes the unified `search`/`fetch` plus all `trash_*` tools — useful as a reference-data-only deployment.

---

## 6. Type Strategy

There is no `src/types.ts`. Three rules:

1. **Co-locate.** *arr-shaped types live next to their client; TRaSH types live next to the TRaSH client; transient dispatcher types (e.g. `SearchEntry`, `QueueCapableClient`) live in `src/index.ts`.
2. **No runtime validation.** Tool inputs are cast (`args as { ... }`) rather than parsed with Zod. This keeps the runtime dependency footprint at exactly one (`@modelcontextprotocol/sdk`) and is acceptable because the MCP SDK already validates the JSON Schema declared on each `Tool`.
3. **Export liberally.** Anything that might be reused by tests or by future split files is `export`ed even if it's currently only used in one place.

---

## 7. Error Model

Three layers:

1. **Network/HTTP layer** — `ArrClient.request()` throws a controlled `Error` with status + body snippet.
2. **Validation/precondition layer** — handlers either throw `Error("X not configured")` (treated identically to layer 1) or return `textError("...")` for expected user-facing failures (e.g. "TRaSH naming key not found for media server X").
3. **Dispatcher layer** — outer try/catch in `src/index.ts:1139` converts any thrown `Error` into the standard MCP error response shape:

```ts
{ content: [{ type: "text", text: `Error: ${err.message}` }], isError: true }
```

Upstream JSON is never forwarded verbatim — we always go through the controlled string in `ArrClient.request()` so API keys / internal endpoints never leak.

---

## 8. Build, Package, Distribute

- **Build:** `tsc` reads `tsconfig.json` (`module: NodeNext`, `target: ES2022`, `strict: true`, `outDir: ./dist`) and emits `.js` + `.d.ts` + source maps. `package.json#files` is `["dist"]`, so only the compiled output ships to npm.
- **Bin:** `package.json#bin.mcp-arr → dist/index.js`. The shebang on line 1 makes it directly executable after `npm i -g`.
- **Docker:** Multi-stage `Dockerfile`. Builder runs `npm ci` + `npm run build`; runtime stage does `npm ci --omit=dev`, copies `dist`, drops to non-root `nodejs` (uid 1001), and `ENTRYPOINT ["node", "dist/index.js"]`.
- **Compose:** `docker-compose.yml` defaults to HTTP transport (`MCP_TRANSPORT=http`, port `3000`) and passes through every `*_URL`/`*_API_KEY` env var with empty defaults so unconfigured services remain absent.

---

## 9. CI/CD

- **`.github/workflows/ci.yml`** — push/PR to main/master. Matrix on Node 18.x/20.x/22.x: `npm ci`, `npm run build`, `npx tsc --noEmit`. Separate `security` job runs `npm audit --audit-level=high` (non-blocking).
- **`.github/workflows/release.yml`** — triggered by tags `v*.*.*` or manual dispatch. Pipeline:
  1. `prepare` — resolves and validates the version, computes major/minor.
  2. `build-test` — build + typecheck + `node --test`.
  3. `docker` — multi-arch (`linux/amd64,linux/arm64`) image push to `ghcr.io/${{ github.repository }}` with version/minor/major/latest tags.
  4. `github-release` — extracts release notes from `CHANGELOG.md` via awk and publishes a GitHub Release.
  5. `npm-publish` — publishes the package to npmjs.com.

---

## 10. Testing Philosophy

Coverage is deliberately narrow. The single test (`test/http-transport.test.mjs`) is a regression test for the v1.6.3 stateless-transport bug — it spawns the built server with `MCP_TRANSPORT=http`, polls `/health`, then sends two consecutive JSON-RPC requests to verify the transport doesn't error with "Stateless transport cannot be reused".

When adding tests:

- Use `node:test` + `node:assert/strict`. **No Jest, no Vitest.**
- Author tests as `.mjs` so they run without TypeScript compilation.
- Prefer black-box tests against the built `dist/index.js` over white-box unit tests of internal classes.

---

## 11. Known Inconsistencies / Tech Debt

- **`CLAUDE.md` is stale.** It describes a `src/tools/<service>.ts` modular layout and a `src/types.ts` central type module — neither exists. The actual layout is the flat three-file structure documented here.
- **`SERVER_VERSION` is hardcoded.** Three sources of truth (`package.json`, `server.json`, `src/index.ts:34`) must be edited in lockstep on release.
- **Dispatcher is monolithic.** ~1300 lines of `switch` cases. A future refactor could split per-service handler modules, but until that refactor is explicitly approved, **stay flat** (see AGENTS.md hard rules).
- **Some handlers don't use `jsonText`/`textError`.** They inline the same response shape manually. New code should always use the helpers.
- **`tools.json` is hand-maintained.** It's the registry catalog snapshot, not generated from `TOOLS[]`. Keep it in sync when you add/remove/rename tools.

---

## 12. File Map

```
mcp-arr/
├── src/
│   ├── index.ts            # entrypoint, tools, dispatcher, transports
│   ├── arr-client.ts       # *arr clients + types
│   └── trash-client.ts     # TRaSH GitHub client + cache
├── test/
│   └── http-transport.test.mjs
├── dist/                   # build output (gitignored, shipped to npm)
├── docs/                   # images only (logo, architecture diagram)
├── docker-registry/        # MCP Docker registry catalog snapshot
├── .github/workflows/
│   ├── ci.yml
│   └── release.yml
├── package.json
├── tsconfig.json
├── server.json             # MCP registry manifest
├── tools.json              # static tool catalog snapshot
├── Dockerfile
├── docker-compose.yml
├── README.md               # end-user docs
├── AGENTS.md               # agent operational guide
├── architecture.md         # this file
├── CHANGELOG.md
├── CLAUDE.md               # ⚠️ stale; AGENTS.md / architecture.md supersede
└── LICENSE                 # MIT
```
