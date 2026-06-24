# Doc Types — Structure & Templates

One template per reader. Adapt, don't pad — cut any section that doesn't serve this reader's task.

## README (front door — optimize time-to-first-success)

```markdown
# Project Name
One-to-two sentence description: what it does and who it's for.

## Quickstart
\```bash
git clone … && cd project
cp .env.example .env        # fill in values
docker compose up           # or: pip install -r requirements.txt && uvicorn app.main:app
\```
Then open http://localhost:8000/docs

## Prerequisites
- Python 3.12 / Docker / …

## Configuration
Link to .env.example; describe required vars in a short table.

## Running tests
\```bash
pytest -q
\```

## Architecture / Docs
Link to /docs and docs/architecture.md — don't duplicate them here.
```

Rules: the quickstart must actually work start-to-finish on a clean machine. Lead with running it, not with a feature tour.

## ADR — Architecture Decision Record (`docs/adr/NNNN-title.md`)

```markdown
# ADR 0007: Use the Repository pattern for data access

- **Status:** Accepted
- **Date:** 2025-06-17
- **Deciders:** (who)

## Context
The forces at play — what problem, what constraints, what we knew at the time.

## Decision
The choice, stated plainly.

## Consequences
What gets easier, what gets harder, what we now can't do. Trade-offs, honestly.

## Alternatives considered
What else was on the table and why it lost.
```

Rules: one decision per ADR. ADRs are **immutable and append-only** — when a decision changes, write a new ADR that supersedes the old one (mark the old `Superseded by ADR-00NN`); never rewrite history. The *why* is the whole point.

## Runbook (`docs/runbooks/<task>.md` — for a tired operator)

```markdown
# Runbook: Rotate the SMTP credential

**When:** credential leaked or on the rotation schedule.
**Impact:** none if done in order; email pauses ~1 min.

## Steps
1. Generate the new credential in <system>.
2. Update the secret: `…exact command…`
3. Roll the service: `…exact command…`
4. Verify: `…exact check…` → expect `…`.

## Rollback
…exact steps to revert…

## If it goes wrong
- Symptom → cause → fix.
```

Rules: literal, copy-pasteable commands; state expected output so the operator knows it worked; always include verify + rollback. No prose they have to interpret under pressure.

## End-user guide (in the user's language)

```markdown
# How to submit a shopfloor request

1. Open … and click **New Request**.
2. Fill in Item and Quantity.
3. Click **Submit** — you'll see a confirmation and get an email.

## Checking status
…

## Troubleshooting
- "I didn't get the email" → check spam; the request still saved.
```

Rules: task-oriented headings ("How to…"), the user's vocabulary (no "endpoint", "payload", "service"), steps tied to what they see on screen, and a short troubleshooting list for the common confusions. No implementation detail.

## Cross-type rules

- Every runnable snippet is tested before publishing.
- Link to the single source of truth (API → `/docs`; config → `.env.example`) instead of copying.
- Date anything decision- or time-sensitive.
- Keep each doc focused on one reader; if it's serving two, split it.
