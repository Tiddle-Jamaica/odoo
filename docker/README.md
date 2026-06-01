# Docker Development Setup

This directory contains the Docker setup described in `hack/SETUP.md`.

Files:

- `compose.yaml`: runs PostgreSQL, the Odoo application container, and pgAdmin.
- `Dockerfile`: builds the Odoo development image from this source checkout.
- `odoo-docker.conf`: Odoo configuration for the Compose network.
- `pgadmin/servers.json`: pre-registers the Compose PostgreSQL server in pgAdmin.

Start the stack from the repository root:

```bash
docker compose -f docker/compose.yaml up --build
```

Initialize the development database once:

```bash
docker compose -f docker/compose.yaml exec odoo \
  ./odoo-bin -c docker/odoo-docker.conf -d odoo_dev -i base --stop-after-init
```

Restart Odoo after initialization:

```bash
docker compose -f docker/compose.yaml restart odoo
```

Open:

```text
http://localhost:8069
```

Open pgAdmin:

```text
http://localhost:5050
```

pgAdmin login:

```text
Email: admin@example.com
Password: admin
```

The `Odoo PostgreSQL` server is pre-registered. Connect with:

```text
Username: odoo
Password: odoo
Host: db
Port: 5432
```

Use debug assets while developing the web client:

```text
http://localhost:8069/web?debug=assets
```
