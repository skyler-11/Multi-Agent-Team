# Multi-Agent Team

A **multi-agent engineering team framework for Claude Code**. It is not an application — there is no API, no database, and no UI in this repository. Instead, it defines a reusable team of Claude Code sub-agents (an architect plus seven specialists) and the "fat skill" knowledge packs they load, so that a single `claude --agent architect` session can plan, delegate, build, test, review, ship, and document a *real* software project on your behalf.

## Used in practice

I use this team for my day-to-day development at work. It drove two production modules on an internal manufacturing platform:

- **Schedule compliance tool** — migrated from a Streamlit prototype to a FastAPI backend and React/Tailwind frontend with Keycloak role-based access. It validates schedules for ~650–700 employees against Philippine (DOLE) labor rules and cut each weekly check from ~2 hours to under 15 minutes.
- **Scrap declaration module** — built on the same platform, reusing the shared auth, RBAC, and API layer instead of standing up a separate service.

Both went through UAT and change-control approval and are live.

**Why it's built this way:** I started with a single do-it-all agent. On real builds it drifted from what I asked for: it wrote its own to-do list and bypassed rules I had set. So I split the work into specialists that each own one lane. The backend agent only writes FastAPI, and the frontend agent only writes React. They never talk to each other directly, so one lane's mistakes can't leak into the other. Everything goes through the architect, which assembles the parts and shows me the result.

**My part:** the design and approach come from me; the agents implement them. I verify the code, steer the agents back when the output drifts from my design, test what's built, and feed the results back to the architect for the next round.

## How it fits together

The team follows three rules, defined in [`.claude/TEAM.md`](.claude/TEAM.md) (the source of truth for the roster, conventions, and build order):

- **Thin agents + fat skills** — each agent file in `.claude/agents/` is just a role, a lane, and a handoff contract. All domain expertise (how to design a FastAPI router, how to structure a React data layer, how to wire UiPath callbacks, etc.) lives in the skill it loads from `.claude/skills/`, not in the agent definition itself.
- **Contract-first** — the backend *serves* the cross-layer API/data contract; the frontend and integration agents *consume* it. Only the architect or the backend agent may *change* it. Everyone else only reads it. This is what stops the three sides of a build from drifting apart.
- **Named handoff artifacts** — every agent reports its work in a predictably-named Markdown file at the repo root (`<ROLE>_HANDOFF.md`), so the next agent in the chain — or a human — can pick up the state of the build without re-reading code.

## The roster

| Agent | Owns (lane) | Does NOT touch | Loads (skill) | Handoff artifact |
|---|---|---|---|---|
| `architect` *(lead)* | The plan + the cross-layer API/data **contract** | Feature code | `system-design` | `PLAN.md`, `CONTRACT.md` |
| `backend-developer` | FastAPI endpoints, SQLAlchemy models, Pydantic schemas, business logic | UI, external adapters, infra | `fastapi-backend` | `BACKEND_HANDOFF.md` |
| `frontend-developer` | React + Tailwind UI, client `api/` layer | Backend, DB, infra | `frontend-react-engineer` | `FRONTEND_HANDOFF.md` |
| `integration-specialist` | UiPath bots, SMTP, SharePoint, webhooks — adapters to external systems | Core app logic, UI | `rpa-integration` | `INTEGRATION_HANDOFF.md` |
| `qa-tester` | pytest, contract + integration tests, coverage | Feature implementation | `testing-standards` | `TEST_REPORT.md` |
| `code-reviewer` | Read-only review of diffs (quality, security) | Any edits (read-only tools) | `review-standards` | inline verdict |
| `devops` | Dockerfile, compose, GitHub Actions CI, env/secrets, deploy step | App source, contract, tests, docs | `devops` | `DEVOPS_HANDOFF.md` |
| `docs-writer` | README, architecture/ADRs, runbooks, user guides | Code, behavior, tests, infra | `documentation` | `DOCS_HANDOFF.md` |

