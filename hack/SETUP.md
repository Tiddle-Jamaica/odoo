# Local Development

This repository is a source checkout of Odoo. The local run loop is:

1. Install system dependencies.
2. Create a PostgreSQL user/database.
3. Create a Python virtual environment.
4. Install `requirements.txt`.
5. Run `./odoo-bin` with this repo's addon paths.
6. Open the web client at `http://localhost:8069`.

## Prerequisites

Recommended baseline:

- Linux, preferably Ubuntu 24.04 or Debian 12.
- Python 3.12 on Ubuntu 24.04, or Python 3.11 on Debian 12.
- PostgreSQL.
- Build tools and development headers for Python packages.
- `wkhtmltopdf` is optional but useful for PDF reports.

Install common packages on Ubuntu/Debian:

```bash
sudo apt update
sudo apt install -y \
  python3 python3-dev python3-venv python3-pip \
  build-essential git \
  postgresql postgresql-client libpq-dev \
  libxml2-dev libxslt1-dev zlib1g-dev libsasl2-dev libldap2-dev \
  libjpeg-dev libffi-dev libssl-dev libtiff-dev libopenjp2-7-dev \
  libwebp-dev libharfbuzz-dev libfribidi-dev libxcb1-dev \
  node-less npm wkhtmltopdf
```

Package names vary slightly by distribution. If `pip install -r requirements.txt` fails, install the missing `-dev` package named in the compiler error and retry.

## PostgreSQL Setup

Create a PostgreSQL role for local Odoo development:

```bash
sudo -u postgres createuser -s "$USER"
```

That gives your Linux user permission to create databases locally. If you prefer a dedicated role, create one:

```bash
sudo -u postgres createuser -s odoo
```

Then use `db_user = odoo` in the config below.

## Python Environment

From the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip wheel setuptools
pip install -r requirements.txt
```

Keep the virtual environment active when running `odoo-bin`.

## Local Config

Create a local config file outside version control, for example `hack/odoo-dev.conf`:

```ini
[options]
admin_passwd = admin
db_host = False
db_port = False
db_user = your_linux_user_or_odoo
db_password = False
addons_path = odoo/addons,addons
data_dir = /tmp/odoo-data
http_port = 8069
dev_mode = xml
```

Notes:

- `addons_path = odoo/addons,addons` is important. `odoo/addons` contains the built-in `base` addon; `addons` contains the standard business modules.
- `admin_passwd` is the master password for database management screens.
- `data_dir` stores filestore data, sessions, and generated assets.
- `dev_mode = xml` makes Odoo read XML views from source files more directly while developing.

If you created the PostgreSQL superuser with your Linux username, set `db_user` to that username.

## First Run

Initialize a database with the base module:

```bash
source .venv/bin/activate
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev -i base --stop-after-init
```

Then start the server:

```bash
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev
```

Open:

```text
http://localhost:8069
```

Default login after initializing `base` is usually:

```text
Email: admin
Password: admin
```

## Installing Apps

Install extra modules from the command line:

```bash
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev -i crm
```

Install or update several modules:

```bash
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev -i crm,sale,account
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev -u crm
```

Useful flags:

- `-i module_name`: install module.
- `-u module_name`: update module.
- `--stop-after-init`: perform install/update, then exit.
- `--dev=xml,qweb,assets`: enable developer reload behavior for XML, QWeb, and assets.

Example dev server:

```bash
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev --dev=xml,qweb,assets
```

## UI Development Notes

Backend web-client source lives mainly in:

```text
addons/web/static/src/
```

View renderers live in:

```text
addons/web/static/src/views/
```

Server-side XML views live in module `views/` folders, for example:

```text
addons/crm/views/crm_team_views.xml
```

When changing server XML views:

```bash
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev -u crm --stop-after-init
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev --dev=xml,qweb,assets
```

When changing web assets under `addons/web/static/src/`, use debug assets in the browser:

```text
http://localhost:8069/web?debug=assets
```

If assets appear stale, restart the server and hard-refresh the browser.

## Running Tests

Run Python tests for a module:

```bash
./odoo-bin -c hack/odoo-dev.conf -d odoo_test --test-enable --stop-after-init -i crm
```

Run tagged tests:

```bash
./odoo-bin -c hack/odoo-dev.conf -d odoo_test --test-enable --test-tags /crm --stop-after-init -i crm
```

For JavaScript tests, start Odoo with test assets enabled and use the built-in test routes:

```text
http://localhost:8069/web/tests?debug=tests
```

## Virtualization

You can run this project with Docker by containerizing two services:

- An Odoo application container built from this source checkout.
- A PostgreSQL database container.
- A pgAdmin container for browser-based PostgreSQL inspection.

Required local tools:

- Docker Engine.
- Docker Compose v2, available as `docker compose`.
- Enough disk space for Python dependencies, PostgreSQL data, Odoo filestore data, and generated assets.
- A bind mount of this repository into the Odoo container for source-code development.

Recommended container layout:

```text
odoo-dev/
  app: runs ./odoo-bin from this repository
  db: runs postgres
  pgadmin: browser UI for PostgreSQL
  db-data volume: stores PostgreSQL data
  odoo-data volume: stores Odoo filestore/session/asset data
  pgadmin-data volume: stores pgAdmin state
