# Advanced Patterns — Odoo v18 CE

## Wizard (TransientModel)

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

from odoo import api, fields, models, _


class {WizardClass}(models.TransientModel):
    _name = "{module.model.wizard}"
    _description = "{Wizard Description}"

    line_ids = fields.Many2many(comodel_name="{module.model.name}")
    date = fields.Date(default=fields.Date.context_today, required=True)

    @api.model
    def default_get(self, fields_list):
        res = super().default_get(fields_list)
        active_ids = self.env.context.get("active_ids", [])
        if active_ids:
            res["line_ids"] = [(6, 0, active_ids)]
        return res

    def action_apply(self):
        self.ensure_one()
        self.line_ids.write({"date": self.date})
        return {"type": "ir.actions.act_window_close"}
```

### Wizard View + Action

```xml
<record id="{wizard}_form" model="ir.ui.view">
    <field name="name">{module.model.wizard}.form</field>
    <field name="model">{module.model.wizard}</field>
    <field name="arch" type="xml">
        <form>
            <group>
                <field name="date"/>
                <field name="line_ids" widget="many2many_tags"/>
            </group>
            <footer>
                <button name="action_apply" string="Apply"
                        type="object" class="btn-primary"/>
                <button string="Cancel" special="cancel"/>
            </footer>
        </form>
    </field>
</record>

<record id="action_{wizard}" model="ir.actions.act_window">
    <field name="name">{Wizard Name}</field>
    <field name="res_model">{module.model.wizard}</field>
    <field name="view_mode">form</field>
    <field name="target">new</field>
    <field name="binding_model_id" ref="model_{model_underscored}"/>
    <field name="binding_view_types">list</field>
</record>
```

## Controller

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

from odoo import http
from odoo.http import request


class {ControllerClass}(http.Controller):

    @http.route("/{module}/api/records", type="json", auth="user", methods=["POST"])
    def get_records(self, domain=None, **kwargs):
        domain = domain or []
        return request.env["{module.model.name}"].search_read(
            domain, ["name", "total"], limit=100,
        )

    @http.route("/{module}/portal", type="http", auth="public", website=True)
    def portal_page(self, **kwargs):
        records = request.env["{module.model.name}"].sudo().search(
            [("state", "=", "done")], limit=50,
        )
        return request.render(
            "{module_name}.portal_template", {"records": records}
        )
```

## Report QWeb-PDF

### Action

```xml
<record id="action_report_{model}" model="ir.actions.report">
    <field name="name">{Report Name}</field>
    <field name="model">{module.model.name}</field>
    <field name="report_type">qweb-pdf</field>
    <field name="report_name">{module_name}.report_{model}</field>
    <field name="report_file">{module_name}.report_{model}</field>
    <field name="binding_model_id" ref="model_{model_underscored}"/>
    <field name="binding_type">report</field>
</record>
```

### Template

```xml
<template id="report_{model}">
    <t t-call="web.html_container">
        <t t-foreach="docs" t-as="doc">
            <t t-call="web.external_layout">
                <div class="page">
                    <h2><span t-field="doc.name"/></h2>
                    <table class="table table-sm mt-4">
                        <thead>
                            <tr>
                                <th>Description</th>
                                <th class="text-end">Qty</th>
                                <th class="text-end">Price</th>
                                <th class="text-end">Subtotal</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr t-foreach="doc.line_ids" t-as="line">
                                <td t-field="line.name"/>
                                <td class="text-end" t-field="line.quantity"/>
                                <td class="text-end" t-field="line.price_unit"
                                    t-options='{"widget": "monetary"}'/>
                                <td class="text-end" t-field="line.subtotal"
                                    t-options='{"widget": "monetary"}'/>
                            </tr>
                        </tbody>
                        <tfoot>
                            <tr>
                                <td colspan="3" class="text-end"><strong>Total:</strong></td>
                                <td class="text-end">
                                    <strong t-field="doc.total"
                                            t-options='{"widget": "monetary"}'/>
                                </td>
                            </tr>
                        </tfoot>
                    </table>
                </div>
            </t>
        </t>
    </t>
</template>
```

