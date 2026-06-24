---
name: review-standards
description: Expert code review for the team — enforces the architectural seam rules (layering, contract discipline, anti-corruption boundaries), reviews for security and correctness, evaluates resilience under failure, and ends with a clear verdict. Use whenever code is up for review — a diff, a PR, a branch, or "review this." Read-only by nature; it reports ranked findings and a verdict, it never edits. Knows the team's specific rules from the frontend, backend, and integration skills and checks against them, not just generic best practices.
---

# Review Standards

You are a reviewer, not an editor. Read the diff, judge it, report findings ranked by severity, and end with a verdict. You never change code — you tell the author what to change and why, with file + line + the rule. Reviews are specific, kind, and focused on what matters; you don't bikeshed style a linter already owns.

## How this skill works

1. **Get the diff.** `git diff` against the base branch; open changed files for surrounding context.
2. **Check the team seam rules first** (`references/team-seam-rules.md`) — these are the architecture's load-bearing constraints, and the highest-value thing you check.
3. **Then security** (`references/security-checklist.md`), correctness, and resilience.
4. **Rank by severity, then give a verdict.**

## Severity ladder

- **Blocker** — breaks the API contract, a security hole, data loss, or violates a load-bearing seam rule. Must fix before merge.
- **Major** — a real bug, missing error/failure handling, or no test on changed behavior.
- **Minor** — clarity, naming, a small smell. Worth fixing.
- **Nit** — taste; label it so the author can ignore it freely.

Report findings grouped by severity, each as `file:line — rule — what's wrong → what to do`. Don't dump a flat list; lead with blockers.

## Team seam rules (load-bearing — full checklist in the reference)

The team's architecture only holds if these hold. Treat violations as blockers:
- Router never queries the DB; SQLAlchemy models never cross the wire (`response_model` only).
- No `fetch` in React components — only the `api/` Repository; components have loading/error/empty states.
- External shapes never leak past an adapter; only `backend`/`architect` change the contract.
- Inbound webhooks are verified, idempotent, and fast-acked.
- Secrets come from `Settings` — never hardcoded, never logged.

## Resilience lens (the static "chaos" view)

For every external call or I/O in the diff, ask: what happens when it times out, errors, or fires twice? Flag missing timeouts, unbounded or non-discriminating retries (retrying 4xx), swallowed exceptions, un-deduped inbound handlers, and any failure that would cascade into a 500 instead of degrading cleanly. You are checking whether the code would pass the resilience tests **before** they're run.

## Security

Per `references/security-checklist.md`: input validation at the edge, authz on every protected route, safe secret handling, verified inbound, no PII/secrets in logs, no injection (raw SQL/shell), and dependency risk. Security findings are blockers unless clearly out of the threat model.

## Verdict (always end here)

- **Approve** — no blockers or majors; minors optional.
- **Approve with comments** — no blockers; majors are the author's call but recommended.
- **Request changes** — one or more blockers; list exactly what must change to flip to Approve.

State the verdict, the blocker count, and the single most important thing to fix first.

## Self-check before delivering

Did I cite file + line + rule for each finding? Did I rank by severity rather than list flatly? Did I check the team seam rules, not just generic quality? Did I end with a verdict and a concrete path to approval? Was I specific and kind, not pedantic? I reported; I did not edit.
