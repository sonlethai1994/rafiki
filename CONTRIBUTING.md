# Contributing

## Prerequisites

- Docker & Docker Compose
- [just](https://github.com/casey/just)
- [uv](https://docs.astral.sh/uv/)

## Setup

```bash
git clone git@github.com:sonlethai1994/rafiki.git
cd rafiki
cp -r .envs/.local.example .envs/.local   # fill in values
just build
just up
just manage migrate
uv run pre-commit install
```

## Branch naming

```
feat/<short-description>     # New feature
fix/<short-description>      # Bug fix
chore/<short-description>    # Maintenance, deps, tooling
docs/<short-description>     # Documentation only
```

## Commit messages

Use the imperative mood, present tense. Keep the subject line under 72 characters.

```
Add user profile update endpoint
Fix celery beat PID cleanup on restart
Update django to 6.0.3
```

Avoid vague messages like `fix stuff`, `WIP`, or `changes`.

## Before opening a PR

Run the full quality check locally:

```bash
uv run pre-commit run --all-files         # Lint, format, type check
docker compose -f docker-compose.local.yml run --rm django pytest  # Tests
just manage makemigrations --check        # No pending migrations
```

All three must pass. CI will also run these checks automatically.

## Pull requests

- Keep PRs focused — one concern per PR
- Reference related issues in the description
- Fill out the PR template fully
- Request review before merging

## Adding a new Django app

See [rafiki/CLAUDE.md](rafiki/CLAUDE.md) for the checklist.

## Code style

- Python: Ruff (linting + formatting), enforced via pre-commit
- Templates: djLint, 2-space indent, 119 char line limit
- Type hints: required for new code, checked by mypy
- No inline styles in templates — use Bootstrap utilities or SCSS

## Environment files

`.envs/` is gitignored. Never commit secrets. If a new env var is required, document it in the relevant `CLAUDE.md`.
