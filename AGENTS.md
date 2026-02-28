# AGENTS.md

Claude Code plugin providing goal management skills for the Ralph autonomous development pipeline.

## Project Structure

```
.claude-plugin/plugin.json    Plugin manifest
skills/pipeline/SKILL.md      Goal lifecycle commands
skills/goal-authoring/SKILL.md  Goal writing guidelines
scripts/goal-*                Ruby CLI scripts (all return JSON)
local-git.md                  Local-first git architecture spec
```

## What This Does

Exposes two Claude Code skills (`/pipeline`, `/goal-authoring`) that let users create, queue, and manage goals. Goals are executable units of work that Ralph (the orchestrator in `ralph-runs`) picks up and runs autonomously in isolated repo clones.

## The Ralph Ecosystem

| Service | Port | Language | Purpose |
|---------|------|----------|---------|
| ralph-shows | 5000 | TypeScript/Deno | Web dashboard |
| ralph-plans | 5001 | Go | Goal state machine + SQLite storage (central API) |
| ralph-runs | 5002 | Ruby | Orchestrator — spawns agents, manages clones |
| ralph-logs | 5003 | Go | Real-time log streaming via WebSocket |
| ralph-counts | 5004 | Python | Execution metrics dashboard |

All services communicate via REST through `RALPH_*_HOST/PORT` env vars (set in `.envrc`).

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

Scripts require `RALPH_PLANS_HOST` and `RALPH_PLANS_PORT` environment variables.

## Key Conventions

- Goals specify **what**, never **how** — Ralph discovers the path
- Body format: `## Objective`, `## Reference`, `## Outcomes`, `## Acceptance`
- Default workflow is goals-first — local changes only when user explicitly requests
- `goal-create` reads body from stdin; `--org` and `--repo` are required
- Originally developed against `ikigai-1` (C11 terminal agent) but not tied to any specific project
