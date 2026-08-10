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

Discontinued after Phase I analysis. The scope of changes required to abstract
the SQL layer (placeholder syntax, `lastrowid`, `executescript`, 16 schema
files, `PRAGMA`/`.backup()` calls) exceeded what could be done cleanly in the
contribution window without risking breakage to existing users. Findings
documented and pivot made to Contribution #2.

---

---

# Contribution #2: Add UI Tests to DCS Simulation Engine
**Contribution Number:** 2
**Dates:** 6/8 – present
**Student:** Anjoelo Calderon
**Issue:** https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/issues/141
**Status:** Phase IV Complete — PR #267 submitted upstream, awaiting review

> **Phase IV Complete.** A Vitest harness with 7 passing hook tests, wired into CI, is submitted
> upstream as a pull request from branch `test/ui-vitest`. Scoped deliberately as harness + first
> slice; `ui/tests/README.md` documents exactly what is covered and what remains.

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

Vitest + React Testing Library + jsdom. Tests assert behavior (derived state, disabled attributes, frame-driven transitions) rather than implementation details. WebSocket traffic is simulated via a `MockWebSocket` class stubbed globally in `ui/tests/setup.ts`, so hook tests run without a live API server.

### Validation performed
- `make test-ui` (and `cd ui && bun run test`) — 7 hook tests pass, Vitest 3.2.7
- `biome check tests vitest.config.ts` — clean, no new lint findings
- `.github/workflows/ci.yml` parsed and confirmed to register the new job (`lint`, `test`,
  `test-ui`, `build-docs`, `deploy-docs`)
- Manual trace of `play/$sessionId.tsx` submit-state booleans against backlog items in `ui/tests/README.md`
- Reproduction confirmed: before this branch, `bun run test` failed with `Missing script: "test"`
- Confirmed no `ui/src/` files changed — additive test infrastructure only, no orval-generated
  code touched

### Hook Tests (`use-session-websocket`) — implemented
- [x] `wsState` transitions: connecting → auth → ready
- [x] `waiting` set on `sendTurn`, cleared on `turn_end`
- [x] `isReplaying` true after `replay_start`, false after `replay_end`
- [x] `exited` set when `turn_end` has `exited: true`
- [x] `error` frame + `failure_type` → `wsState === 'closed'`, `exited === true`
- [x] `error` frame without `failure_type` → `wsState === 'error'`
- [x] `enabled: false` → socket never opened

### Component Tests (`PlayPage` submit-state) — planned for Phase IV
- [ ] Can draft text while game loading (textarea enabled during `connecting`)
- [ ] Send button disabled until `wsState === 'ready'` + `turns > 0` + not replaying
- [ ] Enter key blocked until game ready
- [ ] Slash command autocomplete: Enter autocompletes before submit enabled, but not before game loaded
- [ ] Submit disabled while `waiting === true`
- [ ] Resumed sessions: submit disabled during replay, re-enabled after `replay_end`
- [ ] Terminal session → read-only transcript, input disabled, no WS connect

### Auth-Mode Tests — planned for Phase IV
- [ ] Anon mode: `ensureAnonymousAuth()` called before WS connects
- [ ] Registration-required mode: `requireAuth()` redirects to `/login`

### CI wiring — implemented
- [x] `make test-ui` target added, included in `make pr` and `make ci`
- [x] `test-ui` job added to `.github/workflows/ci.yml`, running in parallel with the Python `test` job
- [x] UI suite documented in `CONTRIBUTING.md` alongside the existing test instructions

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
- Revised `CLAUDE.md` to add missing make targets, packages, game structure, and MongoDB collections reference

