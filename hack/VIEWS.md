# Different XML Views

Odoo actually has **two different XML systems**, and that's where a lot of the confusion comes from.

### 1. XML in `views/` (Server-side Odoo Views)

These files are loaded into the database when the module is installed or upgraded.

Example:

```xml
<!-- addons/my_module/views/customer_views.xml -->
<odoo>
    <record id="view_customer_form" model="ir.ui.view">
        <field name="name">customer.form</field>
        <field name="model">res.partner</field>
        <field name="arch" type="xml">
            <form>
                <sheet>
                    <field name="name"/>
                </sheet>
            </form>
        </field>
    </record>
</odoo>
```

This XML:

* Creates form views
* Creates tree views
* Defines menus
* Defines actions
* Defines search views

It is stored in Odoo's database (`ir.ui.view`).

---

### 2. XML in `static/src/xml/` (OWL Templates)

These are frontend templates used by JavaScript OWL components.

Example:

```xml
<!-- addons/my_module/static/src/xml/customer_dashboard.xml -->
<templates xml:space="preserve">
    <t t-name="my_module.CustomerDashboard">
        <div>
            <h1 t-esc="state.title"/>
        </div>
    </t>
</templates>
```

This XML:

* Never becomes an `ir.ui.view`
* Is bundled into frontend assets
* Is consumed by JavaScript

Example component:

```javascript
/** @odoo-module **/

import { Component } from "@odoo/owl";

export class CustomerDashboard extends Component {
    static template = "my_module.CustomerDashboard";
}
```

The template name points to the XML in `static/src/xml`.

---

## How They Work Together

A common flow looks like this:

### Step 1: Server-side view

```xml
<record id="customer_dashboard_action" model="ir.actions.client">
    <field name="tag">customer_dashboard</field>
</record>
```

This tells Odoo:

> When this action runs, launch a client-side application.

---

### Step 2: JavaScript registers the action

```javascript
registry.category("actions").add(
    "customer_dashboard",
    CustomerDashboard
);
```

Now the action tag maps to an OWL component.

---

### Step 3: OWL component renders template

```javascript
export class CustomerDashboard extends Component {
    static template = "my_module.CustomerDashboard";
}
```

The template comes from:

```text
static/src/xml/customer_dashboard.xml
```

---

## Think of it this way

| Folder             | Purpose                             |
| ------------------ | ----------------------------------- |
| `views/`           | Backend metadata stored in database |
| `static/src/js/`   | OWL component logic                 |
| `static/src/xml/`  | OWL HTML templates                  |
| `static/src/scss/` | Styling                             |

The `views/` XML decides **when** and **where** something appears in Odoo.

The `static/src/xml/` templates decide **what the OWL component renders** once the browser is running the JavaScript.

So when you click a menu item and an OWL screen appears, the chain is typically:

```
Menu
  ↓
Action (views/)
  ↓
Client Action (views/)
  ↓
JS Component (static/src/js/)
  ↓
OWL Template (static/src/xml/)
  ↓
Rendered UI
```

That's the most common interaction between the XML in `views/` and the XML in `static/src/xml/`.

---

## Different Kind of Views

The XML files in `static/src/xml/` and the XML files in `views/` may look similar, but they serve completely different purposes.

| Location               | Purpose                                                                       | Processed By          |
| ---------------------- | ----------------------------------------------------------------------------- | --------------------- |
| `views/*.xml`          | Odoo metadata (forms, lists, menus, actions, security-related UI definitions) | Odoo server           |
| `static/src/xml/*.xml` | OWL frontend templates                                                        | Browser/OWL framework |

### `views/`

```text
my_module/
├── views/
│   ├── sale_order_views.xml
│   └── menu.xml
```

Contains things like:

```xml
<record model="ir.ui.view">
    ...
</record>

<menuitem .../>

<record model="ir.actions.act_window">
    ...
</record>
```

These are loaded into the database during module installation or upgrade.

---

### `static/src/xml/`

```text
my_module/
├── static/
│   └── src/
│       └── xml/
│           └── dashboard.xml
```

Contains OWL templates:

```xml
<templates xml:space="preserve">
    <t t-name="my_module.Dashboard">
        <h1>Hello</h1>
    </t>
</templates>
```

These are bundled as frontend assets and sent to the browser.

---

A useful way to think about it:

### `views/`

Defines:

> "Odoo, add a menu called Customers."

> "Open this action when clicked."

> "Show this form view for this model."

---

### `static/src/xml/`

Defines:

> "When my OWL component renders, generate this HTML."

---

The naming is unfortunate because both are XML "views," but one belongs to the **server-side Odoo framework** and the other belongs to the **client-side OWL framework**.

If you're coming from a web application background:

```text
views/
    ≈ routing + configuration + metadata

static/src/xml/
    ≈ React JSX templates / Vue templates
```

That analogy is usually the quickest way to build the correct mental model.
