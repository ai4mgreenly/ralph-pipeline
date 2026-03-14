# AGENTS.md

> **All changes to this repo must go through ralph-goals — not local edits.**
> Working clones are ephemeral: each goal run starts from a fresh checkout of `main@origin`.
> Any local changes that are not committed and merged via a goal will be lost when the next goal runs.

Claude Code plugin providing goal management skills for the Ralph autonomous development pipeline.

## Project Structure

```
.claude-plugin/plugin.json    Plugin manifest
skills/                       Plugin skills (shipped to consumers)
scripts/                      Plugin scripts (shipped to consumers)
.claude/library/              Project-local skills (for working on this repo)
```

## What This Does

Exposes a Claude Code skill (`/ralph-goal`) that let users create, queue, and manage goals. Goals are executable units of work that Ralph (the orchestrator in `ralph-runs`) picks up and runs autonomously in isolated repo clones.

## The Ralph Ecosystem

| Service | URL | Language | Purpose |
|---------|-----|----------|---------|
| ralph-herds | `localhost:8000` | Go | Process supervisor and reverse proxy |
| ralph-shows | `localhost:8000` (default) | TypeScript/Deno | Web dashboard |
| ralph-plans | `ralph-plans.localhost:8000` | Go | Goal state machine + SQLite storage (central API) |
| ralph-runs | (no HTTP) | Ruby | Orchestrator — spawns agents, manages clones |
| ralph-logs | `ralph-logs.localhost:8000` | Go | Real-time log streaming via WebSocket |
| ralph-counts | `ralph-counts.localhost:8000` | Python | Execution metrics dashboard |
| ralph-remembers | `ralph-remembers.localhost:8000` | Go | Long-term memory with hybrid search |

All services communicate via REST through `RALPH_*_URL` env vars (set in `.envrc`).

## Goal Lifecycle

```
draft → queued → running → done
  │       │        │
  └───────┴────────┴──→ cancelled
                   └──→ stuck → queued (retry)
```

No PR step. Local bare repos on `/mnt/store/git/<org>/<repo>` are source of truth. GitHub is a push-only backup mirror.

## Scripts

All in `scripts/`, all Ruby, all return `{"ok": true/false, ...}`:

`goal-create` (stdin body), `goal-list`, `goal-get`, `goal-queue`, `goal-start`, `goal-done`, `goal-stuck`, `goal-retry`, `goal-cancel`, `goal-comment`, `goal-comments`

Scripts require `RALPH_PLANS_URL` environment variable.

## Key Conventions

- Goals specify **what**, never **how** — Ralph discovers the path
- Body format: `## Objective`, `## Reference`, `## Outcomes`, `## Acceptance`
- All changes go through ralph-goals — never make local edits directly to this repo
- `goal-create` reads body from stdin; `--org` and `--repo` are optional (auto-derived from `git remote get-url origin`)
- Originally developed against `ikigai-1` (C11 terminal agent) but not tied to any specific project
