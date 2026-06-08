# Contribution #1: Support for Additional SQL Databases (PostgreSQL)
**Contribution Number:** 1
**Student:** Anjoelo Calderon
**Issue:** https://github.com/camel-ai/oasis/issues/214
**Status:** Phase I Complete

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

## Reproduction Process
### Environment Setup
[Notes on setting up your local development environment - challenges you faced, how you solved them]
### Steps to Reproduce
1. [Step 1]
2. [Step 2]
3. [Observed result]
### Reproduction Evidence
- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]
  
---

## Solution Approach
### Analysis
[Your analysis of the root cause - what's causing the issue?]
### Proposed Solution
[High-level description of your fix approach]
### Implementation Plan
Using UMPIRE framework (adapted):
**Understand:** [Restate the problem]
**Match:** [What similar patterns/solutions exist in the codebase?]
**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]
**Implement:** [Link to your branch/commits as you work]
**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]
**Evaluate:** [How will you verify it works?]

---

## Testing Strategy
### Unit Tests
- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]
### Integration Tests
- [ ] Integration scenario 1
- [ ] Integration scenario 2
### Manual Testing
[What you tested manually and results]

---

## Implementation Notes
### Week [X] Progress
[What you built this week, challenges faced, decisions made]
### Week [Y] Progress
[Continue documenting as you work]
### Code Changes
- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]
  
---

## Pull Request
**PR Link:** [GitHub PR URL when submitted]
**PR Description:** [Draft or final PR description - much of the content above can be adapted]
**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]
**Status:** [Awaiting review / Iterating / Approved / Merged]
  
---

## Learnings & Reflections
### Technical Skills Gained
[What you learned technically]
### Challenges Overcome
[What was hard and how you solved it]
### What I'd Do Differently Next Time
[Reflection on your process]

---

## Resources Used
- OASIS issue #214: https://github.com/camel-ai/oasis/issues/214
- OASIS repository: https://github.com/camel-ai/oasis
- OASIS documentation: https://docs.oasis.camel-ai.org/
