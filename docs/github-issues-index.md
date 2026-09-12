# GitHub Issues Index

Last updated: 10 Sep 2026

This indexes the issues created from the roadmap docs (`vastask-roadmap.md`, `vastask-prd-v2.md`, `vastask-v0-scope-and-buildplan.md`, `vastask-landing-page-planning.md`). It exists for the same reason the docs index asks for a decision log: reasoning and structure scattered across chat history is exactly the cold-start cost this project is meant to solve.

## A note on "milestones"

The GitHub tooling available for this repo has no way to create or list native GitHub Milestones (no API endpoint is exposed for it). The substitute used instead: **one tracking issue per roadmap phase**, with the phase's distinct pieces of work attached as native GitHub **sub-issues**. GitHub renders a progress bar and completion count on each phase issue, similar to a milestone — arguably more visible day-to-day than the classic Milestones tab, and usable right now. It doesn't give the dedicated Issues → Milestones filter/grouping view, though — see below if you want that too.

If you'd still like literal GitHub Milestones as well (e.g. for the Issues → Milestones filter view), create them in the GitHub UI (Issues → Milestones → New milestone) and let me know the numbers — issues can be assigned to them retroactively in one pass.

Every issue also carries an `app` or `landing-page` label, plus a GitHub Issue Type (`Task` / `Feature`) and, where relevant, `go-no-go-gate` or `decision-gate`.

## App track (`vastask` — the tool itself)

Phases 0–7 are the committed build. Phase 1 is a hard go/no-go gate — nothing after it should start until it passes. Phases 8–9 are not code; they're a 30-day usage period and a decision gate. Phase 10 (business exploration) is **deliberately excluded** — the roadmap is explicit that scoping it now would repeat the exact mistake the competitive-landscape doc warned against.

| Phase | Issue | Sub-issues |
|---|---|---|
| 0 — Prep | [#1](../../../issues/1) | — |
| 1 — The Spike (GO/NO-GO gate) | [#2](../../../issues/2) | — (deliberately one script, not split) |
| 2 — Persistence | [#3](../../../issues/3) | [#15](../../../issues/15) schema, [#16](../../../issues/16) run lifecycle + CLI + events |
| 3 — GitHub Sync | [#4](../../../issues/4) | — |
| 4 — Read-Only Board | [#5](../../../issues/5) | [#17](../../../issues/17) scaffold + API, [#18](../../../issues/18) board/columns/cards, [#19](../../../issues/19) detail view |
| 5 — Dispatch + Live Log | [#6](../../../issues/6) | [#20](../../../issues/20) dispatch modal + run creation, [#21](../../../issues/21) SSE endpoint, [#22](../../../issues/22) frontend log rendering, [#23](../../../issues/23) reopen-mid-run replay |
| 6 — Close the Loop | [#7](../../../issues/7) | [#24](../../../issues/24) PR detection, [#25](../../../issues/25) approve/merge/reject |
| 7 — Sharp Edges | [#8](../../../issues/8) | [#26](../../../issues/26) cancel, [#27](../../../issues/27) failure categorisation, [#28](../../../issues/28) cost + timeout, [#29](../../../issues/29) worktree cleanup policy, [#30](../../../issues/30) drag-and-drop (optional) |
| 8–9 — 30-Day Usage & Revisit | [#9](../../../issues/9) | [#31](../../../issues/31) usage period, [#32](../../../issues/32) revisit decision gate |

## Landing page track

Gated as a whole on the app's Phase 1 passing (see `vastask-landing-page-planning.md` §0) — except Phase 0 (positioning), which is deliberately done in parallel, any time, since it's writing not building.

| Phase | Issue | Sub-issues |
|---|---|---|
| 0 — Positioning | [#10](../../../issues/10) | — (do now, no dependency) |
| 1 — Static Page | [#11](../../../issues/11) | [#33](../../../issues/33) content/copy/layout, [#34](../../../issues/34) design system |
| 2 — Email Capture | [#12](../../../issues/12) | — |
| 3 — Hero Recording | [#13](../../../issues/13) | — (gated on app Phase 5 dispatch working) |
| 4 — Send It | [#14](../../../issues/14) | — |

Landing page "Phase 5 — Revisit" has no issue of its own; its inputs fold into the app's Phase 9 revisit gate (#9 / #32) rather than being tracked twice.

## Dependency summary

- Everything in the app track past Phase 1, and the entire landing page build (Phases 1–4), wait on **#2 (the spike)** passing its go/no-go check.
- Landing Page Phase 3 (hero recording, #13) additionally needs a real dispatch to record — realistically app Phase 5 (#6), not just Phase 1.
- App Phase 9 (#32) is the single re-entry point for the business question — see the competitive-landscape doc's Option D. Nothing past it is scoped on purpose.