### Week of 6/23 (Phase III — Working Test Harness)
- Installed Vitest, jsdom, React Testing Library, and jest-dom as `ui/` dev dependencies
- Added `ui/vitest.config.ts` (jsdom environment, `@/` path alias, setup file)
- Added `ui/tests/setup.ts` — jest-dom matchers, global `MockWebSocket` stub, per-test cleanup
- Added `ui/tests/helpers/mock-websocket.ts` — controllable fake WebSocket for frame injection
- Added `ui/tests/use-session-websocket.test.ts` — 7 behavioral tests covering the WS state machine
- Added `"test"` and `"test:watch"` scripts to `ui/package.json`
- Verified: `vitest run` passes all 7 tests locally

**Phase III scope:** First working slice — hook-level tests for the WebSocket state machine. Component and auth-mode tests remain for Phase IV polish before upstream PR merge.

### Phase IV — Submission Prep

Pre-submission review against the repo's own requirements (`CONTRIBUTING.md`, `AGENTS.md`) turned up
four problems with the Phase III branch. All were fixed on a fresh branch, `test/ui-vitest`, cut from
`main`:

1. **Unrelated files in the diff.** `test/ui` carried `contribution_README.md`, `task_plan.md`,
   `findings.md`, `progress.md`, and `.gitattributes` — coursework and a repo-wide line-ending policy,
   none of which belong in a test PR. Rebuilt the branch with only the `ui/` changes. Process docs
   stay here on `test/ui`.
2. **Lockfile out of sync.** `ui/bun.lock` is tracked, but the Phase III commit added six dev
   dependencies to `package.json` without regenerating it — `grep -c '"vitest"' bun.lock` returned
   `0`. Anyone running `bun install --frozen-lockfile` would have hit a hard failure, while CI's plain
   `bun install` would have resolved anyway and hidden it. Installed bun and reran `bun install`.
3. **Tests not enforced anywhere.** `make test` is pytest-only; `make lint` reaches `ui/` for Biome but
   never Vitest. Added a `test-ui` make target and a matching CI job so the suite actually gates PRs.
4. **Backlog file still stale.** `ui/tests/README.md` opened with "When frontend tests are added" after
   the tests existed. Rewrote it as a coverage map with `[x]` / `[~]` / `[ ]` status per item, with the
   four partially-covered items annotated to say the hook side is covered but the component assertion
   is not.

**False alarm worth recording:** `biome check .` reported 48 errors locally. Not real — the global
`core.autocrlf=true` setting produced a CRLF working tree while the git index stores LF
(`git ls-files --eol` showed `i/lf` for every file, including untouched upstream `src/` files that
"failed" identically). CI checks out LF on Linux, so it was never going to fail there. Normalizing the
new files to LF made Biome pass with zero content change against the index.

### Code Changes
- **PR branch (submitted upstream):** https://github.com/jelo-ca/dcs-simulation-engine_issue-141/tree/test/ui-vitest
- **Working branch (coursework docs, not submitted):** https://github.com/jelo-ca/dcs-simulation-engine_issue-141/tree/test/ui
- **PR commit:** `6f5649e` — Add Vitest test harness and WebSocket hook tests for the UI
- **Files created:**
  - `ui/vitest.config.ts`
  - `ui/tests/setup.ts`
  - `ui/tests/helpers/mock-websocket.ts`
  - `ui/tests/use-session-websocket.test.ts`
- **Files modified:**
  - `ui/package.json` — test scripts and dev dependencies
  - `ui/bun.lock` — regenerated for the new dev dependencies
  - `ui/tests/README.md` — rewritten as a coverage map
  - `Makefile` — `test-ui` target, added to `pr` and `ci`
  - `.github/workflows/ci.yml` — `test-ui` job
  - `CONTRIBUTING.md` — how to run the UI suite
- **Diff size:** 10 files, +648 / −41
- **Not in the PR (kept on `test/ui`):** `contribution_README.md`, `task_plan.md`, `findings.md`,
  `progress.md`, `.gitattributes`

---

## Pull Request

**PR Link:** https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/pull/267

**Title:** Add Vitest test harness and WebSocket hook tests for the UI

