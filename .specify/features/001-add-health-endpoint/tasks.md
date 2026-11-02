---
description: "Task list for PCS-3932 — Add GET /health endpoint"
---

# Tasks: Add GET /health Endpoint

**Input**: Spec + plan under `.specify/features/001-add-health-endpoint/`
**Prerequisites**: `spec.md` (required), `plan.md` (required), `contracts/health.http` (required)
**Tests**: Not required by the ticket. One optional smoke-test task is listed and may be skipped.

**Organization**: Grouped by user story. There is one user story (US1) because the ticket is a single-concern feature.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to
- File paths are concrete, repo-rooted.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: None required. The repo already has `index.js` and `README.md` at the root; no scaffolding is needed and no dependency manager is being introduced. This phase is intentionally empty — recorded here so the pipeline sees it was considered, not skipped by oversight.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: None required. No framework install, no new module layout, no shared middleware. Single-file implementation per plan.md "Structure Decision".

**Checkpoint**: Proceed directly to User Story 1.

---

## Phase 3: User Story 1 - Liveness probe returns `{"status":"ok"}` (Priority: P1) 🎯 MVP

**Goal**: `GET /health` on the running process returns HTTP 200 with body `{"status":"ok"}` and JSON content type, per spec.md FR-001 through FR-008.

**Independent Test**: Run the process (`node index.js`) and execute `curl -fsS http://localhost:3000/health`; command exits `0` and prints `{"status":"ok"}`. See `contracts/health.http` for the full matrix.

### Implementation for User Story 1

- [ ] **T001** [US1] Replace `console.log('hello')` in `index.js` with an `http.createServer` listener that:
  - Imports `node:http`.
  - Reads `PORT` from `process.env.PORT`, defaulting to `3000`; fails fast with a non-zero exit if the value is set but not a positive integer (satisfies FR-007).
  - Normalizes the request URL by stripping any query string and a single trailing slash (satisfies FR-004).
  - For path `/health` and method `GET`: responds `200` with headers `Content-Type: application/json; charset=utf-8` and `Content-Length: 15`, and the exact body `{"status":"ok"}` (satisfies FR-001, FR-002, FR-003).
  - For path `/health` and method `HEAD`: responds `200` with the same headers as `GET` and no body (satisfies FR-005, HEAD half).
  - For path `/health` and any other method: responds `405` with header `Allow: GET, HEAD` and empty body (satisfies FR-005, 405 half).
  - For any other path: responds `404` with empty body.
  - Binds `0.0.0.0` and on `listen` success logs `health server listening on http://0.0.0.0:<port>/health` to stdout (satisfies FR-008).
  - Introduces no new dependencies and does not create a `package.json` (satisfies NFR-002).

- [ ] **T002** [US1] Append a "How to run" section to `README.md`:
  - `node index.js` to start.
  - `curl -s http://localhost:3000/health` and the expected `{"status":"ok"}` body.
  - Mention that `PORT=<n>` overrides the default. (Satisfies SC-004.)

- [ ] **T003** [P] [US1] *(Optional)* Add a smoke test using the built-in `node:test` runner at `test/health.test.js` that spawns `index.js` as a child process on a free high port (e.g. by setting `PORT=0` is not supported here — use a fixed test port such as `PORT=37329`), waits for the "listening" log line, runs the five contract cases from `contracts/health.http`, and asserts each. Zero deps. Skip this task if the implementer prefers to rely on manual `curl` verification; the ticket does not require automated tests.

**Checkpoint**: Running `node index.js` and executing each case in `contracts/health.http` matches the expected status, headers (where listed), and body. SC-001 and SC-002 are satisfied.

---

## Phase 4: Polish & Cross-Cutting Concerns

- [ ] **T004** [US1] Manual verification run of all five cases in `.specify/features/001-add-health-endpoint/contracts/health.http`. Record pass/fail per case in the PR description.
- [ ] **T005** [P] [US1] Verify no stray files were created (no `node_modules`, no `package.json`, no `package-lock.json`) — `git status` should show only `index.js` and `README.md` as modified. (Satisfies SC-003 for the diff shape.)

---

## Dependencies & Execution Order

### Phase Dependencies

- Setup and Foundational phases are empty for this feature.
- Phase 3 (US1) depends on nothing and can start immediately.
- Phase 4 (Polish) depends on T001 and T002 being merged into the working tree.

### Within User Story 1

- T001 before T002 (so the README references real behavior).
- T003 is optional and parallelizable with T002 once T001 is committed (both touch different files).
- T004 after T001 at minimum.
- T005 can run any time after T001.

### Parallel Opportunities

- T003 (optional smoke test) and T002 (README) touch different files and can be done in parallel after T001.
- T005 (diff-shape check) is a `git status` read; can run any time after T001.

---

## Notes

- [P] tasks = different files, no dependencies.
- The implementation is intentionally one file. Avoid introducing a framework or a module split to satisfy the ticket.
- Keep the body exactly `{"status":"ok"}` — no trailing newline, no pretty-printing. The `Content-Length: 15` in the contract depends on this.
- Commit the implementation and the README update together or in two small commits on the implementation branch (not the spec branch).
