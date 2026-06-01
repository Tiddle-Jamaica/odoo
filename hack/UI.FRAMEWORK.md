# UI Framework Notes

The UI framework used in `addons/web/static/src/` is Odoo's Owl framework, imported in code as `@odoo/owl`.

Owl is a component-based JavaScript UI framework with XML templates. It is strongly inspired by modern reactive component frameworks such as React and Vue, but it keeps Odoo's XML/QWeb-style template language. In this project, the views folder is built around Owl components paired with XML templates.

## What This Looks Like in `addons/web/static/src/views`

Most view UI pieces are split into:

- A JavaScript component/controller/model/parser file, for behavior and state.
- A matching XML template file, for declarative markup.

For example:

- `addons/web/static/src/views/kanban/kanban_renderer.js`
- `addons/web/static/src/views/kanban/kanban_renderer.xml`
- `addons/web/static/src/views/kanban/kanban_record.js`
- `addons/web/static/src/views/kanban/kanban_record.xml`
- `addons/web/static/src/views/list/list_renderer.js`
- `addons/web/static/src/views/list/list_renderer.xml`
- `addons/web/static/src/views/form/form_renderer.js`

The JavaScript side imports Owl primitives:

```js
import { Component, useState, useRef, useEffect } from "@odoo/owl";
```

Components then bind to XML templates by name:

```js
export class KanbanRenderer extends Component {
    static template = "web.KanbanRenderer";
}
```

The XML side defines that template:

```xml
<t t-name="web.KanbanRenderer">
    ...
</t>
```

This pairing is the core pattern that shapes the `views/` folder.

## Template Language

The browser-side templates use Owl XML templates, which are closely related to QWeb syntax. They use directives such as:

- `t-name` to name a template.
- `t-if`, `t-elif`, and `t-else` for conditional rendering.
- `t-foreach`, `t-as`, and `t-key` for loops.
- `t-out` and `t-esc` for output.
- `t-att-*` and `t-attf-*` for dynamic attributes.
- `t-on-click` and other `t-on-*` handlers for DOM events.
- `t-ref` for element references.
- `t-slot` and `t-set-slot` for slots.
- Component tags such as `<KanbanRecord />`, `<Dropdown />`, and `<ActionHelper />`.

Example from the Kanban renderer template:

```xml
<t t-foreach="getGroupsOrRecords()" t-as="groupOrRecord" t-key="groupOrRecord.key">
    <KanbanRecord
        archInfo="props.archInfo"
        record="groupOrRecord.record"
        openRecord="props.openRecord"
    />
</t>
```

That is Owl component rendering, not server-side HTML rendering.

## View Architecture Pattern

The `views/` folder is organized around Odoo view types. Each view type usually has several cooperating parts:

- `*_view.js`: registers the view type in the Odoo registry.
- `*_controller.js`: handles user interaction and coordinates services.
- `*_renderer.js`: Owl component responsible for visual rendering.
- `*_model.js`: client-side data model where the view needs one.
- `*_arch_parser.js`: parses server-provided XML view architecture.
- `*_compiler.js`: converts Odoo view XML into Owl-compatible template structures when needed.
- `*.xml`: Owl templates for components.

For Kanban, `addons/web/static/src/views/kanban/kanban_view.js` registers the view:

```js
registry.category("views").add("kanban", kanbanView);
```

The registered definition points to:

- `KanbanArchParser`
- `KanbanCompiler`
- `KanbanController`
- `RelationalModel`
- `KanbanRenderer`

That means the server-provided XML view architecture is parsed, compiled, attached to a client model, and finally rendered by Owl components.

## Relationship to Server XML Views

There are two XML systems meeting here:

- Server-side Odoo XML views are stored in `ir.ui.view`. These define business views such as forms, lists, kanbans, and searches.
- Browser-side Owl XML templates in `addons/web/static/src/` define the reusable UI components that render those views.

The flow is:

1. The server resolves an Odoo XML view through `ir.ui.view`.
2. The browser receives the processed architecture through `get_views()`.
3. A view-specific parser reads the architecture.
4. A compiler may transform parts of that architecture into Owl template fragments.
5. Owl components render the final interactive UI.

In Kanban specifically, `kanban_compiler.js` extends the generic `ViewCompiler` and transforms kanban XML nodes such as fields, buttons, images, and `t-call` nodes into Owl-renderable template output.

## Why the Folder Looks This Way

The `addons/web/static/src/views` folder is inspired by a component architecture:

- Templates are declarative XML.
- Behavior is in JavaScript classes.
- State is managed through Owl hooks such as `useState`, `useRef`, `useEffect`, and lifecycle hooks.
- Services are injected with Odoo hooks such as `useService`.
- View types are pluggable through the Odoo `registry`.
- Shared components live under `views/view_components`, `views/fields`, and `core/`.

So the concise answer is: this project uses Owl, Odoo's reactive component framework, with XML/QWeb-like templates. The `views` folder is the client-side view framework that turns server-defined Odoo XML view architecture into interactive browser UI.
