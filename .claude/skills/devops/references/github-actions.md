# GitHub Actions CI

A pipeline that gates every merge on lint + types + tests, then builds and publishes the image. Deploy is a separate, optional job so swapping targets never touches build/test.

## CI workflow (`.github/workflows/ci.yml`)

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip                       # cache deps for speed
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: ruff check .                  # lint  — fails the job on error
      - run: mypy app                      # type-check
      - run: pytest -q                     # the qa-tester suite gates the merge

  build:
    needs: test                            # only build if tests pass
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha            # cache Docker layers across runs
          cache-to: type=gha,mode=max
```

## Principles baked in

- **Fail fast, no masking.** Each check fails the job on a non-zero exit. Never `ruff check . || true` — a check that can't fail isn't a gate.
- **`build needs test`** — the image is only built after lint/types/tests pass, so a broken commit never produces a publishable artifact.
- **Caching** — `cache: pip` and GitHub Actions Docker layer cache keep runs fast without sacrificing reproducibility.
- **SHA tags** — images are tagged with `github.sha`, traceable to the exact commit (add a `latest` or semver tag on releases if wanted).
- **Pinned actions** — pin to major versions (`@v4`); pin to a SHA for stricter supply-chain safety.
- **Least-privilege token** — `permissions:` grants only what the job needs.

## Deploy (separate, pluggable job)

Keep deploy out of `ci.yml` or behind an environment gate, consuming the already-published image:

```yaml
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production            # enables required reviewers / secrets
    steps:
      - run: echo "swap this step for the project's target"
      # VPS:        ssh + `docker compose pull && up -d`
      # Cloud:      provider CLI updates the service to :$GITHUB_SHA
      # Kubernetes: kubectl set image / apply with the new tag
```

If the project has no deploy target yet, stop at `build` (publish only) and document the manual run command in `DEVOPS_HANDOFF.md`. Don't invent infrastructure that isn't there.

## Secrets

All credentials come from `secrets.*` (repo/org/environment secrets) injected at runtime — never written into the workflow file, the image, or logs. Use GitHub **environments** with required reviewers for production deploys.
