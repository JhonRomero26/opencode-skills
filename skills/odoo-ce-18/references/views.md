# Views Reference — Odoo v18 CE

## Critical Changes in v18

```xml
<!-- ✅ v18: direct attributes -->
<button invisible="state != 'draft'"/>
<field name="amount" readonly="state != 'draft'"/>
<field name="partner_id" required="state == 'confirmed'"/>
<field name="internal" column_invisible="parent.is_public"/>

<!-- ❌ OBSOLETE — do not use in v18 -->
<button attrs="{'invisible': [('state', '!=', 'draft')]}"/>

<!-- ✅ v18: <list> replaces <tree> -->
<list multi_edit="1">...</list>

<!-- ✅ v18: simplified chatter -->
<chatter/>
<!-- instead of <div class="oe_chatter"><field name="message_ids"/>...</div> -->

<!-- ✅ Buttons must call public methods only -->
<button name="action_open_form" type="object"/>

<!-- ❌ Never call private methods from XML -->
<button name="_action_open_form" type="object"/>
```

## Button Rule

In Odoo XML views, `type="object"` buttons resolve against model methods.
Always expose a public `action_*` method for anything triggered from the UI.
Private helper methods (prefixed with `_`) are for internal Python use only.

## View Inheritance

```xml
<!-- Add a field after another field -->
<field name="{existing_field}" position="after">
    <field name="{new_field}"/>
</field>

<!-- Add a field before another field -->
<field name="{existing_field}" position="before">
    <field name="{new_field}"/>
</field>

<!-- Replace a field -->
<field name="{field}" position="replace">
    <field name="{field}" widget="many2many_tags"/>
</field>

<!-- Add a tab to the notebook -->
<xpath expr="//notebook" position="inside">
    <page string="{Tab Name}" name="{tab_name}">
        <field name="{field}"/>
    </page>
</xpath>

<!-- Add a button to the header -->
<xpath expr="//header" position="inside">
    <button name="action_custom" string="Custom" type="object"/>
</xpath>

<!-- Add to the button_box -->
<div name="button_box" position="inside">
    <button name="action_view_related"
            type="object" class="oe_stat_button"
            icon="fa-list">
        <field name="related_count" widget="statinfo"
               string="Related"/>
    </button>
</div>

<!-- Hide an existing field -->
<field name="{field}" position="attributes">
    <attribute name="invisible">1</attribute>
</field>
```

## Common v18 Widgets

| Widget            | Use                        |
| ----------------- | -------------------------- |
| `badge`           | Status colors in list      |
| `statusbar`       | Status bar in form header  |
| `many2many_tags`  | Inline tags for M2M        |
| `handle`          | Drag-and-drop for sequence |
| `monetary`        | Currency with symbol       |
| `percentage`      | Percentage                 |
| `progressbar`     | Progress bar               |
| `image`           | Image preview              |
| `html`            | Rich HTML editor           |
| `statinfo`        | Stat button counter        |
| `selection_badge` | Selection as badges        |
| `radio`           | Selection as radio buttons |
| `priority`        | Priority stars             |
| `color_picker`    | Color picker               |

## Kanban View

```xml
<record id="{module_model}_kanban" model="ir.ui.view">
    <field name="name">{module.model.name}.kanban</field>
    <field name="model">{module.model.name}</field>
    <field name="arch" type="xml">
        <kanban default_group_by="state" class="o_kanban_small_column">
            <templates>
                <t t-name="card">
                    <field name="name" class="fw-bold"/>
                    <field name="partner_id"/>
                    <field name="total" widget="monetary"/>
                </t>
            </templates>
        </kanban>
    </field>
</record>
```

Add `kanban` to the action `view_mode`: `<field name="view_mode">list,kanban,form</field>`
