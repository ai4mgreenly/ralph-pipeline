---
name: ralph-goal
description: Create and manage goals for Ralph execution
---

# Ralph Goals

Autonomous development pipeline. Goals are executable units of work. Ralph (the agent loop) picks up queued goals, executes them in isolated clones, and creates pull requests from the results. All goal scripts return JSON (`{"ok": true/false, ...}`).

## Setup

Scripts require two environment variables (typically set in the consuming project's `.envrc`):

```
RALPH_PLANS_HOST=localhost
RALPH_PLANS_PORT=5001
```

Scripts are located at `${CLAUDE_PLUGIN_ROOT}/scripts/`. When invoking any goal command, use this path to resolve the script location.

## Flow

```
Human writes goal → goal-create (draft) → goal-queue (queued) → Ralph executes → PR merges
```

## Default Workflow: Goals-First

**The goals-first workflow is the default for all work.** Local changes are rare exceptions that require explicit user instruction.

**Standard workflow:**

1. **Discuss** — User and Claude discuss the change and approach
2. **Create goal** — Claude creates the goal with clear acceptance criteria
3. **Queue immediately** — Goal is queued right after creation (default behavior)
4. **Ralph executes** — The orchestrator picks up the goal and runs it autonomously
5. **PR merges** — Completed work is merged via PR

**Default behaviors:**

- **Always queue after creation** — No manual testing or "trying it first" unless user explicitly requests it
- **No local changes** — Claude does not make local changes directly; work goes through Ralph

**When to make local changes (exceptions only):**

- User explicitly requests direct changes: "make this change now", "edit this file", "fix this directly"
- User explicitly says: "don't create a goal for this", "do this locally", "make this change here"

**If unsure:** Default to creating and queuing a goal. The user will specify if they want an exception.

## Goal Statuses

`draft` → `queued` → `running` → `done` (or `stuck`/`cancelled`)

## Goal Commands

All commands below are in `${CLAUDE_PLUGIN_ROOT}/scripts/`.

| Command | Usage | Does |
|---------|-------|------|
| `goal-create` | `--title "..." --org ORG --repo REPO [--model MODEL] [--reasoning LEVEL] < body.md` | Create goal (draft). Body via stdin. |
| `goal-list` | `[--status STATUS] [--org ORG] [--repo REPO]` | List goals, optionally filtered |
| `goal-get` | `<id>` | Read goal body + status |
| `goal-queue` | `<id>` | Transition draft → queued |
| `goal-cancel` | `<id>` | Cancel a non-terminal goal |
| `goal-comment` | `<id> --body "..."` | Add comment to a goal |
| `goal-comments` | `<id>` | List comments on a goal |
| `goal-done` | `<id>` | Transition running → done |
| `goal-retry` | `<id>` | Requeue a stuck goal |
| `goal-start` | `<id>` | Mark goal as running |
| `goal-stuck` | `<id>` | Mark goal as stuck |
| `goal-depend` | `add <id> <dep_id>` / `remove <id> <dep_id>` / `list <id>` | Add, remove, or list goal dependencies |

## Deriving org and repo

**Always derive `--org` and `--repo` from the current project's git remote origin.** Never guess, never ask the user — read the remote URL directly.

```bash
git remote get-url origin
```

Parse the result:

| Remote URL format | Example | org | repo |
|-------------------|---------|-----|------|
| `/mnt/store/git/<org>/<repo>` | `/mnt/store/git/acme/myapp` | `acme` | `myapp` |
| `git@github.com:<org>/<repo>.git` | `git@github.com:acme/myapp.git` | `acme` | `myapp` |

For the local bare repo path, split on `/` and take the last two segments (org = second-to-last, repo = last).

For the GitHub SSH URL, extract the `<org>/<repo>` part after the `:`, then strip the trailing `.git`.

**If the remote origin cannot be read** (e.g. `git remote get-url origin` fails or returns empty), stop and tell the user — do not guess or proceed with placeholder values.

## Creating a Goal

First, read the remote origin to get org and repo:

```bash
git remote get-url origin
# e.g. /mnt/store/git/acme/myapp  →  org=acme  repo=myapp
# e.g. git@github.com:acme/myapp.git  →  org=acme  repo=myapp
```

Then create the goal using those values:

```bash
cat <<'EOF' | goal-create --title "Add feature X" --org acme --repo myapp
## Objective
What should be accomplished.

## Reference
Relevant files, docs, and examples.

## Outcomes
Measurable, verifiable results.

## Acceptance
Success criteria.
EOF
```

Then queue it:

```bash
goal-queue <id>
```

## Key Rules

- **Body via stdin** — `goal-create` reads body from stdin
- **org + repo from remote** — Always run `git remote get-url origin` and parse it; never guess or ask the user unless the remote can't be read
- Goals go through Ralph — don't make local changes unless explicitly asked

## Goal Authoring

### Core Principle

**Ralph has unlimited context through iteration.** Don't artificially limit goals or references — Ralph can read all project documents, iterate through failures, and persist until outcomes are achieved.

### Key Principles

1. **Specify WHAT, never HOW** — Outcomes, not steps/order
2. **Reference liberally** — All relevant docs, Ralph reads across iterations
3. **One cohesive objective** — Not artificially small, not entire release
4. **Complete acceptance criteria** — Ralph needs to know when done
5. **Trust Ralph to iterate** — Discovers path, learns from failures
6. **Be explicit about discovery** — If work requires finding all instances of X, state it clearly in both objective and outcomes. Ralph has gotten stuck when discovery wasn't explicit. Write "Discover and fix all hardcoded paths" not "Fix hardcoded paths" or "Fix paths at lines 10, 25, 40"

### Example: Bad vs Good

**Bad (context-limited thinking):**
```markdown
## Objective
Create the config struct.

## Reference
docs/config.md section 2.1

## Outcomes
- Struct defined in src/config.h
```

Problems: Artificially small, minimal reference, no acceptance criteria.

**Good (leverages unlimited context):**
```markdown
## Objective
Implement configuration loading with environment variable overrides per docs/config.md.

## Reference
- docs/config.md — Format spec and defaults
- docs/environment.md — Env var override behavior
- src/main.c — Entry point where config is loaded
- tests/unit/config/ — Existing test structure

## Outcomes
- Config loading implemented per docs/config.md
- Environment variable overrides working
- Default values applied when env vars absent
- Unit tests in tests/unit/config/ pass

## Acceptance
- All tests pass
- CONFIG_DIR=/tmp/test ./app starts with correct config path
```

Why better: Complete objective, comprehensive references, measurable outcomes, clear acceptance.

**Discovery example:**
```markdown
## Objective
Discover all hardcoded /usr/local/etc path references in src/ and update them to use CONFIG_DIR environment variable.

## Reference
- src/paths.c — Central path resolution
- .envrc — Shows CONFIG_DIR=$PWD/etc

## Outcomes
- All hardcoded /usr/local/etc references in src/ discovered via grep
- All discovered files updated to use getenv("CONFIG_DIR")
- No hardcoded /usr/local/etc paths remain in src/

## Acceptance
- All tests pass
- grep -r "/usr/local/etc" src/ returns no hardcoded paths
```

Why explicit discovery matters: Objective starts with "Discover", first outcome confirms discovery happened. Ralph won't skip the grep step.

### Anti-Patterns

- **Step-by-step instructions** — "First do X, then Y" (Ralph discovers path)
- **Minimal references** — "Save context" (Ralph iterates, include all relevant)
- **Vague outcomes** — "Feature implemented" (Be specific and measurable)
- **Tiny goals** — Breaking cohesive work into artificial steps
- **Pre-discovered work** — Listing specific file:line locations to fix (Ralph should discover)

**Do this:** Comprehensive objective, all references, measurable outcomes, trust Ralph to iterate and discover.
