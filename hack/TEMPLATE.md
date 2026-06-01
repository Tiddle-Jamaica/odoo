# CRM XML Templates And View Files

The XML files in `addons/crm/views/` are Odoo module data files. They are loaded because `addons/crm/__manifest__.py` lists them in the module's `data` section.

These files can contain several different kinds of XML elements. They look similar because they all live under `<odoo>`, but they do different jobs:

- `<record>` creates or updates database records.
- `<field>` sets fields on those records, or describes model fields inside a view architecture.
- `<menuitem>` is a shortcut for creating menu records.
- `<template>` defines a QWeb template, often HTML-like markup with dynamic `t-*` directives.

## `<record>`: Create Or Update Odoo Records

The `<record>` tag is generic Odoo data loading syntax.

Example from `addons/crm/views/crm_lead_views.xml`:

```xml
<record id="crm_lead_view_form" model="ir.ui.view">
    <field name="name">crm.lead.form</field>
    <field name="model">crm.lead</field>
    <field name="arch" type="xml">
        <form class="o_lead_opportunity_form" js_class="crm_form">
            ...
        </form>
    </field>
</record>
```

This does not define a Python model. It creates or updates a row in the database model named `ir.ui.view`.

Important parts:

- `id="crm_lead_view_form"` creates an external XML id, usually referenced as `crm.crm_lead_view_form`.
- `model="ir.ui.view"` says which Odoo ORM model receives the row.
- Child `<field>` elements set column values on that row.
- `<field name="arch" type="xml">` stores XML view architecture inside the `arch` field of `ir.ui.view`.

So the outer `<record>` affects database metadata. In this example, it stores the CRM lead form view definition in the database.

## `<field>` Has Two Meanings

The meaning of `<field>` depends on where it appears.

Inside a `<record>`, `<field>` sets an ORM field value on the record being loaded:

```xml
<field name="res_model">crm.lead</field>
<field name="view_mode">kanban,list,form</field>
```

Those values may be saved on models like:

- `ir.ui.view`
- `ir.actions.act_window`
- `ir.actions.act_window.view`
- `res.config.settings`
- `ir.rule`

Inside a view architecture, `<field>` means "show this model field in the UI":

```xml
<form>
    <sheet>
        <field name="name"/>
        <field name="partner_id"/>
        <field name="expected_revenue"/>
    </sheet>
</form>
```

That inner `<field>` does not create a database column. It tells the web client to render existing ORM fields from the target model, such as fields declared on `crm.lead` in `addons/crm/models/crm_lead.py`.

## `<record model="ir.ui.view">`: Database Row For UI Architecture

Most CRM view files define records on `ir.ui.view`.

These rows store UI architecture for form, list, kanban, search, graph, pivot, and calendar views. The actual fields must already exist on the Python model.

Common fields on `ir.ui.view` records:

- `name`: technical view name.
- `model`: target ORM model, for example `crm.lead`.
- `inherit_id`: optional parent view to extend.
- `priority`: ordering when multiple inherited views apply.
- `arch`: XML architecture.

Example extension view from CRM-related files:

```xml
<record model="ir.ui.view" id="utm_campaign_view_form">
    <field name="name">utm.campaign.view.form</field>
    <field name="model">utm.campaign</field>
    <field name="inherit_id" ref="utm.utm_campaign_view_form"/>
    <field name="arch" type="xml">
        ...
    </field>
</record>
```

This extends an existing UTM campaign form view rather than replacing the whole form.

## `<record model="ir.actions.act_window">`: Database Row For Navigation Actions

Some CRM XML records define actions, not visual layouts.

An `ir.actions.act_window` record tells Odoo what to open when the user clicks a menu or button.

Typical fields:

- `name`: label shown to the user.
- `res_model`: model to open, for example `crm.lead`.
- `view_mode`: allowed view modes, such as `kanban,list,form`.
- `domain`: default record filter.
- `context`: default values and search flags.
- `view_id` or `view_ids`: preferred views.
- `search_view_id`: search view to use.
- `help`: empty-state help content.

Menus often point to these actions.

## `<menuitem>`: Shortcut For Menu Records

