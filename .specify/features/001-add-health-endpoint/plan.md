# Implementation Plan: Add GET /health Endpoint

**Branch**: `paperclip/PCS-3932-spec`
**Date**: 2026-11-03
**Spec**: [spec.md](./spec.md)
**JIRA**: PCS-3932

## Summary

Add a Node.js HTTP listener to the existing single-file entry point `index.js` that serves `GET /health` with a static JSON body `{"status":"ok"}` and HTTP 200. Keep the repository dependency-free by using Node's built-in `http` module. Port is driven by `process.env.PORT` with a default of `3000`.

## Technical Context

**Language/Version**: Node.js (LTS; the repository has no `package.json` or `.nvmrc`, so the plan assumes any Node ≥ 18 — modern enough to be the default on common CI images, old enough that `http` behavior is identical). If a `package.json` is introduced during implementation, `engines.node` SHOULD pin `>=18`.
**Primary Dependencies**: None. Node standard library module `node:http`.
**Storage**: N/A — no persisted state.
**Testing**: No test framework is present in the repo today. Acceptance is a `curl` command (see `contracts/health.http`). An optional lightweight test (`node:test` built-in, zero-dep) can be added if the tasks phase includes it; it is not required by the ticket.
**Target Platform**: Any Node-capable host (local dev, container, PaaS). Binds to `0.0.0.0` on `process.env.PORT || 3000`.
**Project Type**: Single-process service (CLI-launched HTTP server) — matches the current `index.js` layout.
**Performance Goals**: Not a bottleneck. Static response under 10 ms per NFR-001. Node's `http` can trivially serve thousands of these per second on commodity hardware.
**Constraints**: Zero third-party runtime dependencies (NFR-002). Minimal diff to the one-file structure.
**Scale/Scope**: Toy repo, one endpoint, one file.

## Stack Decision & Alternatives Considered

| Option | Decision | Why |
|---|---|---|
| **Node stdlib `http`** | ✅ Chosen | Zero deps, preserves current single-file layout, fits a "toy repo" and FR-006/NFR-002. |
| Express | ❌ Rejected | Pulls in a dependency tree (and would also require creating `package.json` + lockfile) for a single static route. Overkill. |
| Fastify | ❌ Rejected | Same reasoning as Express; even more surface area. |
| Rewriting to Deno/Bun | ❌ Rejected | Out of scope — the ticket does not justify a runtime migration. |

**Structure Decision**: Keep the single-file layout. The listener is wired directly in `index.js`. No `src/` tree is introduced. If future tickets add more routes, a split into `src/server.js` + `src/routes/health.js` becomes justified; it is not justified now (YAGNI).

## Project Structure

### Documentation (this feature)

```text
.specify/features/001-add-health-endpoint/
├── spec.md              # Feature specification (/speckit.specify output)
├── plan.md              # This file (/speckit.plan output)
├── tasks.md             # Task breakdown (/speckit.tasks output)
└── contracts/
    └── health.http      # Request/response contract used for acceptance
```

### Source Code (repository root)

```text
# Current repo (pre-change)
index.js                 # console.log('hello')
README.md                # one-line title

# Target repo (post-change)
index.js                 # Node http server; GET /health → {"status":"ok"}
README.md                # title + "How to run" + probe URL
# (no new directories, no package.json, no node_modules)
```

**Structure Decision**: Single-file. The entry point `index.js` is the whole app.

## Constitution Check

No project constitution file is present under `.specify/memory/`. The implicit gates applied by this plan:

- ✅ **Simplicity**: One file, one route, zero deps.
- ✅ **Reversibility**: The diff is a drop-in replacement for `index.js` and a README append; `git revert` cleanly restores prior state.
- ✅ **Observability of the acceptance test**: The server logs its listening port on startup so an operator can reliably point `curl` at it.
- ✅ **Scope**: Only `index.js` and `README.md` under application code are touched. `.specify/features/001-add-health-endpoint/` under docs is also touched (expected).

## Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| `PORT=3000` already bound on the dev machine during manual verification | Medium | FR-007: respect `PORT` env var so the operator can set `PORT=3001` without edit. Startup log line names the port. |
| Someone extends `/health` later with a readiness check that calls external systems | Low | FR-006 explicitly forbids it in this spec; follow-up tickets would amend the spec. |
| Node version drift (older Node without a feature we use) | Very Low | Only `node:http` is used — stable since Node 0.1. No modern-syntax trap. |
| Trailing-slash handling diverges from the spec | Low | FR-004 is tested in acceptance; the single router comparison must normalize once. |

## Phase 0 Research

No research needed. The ticket, the current `index.js`, and the Node `http` module docs are sufficient. There are no unknown platforms, no third-party library choices, and no data models.

## Phase 1 Design

### Request/Response Contract

Documented machine-readably in `./contracts/health.http`. Human summary:

- **Request**: `GET /health HTTP/1.1` (also `GET /health/`, also `HEAD /health`).
- **Success response**:
  - Status: `200 OK`
  - Headers: `Content-Type: application/json; charset=utf-8`, `Content-Length: 15` (for `GET`), omitted body for `HEAD`.
  - Body (for `GET`): `{"status":"ok"}` — exactly, no trailing newline, no extra whitespace. The byte length is 15.
- **Method-not-allowed response**:
  - Status: `405 Method Not Allowed`
  - Headers: `Allow: GET, HEAD`
  - Body: empty.
- **Unknown path**:
  - Status: `404 Not Found`
  - Body: empty.

### Server-startup contract

- Reads `process.env.PORT`. If unset or empty string, defaults to `3000`. Parsed as integer; a non-integer value is a hard start-up failure (process exits with non-zero so the orchestrator notices immediately).
- Binds `0.0.0.0`.
- On `listen` success, logs a single line: `health server listening on http://0.0.0.0:<port>/health` to stdout.

### Pseudocode (plan-level, not production code)

```
http.createServer((req, res) => {
  const path = normalize(req.url)          // strip query, strip trailing slash
  if (path !== '/health') return send404()
  if (req.method === 'GET')  return sendJson(200, { status: 'ok' })
  if (req.method === 'HEAD') return sendHead(200)
  return sendAllow('GET, HEAD', 405)
}).listen(PORT, '0.0.0.0', () => log(...))
```

## Complexity Tracking

No constitution violations to justify. No table rows.
