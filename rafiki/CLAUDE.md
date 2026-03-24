# rafiki/

Main Django application directory. All business logic lives here.

## Structure

```
rafiki/
  users/         # User model, auth, profile, API
  contrib/       # Framework customizations (sites migrations)
  templates/     # HTML templates (all apps)
  static/        # Frontend assets (SCSS, JS, images, fonts)
  conftest.py    # Shared pytest fixtures
  __init__.py
```

## Creating a new Django app

```bash
just manage startapp <appname>
# then move it into rafiki/
mv <appname> rafiki/<appname>
```

Checklist after creating an app:

- [ ] Add to `LOCAL_APPS` in `config/settings/base.py`
- [ ] Create `rafiki/<app>/apps.py` with correct `name = "rafiki.<app>"`
- [ ] Add URL patterns to `config/urls.py` if needed
- [ ] Register API ViewSet in `config/api_router.py` if needed
- [ ] Add templates under `rafiki/templates/rafiki/<app>/`

## Conventions

- **Models:** Always define `__str__`, use `BigAutoField` (default), add `get_absolute_url()` where relevant.
- **Views:** Prefer class-based views (CBV). Use `LoginRequiredMixin` for protected views.
- **Tasks:** Place Celery tasks in `<app>/tasks.py`, use `@shared_task` decorator.
- **Tests:** Put tests in `<app>/tests/` as a package. Use `factory-boy` factories — never call `Model.objects.create()` directly in tests.
- **Migrations:** Never edit migrations manually. Always run `just manage makemigrations` after model changes.

## Test fixtures

Shared fixtures are in `rafiki/conftest.py`. App-level fixtures go in `<app>/tests/conftest.py`.
