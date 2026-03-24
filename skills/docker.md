# Docker — Expert Best Practices

## Mental model

A Docker image is an immutable artifact. Build once, run anywhere. The Dockerfile is a recipe for that artifact — layer order determines cache efficiency. A poorly ordered Dockerfile rebuilds from scratch on every code change. A well-ordered one rebuilds only what changed.

---

## Expert rules

### Layer caching

**1. Order Dockerfile instructions from least to most frequently changing.**

```dockerfile
# Bad — code copy before dependency install — cache busted on every code change
COPY . /app
RUN uv sync

# Good — install deps first (changes rarely), copy code last (changes often)
COPY pyproject.toml uv.lock /app/
RUN uv sync --frozen
COPY . /app
```

**2. Use `--mount=type=cache` (BuildKit) to persist package manager caches across builds.**

```dockerfile
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project
```

This is already used in Rafiki's local Dockerfile — never remove it.

**3. Use `.dockerignore` aggressively. Exclude `.git`, `node_modules`, `__pycache__`, test files, and local env files.**

---

### Multi-stage builds

**4. Use multi-stage builds to keep the runtime image minimal. Only copy what the app needs to run.**

```dockerfile
# Stage 1: build
FROM ghcr.io/astral-sh/uv:python3.13-bookworm AS builder
COPY pyproject.toml uv.lock .
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# Stage 2: runtime — no uv, no build tools
FROM python:3.13-slim-bookworm
COPY --from=builder /app/.venv /app/.venv
COPY . /app
```

**5. Name stages semantically (`AS builder`, `AS runner`) not generically (`AS stage1`).**

---

### Security

**6. Never run as root in production. Create a non-root user and switch to it.**

```dockerfile
RUN groupadd -r django && useradd -r -g django django
USER django
```

Rafiki's production Dockerfile already does this. Never remove it.

**7. Never `COPY` `.env` files or secrets into images. Use environment variables at runtime.**

**8. Pin base image versions to a specific digest for reproducible builds.**

```dockerfile
FROM python:3.13.3-slim-bookworm  # pinned minor version
```

---

### Signal handling (PID 1)

**9. Use `exec form` (JSON array) for `CMD` and `ENTRYPOINT` — ensures the process receives signals directly, not through a shell.**

```dockerfile
# Bad — shell form: signals go to sh, not your process
CMD gunicorn config.wsgi

# Good — exec form: process is PID 1, receives SIGTERM
CMD ["gunicorn", "config.wsgi", "--bind", "0.0.0.0:5000"]
```

**10. Use `tini` or `dumb-init` as PID 1 if your process doesn't handle signals natively. Prevents zombie processes.**

```dockerfile
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["gunicorn", "config.wsgi"]
```

---

### Compose

**11. Use YAML anchors (`&anchor` / `<<: *anchor`) to DRY up repeated service config — already used in Rafiki for celery services.**

```yaml
services:
  django: &django
    build: .
    env_file: .envs/.local/.django

  celeryworker:
    <<: *django
    command: /start-celeryworker
```

**12. Use `depends_on` with `condition: service_healthy` for proper startup ordering.**

```yaml
depends_on:
  postgres:
    condition: service_healthy
```

**13. Use `profiles` to opt-in services for specific workflows (e.g., only start docs service when needed).**

```yaml
services:
  docs:
    profiles: ["docs"]
```

**14. Use `docker compose override` files for local developer customization without modifying the base compose file.**

---

### Healthchecks

**15. Define healthchecks for all services that others depend on.**

```yaml
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
    interval: 10s
    timeout: 5s
    retries: 5
```

---

## Common mistakes

- Using `latest` as the base image tag — breaks reproducibility silently
- Installing dev dependencies in the production image — bloats image, exposes tooling
- Copying the entire repo before installing dependencies — destroys layer cache on every commit
- Running multiple processes in one container — breaks the one-process-per-container principle
- Not setting resource limits — one runaway container can starve others

---

## References

- [Docker BuildKit docs](https://docs.docker.com/build/buildkit/)
- [Dockerfile best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Compose file reference](https://docs.docker.com/compose/compose-file/)