**Target:** `jelo-ca:test/ui-vitest` → `diverse-cognitive-systems-group/dcs-simulation-engine:main`

**Status:** Awaiting review — opened as PR #267 against `main`; GitHub Copilot's automated review ran
on open, no human reviewer assigned yet.

### What I contributed

Stood up the React UI's first automated test suite. Added a Vitest + jsdom + React Testing Library
harness to `ui/`, a controllable `MockWebSocket` that lets tests inject individual server frames
without a running API, and 7 tests over the session WebSocket state machine that the play page derives
all of its input gating from. Wired the suite into CI via a `make test-ui` target and a GitHub Actions
job so it is enforced rather than optional, and rewrote `ui/tests/README.md` as a coverage map of the
23-item backlog.

### Scope decision

Submitted as harness + first slice, covering 7 of the 23 backlog behaviors. The remaining items need
component-level tests against `play/$sessionId.tsx` and route-level auth tests. Rather than let a
reviewer discover the gap, the PR states it in the second paragraph, the README marks each item's real
status (including four partial ones), and the PR offers to fold more of the backlog in if the
maintainers would rather review it as a single piece.

Used the repo's own PR template from `CONTRIBUTING.md` rather than a generic one, and wrote
`Addresses issue #141` rather than `Closes #141` — the PR does not resolve the whole issue, and
auto-closing a high-priority issue on partial work would force a maintainer to notice and reopen it.

### Maintainer Feedback

<!-- Record each round here: what was asked, what I changed, commit hash, and my reply. -->
_No feedback yet — PR just opened._

### Next steps

- Play-page component tests: Send button and Enter-key gating, slash-command autocomplete behavior,
  read-only terminal sessions
- `beforeLoad` auth tests: anonymous mode vs. registration-required routing
- Respond to review within 24 hours; follow up politely if nothing after 5–7 business days

---

## Learnings & Reflections

### Technical

- **A test suite CI doesn't run is not a contribution.** The tests passed on my machine for weeks
  while `make test` stayed pytest-only. Wiring the suite into `ci.yml` was a small diff and the
  difference between something a maintainer inherits as a liability and something that protects them.
- **Tracked lockfiles are part of the change.** Adding dev dependencies to `package.json` without
  regenerating `bun.lock` produced a branch that CI would have passed and a contributor running
  `--frozen-lockfile` would have failed on. Worth checking `git ls-files` for a lockfile before
  assuming dependencies are "just a package.json edit."
- **Read the tool output before believing it.** 48 Biome errors looked like a real problem. They were
  a Windows `core.autocrlf` artifact — the index was LF the whole time, and untouched upstream files
  "failed" identically. One `git ls-files --eol` distinguished a local environment quirk from a
  genuine defect.
