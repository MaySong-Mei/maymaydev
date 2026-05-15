# maymaydev v2 protocol — design sketch

**Status:** draft 0.1, expects refinement during early CLI/server implementation.

Goal: describe the wire-level shape of the maymaydev control plane so
that the server, CLI, and (later) GUI clients can be built against a
stable contract.

## Goals

- One protocol drives all clients (CLI, web, native shells, third-party,
  agent-triggered)
- Server returns **view objects** — client only renders
- Multi-tenant safe from day 1 (`workspace_id` on every record)
- Executor daemons connect to the control plane via reverse WebSocket
- Domain events only — no UI vocabulary in the protocol

## Non-goals (for v0.1)

- A2A inter-agent protocol — assume single-agent-per-run; A2A is a
  future expansion of `Run`
- Full enterprise features (SSO, audit export, policy engine) — those
  ship as plugins later
- Backward compatibility with offload v1 — clean break

## Object model

Concrete fields are illustrative; types finalized during implementation.

```
Workspace
  id, name, created_at, plan

User
  id, email, workspaces[] (with role per workspace)

Endpoint  -- execution endpoint registered to a workspace
  id, workspace_id, name, kind ("machine" | "cloud_task" | "agent_service")
  capabilities: { tags: [...], custom: {...} }
  manifest: { hostname, os, agents: [{name, kind, version}], ...}
  status: "online" | "offline" | "draining"
  last_seen, registered_at

Topic  -- first-class intent unit (replaces v1 "session" as API root)
  id, workspace_id, project_id (nullable), parent_topic_id (nullable)
  title, raw_input, summary
  requirement_state: "captured" | "clarifying" | "discussed" | "specified" | "approved"
  execution_state: "idle" | "queued" | "implementing" | "implemented" | "human_testing" | "passed" | "failed" | "paused"
  decision_state: "none" | "needs_feedback" | "blocked" | "pending_implementation" | "archived"
  created_at, updated_at, requirement_approved_at, plan_approved_at

Run  -- one execution attempt of a Topic on an Endpoint
  id, workspace_id, topic_id, endpoint_id
  agent: "claude_code" | "openai_codex" | "local_model" | ...  (executor-declared)
  status: "queued" | "running" | "succeeded" | "failed" | "cancelled"
  started_at, ended_at, exit_code, artifacts[]

Feedback
  id, workspace_id, topic_id, run_id (nullable)
  kind: "choose_one" | "choose_many" | "approve_reject" | "confirm_requirement"
  prompt, options[], status: "pending" | "resolved"
  response (when resolved), resolved_at

Event  -- audit + realtime
  id, workspace_id, type (domain-level, see taxonomy)
  topic_id, run_id, endpoint_id (any nullable)
  ts, payload (type-specific, structured)

Project  -- intent grouping (repo-bound OR virtual)
  id, workspace_id, name, kind: "repo" | "virtual"
  repo_path (when kind=repo, on which endpoint), description
```

## State machines (preserved from v1, validated as useful)

The three-axis Topic model is retained. Axes are independent — Topic
state is a tuple `(requirement_state, execution_state, decision_state)`.

This is unusual but worked in v1 because clarification, execution, and
human decision-making run on different timelines. Single-axis state
machines forced false serialization.

**Transitions are server-computed.** The server returns
`available_actions: [...]` on Topic view; the client only renders.

## API pattern — view objects

Every GET returns a view object. View objects are functions of state
plus derivations the client would otherwise have to compute.

### GET /workspaces/{w}/topics/{id}

```json
{
  "topic": {
    "id": "t_abc",
    "title": "Add rate limiting to /api/upload",
    "raw_input": "we keep getting hammered on uploads...",
    "summary": "Rate limit upload endpoint to 10 req/min per user",
    "requirement_state": "approved",
    "execution_state": "implementing",
    "decision_state": "none",
    "created_at": "2026-05-15T10:30:00Z",
    "project": { "id": "p_xyz", "name": "api-server", "kind": "repo" },
    "parent": null
  },
  "derived": {
    "needs_attention": false,
    "can_archive": false,
    "pipeline_stage": "implementing",
    "blocking_reason": null
  },
  "available_actions": [
    { "action": "cancel_run", "enabled": true, "label": "Cancel current run" },
    { "action": "request_feedback", "enabled": true, "label": "Ask for feedback" },
    { "action": "approve_plan", "enabled": false, "reason": "no plan to approve" }
  ],
  "current_run": { "id": "r_001", "status": "running", "endpoint_id": "e_mac1", "started_at": "..." },
  "runs": [{ "id": "r_001", ... }, ...],
  "pending_feedback": [],
  "resolved_feedback": [{ "id": "f_01", ... }],
  "events_summary": { "total": 47, "last_event_type": "tool_call_completed", "last_event_at": "..." }
}
```

### GET /workspaces/{w}/dashboard

```json
{
  "workspace": { "id": "w_001", "name": "personal" },
  "endpoints": {
    "online": [
      { "id": "e_mac1", "name": "MacBook Pro", "current_run_id": "r_001", "last_seen": "..." },
      { "id": "e_gpu", "name": "GPU box", "current_run_id": null, "last_seen": "..." }
    ],
    "offline": [{ "id": "e_pi", "name": "sensor pi", "last_seen": "..." }]
  },
  "pipeline": {
    "captured": { "count": 5, "topics": [...] },
    "specified": { "count": 10, "topics": [...] },
    "approved": { "count": 8, "topics": [...] },
    "implementing": { "count": 2, "topics": [...] },
    "human_testing": { "count": 1, "topics": [...] },
    "passed": { "count": 15, "topics": [...] },
    "archived": { "count": 1, "topics": [...] }
  },
  "needs_attention": { "count": 3, "topics": [...] },
  "recent_runs": [...]
}
```

