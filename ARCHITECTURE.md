# Architecture

High-level decisions behind Rafiki's design.

## Overview

Rafiki is a personal task automation platform. The architecture prioritises developer experience, correctness, and production readiness over micro-optimisation.

```
Browser / API Client
       │
   Traefik (TLS, routing)
       │
  ┌────┴─────┐
  │  Django   │  ← Gunicorn (prod) / runserver_plus (dev)
  │  + DRF    │
  └────┬─────┘
       │
  ┌────┴──────────────────┐
  │  PostgreSQL  │  Redis  │
  └─────────────┴─────────┘
       │              │
  (data store)   (cache + broker)
                       │
               Celery Workers
               Celery Beat
```

## Key decisions

### Email-only authentication (no username)

**Why:** Usernames add complexity with no user value for a personal tool. Email is unique, familiar, and sufficient as an identifier. django-allauth supports email-only login natively.

**Impact:** `USERNAME_FIELD = "email"`. The `User` model has a single `name` field. Allauth adapters are customised in `rafiki/users/adapters.py`.

---

### Django + DRF (not FastAPI or async framework)

**Why:** Django's batteries — admin, ORM, migrations, allauth, celery integration — significantly reduce boilerplate for a data-heavy application. The overhead of Django is negligible for a personal tool. DRF adds API serialization and OpenAPI schema generation with minimal effort.

**Impact:** Synchronous request handling. Celery handles all async/background work. ASGI is available if async views are needed in future.

---

### Celery + Redis for task queue

**Why:** Cron-based scheduling is fragile and hard to introspect. Celery Beat with `django-celery-beat` allows schedules to be managed via the Django admin UI at runtime without redeployment. Redis doubles as both the message broker and the cache backend.

**Impact:** `celery_app.py` defines the app. Tasks are in `<app>/tasks.py` using `@shared_task`. Beat schedules are stored in the database (via `django-celery-beat`).

---

### uv for Python package management

**Why:** uv is significantly faster than pip/poetry for installation and resolution. It is drop-in compatible with `pyproject.toml` and `uv.lock` provides reproducible builds. The official uv Docker image is used as the base for all Python containers.

**Impact:** Use `uv add <package>` to add dependencies. Lock file is committed (`uv.lock`). Never use `pip install` directly.

---

### Docker for all environments

**Why:** Eliminates "works on my machine" issues. Local and production environments are structurally identical. Docker layer caching + uv makes builds fast.

**Impact:** All development commands run via `just` (wrapping `docker compose`). The production image uses a multi-stage build to keep the runtime image minimal.

---

### Traefik as reverse proxy (production)

**Why:** Traefik handles TLS certificate provisioning (Let's Encrypt ACME) automatically. It integrates natively with Docker labels for routing. No manual nginx TLS config needed.

**Impact:** Production has Traefik → Nginx (static) + Django (gunicorn). Local dev exposes Django directly on port 8000.

---

### Ruff for linting and formatting

**Why:** Ruff replaces flake8, isort, pyupgrade, and black with a single fast tool. Pre-commit integration ensures code is always clean before commit.

**Impact:** Extensive rule set defined in `pyproject.toml`. Migrations are excluded. Run `uv run ruff check . --fix` to auto-fix.

---

### drf-spectacular for API schema

**Why:** Auto-generates OpenAPI 3 schema from DRF ViewSets. Swagger UI at `/api/docs/` provides interactive documentation with zero manual maintenance.

**Impact:** Schema is available at `/api/schema/`. Add `@extend_schema` decorators to ViewSets to enrich documentation.

---

### Sentry for error tracking

**Why:** Silent failures in a personal automation tool are hard to debug. Sentry captures exceptions, Celery task failures, and performance data with minimal setup.

**Impact:** Configured in `production.py`. Requires `SENTRY_DSN` env var. Celery and Redis integrations are enabled.

---

### Mailgun for transactional email

**Why:** Reliable deliverability for email verification and notifications. `django-anymail` provides a unified backend interface — switching providers requires only a config change.

**Impact:** Configured in `production.py`. Local dev uses console email backend (printed to stdout).
