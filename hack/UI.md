# XML UI View Rendering Flow

Odoo maintains most of its UI as XML records stored in the database and resolved at runtime through the `ir.ui.view` model. There are two related but different XML UI paths:

- Application views such as `form`, `list`, `kanban`, `search`, `calendar`, `graph`, and `pivot`. These describe screen structure for the browser web client. The server resolves and post-processes them, then the JavaScript/Owl client renders the final interactive UI.
- QWeb templates, usually `type="qweb"`, which are XML templates rendered server-side into HTML. These are used for the web client bootstrap page, login pages, website pages, reports, and reusable layouts.

The central implementation is `odoo/addons/base/models/ir_ui_view.py`. The main web client UI lives under `addons/web/`, especially `addons/web/static/src/` and `addons/web/views/webclient_templates.xml`.

## Where XML UI Views Live

XML view definitions are shipped by modules and loaded into the database during module install/update.

- `addons/<module>/views/*.xml` defines menus, actions, record views, report templates, and QWeb templates.
- `addons/<module>/__manifest__.py` lists XML files in `data`, `demo`, and declares frontend/backend asset bundles under `assets`.
- `odoo/addons/base/models/ir_ui_view.py` defines the `ir.ui.view` database model that stores and resolves XML architecture.
- `addons/web/views/webclient_templates.xml` contains important QWeb templates such as `web.layout`, `web.frontend_layout`, `web.login`, and `web.webclient_bootstrap`.
- `addons/web/static/src/` contains the browser renderers and components that interpret resolved record-view XML.

## Storage Model

XML views become `ir.ui.view` records. Key fields include:

- `name`: human-readable view name.
- `model`: target ORM model for record views, for example `res.partner` or `account.move`.
- `type`: view type, such as `form`, `list`, `kanban`, `search`, or `qweb`.
- `arch_db`: the XML architecture stored in the database.
- `arch_fs`: the original source file path when loaded from module XML.
- `arch`: computed architecture. In normal mode it reads `arch_db`; in XML dev mode it may read the source file through `arch_fs`.
- `inherit_id`: parent view used by XML inheritance.
- `mode`: controls whether an inherited view is a primary view or an extension view.
- `key` / external XML id: template lookup key used heavily for QWeb templates.
- `group_ids` and XML `groups` attributes: restrict view availability or individual nodes.

There is also `ir.ui.view.custom`, used for per-user customizations. The controller endpoint `addons/web/controllers/view.py` exposes `/web/view/edit_custom` to update those custom arches.

## XML Inheritance

Odoo avoids duplicating complete views by using XML inheritance. A module can inherit another view and apply precise changes with `xpath`, `position`, and related specs.

The important flow is:

1. A base view provides the initial XML architecture.
2. Extension views point to that base view through `inherit_id`.
3. `ir.ui.view._get_combined_arch()` calls `_get_combined_archs()`.
4. `_get_combined_archs()` walks from the requested view to its root primary view.
5. It finds applicable inheriting views with `_get_inheriting_views()`.
6. It builds an inheritance hierarchy and combines it through `_combine(...)`.
7. The result is one final XML tree with all installed module customizations applied in priority order.

This is the key maintenance mechanism: modules extend existing UI without editing the original XML file. For example, an accounting module can add fields to a partner form by inheriting the partner form view and inserting nodes at defined XPath locations.

## Web Client Bootstrap

The main backend UI starts from the `/web`, `/odoo`, or `/odoo/<path>` routes in `addons/web/controllers/home.py`.

Server-side flow:

1. `Home.web_client()` ensures a database is selected.
2. It verifies that the user is logged in and allowed to use the internal web client.
3. It refreshes the session and restores the request environment with the logged-in user.
4. It calls `request.env['ir.http'].webclient_rendering_context()`.
5. `addons/web/models/ir_http.py` builds `session_info`, including user data, companies, currencies, debug state, asset bundle parameters, registry hash, and `view_info`.
6. `request.render('web.webclient_bootstrap', qcontext=context)` renders the bootstrap QWeb template.
7. The response is returned with `X-Frame-Options: DENY` and `Cache-Control: no-store`.

At this stage the server returns an HTML shell. The full backend interface is then driven by JavaScript assets from bundles such as `web.assets_backend` and `web.assets_web`, declared in `addons/web/__manifest__.py`.

## Server-Side QWeb Rendering

QWeb templates are rendered server-side when code calls `request.render(...)` or `env['ir.ui.view']._render_template(...)`.

The key execution path is:

1. A controller calls `request.render('template.xml_id', values)`.
2. `odoo/http.py` delegates rendering to `env['ir.ui.view']._render_template(...)`.
3. `ir.ui.view._render_template()` calls `env['ir.qweb']._render(template, values)`.
4. `odoo/addons/base/models/ir_qweb.py` prepares the rendering environment.
5. QWeb resolves the template by XML id or database id using `ir.ui.view`.
6. `ir.ui.view._preload_views()` fetches and caches template view records.
7. Inherited templates are combined into a final XML tree.
8. QWeb compiles and evaluates directives such as `t-call`, `t-if`, `t-foreach`, `t-set`, `t-out`, `t-esc`, `t-att-*`, and `t-call-assets`.
9. The renderer returns markup-safe HTML.

Examples of this path include login pages, frontend layouts, test pages, website pages, report HTML, and the backend web client bootstrap HTML.

## Record View Loading

Record views are not usually rendered to final HTML on the server. Instead, the server returns processed XML architecture and field metadata to the web client.

The usual backend flow is:

