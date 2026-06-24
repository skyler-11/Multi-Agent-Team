# Dockerfile + Compose

Goal: a small, reproducible, non-root image that builds the same on any machine, plus a compose file that runs the app with its dependencies locally.

## Multi-stage Dockerfile (FastAPI)

```dockerfile
# ---- builder: install deps into a venv ----
FROM python:3.12-slim AS builder
ENV PIP_NO_CACHE_DIR=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install -r requirements.txt          # or: poetry/uv export then install

# ---- runtime: slim, non-root, only what's needed ----
FROM python:3.12-slim AS runtime
ENV PATH="/opt/venv/bin:$PATH" PYTHONUNBUFFERED=1
WORKDIR /app
COPY --from=builder /opt/venv /opt/venv       # bring the built venv, not build tools
COPY . .
RUN useradd -m appuser && chown -R appuser /app
USER appuser                                  # never run as root
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request;urllib.request.urlopen('http://localhost:8000/health')" || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Key choices: the runtime stage copies only the prebuilt venv + app, so build tools never ship; `slim` base + pinned Python version keeps it small and reproducible; non-root `appuser`; a real health check that hits an app endpoint. The app should expose a cheap `GET /health` for this and for orchestrators.

## `.dockerignore` (keep the image small and clean)

```
.git
.venv
__pycache__/
*.pyc
tests/
.env
.env.*
*.md
.github/
```

Excludes secrets (`.env`), VCS, tests, and docs from the build context — smaller images and no accidental secret in a layer.

## docker-compose (local full-stack)

```yaml
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env            # local only; never committed
    depends_on:
      db: { condition: service_healthy }
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 5
    volumes: ["pgdata:/var/lib/postgresql/data"]
volumes:
  pgdata:
```

Notes: `depends_on` with `service_healthy` means the API waits for a ready DB, not just a started container. Use a volume so DB data survives restarts. For dev hot-reload, mount the source and run uvicorn with `--reload` in a compose override, not in the production image.

## Build & run

```bash
docker build -t myapp:$(git rev-parse --short HEAD) .   # tag with commit SHA
docker compose up --build                               # full stack locally
```

Tag with the commit SHA (not only `latest`) so every image is traceable to a commit.
