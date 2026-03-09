---
name: ralph-goal
description: Create and manage goals for Ralph execution
---

# Ralph Goals

Autonomous development pipeline. Goals are executable units of work. Ralph (the agent loop) picks up queued goals, executes them in isolated clones, and creates pull requests from the results. All goal scripts return JSON (`{"ok": true/false, ...}`).

## Setup

Scripts require one environment variable (typically set in the consuming project's `.envrc`):

```
RALPH_PLANS_URL=http://ralph-plans.localhost:8000
```

> **CRITICAL: Always use the full absolute path when invoking goal scripts.**
> Scripts live at `/home/ai4mgreenly/projects/ralph-pipeline/scripts/`.
> **Never invoke bare command names like `goal-list` or `goal-create`** — they are not on `$PATH` and will fail with "command not found".
> Every invocation must use the full path, e.g. `/home/ai4mgreenly/projects/ralph-pipeline/scripts/goal-create`.

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

All commands below must be invoked with their full absolute path: `/home/ai4mgreenly/projects/ralph-pipeline/scripts/<command-name>`. The table shows the script name for readability — always prepend the full path in actual invocations.

| Command | Usage | Does |
|---------|-------|------|
| `goal-create` | `--title "..." [--org ORG] [--repo REPO] [--model MODEL] [--reasoning LEVEL] < body.md` | Create goal (draft). Body via stdin. org/repo auto-derived from remote origin. |
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
| `goal-abort` | `<id>` | Abort a running goal (moves to stuck) |
| `goal-depend` | `add <id> <dep_id>` / `remove <id> <dep_id>` / `list <id>` | Add, remove, or list goal dependencies |
| `goal-attachment-create` | `<goal_id> --name "name.md" < body.md` | Create attachment from stdin |
| `goal-attachments` | `<goal_id>` | List attachments for a goal |
| `goal-attachment-get` | `<goal_id> <attachment_id>` | Get single attachment with body |
| `goal-attachment-edit` | `<goal_id> <attachment_id> --old-str "..." --new-str "..."` or `--body "..."` | Edit attachment body (substring or full replacement) |
| `goal-attachment-delete` | `<goal_id> <attachment_id>` | Delete attachment |

## org and repo: Auto-Derived from Remote Origin

**`goal-create` automatically derives org and repo from `git remote get-url origin`** when `--org` and `--repo` are omitted. You do not need to pass these flags for the current directory's repo.

**Never guess, never invent, never hardcode org or repo values.** They must always come from the git remote origin — either auto-derived by the script or explicitly passed as flags that you read from the remote yourself.

The script handles both URL formats:

| Remote URL format | Example | org | repo |
|-------------------|---------|-----|------|
| `/mnt/store/git/<org>/<repo>` | `/mnt/store/git/acme/myapp` | `acme` | `myapp` |
| `git@github.com:<org>/<repo>.git` | `git@github.com:acme/myapp.git` | `acme` | `myapp` |

Works in both git and jj repositories (jj uses git remotes under the hood).

**If the remote origin cannot be read**, the script exits with a JSON error — do not guess or proceed with placeholder values.

**Use `--org`/`--repo` only when targeting a different repo** than the current directory. For the current repo, omit them entirely.

## Creating a Goal

```bash
cat <<'EOF' | /home/ai4mgreenly/projects/ralph-pipeline/scripts/goal-create --title "Add feature X"
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

org and repo are auto-derived from `git remote get-url origin` in the current directory.

To target a different repo, pass the flags explicitly — but still read the values from that repo's remote origin, never guess:

```bash
# Only when targeting a different repo:
cat <<'EOF' | /home/ai4mgreenly/projects/ralph-pipeline/scripts/goal-create --title "Add feature X" --org acme --repo other-repo
...
EOF
```

Then queue it:

```bash
/home/ai4mgreenly/projects/ralph-pipeline/scripts/goal-queue <id>
```

## Attachments

Attachments are markdown text files associated with a goal. Each attachment has a unique name per goal and a body (markdown text). They are useful for storing supplementary context, reports, or notes alongside a goal.

| Command | Usage | Does |
|---------|-------|------|
| `goal-attachment-create` | `<goal_id> --name "name.md" < body.md` | Create attachment from stdin |
| `goal-attachments` | `<goal_id>` | List attachments for a goal |
| `goal-attachment-get` | `<goal_id> <attachment_id>` | Get single attachment with body |
| `goal-attachment-edit` | `<goal_id> <attachment_id> --old-str "..." --new-str "..."` or `--body "..."` | Edit attachment body |
| `goal-attachment-delete` | `<goal_id> <attachment_id>` | Delete attachment |

### Edit Modes

`goal-attachment-edit` supports two modes:

- **Substring replacement** — `--old-str "find" --new-str "replace"`: Replaces the first unique occurrence of `old-str` in the body. Fails if the string is not found or matches more than once.
- **Full body replacement** — `--body "new content"` or `--body -` (reads from stdin): Replaces the entire attachment body.

### Examples

```bash
SCRIPTS=/home/ai4mgreenly/projects/ralph-pipeline/scripts

# Create an attachment
echo "# Notes\nSome context here." | $SCRIPTS/goal-attachment-create 42 --name "notes.md"

# List attachments
$SCRIPTS/goal-attachments 42

# Get a single attachment
$SCRIPTS/goal-attachment-get 42 7

# Edit with substring replacement
$SCRIPTS/goal-attachment-edit 42 7 --old-str "old text" --new-str "new text"

# Edit with full body replacement (via stdin)
cat updated-notes.md | $SCRIPTS/goal-attachment-edit 42 7 --body -

# Delete an attachment
$SCRIPTS/goal-attachment-delete 42 7
```

## Key Rules

- **Body via stdin** — `goal-create` reads body from stdin
- **org + repo auto-derived** — Omit `--org`/`--repo` for the current directory; the script derives them from `git remote get-url origin`
- **Never guess org/repo** — If you must pass them explicitly, read them from the target repo's remote origin first
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

**Implementation recipe example:**

Bad (outcomes as pseudocode):
```markdown
## Outcomes
- 1. Call `db.connect(config.dsn)` in `main()` before handler registration
- 2. Wrap each handler with `withDB(db)` middleware using a closure
- 3. Call `rows.Scan(&user.ID, &user.Name, &user.Email)` in `GetUser()`
- 4. Add `defer db.Close()` at end of `main()`
```

Problems: This is an implementation recipe, not an end state. Ralph reads numbered steps and executes them all at once instead of iterating. If step 3 fails, Ralph has no way to verify step 2 independently. The steps also encode brittle assumptions — field order in `Scan`, the exact middleware shape — that Ralph may need to adapt.

Good (outcomes as observable end-state properties):
```markdown
## Outcomes
- Database connection is established at startup using `config.dsn`
- All HTTP handlers have access to the database connection
- `GET /users/:id` returns correct user fields (id, name, email) from the database
- Application shuts down cleanly without connection leaks
```

Why better: Each outcome is independently verifiable. Ralph can iterate on each one separately, adapt the implementation to the codebase, and verify correctness through tests rather than checking off pseudocode steps.

### Anti-Patterns

- **Step-by-step instructions** — "First do X, then Y" (Ralph discovers path)
- **Minimal references** — "Save context" (Ralph iterates, include all relevant)
- **Vague outcomes** — "Feature implemented" (Be specific and measurable)
- **Tiny goals** — Breaking cohesive work into artificial steps
- **Pre-discovered work** — Listing specific file:line locations to fix (Ralph should discover)
- **Implementation recipes** — Numbered steps, pseudocode, or function call sequences in Outcomes. Outcomes describe the end state, not the steps to get there. Ralph sees numbered steps and executes them all at once instead of iterating.

**Do this:** Comprehensive objective, all references, measurable outcomes, trust Ralph to iterate and discover.
