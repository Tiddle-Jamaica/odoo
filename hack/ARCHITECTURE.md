# Project Architecture Notes

This repository is the Odoo server and standard addons source tree. It is organized as a Python web application with a modular addon system: the core framework lives in `odoo/`, installable business applications live in `addons/`, and the foundational `base` addon is bundled in `odoo/addons/base/`.

## Runtime Entry Points

- `odoo-bin` is the executable entry point. It imports `odoo.cli` and starts the selected command.
- `odoo/cli/server.py` implements the default server command. It parses configuration, reports addon paths and database connection settings, initializes requested databases, then starts the service layer through `odoo.service.server`.
- `odoo/http.py` is the WSGI/request layer. It routes static files, no-database routes, and database-backed requests, then dispatches matching controller methods decorated with `@odoo.http.route`.
- `odoo/service/` contains server, database, model, common, and security service code used by the runtime.

## Core Framework

- `odoo/orm/` contains the ORM implementation: model classes, fields, environments, domains, registries, and table abstractions.
- `odoo/modules/` handles module discovery, dependency graphs, loading, migrations, and registry updates.
- `odoo/tools/` contains shared infrastructure such as configuration, XML/data conversion, caching, safe evaluation, SQL helpers, translations, and asset utilities.
- `odoo/sql_db.py` is the lower-level PostgreSQL connection/cursor layer used beneath the ORM.

## Addon System

Odoo functionality is delivered through addons. Each addon is normally a directory with:

- `__manifest__.py` for metadata, dependencies, data files, and asset bundle declarations.
- `models/` for Python ORM models and extensions.
- `controllers/` for HTTP/JSON routes.
- `views/` for XML UI view definitions, menus, actions, reports, and templates.
- `static/src/` for JavaScript, XML component templates, SCSS, images, and other web assets.
- `security/` for access-control CSV and record-rule XML files.
- `data/`, `demo/`, `report/`, `wizard/`, and `tests/` where needed.

The root `addons/` directory contains the standard business modules such as `account`, `sale`, `stock`, `crm`, `website`, `point_of_sale`, and many localization modules. The built-in base module is at `odoo/addons/base/`.

## UI Location

The main web UI is in `addons/web/`.

- `addons/web/static/src/` contains the core JavaScript/Owl web client, UI components, services, views, search UI, webclient shell, SCSS, and frontend assets.
- `addons/web/static/src/main.js` and `addons/web/static/src/start.js` are part of the browser startup bundle.
- `addons/web/views/webclient_templates.xml` and related XML files define server-rendered web client templates.
- Feature modules extend the UI from their own `static/src/` directories, for example `addons/account/static/src/`, `addons/point_of_sale/static/src/`, and `addons/website/static/src/`.
- Addon manifests declare which UI files are included in asset bundles. For example, `addons/web/__manifest__.py` defines bundles such as `web.assets_backend`, `web.assets_web`, and `web.assets_frontend`.

## Database Models Location

Database models are Odoo ORM models written in Python.

- The ORM engine itself is in `odoo/orm/`, especially `odoo/orm/models.py`, `odoo/orm/fields*.py`, `odoo/orm/environments.py`, and `odoo/orm/registry.py`.
- Core database models are in `odoo/addons/base/models/`. This includes models such as users, companies, partners, countries, currencies, attachments, access rules, menus, views, actions, cron jobs, sequences, and module metadata.
- Business-module database models are in each addon under `addons/<module>/models/`. Examples include `addons/account/models/`, `addons/sale/models/`, `addons/stock/models/`, and `addons/crm/models/`.
- Models usually declare `_name` to create a model/table, `_inherit` to extend an existing model, and `fields.*` attributes to define stored or computed fields. The registry loads these definitions according to installed module manifests.

## Request and Data Flow

1. A process starts through `odoo-bin`, which delegates to the CLI command system.
2. The server command loads configuration, addon paths, and database settings.
3. HTTP requests enter through `odoo/http.py`.
4. Static asset requests are served from addon `static/` directories.
5. Database-backed requests open an Odoo registry/environment and dispatch to controller routes.
6. Controllers call ORM models through `request.env` or service helpers.
7. ORM models use `odoo/sql_db.py` to read and write PostgreSQL.
8. Views, QWeb templates, and asset bundles combine server metadata with the browser-side web client under `addons/web/static/src/`.

## Testing and Packaging

- Python dependencies are listed in `requirements.txt`.
- Packaging and distribution helpers are in `setup.py`, `setup.cfg`, `MANIFEST.in`, `setup/`, and `debian/`.
- Tests are distributed throughout `odoo/tests/`, `odoo/addons/base/tests/`, and addon-specific `tests/` directories.
