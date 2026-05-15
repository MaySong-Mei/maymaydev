# maymaydev

> Intent platform — capture, manage, route, and govern AI work across machines.

## Status

Greenfield. No code yet. Design phase.

A predecessor implementation ("offload") existed and is preserved as
lessons-learned only; see [docs/v1-lessons.md](docs/v1-lessons.md) and
[docs/v1-diary.md](docs/v1-diary.md).

## What this is

A control plane for AI work, organized around four verbs:

- **Capture** — accept intent from any client (web, native, CLI, agent)
- **Manage** — track intent state, dependencies, feedback, artifacts
- **Connect** — route intent to execution endpoints
- **Govern** — manage endpoints themselves (manifest, policy, audit)

Execution endpoints — machines running an executor daemon — stay
sovereign. Code, credentials, and environment never leave the user's
hardware. The control plane mediates intent and state, not files.

## Architectural commitments

- **Headless service**: server returns view objects; UIs (web, native,
  CLI) are interchangeable consumers; no business logic in clients.
- **Dual deployment**: same codebase ships as hosted SaaS and as
  self-host (Docker, single-tenant); difference is config, not fork.
- **Multi-tenant from day 1**: every record carries `workspace_id`;
  auth is a pluggable module.
- **Postgres from day 1**: no SQLite migration debt.
- **CLI is the conformance test**: any feature must be doable from the
  terminal before any GUI client gets it.

## License

AGPL-3.0 — see [LICENSE](LICENSE). The OSS core stays AGPL; future
enterprise plugins may carry separate licensing.
