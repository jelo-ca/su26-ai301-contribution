# Contribution #1: Support for Additional SQL Databases (PostgreSQL)
**Contribution Number:** 1
**Dates:** 6/1 – 6/7
**Student:** Anjoelo Calderon
**Issue:** https://github.com/camel-ai/oasis/issues/214
**Status:** Phase I Complete (Discontinued)

---

## Why I Chose This Issue

OASIS is a large-scale LLM social-simulation platform that currently persists
every run to a local SQLite file. That works for local evaluation but doesn't
share well in multi-process or hosted environments — which is the gap this
feature request describes. I've been working with Cloud SQL, AlloyDB, and
Postgres on projects so this lands within my skillset.

It's also a good fit for a high-merge-probability contribution: the change is
additive and non-breaking rather than disruptive, with a clear default to
preserve. What I hope to learn is how to introduce a cross-cutting abstraction
into an existing codebase without a large invasive diff, and how to coordinate
on the social side of OSS.

---

## Understanding the Issue

### Problem Description
OASIS can only persist simulation state to a local SQLite database. There is no
supported way to point it at a server database such as PostgreSQL, which is a
poor fit for runs at scale or when multiple processes/machines need shared data.

### Expected Behavior
A user should be able to opt in to another SQL backend — at minimum
PostgreSQL — via configuration (a constructor argument and/or environment
variable), while SQLite remains the default for existing users.

### Current Behavior
`oasis/social_platform/database.py::create_db()` calls `sqlite3.connect(...)`
directly and builds the schema with `cursor.executescript()` (a SQLite-only
method). All queries use SQLite's `?` placeholder style, and inserts read the
row id via `cursor.lastrowid`. There is no abstraction point to swap engines.

### Affected Components
- `oasis/social_platform/database.py` — connection creation, schema loading,
  and introspection helpers.
- `oasis/social_platform/platform.py` — holds the connection/cursor from
  `create_db()`; uses a SQLite `PRAGMA` and a SQLite-only `.backup()` path, and
  reads `lastrowid` in ~13 places.
- `oasis/social_platform/platform_utils.py` — `_execute_db_command()` /
  `_execute_many_db_command()`, the central execution methods.
- `oasis/social_platform/schema/*.sql` — 16 SQLite-dialect schema files.
- `pyproject.toml` — dependency/extras declaration.

**Key finding:** DB access is centralized — nearly all reads/writes funnel
through `_execute_db_command()` on the cursor returned by `create_db()`. That
single chokepoint is where a backend could be introduced later without
rewriting the ~100 existing query strings.

---

## Outcome

Discontinued after Phase I analysis. Maintainers never got back to me so I decided to find another project. 
Findings documented and pivot made to Contribution #2.

---

---

# Contribution #2: Add UI Tests to DCS Simulation Engine
**Contribution Number:** 2
**Dates:** 6/8 – present
**Student:** Anjoelo Calderon
**Issue:** https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/issues/141
**Status:** In Progress — Analysis & Planning Complete, Infrastructure Pending

---

## Why I Chose This Issue

The dcs-simulation-engine is a gameplay framework for interacting with diverse cognitive systems, developed at Georgia Tech. The React frontend has zero test coverage — the issue is explicitly labeled HIGH PRIORITY and good first issue. Adding a test suite is a meaningful, scoped contribution: it improves long-term maintainability without requiring deep domain knowledge of the simulation logic.

This also fills a gap in my skills — I've done backend testing but haven't set up a frontend test harness from scratch on a real OSS project.

---

## Understanding the Issue

### Problem Description
The React UI (`ui/`) has no automated tests. A preliminary backlog of 23 test behaviors exists in `ui/tests/README.md` but no framework is installed and no test files exist.

### Expected Behavior
A working test suite runnable via `bun run test` that covers the behavioral items in `ui/tests/README.md` — primarily input submission gating, WebSocket state transitions, auth-mode routing, and session restoration.

