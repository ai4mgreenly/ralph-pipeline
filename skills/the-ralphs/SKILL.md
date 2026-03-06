---
name: the-ralphs
description: Overview of the Ralph nano-services and how they fit together
---

# The Ralphs

An ad-hoc collection of nano-services that grew into an autonomous software development pipeline. Each service started as a quick hack to solve an immediate friction point — a few hundred lines to scratch an itch — and over time they settled into a coherent system.

## The Services

**ralph-plans** (Go/SQLite, port 5001) — Where goals live. A REST API wrapping a SQLite database that tracks goals through their lifecycle: `draft → queued → running → done` (with `stuck` for failures and retry). Started as a replacement for GitHub Issues when those proved too clunky for goal management. Handles dependencies, comments, and attachments.

**ralph-runs** (Ruby, port 5002) — The orchestrator. Polls ralph-plans for queued goals, clones repos into isolated directories, spawns Claude agents to work on them, and merges results back to main. Retries up to 3 times on failure. Uses jujutsu (jj) for version control. This was the first piece — originally just a script to stop manually running agents.

**ralph-shows** (TypeScript/Deno, port 5000) — Web dashboard. Polls ralph-plans every 2 seconds and shows what's running, queued, stuck, done. Read-only. Built because constantly curling the API got old.

**ralph-logs** (Go, port 5003) — Log tail in a browser. Watches log files via glob patterns and streams them over WebSocket. Handles file rotation. Built because `tail -f` across multiple agent runs was unmanageable.

**ralph-counts** (Python, port 5004) — Metrics dashboard. Reads an append-only JSONL file of completed runs and shows cost, tokens, time, iterations, and lines changed. Built to answer "is this actually saving time?"

**ralph-remembers** (Go, no port yet) — Long-term memory service. Persistent document store with hybrid search (vector + BM25) so agents can retain knowledge across sessions and projects. Still taking shape.

**ralph-herds** (not yet implemented) — Process supervisor. A single entry point to launch, monitor, and restart all the other nano-services. Built because manually starting six services got tedious. Herding cats.

## How They Talk

All services run on localhost. Communication is REST via `RALPH_*_HOST/PORT` environment variables (set in each project's `.envrc`). No service discovery, no message queues, no shared databases. Each service owns its own storage.

## Three-Tier Git

The system grew into a three-tier git architecture:

1. **Bare repos** at `/mnt/store/git/<org>/<repo>` — the source of truth
2. **GitHub** — push-only backup mirror (not the operational center)
3. **Working clones** — `~/projects/<repo>` for humans, disposable per-goal clones for agents

Each goal gets a fresh, isolated clone. Agents don't share state. Results merge back to the bare repo, which pushes to GitHub.

## How Goals Work

Goals specify **what**, never **how**. A goal has an objective, reference material, expected outcomes, and acceptance criteria. Ralph (the agent) figures out the implementation. Validation runs through a `.ralph/check` script in each repo — no human code review in the loop.

All scripts return JSON: `{"ok": true/false, ...}`. The whole system is glued together with small CLI scripts that parse flags and talk to APIs.

## The Pattern That Emerged

No grand design. Each service was built in under an hour when something became painful enough to automate. The architecture fell out of a few recurring choices: keep services tiny, don't share databases, use files and REST, make agents disposable. The result is a pipeline where intelligence lives at the top (goal composition) and everything below is mechanical execution.
