# .github/

GitHub Actions CI and repository configuration.

## CI Workflow (`workflows/ci.yml`)

Triggers on push and pull requests to `main` (ignores `docs/**`).

### Jobs

**linter**
- Sets up Python from `.python-version`
- Runs all pre-commit hooks (Ruff, mypy, djLint, prettier, etc.)
- Fails fast — fix lint before tests

**pytest**
- Builds Django and docs Docker images with GitHub Actions cache (GHA cache, scoped per image)
- Checks for pending migrations (`makemigrations --check`)
- Runs migrations
- Runs full pytest suite inside Docker
- Tears down the stack

### Caching strategy

Docker layer caching uses `type=gha` (GitHub Actions cache). Scopes:
- `django-cached-tests` — Django image
- `postgres-cached-tests` — Postgres image
- `cached-docs` — Docs image

Caches are invalidated when the relevant Dockerfile or dependencies change.

## Dependabot (`dependabot.yml`)

Configured for weekly updates to:
- Python packages (`pip` ecosystem, `pyproject.toml`)
- GitHub Actions versions

PRs are created automatically — review and merge or dismiss.

## Adding a new workflow

Create a new `.yml` file in `workflows/`. Follow the existing pattern:
- Pin action versions (e.g., `actions/checkout@v6`)
- Use `concurrency` to cancel in-progress runs on new push
- Use `paths-ignore` to skip docs-only changes

## Secrets

Production secrets (Sentry DSN, Mailgun keys, etc.) should be stored as GitHub repository secrets and referenced in workflows via `${{ secrets.MY_SECRET }}`.