### GET /workspaces/{w}/endpoints

```json
{
  "endpoints": [
    {
      "id": "e_mac1",
      "name": "MacBook Pro",
      "kind": "machine",
      "status": "online",
      "manifest": {
        "hostname": "may-mbp.local",
        "os": "darwin 25.4",
        "agents": [{ "name": "claude_code", "kind": "cc", "version": "..." }]
      },
      "capabilities": { "tags": ["dev", "git", "node", "python"] },
      "current_runs": [{ "id": "r_001", "topic_id": "t_abc", ... }],
      "last_seen": "..."
    }
  ]
}
```

**Notice:** the client never computes "is this endpoint online" from
`last_seen`. The server sets `status` based on its own policy and
returns it directly.

## Events — domain taxonomy

Events are emitted by the server and delivered via WebSocket to
subscribed clients. They are **never UI-specific**.

```
intent.captured                  -- Topic created
topic.summary_updated
topic.state_changed              -- any of three axes
topic.requirement_approved
topic.plan_approved

run.queued
run.started
run.tool_invoked                 -- aggregated from underlying agent events
run.message_appended             -- pre-assembled message, no raw stream
run.completed                    -- {status: succeeded|failed|cancelled}

feedback.requested
feedback.resolved

endpoint.registered
endpoint.connected
endpoint.disconnected
endpoint.capability_changed
```

**Pre-assembly rule:** if an underlying agent (Claude Code, Codex, etc.)
emits raw token / tool-call stream events, the server aggregates them
into a coherent `run.message_appended` event before forwarding. Clients
never see raw streams.

## Connection topology

```
Client (CLI / web / native)
  ↓ HTTPS + WSS
Control plane (FastAPI + Postgres)
  ↑ WSS reverse connection
Executor daemon (one per machine)
  ↓ subprocess
Local agent (Claude Code / Codex / model)
```

**Executor daemon ↔ control plane:** daemon initiates outbound WSS to
the configured control plane URL (offload.com or self-hosted). No
inbound ports needed on user machines. Auth via daemon-issued token.

**Client ↔ control plane:** standard HTTP for state, WS for events.
Client auth via user session token (cookie or bearer).

## CLI surface (sketch)

The CLI is the protocol conformance test. Anything possible via API
must be possible via CLI.

```
maymay auth login
maymay workspace list
maymay workspace use <name>

maymay topic create "<raw intent>" [--project <p>]
maymay topic show <id>                  # renders view object
maymay topic list [--filter active|attention|archived]
maymay topic update <id> --summary "..."
maymay topic feedback <id>              # interactive resolve pending
maymay topic archive <id>

maymay run start <topic_id> --endpoint <e_id> --agent <agent>
maymay run logs <run_id> [--follow]
maymay run cancel <run_id>

maymay endpoint list
maymay endpoint show <id>
maymay endpoint register               # to be run on the executor machine itself

maymay dashboard                       # render workspace dashboard view
maymay events tail [--topic <id>]      # subscribe to event stream
```

## Open questions (to resolve during implementation)

1. **Endpoint identity & pairing flow.** How does a fresh machine
   register? Magic-link from dashboard? Manual token paste? Need a
   spec before executor daemon work starts.

2. **Capability vocabulary.** Free-form tags vs predefined vocab vs
   both. Lean: both — common tags (`gpu`, `python`, `git`, `internet`)
   are predefined; user-defined tags allowed for extensibility.

3. **Routing decisions.** v1: user manually picks endpoint per run.
   v2 day 1: same (UI dropdown picks from compatible endpoints based
   on capability). v2 future: rule-based, then LLM-assisted. The API
   should accept either an explicit `endpoint_id` or a `requirements`
   block; if `requirements`, server picks.

4. **Multi-tenant boundaries.** Workspaces are tenants. Cross-workspace
   sharing of Endpoints or Topics: deferred to v1.1 or later. Day 1:
   strict workspace isolation, easier security model.

5. **Auth.** Single-tenant self-host: bearer token from config file.
   Multi-tenant SaaS: OAuth via offload.com (when domain decided) +
   workspace invites. Both share the same internal token model.

6. **Artifact storage.** Where do `run.artifacts[]` live? On the
   executor daemon's filesystem (referenced by URL the daemon serves)?
   In control-plane object storage? Day 1: executor-local, control
   plane has URLs only.

7. **Pre-assembled message granularity.** How aggressive is the server
   aggregating raw tool events? Lean: one `run.message_appended` per
   logical agent turn (system message, user message, assistant
   response — each one event). Tool calls are sub-events within
   `run.tool_invoked`.

## What this protocol does NOT include yet

- Web push / mobile push notifications (delivered to clients via
  separate notification service)
- Billing / subscription endpoints (commerce module, separate)
- Admin APIs (workspace user management, plugin config — later)
- Agent-to-agent (A2A) message passing — future expansion

## Implementation order

This protocol document is the contract. Implementation order:

1. Skeleton: FastAPI + Postgres + SQLModel + workspace/user/auth
2. Topic CRUD with view object API + CLI commands
3. Endpoint registration + reverse WS daemon ↔ control plane
4. Run lifecycle: dispatch Topic → Endpoint, event aggregation
5. Feedback request/resolve
6. Dashboard view aggregation
7. (deferred) Web UI consuming the same API
8. (deferred) Native shells