- **The repo's own contribution docs beat a generic checklist.** `CONTRIBUTING.md` had its own PR
  template and `AGENTS.md` had explicit testing guidance ("prefer functional tests over excessive unit
  tests", "use bun instead of npm"). Following those and naming them in the PR shows I read the
  project rather than pattern-matched a template.

### Process

- **Scoping honestly is a feature, not an admission.** 7 of 23 items could read as unfinished. Stated
  up front, with a coverage map and an offer to expand, it reads as a reviewable increment instead —
  and it gives the maintainer a real decision to make about scope.
- **Keeping coursework out of the PR mattered more than I expected.** The Phase III branch mixed my
  planning docs and a repo-wide `.gitattributes` into a test PR. Separating the branches cost fifteen
  minutes and removed the single most likely reason a reviewer would have bounced it on sight.

### Personal reflection

<!-- Add your own reflection here: what was hardest, what you'd do differently, how it felt to submit
     to a real project. -->


---

## Resources Used
- Issue #141: https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/issues/141
- Repository: https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine
- UI test backlog: `ui/tests/README.md`
- Vitest docs: https://vitest.dev
- React Testing Library: https://testing-library.com/docs/react-testing-library/intro
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

Discontinued after Phase I analysis. The scope of changes required to abstract
the SQL layer (placeholder syntax, `lastrowid`, `executescript`, 16 schema
files, `PRAGMA`/`.backup()` calls) exceeded what could be done cleanly in the
contribution window without risking breakage to existing users. Findings
documented and pivot made to Contribution #2.

---

---

# Contribution #2: Add UI Tests to DCS Simulation Engine
**Contribution Number:** 2
**Dates:** 6/8 – present
**Student:** Anjoelo Calderon
**Issue:** https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/issues/141
**Status:** Phase IV Complete — PR #267 submitted upstream, awaiting review

> **Phase IV Complete.** A Vitest harness with 7 passing hook tests, wired into CI, is submitted
> upstream as a pull request from branch `test/ui-vitest`. Scoped deliberately as harness + first
> slice; `ui/tests/README.md` documents exactly what is covered and what remains.

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

Vitest + React Testing Library + jsdom. Tests assert behavior (derived state, disabled attributes, frame-driven transitions) rather than implementation details. WebSocket traffic is simulated via a `MockWebSocket` class stubbed globally in `ui/tests/setup.ts`, so hook tests run without a live API server.

### Validation performed
- `make test-ui` (and `cd ui && bun run test`) — 7 hook tests pass, Vitest 3.2.7
- `biome check tests vitest.config.ts` — clean, no new lint findings
- `.github/workflows/ci.yml` parsed and confirmed to register the new job (`lint`, `test`,
  `test-ui`, `build-docs`, `deploy-docs`)
- Manual trace of `play/$sessionId.tsx` submit-state booleans against backlog items in `ui/tests/README.md`
- Reproduction confirmed: before this branch, `bun run test` failed with `Missing script: "test"`
- Confirmed no `ui/src/` files changed — additive test infrastructure only, no orval-generated
  code touched

### Hook Tests (`use-session-websocket`) — implemented
- [x] `wsState` transitions: connecting → auth → ready
- [x] `waiting` set on `sendTurn`, cleared on `turn_end`
- [x] `isReplaying` true after `replay_start`, false after `replay_end`
- [x] `exited` set when `turn_end` has `exited: true`
- [x] `error` frame + `failure_type` → `wsState === 'closed'`, `exited === true`
- [x] `error` frame without `failure_type` → `wsState === 'error'`
- [x] `enabled: false` → socket never opened

### Component Tests (`PlayPage` submit-state) — planned for Phase IV
- [ ] Can draft text while game loading (textarea enabled during `connecting`)
- [ ] Send button disabled until `wsState === 'ready'` + `turns > 0` + not replaying
- [ ] Enter key blocked until game ready
- [ ] Slash command autocomplete: Enter autocompletes before submit enabled, but not before game loaded
- [ ] Submit disabled while `waiting === true`
- [ ] Resumed sessions: submit disabled during replay, re-enabled after `replay_end`
- [ ] Terminal session → read-only transcript, input disabled, no WS connect

### Auth-Mode Tests — planned for Phase IV
- [ ] Anon mode: `ensureAnonymousAuth()` called before WS connects
- [ ] Registration-required mode: `requireAuth()` redirects to `/login`

### CI wiring — implemented
- [x] `make test-ui` target added, included in `make pr` and `make ci`
- [x] `test-ui` job added to `.github/workflows/ci.yml`, running in parallel with the Python `test` job
- [x] UI suite documented in `CONTRIBUTING.md` alongside the existing test instructions

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
- Revised `CLAUDE.md` to add missing make targets, packages, game structure, and MongoDB collections reference

### Week of 6/23 (Phase III — Working Test Harness)
- Installed Vitest, jsdom, React Testing Library, and jest-dom as `ui/` dev dependencies
- Added `ui/vitest.config.ts` (jsdom environment, `@/` path alias, setup file)
- Added `ui/tests/setup.ts` — jest-dom matchers, global `MockWebSocket` stub, per-test cleanup
- Added `ui/tests/helpers/mock-websocket.ts` — controllable fake WebSocket for frame injection
- Added `ui/tests/use-session-websocket.test.ts` — 7 behavioral tests covering the WS state machine
- Added `"test"` and `"test:watch"` scripts to `ui/package.json`
- Verified: `vitest run` passes all 7 tests locally

**Phase III scope:** First working slice — hook-level tests for the WebSocket state machine. Component and auth-mode tests remain for Phase IV polish before upstream PR merge.

### Phase IV — Submission Prep

Pre-submission review against the repo's own requirements (`CONTRIBUTING.md`, `AGENTS.md`) turned up
four problems with the Phase III branch. All were fixed on a fresh branch, `test/ui-vitest`, cut from
`main`:

1. **Unrelated files in the diff.** `test/ui` carried `contribution_README.md`, `task_plan.md`,
   `findings.md`, `progress.md`, and `.gitattributes` — coursework and a repo-wide line-ending policy,
   none of which belong in a test PR. Rebuilt the branch with only the `ui/` changes. Process docs
   stay here on `test/ui`.
2. **Lockfile out of sync.** `ui/bun.lock` is tracked, but the Phase III commit added six dev
   dependencies to `package.json` without regenerating it — `grep -c '"vitest"' bun.lock` returned
   `0`. Anyone running `bun install --frozen-lockfile` would have hit a hard failure, while CI's plain
   `bun install` would have resolved anyway and hidden it. Installed bun and reran `bun install`.
3. **Tests not enforced anywhere.** `make test` is pytest-only; `make lint` reaches `ui/` for Biome but
   never Vitest. Added a `test-ui` make target and a matching CI job so the suite actually gates PRs.
4. **Backlog file still stale.** `ui/tests/README.md` opened with "When frontend tests are added" after
   the tests existed. Rewrote it as a coverage map with `[x]` / `[~]` / `[ ]` status per item, with the
   four partially-covered items annotated to say the hook side is covered but the component assertion
   is not.

**False alarm worth recording:** `biome check .` reported 48 errors locally. Not real — the global
`core.autocrlf=true` setting produced a CRLF working tree while the git index stores LF
(`git ls-files --eol` showed `i/lf` for every file, including untouched upstream `src/` files that
"failed" identically). CI checks out LF on Linux, so it was never going to fail there. Normalizing the
new files to LF made Biome pass with zero content change against the index.

### Code Changes
- **PR branch (submitted upstream):** https://github.com/jelo-ca/dcs-simulation-engine_issue-141/tree/test/ui-vitest
- **Working branch (coursework docs, not submitted):** https://github.com/jelo-ca/dcs-simulation-engine_issue-141/tree/test/ui
- **PR commit:** `6f5649e` — Add Vitest test harness and WebSocket hook tests for the UI
- **Files created:**
  - `ui/vitest.config.ts`
  - `ui/tests/setup.ts`
  - `ui/tests/helpers/mock-websocket.ts`
  - `ui/tests/use-session-websocket.test.ts`
- **Files modified:**
  - `ui/package.json` — test scripts and dev dependencies
  - `ui/bun.lock` — regenerated for the new dev dependencies
  - `ui/tests/README.md` — rewritten as a coverage map
  - `Makefile` — `test-ui` target, added to `pr` and `ci`
  - `.github/workflows/ci.yml` — `test-ui` job
  - `CONTRIBUTING.md` — how to run the UI suite
- **Diff size:** 10 files, +648 / −41
- **Not in the PR (kept on `test/ui`):** `contribution_README.md`, `task_plan.md`, `findings.md`,
  `progress.md`, `.gitattributes`

---

## Pull Request

**PR Link:** https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/pull/267

**Title:** Add Vitest test harness and WebSocket hook tests for the UI

**Target:** `jelo-ca:test/ui-vitest` → `diverse-cognitive-systems-group/dcs-simulation-engine:main`

**Status:** Awaiting review — opened as PR #267 against `main`; GitHub Copilot's automated review ran
on open, no human reviewer assigned yet.

### What I contributed

Stood up the React UI's first automated test suite. Added a Vitest + jsdom + React Testing Library
harness to `ui/`, a controllable `MockWebSocket` that lets tests inject individual server frames
without a running API, and 7 tests over the session WebSocket state machine that the play page derives
all of its input gating from. Wired the suite into CI via a `make test-ui` target and a GitHub Actions
job so it is enforced rather than optional, and rewrote `ui/tests/README.md` as a coverage map of the
23-item backlog.

### Scope decision

Submitted as harness + first slice, covering 7 of the 23 backlog behaviors. The remaining items need
component-level tests against `play/$sessionId.tsx` and route-level auth tests. Rather than let a
reviewer discover the gap, the PR states it in the second paragraph, the README marks each item's real
status (including four partial ones), and the PR offers to fold more of the backlog in if the
maintainers would rather review it as a single piece.

Used the repo's own PR template from `CONTRIBUTING.md` rather than a generic one, and wrote
`Addresses issue #141` rather than `Closes #141` — the PR does not resolve the whole issue, and
auto-closing a high-priority issue on partial work would force a maintainer to notice and reopen it.

### Maintainer Feedback

<!-- Record each round here: what was asked, what I changed, commit hash, and my reply. -->
_No feedback yet — PR just opened._

### Next steps

- Play-page component tests: Send button and Enter-key gating, slash-command autocomplete behavior,
  read-only terminal sessions
- `beforeLoad` auth tests: anonymous mode vs. registration-required routing
- Respond to review within 24 hours; follow up politely if nothing after 5–7 business days

---

## Learnings & Reflections

### Technical

- **A test suite CI doesn't run is not a contribution.** The tests passed on my machine for weeks
  while `make test` stayed pytest-only. Wiring the suite into `ci.yml` was a small diff and the
  difference between something a maintainer inherits as a liability and something that protects them.
- **Tracked lockfiles are part of the change.** Adding dev dependencies to `package.json` without
  regenerating `bun.lock` produced a branch that CI would have passed and a contributor running
  `--frozen-lockfile` would have failed on. Worth checking `git ls-files` for a lockfile before
  assuming dependencies are "just a package.json edit."
- **Read the tool output before believing it.** 48 Biome errors looked like a real problem. They were
  a Windows `core.autocrlf` artifact — the index was LF the whole time, and untouched upstream files
  "failed" identically. One `git ls-files --eol` distinguished a local environment quirk from a
  genuine defect.
- **The repo's own contribution docs beat a generic checklist.** `CONTRIBUTING.md` had its own PR
  template and `AGENTS.md` had explicit testing guidance ("prefer functional tests over excessive unit
  tests", "use bun instead of npm"). Following those and naming them in the PR shows I read the
  project rather than pattern-matched a template.

### Process

- **Scoping honestly is a feature, not an admission.** 7 of 23 items could read as unfinished. Stated
  up front, with a coverage map and an offer to expand, it reads as a reviewable increment instead —
  and it gives the maintainer a real decision to make about scope.
- **Keeping coursework out of the PR mattered more than I expected.** The Phase III branch mixed my
  planning docs and a repo-wide `.gitattributes` into a test PR. Separating the branches cost fifteen
  minutes and removed the single most likely reason a reviewer would have bounced it on sight.

### Personal reflection

<!-- Add your own reflection here: what was hardest, what you'd do differently, how it felt to submit
     to a real project. -->


---

## Resources Used
- Issue #141: https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine/issues/141
- Repository: https://github.com/diverse-cognitive-systems-group/dcs-simulation-engine
- UI test backlog: `ui/tests/README.md`
- Vitest docs: https://vitest.dev
- React Testing Library: https://testing-library.com/docs/react-testing-library/intro
