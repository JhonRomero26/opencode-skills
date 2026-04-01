# Templates de Scaffolding — Odoo v18 OCA

Usa estos templates como base al generar cada archivo. Reemplaza los placeholders `{...}`.

## Modelo Completo

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

import logging

from odoo import api, fields, models, _
from odoo.exceptions import UserError, ValidationError

_logger = logging.getLogger(__name__)


class {ClassName}(models.Model):
    _name = "{module.model.name}"
    _description = "{Model Human Name}"
    _inherit = ["mail.thread", "mail.activity.mixin"]
    _order = "sequence, id desc"

    # --- Defaults ---
    def _default_company_id(self):
        return self.env.company

    # --- Fields: primitivos → relacionales → computados ---
    active = fields.Boolean(default=True)
    name = fields.Char(required=True, index=True, tracking=True)
    description = fields.Text()
    sequence = fields.Integer(default=10)
    state = fields.Selection(
        selection=[
            ("draft", "Draft"),
            ("confirmed", "Confirmed"),
            ("done", "Done"),
            ("cancelled", "Cancelled"),
        ],
        default="draft",
        required=True,
        tracking=True,
    )
    company_id = fields.Many2one(
        comodel_name="res.company",
        default=_default_company_id,
        required=True,
    )
    partner_id = fields.Many2one(
        comodel_name="res.partner",
        required=True,
        tracking=True,
    )
    line_ids = fields.One2many(
        comodel_name="{module.model.line}",
        inverse_name="parent_id",
    )
    tag_ids = fields.Many2many(comodel_name="{module.model.tag}")
    total = fields.Float(compute="_compute_total", store=True)

    # --- SQL Constraints ---
    _sql_constraints = [
        ("name_company_uniq", "unique(name, company_id)",
         "Name must be unique per company!"),
    ]

    # --- Python Constraints ---
    @api.constrains("name")
    def _check_name(self):
        for record in self:
            if record.name and len(record.name) < 3:
                raise ValidationError(
                    _("Name must be at least 3 characters for '%s'.")
                    % record.name
                )

    # --- Compute ---
    @api.depends("line_ids.subtotal")
    def _compute_total(self):
        for record in self:
            record.total = sum(record.line_ids.mapped("subtotal"))

    # --- Onchange ---
    @api.onchange("partner_id")
    def _onchange_partner_id(self):
        if self.partner_id:
            self.name = self.partner_id.name

    # --- CRUD ---
    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if not vals.get("name"):
                vals["name"] = self.env["ir.sequence"].next_by_code(
                    "{module.model.name}"
                ) or _("New")
        return super().create(vals_list)

    def unlink(self):
        if self.filtered(lambda r: r.state != "draft"):
            raise UserError(_("Only draft records can be deleted."))
        return super().unlink()

    # --- Actions ---
    def action_confirm(self):
        for record in self:
            if record.state != "draft":
                raise UserError(_("Only draft records can be confirmed."))
            record.state = "confirmed"

    def action_done(self):
        self.filtered(lambda r: r.state == "confirmed").write({"state": "done"})

    def action_cancel(self):
        self.write({"state": "cancelled"})

    def action_draft(self):
        self.filtered(
            lambda r: r.state == "cancelled"
        ).write({"state": "draft"})
```

## Line Model (One2many child)

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

from odoo import api, fields, models


class {ClassName}Line(models.Model):
    _name = "{module.model.line}"
    _description = "{Model} Line"
    _order = "sequence, id"

    parent_id = fields.Many2one(
        comodel_name="{module.model.name}",
        required=True,
        ondelete="cascade",
        index=True,
    )
    sequence = fields.Integer(default=10)
    name = fields.Char(required=True)
    quantity = fields.Float(default=1.0)
    price_unit = fields.Float(digits="Product Price")
    subtotal = fields.Float(
        compute="_compute_subtotal",
        store=True,
    )

    @api.depends("quantity", "price_unit")
    def _compute_subtotal(self):
        for line in self:
            line.subtotal = line.quantity * line.price_unit
```

## Complete View (Form + List + Search + Action + Menu)

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- Copyright 2025 {Company}
     License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl). -->
