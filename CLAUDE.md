# maymaydev — agent orientation

When you (Claude) start working in this repo, read in order:

1. `README.md` — product framing + architectural commitments
2. `docs/protocol.md` — the v2 API design (Topic, Endpoint, Run, view objects)
3. `docs/v1-lessons.md` — what was learned from the predecessor "offload"

## Status

Greenfield. Design phase, no code yet. Anything you write is the first
implementation.

## Discipline rules — non-negotiable

1. **Server returns view objects, never raw models.** If a client must
   compute derived state, filter/group/sort, or aggregate raw events
   into UI messages — the API is wrong, fix the API.
2. **No client-type branching in server.** No `if user_agent == "ios"`,
   no per-platform endpoints. The API contract is the contract.
3. **CLI is the conformance test.** Any feature lands in CLI first or
   simultaneously with any GUI. If you can't write the CLI command,
   the API is wrong.
4. **Multi-tenant from day 1.** Every persistent record carries
   `workspace_id`. Single-tenant deployments use a default value.
5. **Postgres from day 1.** No SQLite. No `if local_mode` shortcuts.
6. **No business logic in clients.** State machines, validation,
   routing — all server-side. Clients only render + forward input.
7. **Domain events only, never UI events.** Emit `task.completed`,
   never `show_toast`. The server has no UI vocabulary.

## Four-verb framing

Every feature must clearly strengthen exactly one of:

- **Capture** — intent enters the system (clients, ingestion, parsing)
- **Manage** — intent state, dependencies, feedback, artifacts
- **Connect** — route intent to execution endpoints
- **Govern** — manage endpoints (manifest, policy, quota, audit)

If a proposed feature doesn't cleanly fit one — push back, the framing
or the feature is wrong.

## Predecessor reference

The v1 "offload" implementation lives at `../offload/` (sibling dir).

**It is reference-only. Do not import code from it.** When in doubt,
re-derive in v2 style. The Swift iOS code in particular contains
significant business-logic leakage (~30-40% of state computation done
client-side) — reading it pollutes thinking. Skip unless explicitly
investigating a specific v1 behavior.

What's worth referencing from v1 with care:

- State machine semantics (three-axis: requirement / execution / decision)
- Feedback type taxonomy (choose_one / choose_many / approve_reject / confirm_requirement)
- `.offload/topics/<id>/` data layout (single source of truth pattern)
- Virtual projects concept (non-repo intent grouping)

What to ignore:
- Session-as-API-root pattern (v2 makes Topic the root)
- SQLite schema (v2 uses Postgres, redesigned)
- Adapter implementation details (cc-bridge.mjs is CC-specific; v2 needs vendor-neutral protocol)
- Any client-side state computation

## Conventions

- Python + FastAPI + SQLModel + Postgres (control plane)
- TypeScript + React (web UI, when it lands)
- Native shells (iOS, etc.) are deferred and will be < 500 LOC each
- License: AGPL-3.0 core; enterprise plugins may differ
- Branch model: trunk-based on `main`; feature branches short-lived
