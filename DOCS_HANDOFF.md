# DOCS_HANDOFF

**Date:** 2026-06-25 · **Author:** docs-writer

## What's documented and where

- **`README.md`** (repo root) — the front door for this repository. Describes what the repo actually is (a multi-agent team framework, not an application), the 8-agent roster table, the skills layer (10 skills, including the 2 shared ones), key conventions from `TEAM.md`, the typical cross-lane build sequence, the repo layout tree, the role of `DESIGN.md`, and how to start a session (`claude --agent architect`).
- Source files read for ground truth (all under the repo root): `.claude/TEAM.md`, all 8 files in `.claude/agents/`, all 10 `SKILL.md` files under `.claude/skills/`, `DESIGN.md`, `PLAN.md`. No application code exists in this repo, so none was read or referenced.

## Intentionally pointed at other sources rather than copied

- **API reference** — not applicable. There is no FastAPI service (or any app) in this repository, so there is no `/docs` or `/redoc` to link to. The README does not fabricate one.
- **Full agent role text** — the README summarizes the roster table from `TEAM.md` plus tool/escalation notes from the agent files, but links to `.claude/agents/*.md` rather than reproducing each agent's full body, since that's the single source of truth and will drift if copied.
- **Full skill content** — the README lists each skill's name, owning agent, and a one-line summary of what it covers, but links to `.claude/skills/*/SKILL.md` for the actual patterns/references rather than transcribing them.
- **`DESIGN.md` contents** — README describes its *role* (binding for `frontend-developer` UI work, not currently consumed by anything in this repo) without restating its tokens/components; links to the file itself.

## Found stale / missing — for the owning agent

- **No `CONTRACT.md` in this repo.** Per `PLAN.md`, this is expected and correct: there is no cross-layer API/data contract to define because there's no application code. Flagging only so a future reader doesn't read its absence as an oversight.
- **Orchestration is still "TBD" in `TEAM.md`** ("decide: main session routes (simple) vs. a lead `architect` agent that delegates. Revisit after 2–3 specialists exist.") — but the `architect.md` agent file and the roster row both already describe the architect as the lead, and the build order shows it as the last agent added. The README documents the *current, built* behavior (architect as main-thread lead via `claude --agent architect`), consistent with `architect.md`'s own description. If `TEAM.md`'s "Orchestration: TBD" line is meant to still be open, `architect`'s owning agent (or the user) should resolve that contradiction in `TEAM.md` — it currently reads as resolved in `architect.md` but unresolved in `TEAM.md`.
- **`code-reviewer` handoff is inline-only.** `TEAM.md` footnote suggests adding a `Write` tool scoped to `REVIEW.md` if persistence is wanted; no such file exists today, which is correct per the current tool grant in `.claude/agents/code-reviewer.md` (read-only, no Write tool). No action needed unless that tool grant changes.
- **`DESIGN.md` has a minor encoding artifact**: literal `[cite: 1]` markers and a few `�` mojibake characters (e.g. "upper third of nearly every marketing page" footnotes, and dashes rendered as `�`) throughout the file, suggesting it was pasted from another source without full encoding/citation cleanup. Not something docs-writer should silently rewrite since it's a generated spec owned outside this task's scope — flagging for whoever owns `DESIGN.md` to clean up if it's going to be used for real frontend work.

## Not done (out of scope for this task)

- No ADRs were written (`/docs/adr`) — no architectural decision was requested or identified for this docs-only task.
- No runbooks or end-user guides were written — there is no deployed system or end user to write them for yet.