The architect runs as the **main-thread lead** and is the only agent that can spawn the other seven (via a scoped `Agent` tool); a delegated specialist cannot spawn anyone else, and cannot ask the user a question mid-task — so the architect resolves ambiguity up front, before delegating. Full role definitions, tool grants, and escalation rules live in [`.claude/agents/`](.claude/agents/) — one file per agent.

## The skills layer

A "fat skill" is where an agent's actual expertise lives: the patterns, references, and quality bar for its lane, loaded fresh into context rather than baked into the agent's own prompt. This keeps the agent definitions thin and lets a skill be improved once and picked up by every agent that loads it. Each skill is a `SKILL.md` plus an optional `references/` folder of deeper material pulled in only when needed.

Skills present in [`.claude/skills/`](.claude/skills/):

| Skill | Used by | Covers |
|---|---|---|
| `system-design` | `architect` | Clarify-first decomposition, contract-first design, lane→agent routing, sequencing/parallelization, the verification loop |
| `fastapi-backend` | `backend-developer` | Layered FastAPI architecture, Pydantic v2 contracts, database-agnostic persistence, pluggable auth |
| `frontend-react-engineer` | `frontend-developer` | React + Tailwind architecture, Repository/Container-Presentational patterns, contract-first backend integration |
| `rpa-integration` | `integration-specialist` | Anti-corruption adapters for UiPath Orchestrator, SMTP, SharePoint (Graph), inbound webhooks; reliability patterns |
| `testing-standards` | `qa-tester` | Test pyramid, behavior-over-internals testing, in-memory SQLite + transactional rollback fixtures, contract and resilience tests |
| `review-standards` | `code-reviewer` | Review process, severity ladder, team seam-rule checks, security checklist, resilience lens, verdict format |
| `devops` | `devops` | Multi-stage Docker image, docker-compose, GitHub Actions CI, 12-factor config/secrets, pluggable deploy step |
| `documentation` | `docs-writer` | Doc types (README/ADR/runbook/guide), ground-truth writing, leaning on auto-generated API docs |
| `repo-conventions` | every builder agent | Detecting and matching an existing repo's style/layout/naming instead of imposing a new one |
| `contract-first` | every builder agent | The shared discipline behind the contract border — serve vs. consume, additive vs. breaking, recording contract changes |

`repo-conventions` and `contract-first` are **shared skills**: per `TEAM.md`, they load into every builder agent rather than being copied into each one individually.

## Key conventions

From the [Conventions table in `.claude/TEAM.md`](.claude/TEAM.md):

| Convention | Rule |
|---|---|
| Shared skills | `repo-conventions` + `contract-first` load into every builder agent — never copied per-agent |
| Handoff naming | `<ROLE>_HANDOFF.md` at repo root |
| The contract border | `backend` *serves* the contract; `frontend` + `integration` *consume* it. Only `architect` (or `backend`) may **change** it; everyone else reads it. Prevents three-way drift |
| Per-project design | `design.md` at the project repo root → `frontend-developer` treats it as binding |
| Thin-agent rule | Zero domain knowledge in agent bodies; all of it lives in skills |

## Typical build sequence

For a feature or build that spans more than one lane, the architect runs roughly this sequence (see the `system-design` skill for the full decomposition and verification loop):

1. **Contract** — architect clarifies scope with the user, then writes `CONTRACT.md` (the API/data shapes everyone builds to) and `PLAN.md` (tasks, owners, order).
2. **Backend** — `backend-developer` serves the contract: endpoints, schemas, models, persistence.
3. **Frontend ‖ Integration** — `frontend-developer` and `integration-specialist` consume the contract in parallel: UI wired to the API on one side, external-system adapters (UiPath/SMTP/SharePoint/webhooks) on the other.
4. **QA** — `qa-tester` writes/runs tests against what was built and reports `TEST_REPORT.md`.
5. **Review** — `code-reviewer` inspects the diff against team seam rules, security, and resilience, and gives a verdict (read-only).
6. **Ship** — `devops` containerizes, wires CI, and handles the deploy step.
7. **Docs** — `docs-writer` documents what was actually built, grounded in the code, the contract, and every `*_HANDOFF.md`.