```

The root `docker/` folder contains the actual virtualization files:

- `docker/compose.yaml`
- `docker/Dockerfile`
- `docker/odoo-docker.conf`
- `docker/README.md`
- `docker/pgadmin/servers.json`

The Compose file follows this shape:

```yaml
name: odoo-dev

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoo
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U odoo -d postgres"]
      interval: 5s
      timeout: 5s
      retries: 20
    volumes:
      - db-data:/var/lib/postgresql/data

  odoo:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8069:8069"
      - "8072:8072"
    volumes:
      - ..:/workspace/odoo
      - odoo-data:/var/lib/odoo
    working_dir: /workspace/odoo
    command: >
      ./odoo-bin
      -c docker/odoo-docker.conf
      -d odoo_dev
      --dev=xml,qweb,assets

  pgadmin:
    image: dpage/pgadmin4:8
    depends_on:
      db:
        condition: service_healthy
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
      PGADMIN_CONFIG_SERVER_MODE: "False"
      PGADMIN_CONFIG_MASTER_PASSWORD_REQUIRED: "False"
    ports:
      - "5050:80"
    volumes:
      - pgadmin-data:/var/lib/pgadmin
      - ./pgadmin/servers.json:/pgadmin4/servers.json:ro

volumes:
  db-data:
  odoo-data:
  pgadmin-data:
```

A matching `docker/odoo-docker.conf` uses the Compose service name as the database host:

```ini
[options]
admin_passwd = admin
db_host = db
db_port = 5432
db_user = odoo
db_password = odoo
addons_path = odoo/addons,addons
data_dir = /var/lib/odoo
http_port = 8069
longpolling_port = 8072
dev_mode = xml
```

The Docker image needs the same system libraries used by the local setup plus Python dependencies from `requirements.txt`. A simple development Dockerfile usually:

1. Starts from a Python image close to the supported distro/Python version, such as `python:3.12-bookworm`.
2. Installs system packages: build tools, PostgreSQL client libraries, XML/XSLT libraries, LDAP/SASL headers, image libraries, Node/npm or Less tooling, and optionally `wkhtmltopdf`.
3. Sets a work directory such as `/workspace/odoo`.
4. Copies `requirements.txt`.
5. Runs `pip install -r requirements.txt`.
6. Uses the bind-mounted repository source at runtime.

First Docker run:

```bash
docker compose -f docker/compose.yaml up --build
```

In another terminal, initialize the database once:

```bash
docker compose -f docker/compose.yaml exec odoo \
  ./odoo-bin -c docker/odoo-docker.conf -d odoo_dev -i base --stop-after-init
```

Then restart the Odoo service:

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

The PostgreSQL server is pre-registered in pgAdmin as `Odoo PostgreSQL`. Use these database credentials if pgAdmin asks for them:

```text
Host: db
Port: 5432
Database: postgres
Username: odoo
Password: odoo
```

Docker development notes:

- Keep PostgreSQL data in a named volume so database state survives container rebuilds.
- Keep Odoo filestore data in a named volume so attachments and generated files survive restarts.
- Keep pgAdmin data in a named volume so saved browser database UI state survives restarts.
- Bind mount the repository so edits to Python, XML, JS, and SCSS files are visible inside the container.
- Use `/web?debug=assets` while working on `addons/web/static/src`.
- Re-run module updates inside the container after changing server XML views, for example:

```bash
docker compose -f docker/compose.yaml exec odoo \
  ./odoo-bin -c docker/odoo-docker.conf -d odoo_dev -u crm --stop-after-init
```

## Troubleshooting

Database connection fails:

- Check PostgreSQL is running: `sudo systemctl status postgresql`.
- Confirm `db_user` matches an existing PostgreSQL role.
- Try `createdb odoo_dev` with the same user to verify permissions.

Missing Python headers or libraries:

- Re-run the failing `pip install` command.
- Look at the package that failed to compile.
- Install the matching Debian/Ubuntu `-dev` package.

Blank or stale UI:

- Restart Odoo with `--dev=xml,qweb,assets`.
- Open `/web?debug=assets`.
- Hard-refresh the browser.
- Update the affected module if the change was in module XML.

Port already in use:

```bash
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev --http-port=8070
```

Reset a local database:

```bash
dropdb odoo_dev
createdb odoo_dev
./odoo-bin -c hack/odoo-dev.conf -d odoo_dev -i base --stop-after-init
```
