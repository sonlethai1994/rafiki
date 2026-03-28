# compose/

Dockerfiles for each service, split by environment.

## Structure

```
compose/
  local/
    django/
      Dockerfile          # Dev Django image (uv, watchfiles, dev-user)
      entrypoint          # Waits for PostgreSQL, then runs command
      start               # Runs migrations then runserver_plus
      start-celeryworker  # Starts worker with watchfiles auto-reload
      start-celerybeat    # Starts beat scheduler with watchfiles
      start-flower        # Starts Flower monitor
    node/
      Dockerfile          # Node image for webpack dev server
  production/
    django/
      Dockerfile          # Multi-stage prod image (Node → Python)
      entrypoint
      start               # Collects static then gunicorn
      start-celeryworker
      start-celerybeat
      start-flower
    postgres/
      Dockerfile          # PostgreSQL 18 with backup scripts
      maintenance/        # Backup/restore utilities
    nginx/
      Dockerfile          # Nginx for static/media files
      nginx.conf
    traefik/
      Dockerfile          # Traefik reverse proxy
      traefik.yml
```

## Local Django image

- Base: `ghcr.io/astral-sh/uv:python3.13-bookworm-slim`
- Uses `uv` for fast dependency installation with layer caching
- Includes `build-essential`, `libpq-dev`, `gettext`, `wait-for-it`
- Creates `dev-user` with sudo (for devcontainer compatibility)
- Start scripts are copied to `/` and made executable

## Production Django image (multi-stage)

1. **client-builder** (Node) — runs `npm run build`, produces `webpack_bundles/`
2. **python-build-stage** — compiles Python wheels with uv
3. **python-run-stage** — minimal runtime, copies wheels + built assets, runs as non-root `django` user

## Start scripts

All scripts follow the same pattern:

1. Wait for dependencies (via `wait-for-it` or entrypoint)
2. Run setup (migrations, static collection)
3. Start the process

Never run `manage.py` commands directly in Dockerfiles — use start scripts.

## Adding a new service

1. Add a `Dockerfile` under `compose/<env>/<service>/`
2. Add the service definition to `docker-compose.<env>.yml`
3. Add start script if the service needs initialization logic
4. Add env file reference if the service needs secrets

## Rebuild vs restart

```bash
just build     # Rebuild images (after Dockerfile or dependency changes)
just up        # Start/restart containers (no rebuild)
just down      # Stop containers
just prune     # Remove containers + volumes (full reset)
```
