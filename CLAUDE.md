# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status: pre-implementation

This repository has no application code yet. It currently contains only project-tracking artifacts: `README.md`, `LICENSE`, and `docs/github-issues-index.md`. All planning and task breakdown lives in GitHub Issues (#1–#34), organized by roadmap phase — read `docs/github-issues-index.md` for the full phase → issue mapping before starting any work, and read the specific issue(s) for the phase you're touching before writing code. Don't assume acceptance criteria from one phase apply to another.

Vastask (the product these issues describe) is a **personal, single-user tool**, not a hosted product: one person, one machine, no accounts, no multi-tenancy, no cloud. It syncs GitHub issues onto a local board, hands one to Claude Code on request, runs it in an isolated git worktree, and holds the result at a pull request until the user approves it.

## Where to start, and what's gated on what

- **Issue #2** ("Phase 1 — The Spike") is an explicit go/no-go gate for the whole app track. It's deliberately one script — no database, no UI, no framework. Nothing in the app Phases 2–9 (issues #3–#9, #15–#32) should start before it passes.
- **Issue #10** ("Landing Page — Positioning") is the only landing-page work *not* gated on the spike — it's writing, not building. Issues #11–#14 (the actual page build) wait on #2, and #13 (hero recording) specifically needs a real dispatch to record, i.e. app Phase 5 (#6).
- Sub-issues (#15–#34) attached under a phase-tracking issue (#1–#14) break that phase into its distinct pieces of work, each with its own acceptance criteria — GitHub's sub-issue relationship, not just a checklist, so the parent issue's progress bar reflects them automatically.
- Issue #9's sub-issues (#31 usage period, #32 revisit gate) are a 30-day observation period and a decision gate, not code to write. There is deliberately no issue for "Phase 10" (business exploration) — it stays unscoped until #32 says go; don't create one preemptively.

## Locked architecture decisions

These apply to all future code in this repo. They exist specifically to stop settled questions from being relitigated — several were already reversed once during planning (multi-tenant product → personal tool, `--bare` on → off, webhooks → polling), and re-opening them costs a full session each time.

- **Single local process.** One Node/Bun process serves the REST API, the static frontend, and runs the agent — no separate runner daemon, no cloud component, no WebSocket-to-a-server.
- **GitHub access via the `gh` CLI**, not the GitHub API directly — it's already authenticated on the machine; this avoids an OAuth/App-installation subsystem entirely.
- **Sync is polling, not webhooks** — every 60s while the app is open, plus immediately after any local mutation. GitHub can't deliver webhooks to `localhost`.
- **SQLite** (`better-sqlite3` or `bun:sqlite`), six tables: `task`, `task_event` (append-only, covers every actor: `me`/`agent`/`github`/`system`), `run`, `run_event` (every streamed `stream-json` line, high write volume), `pull_request`, `repo`/`config`. `task.github_issue_number` is nullable — local-only tasks are allowed. `task` ↔ `run` is one-to-many; never assume a single run per task.
- **SSE, not WebSocket**, for streaming run logs to the browser — traffic is one-directional (server → client) only.
- **Agent invocation contract**: `claude -p "<prompt>" --output-format stream-json --verbose --include-partial-messages`, run inside the dispatch worktree, no `--bare`, `ANTHROPIC_API_KEY` set explicitly on a dedicated key, `--permission-mode acceptEdits` plus a narrow `--allowedTools` allowlist (read/edit/git only). `--dangerously-skip-permissions` is never used. Capture `session_id` (from `system/init`) and `total_cost_usd` (from the final `result` event) on every run.
- **Every dispatch runs in a fresh `git worktree`** (branch `vastask/task-<issue-number>-<short-id>`, cut from a freshly-fetched default branch, in a directory outside the normal working tree). A run must proceed normally even with uncommitted changes sitting in the primary working directory, and never touch it — this is flagged repeatedly as the single most important behavior to verify by test, not assume.
- **Board is five fixed, non-configurable columns**: `Backlog → Ready → In Progress → In Review → Done`. `Ready` is the dispatch-eligible column. Drag-and-drop is polish, never a requirement — dropdown status change is sufficient.

## Non-negotiable invariants

Flagged across the planning docs as "never cut," regardless of time pressure on any given phase:

1. **Worktree isolation** — a run can never touch the user's working directory.
2. **The editable prompt** — the full composed prompt is always shown to the user, editable, before any run starts. Never dispatch a prompt the user hasn't seen.
3. **The append-only event log** (`task_event` / `run_event`) — every actor's mutations are recorded, never overwritten or batched away.
4. **The approval gate** — no agent-authored change reaches the default branch without an explicit, recorded approval action. A code path that could bypass this is a broken core promise, not a minor bug.

## Commands

None yet — there is no build, lint, or test tooling in this repository, because there is no code. The first code-producing work is Issue #2, which starts as a single standalone script with no framework, database, or UI by design. Don't scaffold a framework, `package.json`, or test suite ahead of the issue that actually needs it; the roadmap is explicit that Phase 1 is deliberately minimal and that added tooling should land only when a specific phase's issue calls for it.