### Current Behavior
`ui/tests/` contains only a README. No test runner, no config, no test files.

### Affected Components
- `ui/src/hooks/use-session-websocket.ts` — WebSocket state machine (core logic under test)
- `ui/src/routes/play/$sessionId.tsx` — Submit-state logic, auth guard, terminal session handling
- `ui/src/routes/run.tsx` — Run assignment status display
- `ui/package.json` — needs `"test"` script added
- New: `ui/vitest.config.ts`, `ui/tests/setup.ts`

**Key finding:** Nearly all backlog items reduce to asserting derived boolean state (`canSubmitTurn`, `inputDisabled`, `isReplaying`, `waiting`) driven by WebSocket frame sequences. A fake `WebSocket` class + `renderHook` covers most cases without a real server.

---

## Reproduction Process
### Environment Setup
- Dev container via `.devcontainer/` (VS Code)
- `bun install` in `ui/`
- API server not needed for component/hook tests — WebSocket is mocked

### Steps to Reproduce
1. Clone the upstream repository: `git clone https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine`
2. Open the folder in VS Code and select **"Reopen in Container"** to launch the dev container.
3. Inside the container, run `cd ui && bun run test`.
4. Observe: `error: Missing script: "test"` — no test script is defined in `ui/package.json`.
5. Check `ui/tests/` — the directory contains only `README.md`; there are no test files, no runner config, and no `vitest.config.ts`.

### Reproduction Evidence
- **Branch in fork:** https://github.com/jelo-ca/dcs-simulation-engine_issue-141/tree/test/ui

---

## Solution Approach
### Analysis
The WebSocket hook (`use-session-websocket.ts`) is a pure state machine driven by incoming frames. It is self-contained and independently testable with `renderHook`. The play page derives all submit-state from that hook's output — so testing the hook covers most behaviors, and shallow component tests cover the rest.

### Proposed Solution
Vitest + React Testing Library + jsdom. Vitest is the idiomatic Vite choice (shares the build config, zero extra transform setup). The issue mentions Jest but the API is identical and no runner is installed yet.

### Implementation Plan (UMPIRE)
**Understand:** Add behavioral tests for the React UI matching the 23-item backlog.

**Match:** Backend already uses pytest with functional + unit markers. UI will follow the same principle: functional tests over unit tests, behavior over implementation.

**Plan:**
- Install `vitest`, `@vitest/coverage-v8`, `jsdom`, `@testing-library/react`, `@testing-library/user-event`, and `@testing-library/jest-dom` as dev deps in `ui/`.
- Add `ui/vitest.config.ts` (jsdom environment, path aliases mirroring `vite.config.ts`, `setupFiles` pointing to `tests/setup.ts`).
- Add `ui/tests/setup.ts` to import jest-dom matchers and stub the global `WebSocket` class.
- Add `"test": "vitest run"` and `"test:watch": "vitest"` to `ui/package.json`.
- Write hook-level tests for `use-session-websocket.ts` using `renderHook` + a `MockWebSocket` that lets us inject server frames directly — covers all WS state transitions, `waiting` lifecycle, `isReplaying`, and `exited` flag.
- Write component-level tests for `PlayPage` mounting with mocked TanStack Router/Query contexts — covers submit-state (`canSubmitTurn`, `sendDisabled`, `inputDisabled`), Enter-key gating, replay blocking, and terminal session read-only mode.
- Write `beforeLoad` auth-mode tests mocking `getServerConfig` to assert `requireAuth` vs. `ensureAnonymousAuth` call paths.
- Wire `cd ui && bun run test` into the `ci` Makefile target (or a new `test-ui` target).

**Implement:** https://github.com/jelo-ca/dcs-simulation-engine_issue-141/tree/test/ui

**Review:** Biome lint passes, no orval-generated files touched, bun used throughout.

**Evaluate:** `bun run test` passes all cases; behaviors from README checked off.

