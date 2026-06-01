# ORM Used By This Project

This project uses Odoo's built-in Object Relational Mapping system, usually called the Odoo ORM.

It is not Django ORM or SQLAlchemy. The ORM is implemented inside this repository and is tightly integrated with Odoo's module system, security model, view metadata, and PostgreSQL database layer.

## Main Locations

- `odoo/orm/`: core ORM implementation.
- `odoo/orm/models.py`: base model behavior, recordsets, CRUD operations, caching, inheritance, and persistence rules.
- `odoo/orm/fields*.py`: field types such as `Char`, `Many2one`, `One2many`, `Many2many`, computed fields, binary fields, temporal fields, and selection fields.
- `odoo/orm/environments.py`: request/user/database context used by model operations.
- `odoo/orm/registry.py`: loaded model registry for each database.
- `odoo/sql_db.py`: lower-level PostgreSQL connection and cursor layer used below the ORM.

## How Application Models Use It

Business modules define models by subclassing Odoo model classes:

```python
from odoo import fields, models

class CrmLead(models.Model):
    _name = "crm.lead"
    _description = "Lead"

    name = fields.Char(required=True)
    user_id = fields.Many2one("res.users")
```

An example in this repository is `addons/crm/models/crm_lead.py`, where `CrmLead` subclasses `models.Model`, declares `_name = 'crm.lead'`, and defines fields with `fields.Char`, `fields.Many2one`, `fields.Properties`, and other Odoo field types.

## Key Concepts

- Models are Python classes registered by Odoo modules.
- Fields are declared as class attributes using `odoo.fields`.
- Records are manipulated through recordsets, for example `self.env["crm.lead"].search(...)`.
- The active database, user, language, company, and context are carried through `self.env`.
- Access rights, record rules, computed fields, onchange behavior, constraints, and inheritance are built into the ORM.
- Data is persisted in PostgreSQL.

## Common ORM APIs

- `create(vals)`: create records.
- `search(domain)`: find records matching an Odoo domain.
- `browse(ids)`: build a recordset from ids.
- `read(fields)`: read field values.
- `write(vals)`: update records.
- `unlink()`: delete records.
- `mapped(name)`, `filtered(fn)`, `sorted(...)`: recordset helpers.
- `@api.depends`, `@api.constrains`, `@api.onchange`: decorators for computed fields, validation, and UI-side onchange behavior.

In short: the ORM tool used by this project is the native Odoo ORM, backed by PostgreSQL and implemented under `odoo/orm/`.
