# Multi-Tenancy

Odoo achieves multi-tenancy primarily through database-level isolation, with an additional multi-company layer inside a single database.

## Primary Tenant Boundary: Database

The strongest tenant boundary is one PostgreSQL database per Odoo tenant.

Each database has its own:

- Odoo registry.
- Installed modules.
- ORM model metadata.
- Business records.
- Users, companies, access rules, menus, views, and configuration.
- Filestore namespace for attachments and generated files.

The registry code makes this explicit: `odoo/orm/registry.py` defines one `Registry` instance per database name. A request is attached to a database, and the environment uses that database's registry and cursor.

Important locations:

- `odoo/orm/registry.py`: one model registry per database.
- `odoo/sql_db.py`: PostgreSQL connection and cursor layer.
- `odoo/http.py`: request dispatch chooses database-backed or no-database routing.
- `odoo/cli/server.py`: startup and database preload/initialization flow.

In practical deployment terms, separate customers or hard-isolated tenants are commonly separated by database. Odoo can host multiple databases from one server process, and the selected database determines the tenant context.

## Registry And Environment Isolation

For each request, Odoo creates an ORM environment around:

- `cr`: the current database cursor.
- `uid`: the current user.
- `context`: request and business context.
- `registry`: the model registry for the current database.

This is implemented in `odoo/orm/environments.py`.

Because the cursor and registry are database-specific, ORM calls such as:

```python
self.env["crm.lead"].search([])
```

only operate inside the currently selected database.

## Secondary Layer: Multi-Company Inside One Database

Inside a database, Odoo supports multiple companies through the `res.company` model.

This is not the same as full tenant isolation. It is a business partitioning layer for organizations that share one database but need company-specific records, accounting, sales teams, users, rules, reports, and configuration.

Important locations:

- `odoo/addons/base/models/res_company.py`: company model.
- `odoo/addons/base/models/res_users.py`: users, current company, and allowed companies.
- `odoo/orm/environments.py`: `env.company` and `env.companies`.
- `odoo/addons/base/models/ir_rule.py`: record rules.

The active company set is passed through context as `allowed_company_ids`. In `odoo/orm/environments.py`, `env.company` uses the first allowed company, and `env.companies` uses the full enabled company set. If a non-superuser tries to use unauthorized company ids, Odoo raises an access error.

## Record Rules Enforce Company Visibility

Company isolation inside one database is enforced by record rules and ORM access checks.

Record rules are defined by the `ir.rule` model in `odoo/addons/base/models/ir_rule.py`. Rule domains can use:

- `user`
- `company_id`
- `company_ids`

The rule evaluation context defines `company_ids` as the active companies selected by the user.

For example, CRM defines a company rule in `addons/crm/security/crm_security.xml`:

```xml
<field name="domain_force">[('company_id', 'in', company_ids + [False])]</field>
```

That means CRM leads are visible only when their `company_id` is one of the user's active companies, or when the lead is shared with no company.

## Company Consistency On Relations

Odoo also prevents cross-company inconsistencies through model and field checks.

Common patterns:

- Models define `company_id` or `company_ids`.
- Relational fields can use `check_company=True`.
- Models can enable `_check_company_auto = True`.
- The ORM method `_check_company()` verifies that related records belong to compatible companies.

An example appears in `addons/crm/models/crm_lead.py`, where CRM leads have company-aware fields and `_check_company_auto = True`.

## How The UI Participates

The web client receives company context during bootstrap.

In `addons/web/models/ir_http.py`, `session_info()` sends company data to the browser, including:

- current company.
- allowed companies.
- disallowed ancestor companies.
- company hierarchy metadata.

The company switcher updates the active company context. Subsequent RPC calls include the selected `allowed_company_ids`, so ORM rules and company-dependent behavior run under the chosen company set.

## Summary

Odoo's tenancy model has two layers:

1. Database-level multi-tenancy: each tenant can be a separate PostgreSQL database with its own registry, modules, records, users, rules, views, and filestore.
2. Multi-company partitioning: one database can contain multiple companies, with access scoped by `allowed_company_ids`, `res.company`, `res.users`, `ir.rule`, and ORM company checks.

For strong tenant separation, use separate databases. For related companies under one organization, use Odoo's multi-company features inside one database.
