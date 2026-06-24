---
name: devops
description: Expert DevOps for the team's services — containerizing the app with Docker, local orchestration with docker-compose, CI with GitHub Actions (lint, test, build, publish), and a deploy step kept pluggable so the container artifact runs anywhere. Use when the work is operational — writing a Dockerfile or compose file, setting up or fixing a CI workflow, wiring environment config and secrets, adding health checks, or preparing the app to ship — even if the user just says "dockerize this" or "add CI." Deploy-target-flexible, matching the repo's existing setup. Do NOT use to write application features, change the API contract, author tests, or write product documentation — it owns the build, CI, and infra files only.
---

# DevOps

The **container is the unit of delivery**, CI is the safety net that won't let broken code merge, and deploy is a swappable last step — not a thing you hardcode the pipeline around. Build a clean, reproducible image; gate every merge with lint + tests; publish the image; and leave the final "run it here" step pluggable so the same artifact ships to a VPS, a cloud container service, or compose without rework.

## How this skill works

1. **Match the repo.** Detect existing Docker/compose/CI and the package manager (pip/poetry/uv), and extend them rather than replacing.
2. **Containerize** the app as a slim, non-root, reproducible image (`references/dockerfile.md`).
3. **Gate merges with CI** — lint, type-check, test, build the image (`references/github-actions.md`).
4. **Keep config external and deploy pluggable** — 12-factor env, secrets injected at runtime, deploy as a final swappable stage.

## Containerization

Multi-stage build (deps layer → slim runtime), pinned base image, non-root user, `.dockerignore`, a `HEALTHCHECK`, and no secrets baked in. `docker-compose` wires the app to its dependencies (DB, etc.) for local/full-stack runs. Full patterns in `references/dockerfile.md`.

## CI — GitHub Actions

A workflow that runs on PR and main: install (cached) → lint → type-check → **run the test suite** (this is where `qa-tester`'s suite earns its keep) → build the image → publish to a registry on main. Fail fast; cache dependencies and Docker layers; pin action versions. The deploy job is a **separate, optional stage** that consumes the published image — so swapping deploy targets never touches the build/test pipeline. Full workflow in `references/github-actions.md`.

## Config & secrets (12-factor)

Everything environment-specific comes from env vars (the same `Settings` the app reads). Secrets live in GitHub Actions secrets / the runtime's secret store — **never** baked into an image, committed, or echoed into logs. Provide a committed `.env.example` (keys only, no values) so others know what to set.

## Deploy — kept flexible

The published image is the artifact; "where it runs" is a thin final step you don't lock the pipeline to:

| Target | Final step |
|--------|-----------|
| Self-host / VPS | `docker compose pull && up -d` (or a pull+restart script) |
| Cloud container service | push to its registry → service picks up the new tag |
| Kubernetes | update the image tag in the manifest / via the cluster's deploy mechanism |

Pick what the project actually uses; if none is set, stop at "image published" and document the run command — don't invent infrastructure.

## Quality floor (non-negotiable, never announced)

Reproducible, pinned builds; non-root runtime; minimal image (no build tools in the final stage); no secrets in image layers or logs; a working health check; CI that genuinely fails on lint/type/test errors (no `|| true` masking). Tag images with the commit SHA, not just `latest`.

## Your boundary

You own `Dockerfile`, `compose.yaml`, `.github/workflows/`, deploy scripts, `.dockerignore`, and `.env.example`. You do NOT edit application source, change behavior, author tests, or write docs. If the app needs a code change to be deployable (e.g. a missing health endpoint), report it to the owning agent rather than editing it yourself.

## Handoff

Write/update **`DEVOPS_HANDOFF.md`**: how to build and run locally, the env vars required (mirroring `.env.example`), the CI stages and what gates merges, the published image name/tag scheme, and the current deploy step (or "publish only" if none). Overwrite stale entries.

## Self-critique before delivering

- **Reproducibility pass:** would this build identically on a clean machine? Are deps and base image pinned?
- **Secret pass:** any secret in a layer, log, or committed file?
- **Gate pass:** does CI actually fail on a real lint/type/test failure, or is something masking it?
