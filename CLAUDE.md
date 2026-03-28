# CLAUDE.md — Rafiki

Personal AI-powered planner built with Django 6.0. Automate tasks, schedule cron jobs, run custom workflows.

## Tech Stack

- **Backend:** Django 6.0.3, Django REST Framework 3.17, drf-spectacular (OpenAPI)
- **Auth:** django-allauth with MFA, email-based login (no username)
- **Task Queue:** Celery 5 + Redis, django-celery-beat for scheduled tasks
- **Database:** PostgreSQL 18, psycopg3
- **Cache/Broker:** Redis 7.2
- **Frontend:** Bootstrap 5, Webpack 5, SCSS
- **Package manager:** uv (Python), npm (Node)
- **Python:** 3.13 | **Node:** 24.14

---

## Local Development

All local development runs in Docker. Requires:

- `.envs/.local/.django`
- `.envs/.local/.postgres`

```bash
just build       # Build Docker images
just up          # Start all services (detached)
just down        # Stop all services
just logs        # Tail logs for all containers
just manage <cmd>  # Run manage.py inside the Django container
```

### First-time setup

```bash
just build
just up
just manage migrate
just manage createsuperuser
```

App available at **http://localhost:8000**
Flower (Celery monitor) at **http://localhost:5555**
Webpack dev server at **http://localhost:3000**

---

## Running Tests

```bash
# Inside Docker (preferred for CI parity)
docker compose -f docker-compose.local.yml run --rm django pytest

# Locally with uv
uv run pytest

# With coverage
uv run coverage run -m pytest
uv run coverage html
open htmlcov/index.html
```

---

## Code Quality

```bash
uv run ruff check .                    # Lint
uv run ruff format .                   # Format
uv run mypy rafiki                     # Type check
uv run djlint .                        # Template lint
uv run pre-commit run --all-files      # Run all hooks
```

Install pre-commit hooks once (runs automatically on git commit):

```bash
uv run pre-commit install
```

> Ruff has an extensive rule set defined in `pyproject.toml`. Check existing ignores before adding new ones. Migrations are excluded from linting and type-checking.

---

## Architecture

```
config/
  settings/
    base.py          # Shared settings (installed apps, DRF, Celery, allauth)
    local.py         # Debug=True, console email, eager Celery, debug toolbar
    production.py    # HTTPS, Redis cache, Mailgun, Sentry, gunicorn
    test.py          # Fast password hashing, fake webpack loader
  urls.py            # Root URL config
  api_router.py      # DRF router — register new ViewSets here
  celery_app.py      # Celery application config

rafiki/
  users/             # Only app currently
    models.py        # Custom User model (email auth, single name field)
    managers.py      # CustomUserManager
    views.py         # UserDetailView, UserUpdateView, UserRedirectView
    api/
      views.py       # UserViewSet with /me action
      serializers.py # HyperlinkedModelSerializer
    urls.py
    admin.py
    tasks.py         # Celery tasks
    tests/           # Full test suite (models, views, API, forms, tasks)

compose/
  local/django/      # Dev Dockerfile (uv, watchfiles, dev-user)
  local/node/        # Node Dockerfile for webpack
  production/django/ # Multi-stage prod build (Node → Python)

webpack/
  common.config.js   # Shared webpack config
  dev.config.js      # Dev server (port 3000, proxies to Django)
  prod.config.js     # Production build with source maps
```

---

## Key Conventions

- **User model:** Email is the login identifier (`USERNAME_FIELD = "email"`). Single `name` field — not `first_name`/`last_name`. Located at `rafiki.users.models.User`.
- **API relations:** Use `HyperlinkedModelSerializer` — serializers link to detail views by URL, not PK.
- **New APIs:** Register ViewSets in `config/api_router.py`. API schema auto-generated at `/api/schema/`, Swagger UI at `/api/docs/`.
- **Settings:** All config via `django-environ`. Never hardcode secrets. Use env files in `.envs/`.
- **Celery tasks:** Define in `<app>/tasks.py`, use `@shared_task`. Beat schedule managed via Django admin (django-celery-beat).
- **Templates:** Django templates with Bootstrap 5 in `rafiki/templates/`. djLint enforced (119 char line limit, 2-space indent).
- **Type hints:** mypy with django-stubs and djangorestframework-stubs. All new code should be typed.

---

## Services & Ports

| Service    | Port     | Notes                    |
| ---------- | -------- | ------------------------ |
| Django     | 8000     | Main app                 |
| Flower     | 5555     | Celery task monitor      |
| Webpack    | 3000     | Dev server (live reload) |
| PostgreSQL | internal |                          |
| Redis      | internal | Broker + cache           |

---

## Environment Files

| File                          | Purpose                                                  |
| ----------------------------- | -------------------------------------------------------- |
| `.envs/.local/.django`        | Local Django settings (secret key, Redis, Flower creds)  |
| `.envs/.local/.postgres`      | Local DB credentials                                     |
| `.envs/.production/.django`   | Production settings (Sentry DSN, Mailgun, allowed hosts) |
| `.envs/.production/.postgres` | Production DB credentials                                |

These are **never committed** to git.

---

## CI/CD

GitHub Actions (`.github/workflows/ci.yml`) runs on push/PR to `main`:

1. **linter** — runs pre-commit hooks (Ruff, mypy, djLint, etc.)
2. **pytest** — builds Docker images, runs migrations, runs full test suite

Dependabot is enabled for weekly dependency updates.
