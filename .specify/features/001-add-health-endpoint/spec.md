# Feature Specification: Add GET /health Endpoint

**Feature Branch**: `paperclip/PCS-3932-spec`
**Created**: 2026-11-03
**Status**: Draft
**JIRA**: [PCS-3932](https://grampower.atlassian.net/browse/PCS-3932)
**Input**: Ticket description — "Add a GET /health endpoint to this toy repo that returns `{\"status\":\"ok\"}`."

## Context

The repository `shubhanshuPolaris/paperclip-test` currently contains a single Node.js source file (`index.js`) whose only behavior is `console.log('hello')`, and a one-line `README.md`. There is no `package.json`, no HTTP server, no dependency manifest, and no test harness. This feature introduces the first runnable HTTP surface on the repo.

Because the repo has no framework commitment yet, the first load-bearing decision is "which runtime wiring hosts the endpoint". The ticket's wording ("toy repo") and the current `console.log` entry point indicate that minimal, zero-dependency wiring is the intended spirit of the change.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Liveness probe returns a successful status payload (Priority: P1) 🎯 MVP

An operator (or a container orchestrator such as Kubernetes, a load balancer health check, or a monitoring probe) issues `GET /health` against the running process. The service responds with HTTP 200 and a JSON body `{"status":"ok"}` so the caller can confirm the process is alive and able to serve requests.

**Why this priority**: This is the entirety of the ticket's ask. Without it, the ticket is not satisfied; with it, the ticket is complete.

**Independent Test**: Start the process, then run `curl -s -o - -w '\n%{http_code}\n' http://localhost:<port>/health`. The output must include a `200` status line and a body that parses as JSON equal to `{"status":"ok"}`.

**Acceptance Scenarios**:

1. **Given** the process is running, **When** a client sends `GET /health`, **Then** the response status is `200 OK`, the `Content-Type` response header is `application/json` (or compatible, e.g. `application/json; charset=utf-8`), and the response body parsed as JSON equals `{"status":"ok"}`.
2. **Given** the process is running, **When** a client sends `GET /health` repeatedly in sequence, **Then** every response is identical and the handler is idempotent (no state is mutated).
3. **Given** the process is running, **When** a client sends `HEAD /health`, **Then** the response status is `200` with no body (or the server MAY respond `405` — see FR-005 for the decided behavior).

### Edge Cases

- A client requests `GET /health/` (trailing slash). The decision is captured in FR-004.
- A client requests `GET /HEALTH` (uppercase). Path matching is case-sensitive per RFC 3986; the response for an unmatched route is `404`.
- A client sends a non-GET verb to `/health` (e.g. `POST`, `DELETE`). See FR-005.
- The process is starting up and has not finished binding its listening socket yet. The caller receives a connection refusal from the OS, not a JSON body; the health endpoint only reports health once the server is accepting connections.
- Two instances of the process attempt to bind the same port. The second fails to start; `/health` on that second instance is unreachable — expected and out of scope.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The service MUST expose an HTTP route at the path `/health` that accepts the `GET` method.
- **FR-002**: On a successful `GET /health`, the service MUST respond with HTTP status code `200`.
- **FR-003**: The `GET /health` response body MUST parse as JSON equal to `{"status":"ok"}`, and the `Content-Type` response header MUST indicate JSON (`application/json`, optionally with a `charset` parameter).
- **FR-004**: The service MUST treat `/health` and `/health/` as the same route (both return the success response). Rationale: common probe tooling varies on trailing slashes and this avoids per-caller surprise.
- **FR-005**: For `/health` requested with a method other than `GET` or `HEAD`, the service MUST respond with `405 Method Not Allowed` and set an `Allow: GET, HEAD` header. `HEAD /health` MUST return the same status and headers as `GET /health` with an empty body (standard HTTP semantics).
- **FR-006**: The handler MUST NOT depend on any external system (no database call, no outbound HTTP, no filesystem read beyond process load). It is a liveness probe, not a readiness probe.
- **FR-007**: The server listening port MUST be configurable via the `PORT` environment variable, defaulting to `3000` when `PORT` is unset or empty. Rationale: standard twelve-factor convention and compatibility with PaaS runners.
- **FR-008**: Logs emitted by the server start-up SHOULD indicate the port the service is listening on, so the operator running the spec's acceptance test knows which port to curl.

### Non-Functional Requirements

- **NFR-001**: The handler SHOULD return in under 10 ms on a warm process on commodity hardware (it does only JSON serialization of a constant object).
- **NFR-002**: The implementation SHOULD introduce zero runtime dependencies if feasible given the chosen stack, consistent with the repository's current zero-dependency state. See the plan for the trade-off.

### Key Entities

None. The response payload is a constant JSON object; there is no persisted state and no domain entity.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: `curl -fsS http://localhost:$PORT/health` exits with status `0` and prints `{"status":"ok"}` (with any trailing newline permitted by curl's output).
- **SC-002**: The acceptance scenarios in User Story 1 all pass against the running process.
- **SC-003**: Starting the service (`node index.js`, or whatever the plan selects) and then stopping it leaves no stray processes, no open file descriptors, and no written files on disk other than what was there before.
- **SC-004**: `README.md` is updated with a one-line "How to run" snippet and the probe URL so a new reader can validate the endpoint without reading the source.

## Assumptions

- The repository will remain Node.js. The existing `index.js` and absence of other language manifests (`pyproject.toml`, `go.mod`, `Cargo.toml`, etc.) make Node the obvious choice; swapping languages is out of scope.
- There is no requirement for TLS, authentication, rate limiting, or structured logging on this endpoint. It is a liveness probe on a toy repo.
- There is no CI pipeline requirement attached to this ticket. If one exists elsewhere, this spec does not add or modify it.
- Hostname binding defaults to `0.0.0.0` (all interfaces), standard for containerized probes. Changing this is out of scope.

## Out of Scope

- Readiness probes (`/ready`, `/readyz`) that check downstream dependencies.
- Versioned health payloads, build-info enrichment (git SHA, build time), or uptime counters.
- Metrics endpoints (`/metrics`, Prometheus exposition).
- Graceful-shutdown behavior where `/health` starts returning non-200 during drain.
- Introducing a web framework (Express, Fastify, Koa) if the plan selects the Node stdlib `http` module. The plan explicitly chooses between these.

## Open Questions

- None load-bearing. The ticket is unambiguous; stack choice is a plan-level decision documented in `plan.md`.
