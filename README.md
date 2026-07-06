# Paperless-NGX Scale Template

Docker Compose template for running Paperless-NGX with separate web, consumer, worker, and scheduler containers.

This template pins Paperless-NGX to `2.20.1` and keeps Redis, PostgreSQL, Paperless data, media, consume, export, and temporary storage in Compose-managed volumes or local folders.

## Services

- `broker`: Redis 7, used by Celery and Paperless task queues.
- `db`: PostgreSQL 17.
- `paperless-web`: web server exposed on host port `8008`.
- `paperless-consumer`: document consume service.
- `paperless-worker`: Celery worker for background tasks and OCR work.
- `paperless-scheduler`: Celery beat scheduler.

## Quick Start

Create local folders used by bind mounts:

```bash
mkdir -p consume export
```

Review `paperless.env`, then start the stack:

```bash
docker compose up -d
```

Open Paperless at:

```text
http://localhost:8008
```

## Configuration

Main settings live in `paperless.env`.

Important defaults:

- `PAPERLESS_SECRET_KEY=change-me-to-a-long-random-string` is a template placeholder. Change it for real deployments.
- `PAPERLESS_URL=http://localhost:8000` may need to match your external URL if deployed behind reverse proxy.
- `USERMAP_UID=1000` and `USERMAP_GID=1000` map the internal `paperless` user to host UID/GID 1000.
- `PAPERLESS_CONVERT_TMPDIR=/tmp/paperless-convert` uses the `convert_tmp` volume for conversion scratch space.

## Split-Container Init Overrides

The official Paperless-NGX image uses s6-overlay with `ENTRYPOINT ["/init"]`. Setting `command:` to an internal service script does not skip container init. Without overrides, every Paperless container tries to run shared init steps before its service starts.

This template intentionally leaves full init enabled for `paperless-web`, then disables selected shared-state init steps in non-web containers by bind-mounting `docker-overrides/skip-init` over those scripts:

- `init-migrations/run`: prevents consumer, worker, and scheduler from running Django database migrations.
- `init-search-index/run`: prevents consumer, worker, and scheduler from concurrently rebuilding the search index.

Only `paperless-web` should run migrations and search-index init.

## Init Steps Left Enabled

Non-web containers still run useful container-local init steps:

- Environment file handling.
- UID/GID mapping.
- Folder creation and permission checks.
- Wait-for-PostgreSQL.
- Wait-for-Redis.
- Django system checks.

If `PAPERLESS_ADMIN_USER` is later added to `paperless.env`, consider also skipping `init-superuser/run` in non-web containers to avoid repeated admin-user DB writes.

If `PAPERLESS_OCR_LANGUAGES` is later added, each Paperless container may try to install Tesseract language packages on startup. Prefer a custom image for production use if extra OCR language packages are required.

## Volumes And Data

Compose-managed volumes:

- `pgdata`: PostgreSQL data.
- `redisdata`: Redis append-only data.
- `data`: Paperless application data and index metadata.
- `media`: Paperless document storage.
- `convert_tmp`: conversion scratch space.
- `paperless_tmp`: Paperless scratch space.

Local bind mounts:

- `./consume`: drop documents here for consumption.
- `./export`: export/import folder.
- `./docker-overrides/skip-init`: no-op init script for non-web containers.

## Useful Commands

Start stack:

```bash
docker compose up -d
```

View logs:

```bash
docker compose logs -f
```

View Paperless logs only:

```bash
docker compose logs -f paperless-web paperless-consumer paperless-worker paperless-scheduler
```

Stop stack:

```bash
docker compose down
```

Stop and remove all named volumes:

```bash
docker compose down -v
```

## Notes

This is a template repo. It is intended to be edited before real deployment, especially secrets, external URL, resource limits, backups, and reverse proxy/TLS configuration.
