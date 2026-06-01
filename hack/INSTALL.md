# Simple Run Instructions

This project can run locally through Docker using the files in `docker/`:

- `docker/compose.yaml`
- `docker/Dockerfile`
- `docker/odoo-docker.conf`
- `docker/README.md`

## Requirements

Install Docker Engine and Docker Compose v2. Confirm Compose works:

```bash
docker compose version
```

## Start The App

From the repository root:

```bash
docker compose -f docker/compose.yaml up --build
```

The first build can take a while because it installs system packages and Python dependencies.

## Initialize The Database

In a second terminal, run this once:

```bash
docker compose -f docker/compose.yaml exec odoo \
  ./odoo-bin -c docker/odoo-docker.conf -d odoo_dev -i base --stop-after-init
```

Then restart the Odoo service:

```bash
docker compose -f docker/compose.yaml restart odoo
```

## Open Odoo

Visit:

```text
http://localhost:8069
```

Default login:

```text
Email: admin
Password: admin
```

For frontend/UI development, use:

```text
http://localhost:8069/web?debug=assets
```

## Stop The App

```bash
docker compose -f docker/compose.yaml down
```

To remove local database and filestore volumes too:

```bash
docker compose -f docker/compose.yaml down -v
```
