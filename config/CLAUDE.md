# config/

Django project configuration — settings, URLs, Celery, and API routing.

## Structure

```
config/
  settings/
    base.py        # Shared across all environments
    local.py       # Local dev (DEBUG=True, console email, eager Celery)
    production.py  # Production (HTTPS, Redis cache, Mailgun, Sentry)
    test.py        # Test suite (fast passwords, fake webpack)
  urls.py          # Root URL config
  api_router.py    # DRF router — register all ViewSets here
  celery_app.py    # Celery application instance
  wsgi.py          # WSGI entry point (production)
  asgi.py          # ASGI entry point
```

## Settings

All settings use `django-environ`. Never hardcode secrets — always use `env()`.

```python
# Good
SECRET_KEY = env("DJANGO_SECRET_KEY")

# Bad
SECRET_KEY = "hardcoded-secret"
```

Settings are split by environment. `base.py` is always loaded; the active settings module is set via `DJANGO_SETTINGS_MODULE`.

| Module                       | When used        |
| ---------------------------- | ---------------- |
| `config.settings.local`      | Local Docker dev |
| `config.settings.production` | Production       |
| `config.settings.test`       | pytest           |

## Adding a new setting

1. Add the default/logic to `base.py`
2. Override in `local.py` or `production.py` only if the value differs per environment
3. If it requires a secret, add the env var to the relevant `.envs/` file

## Adding a new Django app

1. Create the app under `rafiki/`
2. Add to `INSTALLED_APPS` in `base.py` (under `LOCAL_APPS`)
3. Add URLs to `config/urls.py` if needed

## API Router

All DRF ViewSets are registered in `api_router.py`:

```python
router.register(r"users", UserViewSet)
router.register(r"tasks", TaskViewSet)  # add new ones here
```

The router is included at `/api/` in `urls.py`. API schema is auto-generated at `/api/schema/`, Swagger UI at `/api/docs/`.

## URLs

| Path                   | Purpose                          |
| ---------------------- | -------------------------------- |
| `/`                    | Home page                        |
| `/about/`              | About page                       |
| `/users/`              | User views (allauth + custom)    |
| `/api/`                | DRF API (router)                 |
| `/api/schema/`         | OpenAPI schema (drf-spectacular) |
| `/api/docs/`           | Swagger UI                       |
| `/api/auth-token/`     | Token auth endpoint              |
| `/<DJANGO_ADMIN_URL>/` | Admin (URL set via env var)      |

## Celery

Celery app is defined in `celery_app.py`, auto-discovers tasks from all installed apps. Configuration lives in `base.py` under `CELERY_*` settings.

- Broker: Redis (`CELERY_BROKER_URL`)
- Beat scheduler: `django-celery-beat` (schedule managed via Django admin)
- Task time limit: 5 minutes (soft: 60s)
- Local dev: `CELERY_TASK_ALWAYS_EAGER=True` (tasks run synchronously)