1. The browser loads the web client shell.
2. The user opens a menu.
3. Menus come from `/web/webclient/load_menus`, implemented in `addons/web/controllers/home.py`.
4. A menu triggers an `ir.actions.act_window` action for a model and a set of view types.
5. The browser calls `/web/dataset/call_kw`, implemented in `addons/web/controllers/dataset.py`.
6. That endpoint delegates to `odoo.service.model.call_kw(...)`.
7. For views, the called ORM method is commonly `get_views(...)` on the target model.
8. `Base.get_views()` in `ir_ui_view.py` calls `self.get_view(view_id, view_type, **options)` for each requested view.
9. The server returns processed XML architecture plus field definitions, toolbar actions, and optional search filters.
10. The JavaScript web client chooses the correct renderer from `addons/web/static/src/views/` and renders the UI in the browser.

So for form/list/kanban/search views, server-side rendering means "resolve, validate, enrich, secure, and serialize XML metadata." The actual interactive HTML is produced client-side by Owl components and Odoo view renderers.

## `get_views()` and `get_view()` Details

The main record-view methods are added to the abstract `base` model in `odoo/addons/base/models/ir_ui_view.py`.

`get_views(views, options)`:

- Receives a list like `[[view_id, 'form'], [False, 'list'], [False, 'search']]`.
- Calls `get_view(...)` for each requested view.
- Aggregates all field names used by the returned XML.
- Calls `fields_get(...)` on every involved model to return field descriptions.
- Adds toolbar actions when `toolbar` is requested.
- Adds saved filters when `load_filters` is requested.

`get_view(view_id, view_type, **options)`:

- Checks read access.
- Calls `_get_view_cache(...)`.
- Parses the cached XML.
- Applies user-specific access-right post-processing.
- Applies debug-mode handling.
- Serializes the XML tree back to a string.

`_get_view_cache(...)`:

- Resolves the requested view using `_get_view(...)`.
- Combines inherited views.
- Post-processes the XML with `postprocess_and_fields(...)`.
- Caches the result when XML dev mode is not enabled.

## View Resolution

`_get_view(view_id=None, view_type='form', **options)` decides which XML architecture to use.

Key behavior:

- If an explicit `view_id` is provided, that view is used.
- If no `view_id` is provided, Odoo checks the context for keys like `form_view_ref`, `list_view_ref`, or `kanban_view_ref`.
- If no context override exists, Odoo asks `ir.ui.view.default_view(model, view_type)` for the lowest-priority matching view.
- If no stored view exists, it may generate a fallback default view using methods such as `_get_default_form_view()`, `_get_default_list_view()`, `_get_default_search_view()`, and `_get_default_kanban_view()`.

This is why a module can define a default view, override a view through context, or rely on an automatically generated emergency fallback.

## Post-Processing

After inheritance is resolved, the XML is post-processed before it is sent to the browser.

Important responsibilities:

- Validate that referenced fields exist on the target model.
- Collect fields used by nodes, domains, contexts, modifiers, decorations, labels, filenames, and nested subviews.
- Add missing invisible fields required by expressions so the browser has data needed to evaluate them.
- Convert group restrictions into internal keys and later remove nodes the current user should not see.
- Add `model_access_rights` hints and create/write/delete flags based on access rights.
- Add `on_change="1"` to fields involved in onchange behavior.
- Embed missing x2many subviews for visible one2many/many2many fields in form views.
- Handle tag-specific behavior for `form`, `list`, `field`, `calendar`, `search`, `groupby`, `label`, and related nodes.

The main methods are:

- `postprocess_and_fields(...)`
- `_postprocess_view(...)`
- `_postprocess_tag_field(...)`
- `_postprocess_access_rights(...)`
- `_postprocess_on_change(...)`
- `_add_missing_fields(...)`

This step is where the raw XML becomes a secure, client-ready view description.

## Security and Visibility

XML views are not trusted as final UI output. The server adjusts them for the current user and model permissions.

Key points:

- `groups` attributes restrict individual XML nodes.
- `group_ids` on `ir.ui.view` restrict complete views.
- `_postprocess_access_rights(...)` removes nodes for groups the current user does not match.
- Model access rights are checked and encoded into the returned XML so the client knows whether create, edit, delete, or relational creation actions are available.
- Field-level groups affect whether field nodes are available.
- Debug-only nodes using `base.group_no_one` are handled specially.

The browser may hide or disable UI controls, but server-side ORM access rules remain the real enforcement point for data operations.

## Asset Inclusion

XML templates can include JavaScript and CSS bundles with `t-call-assets`.

The asset flow is:

1. Addon manifests declare bundle membership in `assets`.
2. QWeb templates call bundles, for example `t-call-assets="web.assets_frontend"` or backend bundles in the web client bootstrap.
3. The asset system resolves files across installed modules.
4. In normal mode assets are bundled/minified/cached.
5. In debug asset mode, individual source files are exposed for easier development.

The backend UI assets are mainly declared in `addons/web/__manifest__.py` and sourced from `addons/web/static/src/`.

## Key Execution Summary

For server-rendered QWeb pages:

1. Controller route handles HTTP request.
2. Controller calls `request.render(template, values)`.
3. `ir.ui.view` resolves the template view.
4. Inheritance is applied.
5. `ir.qweb` evaluates QWeb directives.
6. HTML is returned to the browser.

For backend record views:

1. Browser opens the web client shell from `/web` or `/odoo`.
2. Browser loads menus and actions.
3. Browser requests model views through `/web/dataset/call_kw`.
4. Model `get_views()` resolves XML views through `ir.ui.view`.
5. Inheritance, validation, field collection, access processing, and caching run on the server.
6. Server returns XML architecture and field metadata.
7. JavaScript/Owl renderers in `addons/web/static/src/views/` render the interactive UI.

The most important architectural point is that XML views are the durable UI contract. The server owns loading, inheritance, security, metadata, and QWeb rendering; the web client owns the live rendering and interaction for business record views.