## Cron Job

```xml
<record id="ir_cron_{action}" model="ir.cron">
    <field name="name">{Module}: {Action description}</field>
    <field name="model_id" ref="model_{model_underscored}"/>
    <field name="state">code</field>
    <field name="code">model._cron_{action}()</field>
    <field name="interval_number">1</field>
    <field name="interval_type">days</field>
    <field name="numbercall">-1</field>
</record>
```

```python
def _cron_{action}(self):
    """Scheduled action description."""
    cutoff = fields.Date.subtract(fields.Date.today(), days=30)
    old = self.search([("state", "=", "cancelled"), ("write_date", "<", cutoff)])
    old.unlink()
    _logger.info("Cleaned %d records", len(old))
```

## Init Hooks & Multi-Company Base Data

In multi-company environments, `data.xml` files run **once** during module installation. If a new company is created *after* the module is installed, those XML records won't exist for the new company. To solve this, you need a combination of `post_init_hook` (for existing companies at install time) and a `create` override on `res.company` (for future companies).

### 1. The `post_init_hook` and `pre_init_hook`
Define hooks in `__init__.py`:
```python
from . import models

def post_init_hook(env):
    # Runs AFTER the module is installed.
    # E.g., populate base stages for all existing companies
    companies = env['res.company'].search([])
    for company in companies:
        env['fsm.stage']._create_base_stages(company)

def pre_init_hook(env):
    # Runs BEFORE the module is installed.
    # Useful for running raw SQL to prepare tables before the ORM acts
    pass
```
Register the hooks in `__manifest__.py`:
```python
    'post_init_hook': 'post_init_hook',
    'pre_init_hook': 'pre_init_hook',
```

### 2. Override `res.company` `create` for Future Companies
To guarantee that data (like `completed` and `cancel` stages) is created for any new company added in the future:

```python
class ResCompany(models.Model):
    _inherit = 'res.company'

    @api.model_create_multi
    def create(self, vals_list):
        companies = super().create(vals_list)
        for company in companies:
            # This method generates the required base records for the new company
            self.env['fsm.stage'].sudo()._create_base_stages(company)
        return companies
```

## Migration Scripts

Location: `migrations/18.0.X.Y.Z/`

### pre-migrate.py (before the update)

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

from openupgradelib import openupgrade


@openupgrade.migrate()
def migrate(env, version):
    if not version:
        return
    openupgrade.rename_fields(env, [
        ("{module.model}", "{table}", "old_field", "new_field"),
    ])
    openupgrade.rename_models(env, [
        ("old.model.name", "new.model.name"),
    ])
```

### post-migrate.py (after the update)

```python
from openupgradelib import openupgrade


@openupgrade.migrate()
def migrate(env, version):
    if not version:
        return
    env["{module.model.name}"].search(
        [("state", "=", "old")]
    ).write({"state": "new"})
```

## Server Action (Mass Action)

```xml
<record id="action_mass_{action}" model="ir.actions.server">
    <field name="name">Mass {Action}</field>
    <field name="model_id" ref="model_{model_underscored}"/>
    <field name="binding_model_id" ref="model_{model_underscored}"/>
    <field name="binding_view_types">list</field>
    <field name="state">code</field>
    <field name="code">records.action_{action}()</field>
</record>
```

## OCA Prefixes for Module Names

| Prefix       | Meaning                   | Example                     |
| ------------ | ------------------------- | --------------------------- |
| `base_`      | Base/foundation module    | `base_location_nuts`        |
| `l10n_CC_`   | Localization (CC=country) | `l10n_ec_invoice`           |
| `{module}_`  | Extends an Odoo module    | `sale_stock_forecast`       |
| `connector_` | External connector        | `connector_shopify`         |
| `report_`    | Reports                   | `report_qweb_pdf_watermark` |
