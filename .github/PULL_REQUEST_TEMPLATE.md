## What

<!-- Brief description of what this PR does -->

## Why

<!-- Why is this change needed? Link to issue if applicable -->

Closes #

## Checklist

- [ ] Pre-commit hooks pass (`uv run pre-commit run --all-files`)
- [ ] Tests pass (`docker compose -f docker-compose.local.yml run --rm django pytest`)
- [ ] No pending migrations (`just manage makemigrations --check`)
- [ ] New env vars documented in the relevant `CLAUDE.md`
- [ ] New features have tests
