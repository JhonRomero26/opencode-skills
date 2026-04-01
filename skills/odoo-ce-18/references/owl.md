# OWL 2 Reference — Odoo v18

## Base Component

```javascript
/** @odoo-module */
// Copyright 2025 {Company}
// License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

import { Component, useState, onWillStart, onMounted } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";
import { _t } from "@web/core/l10n/translation";

export class {ComponentName} extends Component {
    static template = "{module_name}.{ComponentName}";
    static props = {};

    setup() {
        this.orm = useService("orm");
        this.action = useService("action");
        this.notification = useService("notification");
        this.state = useState({ records: [], loading: true });
        onWillStart(() => this.loadData());
    }

    async loadData() {
        try {
            this.state.records = await this.orm.searchRead(
                "{module.model.name}",
                [["state", "=", "confirmed"]],
                ["name", "total", "partner_id"]
            );
        } catch (e) {
            this.notification.add(_t("Error loading data"), { type: "danger" });
        }
        this.state.loading = false;
    }

    openRecord(id) {
        this.action.doAction({
            type: "ir.actions.act_window",
            res_model: "{module.model.name}",
            res_id: id,
            views: [[false, "form"]],
        });
    }
}

registry.category("actions").add("{module_name}.{tag}", {ComponentName});
```

## Template XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- Copyright 2025 {Company}
     License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl). -->
<templates xml:space="preserve">
    <t t-name="{module_name}.{ComponentName}">
        <div class="o_action">
            <div class="o_control_panel d-flex align-items-center p-3">
                <h2 class="mb-0">Dashboard</h2>
            </div>
            <div class="o_content p-3">
                <t t-if="state.loading">
                    <div class="text-center p-5">
                        <i class="fa fa-spinner fa-spin fa-2x"/>
                    </div>
                </t>
                <t t-else="">
                    <div class="row">
                        <t t-foreach="state.records" t-as="rec" t-key="rec.id">
                            <div class="col-md-4 mb-3">
                                <div class="card cursor-pointer"
                                     t-on-click="() => this.openRecord(rec.id)">
                                    <div class="card-body">
                                        <h5 t-out="rec.name"/>
                                        <span class="badge text-bg-primary"
                                              t-out="rec.total"/>
                                    </div>
                                </div>
                            </div>
                        </t>
                    </div>
                </t>
            </div>
        </div>
    </t>
</templates>
```

## Registro como Client Action

```xml
<record id="action_{tag}" model="ir.actions.client">
    <field name="name">{Action Name}</field>
    <field name="tag">{module_name}.{tag}</field>
</record>
```

## Assets en **manifest**.py

```python
"assets": {
    "web.assets_backend": [
        "{module_name}/static/src/**/*",
    ],
},
```

## Patching Existing Components

```javascript
/** @odoo-module */
import { patch } from "@web/core/utils/patch";
import { FormController } from "@web/views/form/form_controller";

patch(FormController.prototype, {
  async onRecordSaved(record) {
    await super.onRecordSaved(record);
    // Custom logic
  },
});
```

## Available Services

| Service                      | Use                                               |
| ---------------------------- | ------------------------------------------------- |
| `useService("orm")`          | `searchRead`, `call`, `create`, `write`, `unlink` |
| `useService("action")`       | `doAction` — navigate/open views                  |
| `useService("notification")` | `.add(msg, {type: "success\|warning\|danger"})`   |
| `useService("dialog")`       | Modals                                            |
| `useService("rpc")`          | Low-level RPC                                     |
| `useService("user")`         | Current user info                                 |
| `useService("company")`      | Current company                                   |

## OWL 2 Hooks

| Hook                    | When                               |
| ----------------------- | ---------------------------------- |
| `useState(obj)`         | Reactive state (auto re-render)    |
| `useRef("name")`        | Reference to a DOM element         |
| `onWillStart(fn)`       | Before the first render (async OK) |
| `onMounted(fn)`         | After mounting in the DOM          |
| `onWillUnmount(fn)`     | Before unmounting (cleanup)        |
| `onWillUpdateProps(fn)` | Before updating props              |
| `onPatched(fn)`         | After re-render                    |
