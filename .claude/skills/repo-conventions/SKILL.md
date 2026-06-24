---
name: repo-conventions
description: Expert discipline for fitting into an existing codebase — detecting and matching its style, layout, naming, and tooling rather than imposing your own. Use this skill whenever you are about to write or change code in a repo you didn't author: before adding a file, an endpoint, a component, or a test, and any time you're tempted to "clean up," reformat, restructure, or pull in a new dependency. It is the shared baseline that loads into every builder agent (backend, frontend, integration, qa, devops) so changes read as if the original authors wrote them. Do NOT use it to decide product behavior, the API contract, or architecture — those belong to the specialist skills; this skill only governs how new work blends into what already exists.
---

# Repo Conventions

You are a **guest in an existing codebase**. The people who built it made hundreds of small
decisions — naming, layout, async vs sync, how errors are raised, how tests are arranged — and a
reader should not be able to tell which lines you added by style alone. Local consistency beats
your personal taste every time. A "better" pattern dropped into a repo that uses a different one
is not better; it's a second dialect a maintainer now has to hold in their head.

This is the team's shared baseline. Every specialist skill assumes it. When a specialist says
"match the repo before imposing anything," this is the *how*.

## How this skill works

1. **Survey before you write.** Read the neighbours of the file you're about to touch — the same
   directory, a sibling endpoint, a peer component. Learn the local idiom first.
2. **Mirror the nearest neighbour.** Copy the structure, naming, and imports of the closest
   existing example, then change only what your task requires.
3. **Respect the tooling.** Let the repo's formatter, linter, and config decide style — don't
   hand-impose yours or fight theirs.
4. **Leave it as you'd want to find it.** Your diff should be the smallest change that does the
   job, touching only lines your task owns.

## Detect before you write

Before the first line, establish the local answers to:

- **Language & framework versions** — read `package.json` / `pyproject.toml` / `go.mod` etc.
  Don't use syntax or APIs the pinned version doesn't have.
- **Async vs sync** — is the code `async def` + `AsyncSession`, or sync `Session`? `async/await`
  or callbacks/promises? Follow it; never convert one to the other unprompted.
- **Project layout** — where do routers, services, components, hooks, tests actually live? Put
  new files where their kind already lives, not where you'd file them.
- **Naming** — casing for files, functions, variables, types; singular vs plural directories;
  prefixes/suffixes (`...Service`, `use...`, `..._test.py`). Match the dominant pattern exactly.
- **Import style** — absolute vs relative, path aliases, ordering, default vs named exports.
- **Error-handling idiom** — how the repo raises, wraps, and logs errors; the existing exception
  types or error envelopes. Reuse them rather than introducing a new scheme.
- **Test layout** — framework, file location (co-located vs `tests/`), fixture and naming
  conventions. New tests look like existing tests.

## Tooling is law

The repo's configured tools are the style authority, not you.

- Honor `.editorconfig`, Prettier/ESLint, Ruff/Black/isort, `.nvmrc`/`.tool-versions` and the
  like. If a formatter is configured, format with it and accept its output.
- **Never reformat unrelated code.** Reflowing, re-sorting imports, or re-indenting lines your
  task didn't touch buries the real change in noise and corrupts `git blame`. Keep whitespace and
  formatting churn out of your diff.
- Respect lockfiles. Don't regenerate or bump them as a side effect.

## No drive-by refactors, no unprompted restructuring

- **Don't refactor what you weren't asked to.** A rename, a "tidy," or a structural improvement is
  a separate change — propose it, don't smuggle it into an unrelated diff.
- **Don't convert styles** (sync↔async, class↔hook, callback↔promise, ORM query strategy) just
  because you'd write it differently.
- **Prefer what's already imported.** Reach for the libraries and helpers the repo already uses
  before adding a dependency. A new dependency is a decision with a maintenance and security cost —
  justify it, and surface it rather than slipping it into the lockfile.

## When conventions conflict or are absent

- **Conflict:** follow the *dominant local* pattern — the one most files in the relevant area use
  — and match the file you're editing. Consistency within a module outranks a repo-wide ideal.
- **Absent:** if the repo gives no precedent for a genuinely new decision, pick a clean, common
  default *and say so*, so a maintainer can redirect — don't invent a convention silently and
  spread it.

## Quality floor (non-negotiable, never announced)

New code uses the repo's versions, layout, naming, imports, and error idioms; passes the
configured formatter/linter with no new warnings; adds no unrequested dependency, reformatting, or
refactor; and keeps the diff scoped to the task. You don't announce that you matched the repo — a
clean blend is the expectation, not an achievement.

## Self-critique before delivering

- **Blend pass:** drop your diff next to a peer file — could a reviewer tell which lines are new by
  style alone? If a naming, layout, or idiom mismatch gives you away, fix it.
- **Scope pass:** is every changed line required by the task? Revert incidental reformatting,
  re-sorted imports, and opportunistic refactors. If you added a dependency or a new pattern, did
  you surface it instead of burying it?
