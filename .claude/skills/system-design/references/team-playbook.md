# Team Playbook — Routing, Sequencing, Handoffs

How to map a request onto the team and run it. This is the architect's routing logic.

## Lane → agent map (who gets which task)

| If the task is about… | Route to | It owns | It produces |
|-----------------------|----------|---------|-------------|
| The plan + cross-layer contract | *you (architect)* | `PLAN.md`, `CONTRACT.md` | the contract others build to |
| API endpoints, schemas, DB, business logic | `backend-developer` | server side; serves the contract | `BACKEND_HANDOFF.md` |
| React + Tailwind UI, client data layer | `frontend-developer` | client side; consumes the contract | `FRONTEND_HANDOFF.md` |
| UiPath, SMTP, SharePoint, webhooks | `integration-specialist` | external adapters | `INTEGRATION_HANDOFF.md` |
| Tests of any kind | `qa-tester` | `tests/` | `TEST_REPORT.md` |
| Reviewing a diff before merge | `code-reviewer` | read-only review | inline verdict |
| Docker, CI, shipping | `devops` | build/CI/infra | `DEVOPS_HANDOFF.md` |
| READMEs, ADRs, runbooks, guides | `docs-writer` | docs | `DOCS_HANDOFF.md` |

If a task seems to span two lanes, it's not yet decomposed — split it at the contract line (e.g. "inbound webhook" = integration owns the handler + contract, backend mounts the route).

## Typical feature sequence

```
1. CONTRACT      you define CONTRACT.md (with backend-developer's input if needed)
2. BACKEND       backend-developer implements endpoints + schemas, serves /openapi.json
   ├─ 3a. FRONTEND      frontend-developer builds UI against the contract  ┐ parallel
   └─ 3b. INTEGRATION   integration-specialist wires external adapters     ┘ (independent)
4. QA            qa-tester writes/runs tests, incl. resilience tests
5. REVIEW        code-reviewer gates the diff (blockers must clear)
6. SHIP          devops containerizes + CI + deploy
7. DOCS          docs-writer documents (can start once contract is stable)
```

## What parallelizes vs. what blocks

- **Blocks on the contract:** backend, frontend, integration all need `CONTRACT.md` first. Don't release any build task until it exists.
- **Parallel once the contract exists:** `frontend` ∥ `integration` (they depend on the contract, not on each other). `docs-writer` can begin on architecture/contract docs in parallel too.
- **Strictly sequential:** review after the code it reviews; ship after review passes; the contract before everything.
- Frontend depends on the *contract*, not on backend's internal completion — if the contract is frozen, frontend can build against a mock while backend implements (the frontend skill's contract-first stance makes this safe).

## Handoff chain (who reads whose artifact)

- `frontend` + `integration` read `BACKEND_HANDOFF.md` (and the contract) to know the shapes.
- `backend` reads `INTEGRATION_HANDOFF.md` to mount any inbound routes integration defined.
- `qa-tester` reads all `*_HANDOFF.md` to know what changed and therefore what to test.
- `code-reviewer` reads the relevant handoff to verify the diff matches its claim (especially breaking contract changes).
- `devops` reads `BACKEND_HANDOFF.md` for env vars / health needs.
- `docs-writer` reads everything + the contract for ground truth.
- *You* keep `PLAN.md` current so the chain stays coherent.

## Clarify-up-front checklist (before any delegation)

Because delegated specialists can't ask the user, resolve these yourself first:

- **Scope** — what's in and explicitly out of this change.
- **Contract** — the exact shapes/endpoints affected; is any change breaking (needs versioning)?
- **Data model** — new/changed entities, constraints.
- **External systems** — which ones, in which direction, with what reliability needs.
- **Acceptance** — what "done" means, so `qa-tester` has something to assert.
- **Non-functionals** — auth, performance, deploy target, if relevant.

Any unknown here that you delegate becomes a wrong assumption you'll pay for two steps later. If you can't fill these in, go back to the user before spawning anyone.
