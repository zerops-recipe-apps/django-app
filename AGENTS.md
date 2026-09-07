# django-app

Minimal Django 5.2 file-upload demo on Zerops with PostgreSQL, S3-compatible object storage, and Mailpit — shared `base` setup extended by `prod` and `dev`.

## Zerops service facts

- HTTP port: `8000`
- Siblings:
  - `db` (PostgreSQL) — env: `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`
  - `storage` (Object Storage) — env: `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_BUCKET_NAME`, `S3_ENDPOINT_URL`
  - `mailpit` (SMTP mock) — env: `MAIL_HOST`, `MAIL_PORT` (override via `MAIL_HOST_OVERRIDE` / `MAIL_PORT_OVERRIDE`)
- Runtime base: `python@3.12` (Alpine on prod / Ubuntu on dev)

## Zerops dev

`setup: dev` has no `start` command — the runtime stays idle after init; the agent starts the dev server.

- Dev command: `python manage.py runserver 0.0.0.0:8000`
- Rebuild static assets without deploy: `python manage.py collectstatic --no-input`

**All platform operations (start/stop/status/logs of the dev server, deploy, env / scaling / storage / domains) go through the Zerops development workflow via `zcp` MCP tools. Don't shell out to `zcli`.**

## Notes

- `APP_DOMAIN_URLS` (defaults to `$zeropsSubdomain`) drives `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS` in `settings.py` — update it when binding a custom domain.
- Prod `initCommands` run `migrate`, `collectstatic`, and a one-time `createsuperuser` under `zsc execOnce` keyed on `$ZEROPS_appVersionId` — `dev` defines no `initCommands`, so the agent runs `python manage.py migrate` manually after SSHing in.
- Python packages install system-wide via `prepareCommands` (no venv) — no `node_modules`-style artifact is deployed.
