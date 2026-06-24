# Agent Team — Source of Truth

**Architecture:** thin agents + fat skills · contract-first · named handoff artifacts.
Each agent is a role + lane + handoff; all domain knowledge lives in the skill it loads.

## Roster

| Agent | Owns (lane) | Does NOT touch | Loads (skill) | Handoff artifact | Model | Status |
|-------|-------------|----------------|---------------|------------------|-------|--------|
| `architect` *(lead)* | The plan + the cross-layer API/data **contract** | Feature code | `system-design` | `PLAN.md`, `CONTRACT.md` | opus | ⬜ planned |
| `backend-developer` | FastAPI endpoints, SQLAlchemy models, Pydantic schemas, business logic | UI, external adapters, infra | `fastapi-backend` | `BACKEND_HANDOFF.md` | sonnet | ✅ built |
| `frontend-developer` | React + Tailwind UI, client `api/` layer | Backend, DB, infra | `frontend-react-engineer` | `FRONTEND_HANDOFF.md` | sonnet | ✅ built |
| `integration-specialist` | UiPath bots, SMTP, SharePoint, webhooks — adapters to external systems | Core app logic, UI | `rpa-integration` | `INTEGRATION_HANDOFF.md` | sonnet | ✅ built |
| `qa-tester` | pytest, contract + integration tests, coverage | Feature implementation | `testing-standards` | `TEST_REPORT.md` | sonnet | ✅ built |
| `code-reviewer` | Read-only review of diffs (quality, security) | Any edits (read-only tools) | `review-standards` | inline verdict* | opus | ✅ built |
| `devops` | Dockerfile, compose, GitHub Actions CI, env/secrets, deploy step | App source, contract, tests, docs | `devops` | `DEVOPS_HANDOFF.md` | sonnet | ✅ built |
| `docs-writer` | README, architecture/ADRs, runbooks, user guides | Code, behavior, tests, infra | `documentation` | `DOCS_HANDOFF.md` | sonnet | ✅ built |

## Conventions

| Convention | Rule |
|------------|------|
| Shared skills | `repo-conventions` + `contract-first` load into **every builder agent** — never copied per-agent |
| Handoff naming | `<ROLE>_HANDOFF.md` at repo root |
| The contract border | `backend` *serves* the contract; `frontend` + `integration` *consume* it. Only `architect` (or `backend`) may **change** it; everyone else reads it. Prevents three-way drift |
| Per-project design | `design.md` at project repo root → `frontend-developer` treats it as binding |
| Thin-agent rule | Zero domain knowledge in agent bodies; all of it in skills |

## Build order

1. ✅ `frontend-developer` (+ `frontend-react-engineer` skill)
2. ✅ `backend-developer` (+ `fastapi-backend` skill) — the contract source the others lean on
3. ✅ `integration-specialist` — the RPA/external glue
4. ✅ `qa-tester` and `code-reviewer` — quality gates
5. ✅ `devops` and `docs-writer` — ship + document
6. `architect` — added once there are enough specialists to coordinate

*\* `code-reviewer` is read-only and outputs its verdict inline. Add a `Write` tool scoped to `REVIEW.md` only if you want the verdict persisted for the handoff chain.*

## Orchestration

**TBD** — decide: main session routes (simple) **vs.** a lead `architect` agent that delegates. Revisit after 2–3 specialists exist.

## Install layout (reminder)

```
~/.claude/  (or project-local .claude/)
├── agents/   → <name>.md per agent
└── skills/   → <skill-name>/SKILL.md (+ references/)
```
Agent and the skill it loads must be at the **same scope**.
