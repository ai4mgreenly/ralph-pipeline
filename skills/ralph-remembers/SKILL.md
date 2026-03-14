---
name: ralph-remembers
description: Store and retrieve documents from Ralph's long-term memory
---

# Ralph Remembers

Long-term memory store for Ralph agents. Documents are scoped by agent and/or project, support full-text search, and automatically create revision snapshots on update or delete. All scripts return JSON (`{"ok": true/false, ...}`).

## Setup

Scripts require one environment variable (typically set in the consuming project's `.envrc`):

```
RALPH_REMEMBERS_URL=http://ralph-remembers.localhost:8000
```

> **CRITICAL: Always use the full absolute path when invoking memory scripts.**
> Scripts live at `/home/ai4mgreenly/projects/ralph-pipeline/scripts/`.
> **Never invoke bare command names like `memory-list` or `memory-create`** — they are not on `$PATH` and will fail with "command not found".
> Every invocation must use the full path, e.g. `/home/ai4mgreenly/projects/ralph-pipeline/scripts/memory-create`.

## Data Model

Documents have: `id` (UUID), `agent`, `project`, `title`, `body`, `created_at`, `updated_at`. Revisions are automatic snapshots — each update or delete captures the prior state as an immutable revision with its own UUID and timestamp.

## Commands

All commands below must be invoked with their full absolute path: `/home/ai4mgreenly/projects/ralph-pipeline/scripts/<command-name>`. The table shows the script name for readability — always prepend the full path in actual invocations.

| Command | Usage | Does |
|---------|-------|------|
| `memory-create` | `[--agent AGENT] [--project PROJECT] [--title TITLE] < body` | Create document. Body via stdin. |
| `memory-get` | `<id>` | Retrieve a document by ID |
| `memory-update` | `<id> [--title TITLE] < body` | Replace body (and optionally title). Snapshots prior state as revision. |
| `memory-delete` | `<id>` | Soft-delete document. Excluded from list/search; revisions remain accessible. |
| `memory-list` | `[--agent AGENT] [--project PROJECT] [--title TITLE] [-q QUERY] [--limit N] [--offset N]` | List documents with optional filtering and full-text search |
| `memory-revisions` | `<id>` | List revision metadata for a document (newest first) |
| `memory-revision-get` | `<id> <rev>` | Retrieve a specific revision with full body |

## Examples

```bash
SCRIPTS=/home/ai4mgreenly/projects/ralph-pipeline/scripts

# Create a document
echo "Architecture decision: use SQLite for local storage" | \
  $SCRIPTS/memory-create --agent ralph --project ralph-pipeline --title "ADR: SQLite"

# Search across all documents
$SCRIPTS/memory-list -q "SQLite"

# Search within a specific agent+project scope
$SCRIPTS/memory-list --agent ralph --project ralph-pipeline -q "architecture"

# Filter by multiple agents
$SCRIPTS/memory-list --agent ralph --agent ralph-runs

# Get a document by ID
$SCRIPTS/memory-get a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Update a document (snapshots current state first)
echo "Updated notes" | $SCRIPTS/memory-update a1b2c3d4-e5f6-7890-abcd-ef1234567890 --title "New title"

# List revisions for a document
$SCRIPTS/memory-revisions a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Get a specific revision
$SCRIPTS/memory-revision-get a1b2c3d4-e5f6-7890-abcd-ef1234567890 rev1-uuid

# Soft-delete a document
$SCRIPTS/memory-delete a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

## Key Rules

- **Body via stdin** — `memory-create` and `memory-update` read body from stdin
- **Soft-delete semantics** — `memory-delete` marks documents as deleted; they disappear from `memory-list` and search results, but revision history remains accessible via `memory-revisions` and `memory-revision-get`
- **Revisions are automatic** — Every `memory-update` and `memory-delete` creates a revision snapshot of the prior state; no explicit action needed
- **Scope with agent/project** — Use `--agent` and `--project` to organize and filter documents; both are optional but help avoid collisions across agents and projects
- **`--agent` and `--project` are repeatable in `memory-list`** — Pass multiple values to match any of them (OR semantics)
- **Full-text search via `-q`** — Searches document bodies; combine with `--agent`/`--project` to narrow scope
- **Title uniqueness** — Titles are unique per agent+project scope; omit title for untitled documents