---

## Testing Strategy
### Hook Tests (`use-session-websocket`)
- [ ] `wsState` transitions: connecting → auth → ready → closed
- [ ] `waiting` set on `sendTurn`, cleared on `turn_end` / `event` / `error`
- [ ] `isReplaying` true after `replay_start`, false after `replay_end`
- [ ] `exited` set when `turn_end` has `exited: true`
- [ ] `error` frame + `failure_type` → `wsState === 'closed'`, `exited === true`
- [ ] `error` frame without `failure_type` → `wsState === 'error'`
- [ ] `enabled: false` → socket never opened

### Component Tests (`PlayPage` submit-state)
- [ ] Can draft text while game loading (textarea enabled during `connecting`)
- [ ] Send button disabled until `wsState === 'ready'` + `turns > 0` + not replaying
- [ ] Enter key blocked until game ready
- [ ] Slash command autocomplete: Enter autocompletes before submit enabled, but not before game loaded
- [ ] Submit disabled while `waiting === true`
- [ ] Resumed sessions: submit disabled during replay, re-enabled after `replay_end`
- [ ] Terminal session → read-only transcript, input disabled, no WS connect

### Auth-Mode Tests
- [ ] Anon mode: `ensureAnonymousAuth()` called before WS connects
- [ ] Registration-required mode: `requireAuth()` redirects to `/login`

### Deferred
- Back/Fwd button behavior (no spec in backlog)
- Second-tab session detection (no client implementation found)

---

## Implementation Notes

### Week of 6/8 – 6/14 (Analysis & Planning)
- Read issue #141: no UI tests exist; backlog in `ui/tests/README.md` (23 items, 3 incomplete)
- Analyzed `use-session-websocket.ts` — WS state machine fully understood; state transitions, `waiting` lifecycle, and `isReplaying` logic documented
- Analyzed `play/$sessionId.tsx` — submit-state logic mapped to derived booleans (`canSubmitTurn`, `sendDisabled`, `inputDisabled`, `gameReady`)
- Analyzed auth guard in `beforeLoad` — mockable via `getServerConfig` / `ensureAnonymousAuth`
- Chose Vitest over Jest (Vite-native, identical API, zero extra transform config)
- Deferred: Back/Fwd (no spec), second-tab (no client implementation visible)
- Created and committed planning docs: `task_plan.md`, `findings.md`, `progress.md`
- Committed: `9c81c92 Add findings and progress documentation for UI tests (#141)`

### Week of 6/15 (Codebase Familiarization)
- Deep-read backend architecture: session lifecycle, WebSocket protocol, DAL, game structure, MongoDB collections
- Revised `CLAUDE.md` to add missing make targets (`make pr`, `make ci`, `make lint-fix`, `make test-live`), missing packages (`helpers/`, `hitl/`, `reporting/`, `autoplay/`, `games/`, `experiments/`, `database_seeds/`), game four-part structure, and MongoDB collections reference
- Working branch: `fix/devcontainer-zshrc-line-endings` (line-ending normalization for dev container files)

### Code Changes
- **Files created:** `task_plan.md`, `findings.md`, `progress.md` (committed)
- **Files modified:** `CLAUDE.md` (improved architecture documentation)
- **Key commits:** `9c81c92` — planning and findings docs
- **Next:** Phase 1 infrastructure (install deps, `vitest.config.ts`, `tests/setup.ts`, `package.json` script)

---

## Pull Request
**PR Link:** TBD
**PR Description:** TBD
**Maintainer Feedback:** TBD
**Status:** In Progress — pending Phase 1

---

## Learnings & Reflections
[To be filled in after PR is submitted]

---

## Resources Used
- Issue #141: https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/issues/141
- Repository: https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine
- UI test backlog: `ui/tests/README.md`
- Vitest docs: https://vitest.dev
- React Testing Library: https://testing-library.com/docs/react-testing-library/intro