`addons/crm/views/crm_menu_views.xml` uses `<menuitem>` heavily.

Example:

```xml
<menuitem
    id="menu_crm_opportunities"
    name="My Pipeline"
    parent="crm_menu_sales"
    action="crm.action_your_pipeline"
    sequence="1"/>
```

`<menuitem>` is shorthand for creating or updating an `ir.ui.menu` record.

Important attributes:

- `id`: external XML id for the menu.
- `name`: displayed menu label.
- `parent`: parent menu XML id.
- `action`: action XML id to execute when clicked.
- `groups`: security groups allowed to see the menu.
- `sequence`: ordering.
- `web_icon`: icon used for app-level menus.

So `crm_menu_views.xml` builds the CRM navigation tree: CRM root menu, Sales, Leads, Reporting, Configuration, and child menu entries.

## `<template>`: QWeb Template With HTML-Like Markup

`addons/crm/views/crm_helper_templates.xml` contains:

```xml
<template id="crm_action_helper" name="crm action helper">
    <t t-if="team.alias_email">
        <p class="o_view_nocontent_smiling_face">
            Create an opportunity to start playing with your pipeline.
        </p>
        <p>Use the <i>New</i> button, or send an email to
            <a t-attf-href="mailto:#{team.alias_email}">
                <t t-esc="team.alias_email"/>
            </a>
            to test the email gateway.
        </p>
    </t>
    <t t-else="">
        ...
    </t>
</template>
```

This is a QWeb template. It is stored as an `ir.ui.view` template record, but its purpose is not to define a form/list/kanban layout. It defines reusable HTML-like output.

QWeb directives in this file:

- `t-if`: render the block only if the expression is true.
- `t-else`: fallback block.
- `t-attf-href`: dynamically format an HTML attribute.
- `t-esc`: output escaped text.

The visible tags like `<p>`, `<i>`, and `<a>` are real HTML-style markup. The `<t>` tags are control tags used by QWeb and do not necessarily become visible DOM elements.

## Why `crm_action_helper` Exists

The CRM helper template provides empty-state helper content for the CRM pipeline.

It can show different text depending on whether a sales team has an email alias:

- If `team.alias_email` exists, it tells the user they can create an opportunity by sending email to that alias.
- If no alias exists, it tells the user to use the New button or configure an email alias.

This is why it looks more like an HTML snippet than a form view. It is not defining database columns or a full app screen; it is a reusable helper fragment.

## How These Pieces Work Together

1. `addons/crm/__manifest__.py` lists CRM XML files in `data`.
2. During module install/update, Odoo's XML loader reads those files.
3. `<record>` tags create or update metadata rows such as views, actions, filters, and settings.
4. `<menuitem>` tags create or update navigation menu rows.
5. `<template>` tags create QWeb template views.
6. When the user opens CRM, menus trigger actions.
7. Actions load `ir.ui.view` records for the target model.
8. The web client renders the view architecture using existing Python ORM fields.
9. QWeb templates render helper HTML snippets where referenced by actions, views, or server rendering code.

## Why Odoo Uses This XML Abstraction

Odoo is not only an application; it is also an application platform. The XML abstraction exists so business apps can be installed, extended, customized, translated, secured, and upgraded as modular metadata instead of hard-coded screens.

The main reason is that Odoo modules need to ship more than Python code. A module must also ship:

- UI views.
- Menus.
- Actions.
- Security rules.
- Reports.
- Email templates.
- Default records.
- Settings.
- Demo data.
- Help snippets and QWeb templates.

XML gives Odoo one declarative format for loading all of that into the database.

## Metadata-Driven UI

Odoo stores much of the UI as metadata in models like `ir.ui.view`, `ir.ui.menu`, and `ir.actions.act_window`.

That means the web client does not need a custom hand-written page for every business object. Instead:

1. Python models define fields and business behavior.
2. XML records define how those fields appear in forms, lists, kanbans, searches, reports, and menus.
3. The generic Odoo web client renders the UI from that metadata.

This is why a model like `crm.lead` can have form, kanban, list, graph, pivot, calendar, and search views without each one being a standalone JavaScript page.

