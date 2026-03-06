---
name: repo
description: Create new repositories with the full ralph three-tier git structure
---

# Repo

Create new repositories with the ralph three-tier git architecture:

1. **GitHub** — remote repository (push-only backup mirror)
2. **Bare repo** — `/mnt/store/git/<org>/<repo>` (source of truth)
3. **Local clone** — `~/projects/<repo>` (working checkout)

## Script

```
${CLAUDE_PLUGIN_ROOT}/scripts/repo-create --org ORG --repo REPO [--private] [--local-path PATH]
```

### Flags

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--org` | yes | — | GitHub organization |
| `--repo` | yes | — | Repository name |
| `--private` | no | public | Create private GitHub repo |
| `--local-path` | no | `~/projects/<repo>` | Local clone path |

### Output

```json
{"ok": true, "org": "foo", "repo": "bar", "github": "foo/bar", "bare": "/mnt/store/git/foo/bar", "local": "/home/user/projects/bar"}
```

## Examples

- "Create a mgreenly/my-app repo" → `repo-create --org mgreenly --repo my-app`
- "Create a private mgreenly/my-app repo" → `repo-create --org mgreenly --repo my-app --private`
- "Create mgreenly/my-app in ~/work/my-app" → `repo-create --org mgreenly --repo my-app --local-path ~/work/my-app`

## Pre-flight Checks

The script validates before making any changes:
- `gh` CLI is authenticated
- Bare repo path doesn't already exist
- Local path doesn't already exist
- `/mnt/store/git` is writable

## Error Handling

On failure, the output includes a `completed` array showing which steps succeeded (e.g., `["github", "bare"]`), so you can report what was created and what wasn't. No automatic rollback — partial state is easier to fix manually.

Network operations retry up to 3 times with a 2-second delay.