<odoo>

    <!-- ==================== FORM ==================== -->
    <record id="{module_model}_form" model="ir.ui.view">
        <field name="name">{module.model.name}.form</field>
        <field name="model">{module.model.name}</field>
        <field name="arch" type="xml">
            <form>
                <header>
                    <button name="action_confirm" string="Confirm"
                            type="object" class="btn-primary"
                            invisible="state != 'draft'"/>
                    <button name="action_done" string="Done"
                            type="object"
                            invisible="state != 'confirmed'"/>
                    <button name="action_cancel" string="Cancel"
                            type="object"
                            invisible="state in ('done', 'cancelled')"/>
                    <button name="action_draft" string="Reset to Draft"
                            type="object"
                            invisible="state != 'cancelled'"/>
                    <field name="state" widget="statusbar"
                           statusbar_visible="draft,confirmed,done"/>
                </header>
                <sheet>
                    <div class="oe_button_box" name="button_box"/>
                    <widget name="web_ribbon" title="Archived"
                            bg_color="text-bg-danger"
                            invisible="active"/>
                    <div class="oe_title">
                        <label for="name"/>
                        <h1><field name="name" placeholder="Name..."/></h1>
                    </div>
                    <group>
                        <group>
                            <field name="partner_id"/>
                        </group>
                        <group>
                            <field name="company_id"
                                   groups="base.group_multi_company"/>
                        </group>
                    </group>
                    <notebook>
                        <page string="Lines" name="lines">
                            <field name="line_ids">
                                <list editable="bottom">
                                    <field name="sequence" widget="handle"/>
                                    <field name="name"/>
                                    <field name="quantity"/>
                                    <field name="price_unit"/>
                                    <field name="subtotal"/>
                                </list>
                            </field>
                            <group class="oe_subtotal_footer">
                                <field name="total"/>
                            </group>
                        </page>
                        <page string="Notes" name="notes">
                            <field name="description"
                                   placeholder="Add notes..."/>
                        </page>
                    </notebook>
                </sheet>
                <chatter/>
            </form>
        </field>
    </record>

    <!-- ==================== LIST ==================== -->
    <record id="{module_model}_list" model="ir.ui.view">
        <field name="name">{module.model.name}.list</field>
        <field name="model">{module.model.name}</field>
        <field name="arch" type="xml">
            <list multi_edit="1"
                  decoration-muted="state == 'cancelled'"
                  decoration-info="state == 'draft'">
                <field name="sequence" widget="handle"/>
                <field name="name"/>
                <field name="partner_id"/>
                <field name="total" sum="Total"/>
                <field name="state" widget="badge"
                       decoration-success="state == 'done'"
                       decoration-info="state == 'draft'"
                       decoration-warning="state == 'confirmed'"
                       decoration-danger="state == 'cancelled'"/>
                <field name="company_id" optional="hide"
                       groups="base.group_multi_company"/>
            </list>
        </field>
    </record>

    <!-- ==================== SEARCH ==================== -->
    <record id="{module_model}_search" model="ir.ui.view">
        <field name="name">{module.model.name}.search</field>
        <field name="model">{module.model.name}</field>
        <field name="arch" type="xml">
            <search>
                <field name="name"/>
                <field name="partner_id"/>
                <filter name="filter_draft" string="Draft"
                        domain="[('state', '=', 'draft')]"/>
                <filter name="filter_confirmed" string="Confirmed"
                        domain="[('state', '=', 'confirmed')]"/>
                <filter name="filter_done" string="Done"
                        domain="[('state', '=', 'done')]"/>
                <separator/>
                <filter name="filter_archived" string="Archived"
                        domain="[('active', '=', False)]"/>
                <group expand="0" string="Group By">
                    <filter name="groupby_state" string="State"
                            context="{'group_by': 'state'}"/>
                    <filter name="groupby_partner" string="Partner"
                            context="{'group_by': 'partner_id'}"/>
                </group>
            </search>
        </field>
    </record>

    <!-- ==================== ACTION ==================== -->
    <record id="{module_model}_action" model="ir.actions.act_window">
        <field name="name">{Model Human Name}</field>
        <field name="res_model">{module.model.name}</field>
        <field name="view_mode">list,form</field>
        <field name="search_view_id" ref="{module_model}_search"/>
        <field name="context">{'search_default_filter_draft': 1}</field>
        <field name="help" type="html">
            <p class="o_view_nocontent_smiling_face">
                Create your first record
            </p>
        </field>
    </record>

    <!-- ==================== MENU ==================== -->
    <menuitem id="menu_{module}_root"
              name="{Module Name}"
              web_icon="{module_name},static/description/icon.png"
              sequence="100"/>
    <menuitem id="menu_{module}_main"
              name="{Section}"
              parent="menu_{module}_root"
              sequence="10"/>
    <menuitem id="menu_{model}"
              name="{Model Name Plural}"
              parent="menu_{module}_main"
              action="{module_model}_action"
              sequence="10"/>

</odoo>
```

## Security Groups + Record Rules

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- Copyright 2025 {Company}
     License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl). -->
<odoo>
    <record id="module_category_{module}" model="ir.module.category">
        <field name="name">{Module Name}</field>
        <field name="sequence">100</field>
    </record>

    <record id="group_user" model="res.groups">
        <field name="name">User</field>
        <field name="category_id" ref="module_category_{module}"/>
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <record id="group_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="category_id" ref="module_category_{module}"/>
        <field name="implied_ids" eval="[(4, ref('group_user'))]"/>
    </record>

    <!-- Multi-company rule -->
    <record id="rule_{model}_company" model="ir.rule">
        <field name="name">{Model}: multi-company</field>
        <field name="model_id" ref="model_{model_underscored}"/>
        <field name="domain_force">
            ['|', ('company_id', '=', False),
             ('company_id', 'in', company_ids)]
        </field>
    </record>
</odoo>
```

## Existing Model Inheritance

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

from odoo import fields, models


class ResPartner(models.Model):
    _inherit = "res.partner"

    {new_field} = fields.Char(string="{Field Label}")
    {relation_ids} = fields.One2many(
        comodel_name="{module.model.name}",
        inverse_name="partner_id",
        string="{Records}",
    )
```

## Existing View Inheritance

```xml
<record id="view_{inherited_model}_form_inherit_{module}" model="ir.ui.view">
    <field name="name">{inherited.model}.form.inherit.{module_name}</field>
    <field name="model">{inherited.model}</field>
    <field name="inherit_id" ref="{base_module}.{original_view_id}"/>
    <field name="arch" type="xml">
        <field name="{existing_field}" position="after">
            <field name="{new_field}"/>
        </field>
        <xpath expr="//notebook" position="inside">
            <page string="{Tab Name}" name="{tab_name}">
                <field name="{relation_ids}"/>
            </page>
        </xpath>
    </field>
</record>
```