## Module Installation And Upgrades

Odoo modules are installed into a database. XML records make the install process repeatable:

- Create this view.
- Create this menu.
- Create this action.
- Add this inherited view customization.
- Add this record rule.
- Load this default configuration.

The `id` on XML records becomes an external XML id, such as `crm.crm_lead_view_form`. Odoo uses those ids to update the same database records during module upgrades instead of creating duplicates.

This is essential for upgrades. A module can evolve its views and actions over time, and Odoo can match the XML file back to existing rows in the database.

## Extension Without Copying

Odoo is built for vertical apps and custom modules that extend each other. XML view inheritance is one of the big reasons for the abstraction.

Instead of copying the whole CRM lead form, another module can write:

```xml
<record id="my_crm_lead_form_extension" model="ir.ui.view">
    <field name="model">crm.lead</field>
    <field name="inherit_id" ref="crm.crm_lead_view_form"/>
    <field name="arch" type="xml">
        <field name="tag_ids" position="after">
            <field name="x_custom_score"/>
        </field>
    </field>
</record>
```

That small XML record says: "take the existing CRM lead form and insert my field after `tag_ids`."

This lets modules layer changes on top of each other. Accounting can extend partners, CRM can extend sales teams, website can extend portal views, and custom modules can extend all of them without rewriting the original screens.

## Customization By Users And Studio

Because views are data records, they can be modified through the database rather than only through source code.

This supports:

- administrator customizations.
- per-database configuration.
- Odoo Studio style changes.
- user-specific custom views.
- enabling/disabling menus and fields through groups.

The same concept behind XML-loaded `ir.ui.view` records also lets Odoo store edited views in the database.

## Security And Visibility

The XML abstraction lets Odoo attach security and visibility rules directly to UI metadata.

Examples:

- `groups` on `<menuitem>` controls who sees a menu.
- `groups` on view nodes controls who sees a field, page, button, or section.
- action domains restrict default record sets.
- record rules loaded from XML enforce server-side access.
- view post-processing removes inaccessible fields or controls before the UI reaches the browser.

This matters because Odoo's UI is generated from metadata. Security can be applied consistently while views are being resolved.

## Translation And Localization

XML views contain labels, help text, placeholders, menu names, report text, and snippets. Odoo's translation system can extract those strings from XML files.

This is a major reason for using declarative XML rather than scattering all UI text inside Python or JavaScript.

Localization modules can also load country-specific data, taxes, reports, fields, and views with the same mechanism.

## Low-Code Business Application Platform

Odoo's XML layer is part of its low-code architecture.

A large amount of business application behavior can be assembled from:

- ORM models.
- XML views.
- XML actions.
- XML menus.
- access rules.
- server actions.
- QWeb templates.

This makes new business modules faster to create. A developer can define a model in Python and immediately expose it through standard Odoo screens using XML metadata.

## Why Not Just Hard-Code HTML?

Hard-coded HTML would make each screen more isolated and harder to extend.

Odoo needs to support:

- many installed modules changing the same screen.
- customer-specific customizations.
- upgrades across versions.
- permissions and record rules.
- generic form/list/kanban/search renderers.
- translations.
- database-specific configuration.
- view editing through admin tools.

XML metadata gives Odoo a stable contract between Python models, database records, and the generic web client.

The tradeoff is that the XML can feel unusual at first: it mixes data loading, UI layout, menus, actions, and templates. But that is also the point. Odoo uses XML as the shared declaration layer for the application platform.

## Quick Mental Model

- Python files in `addons/crm/models/` define actual data models and fields.
- XML `<record>` elements load database metadata and configuration.
- XML `<field>` elements inside `<record>` set values on metadata records.
- XML `<field>` elements inside `<form>`, `<kanban>`, `<list>`, or `<search>` display existing ORM fields.
- XML `<menuitem>` elements create navigation entries.
- XML `<template>` elements define QWeb snippets that can contain HTML and dynamic `t-*` directives.

In short: CRM XML view files are not one single template language. They are Odoo module data files that mix database metadata, UI architecture, menu definitions, actions, and QWeb HTML fragments.
