# Phase 0 — Prep Notes

Working notes and decision record for [Issue #1](../../../issues/1) ("Phase 0 — Prep"). Exists for the same reason `github-issues-index.md` does: the research and reasoning behind these decisions shouldn't live only in chat history, since Phase 2 (`run_event` schema, #3/#15) and Phase 7 (worktree cleanup, #29) will need to reference it later.

## Vibe Kanban research (informs the Phase 2 `run_event` schema)

Source: [BloopAI/vibe-kanban](https://github.com/BloopAI/vibe-kanban) (Apache-2.0) — a Rust/React Kanban board that dispatches coding agents (Claude Code, Codex, etc.) into isolated git worktrees. Closest existing prior art to what Vastask is building. Bloop (the company) shut down in April 2026; the project continues as community-maintained.

### Finding 1 — they moved append-only agent logs *off* SQLite, onto the filesystem

Blog post: ["Goodbye SQLite (for logs)"](https://www.vibekanban.com/blog/goodbye-sqlite-for-logs) (not directly fetchable from this session — network egress to vibekanban.com is blocked here; summarized from search results, so treat specifics as approximate, not verbatim quotes).

- Under real workloads they hit recurring `database is locked` errors. SQLite writes serialize behind a database-level lock, so a long write can stall unrelated reads and writes elsewhere in the app.
- ~98% of their database's on-disk size was logs, and lock incidents correlated with large log volumes being written in parallel (multiple agents running concurrently).
- Their logs table was effectively an append-only, non-relational blob store — never queried with joins or filters, just read back sequentially — so it wasn't buying them anything by being *in* the relational database.
- Fix: they moved execution logs to JSONL files on the filesystem, one file per execution process, at a path shaped like `/<first two chars of session id>/<session id>/<execution_process_id>.jsonl`. Result: DB size dropped from gigabytes to ~20 MiB, lock errors disappeared, and reads got faster since they no longer scanned a huge indexed logs table.

**Relevance to Vastask:** AGENTS.md already locks `run_event` in as a SQLite table ("every streamed `stream-json` line, high write volume") — that's settled, not being relitigated here. But VK's failure mode is exactly the shape of table we're about to design in Phase 2, so it's worth designing defensively rather than rediscovering the same wall:

- Vastask's concurrency profile is much lighter than VK's — one person, one machine, realistically one dispatch running at a time (the architecture doesn't call for 4-5 parallel agents the way VK's multi-user board does). The specific lock-contention failure VK hit is less likely to bite at this scale.
- Still worth carrying into the Phase 2 schema design (#15):
  - Use SQLite's WAL mode (`PRAGMA journal_mode=WAL`) so `run_event` writes don't block concurrent reads (e.g. the SSE stream reading while the agent is still writing).
  - Store each `stream-json` line as an opaque `TEXT`/`BLOB` column, not parsed into indexed columns — nothing here needs to be queried relationally, only replayed sequentially (matches the "reopen-mid-run replay" requirement in #23).
  - Don't add indexes to `run_event` beyond what replay-by-`run_id`-in-order needs. VK's pain came partly from an indexed logs table growing huge; a minimal index avoids that regardless of storage backend.
  - If disk usage or lock contention ever becomes a real, *measured* problem post-launch, VK's filesystem-JSONL-per-run approach is the proven fallback pattern — but that's a "if it happens" note, not a pre-emptive redesign. Locked is locked.

### Finding 2 — worktree lifecycle

- Branch naming: `vk/<4-char-id>-<task-slug>` (configurable prefix in settings). Structurally the same shape as Vastask's already-locked `vastask/task-<issue-number>-<short-id>` — good corroboration that this pattern is proven, not just theorized.
- **Orphan cleanup**: on startup, VK removes worktree directories that have no matching workspace record in the database (handles the case where the app crashed mid-deletion, leaving a dangling directory).
- **Expired cleanup**: a worktree tied to a task that's moved to Done/archived gets cleaned up after a 1-hour grace interval. Deleting a task removes its worktree immediately.
- Cleanup is fully disableable via an env var, kept as an explicit debugging escape hatch.
- Rough edges visible in their public issue tracker even after this shipped (titles only, not read in full): a run/`ExecutionProcess` can stay stuck "running" after its worktree has already gone missing (a "ghost run" with no further logs), and worktrees have shipped bugs where they weren't cleaned up after a merge. Both read as reconciliation gaps between DB state and on-disk state, not fundamental design flaws.

**Relevance to Vastask:** confirms the locked worktree-per-dispatch design is sound. Two concrete things worth remembering when Phase 2's run lifecycle (#16) and Phase 7's cleanup policy (#29) get built:
- Whatever marks a `run` as terminal (done/failed/cancelled) should be able to tolerate the worktree already being gone by the time cleanup runs, rather than assuming the two stay in lockstep.
- An orphan-sweep-on-startup (worktree dirs with no matching `run`/`task` row) is a cheap, proven safety net worth planning for, independent of whatever the eventual cleanup *policy* (time-based, manual, on-merge) turns out to be.
- No decision is being made here — per the recommendation below, cleanup policy is explicitly deferred to Phase 7 (#29). This is just a reference point for whoever picks that up.

## Decisions ("decide as you build" items from the v0 doc)

### Local-only tasks (no linked GitHub issue) — decided: allow them

Recommendation from the issue was to allow them and make `github_issue_number` nullable. This is **already recorded** as a locked architecture decision in `AGENTS.md`: *"`task.github_issue_number` is nullable — local-only tasks are allowed."* No further action needed here — noting it in this doc just so Issue #1 has a visible record of where the decision lives, rather than requiring cross-referencing chat history.

### Worktree cleanup policy — decided: don't decide yet

Recommendation was to defer and feel out real disk usage first. That's consistent with `AGENTS.md`, which describes the worktree mechanism itself but deliberately says nothing about a cleanup policy, and with the roadmap, which already carries this as its own dedicated Phase 7 sub-issue (#29). Vibe Kanban's model (orphan-sweep-on-startup + time-based expiry for archived tasks, see above) is parked here as a reference for whoever picks up #29 — not adopted now.

## Environment checklist — must be verified on your actual dev machine

Vastask is a single local process by design (no cloud component). This session runs in an isolated cloud container that is **not** that machine — it has neither `gh` nor `ANTHROPIC_API_KEY`, which is expected and unrelated to whether your real dev machine is ready. These three checks need to happen in a shell on the machine you'll actually run Vastask on:

- [ ] `gh auth status` succeeds, and `gh issue list` works against the target repo.
- [ ] `ANTHROPIC_API_KEY` is set in that shell's environment, on a dedicated key (not shared with other tools) so per-run cost tracking starts clean — no `--bare`.
- [ ] `claude -p "hello" --output-format stream-json` runs successfully, standalone, before Phase 1 tries to combine it with `gh`.

## Spike target repo — decided: `shazzar00ni/vast-task` (this repo)

Phase 1 (#2) will dispatch its go/no-go spike against this repo itself, rather than a separate throwaway sandbox. Real enough to be a meaningful test (the `gh` + `claude -p` combination has actual issues to read and a real worktree to branch from), low-stakes enough that a spike-generated branch/PR here isn't consequential.