## Repo layout

```
.
├── .claude/
│   ├── TEAM.md                  # Source of truth: roster, conventions, build order
│   ├── agents/                  # One role definition per agent (thin)
│   │   ├── architect.md
│   │   ├── backend-developer.md
│   │   ├── frontend-developer.md
│   │   ├── integration-specialist.md
│   │   ├── qa-tester.md
│   │   ├── code-reviewer.md
│   │   ├── devops.md
│   │   └── docs-writer.md
│   └── skills/                  # One "fat skill" knowledge pack per lane
│       ├── system-design/SKILL.md (+ references/)
│       ├── fastapi-backend/SKILL.md (+ references/)
│       ├── frontend-react-engineer/SKILL.md (+ references/)
│       ├── rpa-integration/SKILL.md (+ references/)
│       ├── testing-standards/SKILL.md (+ references/)
│       ├── review-standards/SKILL.md (+ references/)
│       ├── devops/SKILL.md (+ references/)
│       ├── documentation/SKILL.md (+ references/)
│       ├── repo-conventions/SKILL.md
│       └── contract-first/SKILL.md
├── DESIGN.md                    # Design-system spec (see below)
└── README.md                    # This file
```

An agent and the skill it loads must live at the **same scope** — both project-local under `.claude/` (as here) or both under the user's `~/.claude/`.

When a real project is built with this team, the build produces these files at the project root: `CONTRACT.md` and `PLAN.md` (architect), plus the `*_HANDOFF.md` files for whichever specialists ran (`BACKEND_HANDOFF.md`, `FRONTEND_HANDOFF.md`, `INTEGRATION_HANDOFF.md`, `TEST_REPORT.md`, `DEVOPS_HANDOFF.md`, `DOCS_HANDOFF.md`). They aren't included here, because this repo is the team definition itself, not a built project.

## Design system

[`DESIGN.md`](DESIGN.md) is an included Stripe-style design-system spec — color tokens, typography scale, spacing/elevation/shape tokens, and component specs (buttons, cards, inputs, nav, pills). It is not consumed by anything in this repository, since there's no UI here. On a real project, if a `design.md` (or equivalent) exists at the project root, `frontend-developer` treats it as **binding** for visual decisions, overriding the `frontend-react-engineer` skill's own defaults wherever the two differ.

<!-- TODO: if DESIGN.md is adapted from a public source, credit it here, e.g. "Adapted from [source](link)." -->

## How to use it

This team is meant to be run, not read end-to-end. Start a session with the architect as the main-thread lead so it can decompose your request and delegate to the specialists above:

```bash
claude --agent architect
```

Then describe the feature, refactor, or build you want. The architect will:

1. Clarify scope and any open questions with you directly (specialists run in isolation and can't ask follow-ups).
2. Write `CONTRACT.md` (the shapes the build will use) and `PLAN.md` (tasks, owners, sequencing).
3. Delegate each task to the named specialist above, in the order the plan calls for, passing along the contract and whatever context that agent needs.
4. Route the result through `qa-tester` and `code-reviewer` as quality gates, then `devops` to ship and `docs-writer` to document.

You can also invoke any specialist directly (e.g. for a single, well-scoped change that doesn't need cross-lane coordination) — each agent file in `.claude/agents/` documents when it's appropriate to use that agent standalone.

## Further reading

- [`.claude/TEAM.md`](.claude/TEAM.md) — roster, conventions, build order (primary reference for this repo)
- [`.claude/agents/`](.claude/agents/) — full role definitions for all eight agents
- [`.claude/skills/`](.claude/skills/) — the knowledge each agent loads
- [`DESIGN.md`](DESIGN.md) — the included design-system spec
