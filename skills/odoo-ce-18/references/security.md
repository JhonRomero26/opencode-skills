# Security Reference — Odoo v18 OCA

Guiding principle: **least privilege**. Each user should access ONLY what they need to perform their role. Never grant access "just in case".

## Role Design - Methodology

Before writing code, analyze the module and answer:

1. **Who uses this module?** -> Define the groups
2. **What can each role do?** -> Define ACLs (CRUD per model)
3. **Which records does each role see?** -> Define record rules
4. **Which fields does each role see?** -> Define `groups=` on fields
5. **Which actions can each role execute?** -> Define `groups=` on buttons/menus

### Typical Group Hierarchy

```
base.group_user (internal Odoo user)
  └── {module}.group_user (module user)
      └── {module}.group_validator (validator / agent)
          └── {module}.group_manager (module manager)
```

Each group inherits the permissions of the previous one via `implied_ids`. The manager can do EVERYTHING the validator can, and the validator can do EVERYTHING the base user can.

Three levels are not always needed. Simple modules may have only user + manager. Complex modules may have 4+ levels (user -> agent -> supervisor -> manager -> admin).

---

## Layer 1 - Groups and Category

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- Copyright 2025 {Company}
     License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl). -->
<odoo>

    <!-- Module category (shown in Settings > Users) -->
    <record id="module_category_{module}" model="ir.module.category">
        <field name="name">{Module Human Name}</field>
        <field name="description">
            Access rights for {Module} management.
        </field>
        <field name="sequence">50</field>
    </record>

    <!-- Level 1: Base user -->
    <record id="group_user" model="res.groups">
        <field name="name">User</field>
        <field name="comment">
            Can create and manage own records.
            Cannot delete or see other users' records.
        </field>
        <field name="category_id" ref="module_category_{module}"/>
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <!-- Level 2: Validator / Agent (OPTIONAL - only if the logic requires it) -->
    <record id="group_validator" model="res.groups">
        <field name="name">Validator</field>
        <field name="comment">
            Can see all records, validate/approve,
            and access intermediate features.
        </field>
        <field name="category_id" ref="module_category_{module}"/>
        <field name="implied_ids" eval="[(4, ref('group_user'))]"/>
    </record>

    <!-- Level 3: Manager -->
    <record id="group_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="comment">
            Full access: create, read, update, delete.
            Can access configuration and reports.
        </field>
        <field name="category_id" ref="module_category_{module}"/>
        <field name="implied_ids" eval="[(4, ref('group_validator'))]"/>
    </record>

</odoo>
```

**Note**: If there is no intermediate level (validator), the manager inherits directly from user:

```xml
<field name="implied_ids" eval="[(4, ref('group_user'))]"/>
```

---

## Layer 2 - ACLs (ir.model.access.csv)

Each row = (model x group x CRUD permissions). EACH model needs at least one line per group.

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
# --- Main model ---
access_{model}_user,{module.model} user,model_{model_under},{module}.group_user,1,1,1,0
access_{model}_validator,{module.model} validator,model_{model_under},{module}.group_validator,1,1,1,0
access_{model}_manager,{module.model} manager,model_{model_under},{module}.group_manager,1,1,1,1
# --- Line model (child of the main model) ---
access_{model}_line_user,{module.model.line} user,model_{model_line_under},{module}.group_user,1,1,1,0
access_{model}_line_manager,{module.model.line} manager,model_{model_line_under},{module}.group_manager,1,1,1,1
# --- Tag / category model ---
access_{model}_tag_user,{module.model.tag} user,model_{model_tag_under},{module}.group_user,1,0,0,0
access_{model}_tag_manager,{module.model.tag} manager,model_{model_tag_under},{module}.group_manager,1,1,1,1
# --- Wizard (only the group that uses it) ---
access_{wizard}_validator,{module.wizard} validator,model_{wizard_under},{module}.group_validator,1,1,1,0
access_{wizard}_manager,{module.wizard} manager,model_{wizard_under},{module}.group_manager,1,1,1,1
# --- Configuration model (manager only) ---
access_{config}_manager,{module.config} manager,model_{config_under},{module}.group_manager,1,1,1,1
```

### Permission Guide by Model Type

| Model type                       | User    | Validator | Manager |
| -------------------------------- | ------- | --------- | ------- |
| Main model (transactional)       | R W C - | R W C -   | R W C U |
| Lines (child O2M)                | R W C - | R W C -   | R W C U |
| Tags / categories (light config) | R - - - | R - - -   | R W C U |
| Configuration model              | - - - - | R - - -   | R W C U |
| Wizard (bulk action)             | - - - - | R W C -   | R W C U |
| Wizard (user-owned)              | R W C - | R W C -   | R W C U |
| Report model (BI/analytics)      | R - - - | R - - -   | R - - - |

**R**=read, **W**=write, **C**=create, **U**=unlink (delete)

---

## Layer 3 - Record Rules (which RECORDS each group sees)

```xml
<!-- ======================== MULTI-COMPANY ======================== -->
<!-- ALWAYS if the model has company_id -->
<record id="rule_{model}_company" model="ir.rule">
    <field name="name">{Model}: multi-company</field>
    <field name="model_id" ref="model_{model_under}"/>
    <field name="domain_force">
        ['|', ('company_id', '=', False),
         ('company_id', 'in', company_ids)]
    </field>
</record>

<!-- ======================== OWNERSHIP ======================== -->
<!-- Base user only sees their own records -->
<record id="rule_{model}_user_own" model="ir.rule">
    <field name="name">{Model}: user sees own records</field>
    <field name="model_id" ref="model_{model_under}"/>
    <field name="domain_force">
        [('user_id', '=', user.id)]
    </field>
    <field name="groups" eval="[(4, ref('group_user'))]"/>
</record>

<!-- Validator sees all records (no restriction) -->
<record id="rule_{model}_validator_all" model="ir.rule">
    <field name="name">{Model}: validator sees all</field>
    <field name="model_id" ref="model_{model_under}"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('group_validator'))]"/>
</record>

<!-- Manager sees all (inherited from validator, but explicit for clarity) -->
<record id="rule_{model}_manager_all" model="ir.rule">
    <field name="name">{Model}: manager sees all</field>
    <field name="model_id" ref="model_{model_under}"/>
    <field name="domain_force">[(1, '=', 1)]</field>
    <field name="groups" eval="[(4, ref('group_manager'))]"/>
</record>
```

### Record Rule Variants

```xml
<!-- DEPARTMENT: user sees records from their department -->
<record id="rule_{model}_department" model="ir.rule">
    <field name="name">{Model}: user sees own department</field>
    <field name="model_id" ref="model_{model_under}"/>
    <field name="domain_force">
        ['|', ('department_id', '=', False),
         ('department_id', 'in', user.department_ids.ids)]
    </field>
    <field name="groups" eval="[(4, ref('group_user'))]"/>
</record>

<!-- TEAM: user sees records from their sales team -->
<record id="rule_{model}_team" model="ir.rule">
    <field name="name">{Model}: user sees own team</field>
    <field name="model_id" ref="model_{model_under}"/>
    <field name="domain_force">
        ['|', ('team_id', '=', False),
         ('team_id', 'in', user.sale_team_ids.ids)]
    </field>
    <field name="groups" eval="[(4, ref('group_user'))]"/>
</record>

<!-- PARTNER: portal user only sees records linked to their partner -->
<record id="rule_{model}_portal" model="ir.rule">
    <field name="name">{Model}: portal user sees own</field>
    <field name="model_id" ref="model_{model_under}"/>
    <field name="domain_force">
        [('partner_id', '=', user.partner_id.id)]
    </field>
    <field name="groups" eval="[(4, ref('base.group_portal'))]"/>
</record>

<!-- STATE: base user CANNOT see records in state 'secret' -->
<record id="rule_{model}_hide_secret" model="ir.rule">
    <field name="name">{Model}: hide secret from users</field>
    <field name="model_id" ref="model_{model_under}"/>
    <field name="domain_force">
        [('state', '!=', 'secret')]
    </field>
    <field name="groups" eval="[(4, ref('group_user'))]"/>
</record>
```

**Important about record rules with groups:**

- Rules WITHOUT a group -> apply to EVERYONE (global AND)
- Rules WITH a group -> apply only to that group (OR between rules in the same group)
- If a user belongs to multiple groups, the rules are combined with OR

---

## Layer 4 - Fields Restricted by Group

In the model definition, use `groups=` to hide sensitive fields:

```python
class ModelName(models.Model):
    _name = "{module.model.name}"

    # --- Fields visible to everyone ---
    name = fields.Char(required=True)
    state = fields.Selection([...])

    # --- Fields for validator+ only ---
    internal_notes = fields.Text(
        groups="{module_name}.group_validator",
        help="Only visible to validators and managers.",
    )
    priority = fields.Selection(
        selection=[("low", "Low"), ("medium", "Medium"), ("high", "High")],
        groups="{module_name}.group_validator",
    )

    # --- Fields for manager only ---
    internal_cost = fields.Float(
        groups="{module_name}.group_manager",
        digits="Product Price",
    )
    margin = fields.Float(
        compute="_compute_margin",
        groups="{module_name}.group_manager",
    )
    config_field = fields.Boolean(
        groups="{module_name}.group_manager",
    )
```

**Effect of `groups=`:**

- The field does NOT appear in views for users without the group
- The field is NOT accessible via API/RPC for users without the group
- The ORM filters it automatically - you do not need extra `invisible` in the view

---

## Layer 5 - Buttons, Menus, and Actions by Group

### Form Buttons

```xml
<!-- Only validator+ can approve -->
<button name="action_approve" string="Approve"
        type="object" class="btn-primary"
        invisible="state != 'pending'"
        groups="{module_name}.group_validator"/>

<!-- Only manager can force close -->
<button name="action_force_close" string="Force Close"
        type="object" class="btn-danger"
        groups="{module_name}.group_manager"/>

<!-- Only manager can delete from the form -->
<button name="unlink" string="Delete"
        type="object" class="btn-link text-danger"
        groups="{module_name}.group_manager"
    confirm="Are you sure you want to delete this record?"/>
```

### Menus

```xml
<!-- Main menu: all users of the module -->
<menuitem id="menu_root" name="{Module}"
          groups="{module_name}.group_user"/>

<!-- Daily operations: everyone -->
<menuitem id="menu_operations" name="Operations"
          parent="menu_root"
          groups="{module_name}.group_user"/>

<!-- Reports: validator+ -->
<menuitem id="menu_reports" name="Reports"
          parent="menu_root"
          groups="{module_name}.group_validator"/>

<!-- Configuration: manager only -->
<menuitem id="menu_config" name="Configuration"
          parent="menu_root"
          groups="{module_name}.group_manager"/>
```

### Server Actions (Mass Actions)

```xml
<!-- Only validator+ can mass approve -->
<record id="action_mass_approve" model="ir.actions.server">
    <field name="name">Mass Approve</field>
    <field name="model_id" ref="model_{model_under}"/>
    <field name="binding_model_id" ref="model_{model_under}"/>
    <field name="binding_view_types">list</field>
    <field name="groups_id" eval="[(4, ref('group_validator'))]"/>
    <field name="state">code</field>
    <field name="code">records.action_approve()</field>
</record>
```

### Wizard - Access Restriction

```xml
<!-- Reassignment wizard: manager only -->
<record id="action_wizard_reassign" model="ir.actions.act_window">
    <field name="name">Reassign Records</field>
    <field name="res_model">{module.reassign.wizard}</field>
    <field name="view_mode">form</field>
    <field name="target">new</field>
    <field name="binding_model_id" ref="model_{model_under}"/>
    <field name="binding_view_types">list,form</field>
    <field name="groups_id" eval="[(4, ref('group_manager'))]"/>
</record>
```

---

## Layer 6 - Security Tests by Role

```python
# tests/common.py - Setup with one user per role
@classmethod
def setUpClass(cls):
    super().setUpClass()
    cls.env = cls.env(context=dict(cls.env.context, tracking_disable=True))

    # One user per group
    cls.user_basic = cls.env["res.users"].create({
        "name": "Basic User",
        "login": "basic@test.com",
        "groups_id": [(6, 0, [
            cls.env.ref("{module_name}.group_user").id,
        ])],
    })
    cls.user_validator = cls.env["res.users"].create({
        "name": "Validator User",
        "login": "validator@test.com",
        "groups_id": [(6, 0, [
            cls.env.ref("{module_name}.group_validator").id,
        ])],
    })
    cls.user_manager = cls.env["res.users"].create({
        "name": "Manager User",
        "login": "manager@test.com",
        "groups_id": [(6, 0, [
            cls.env.ref("{module_name}.group_manager").id,
        ])],
    })
```

```python
# tests/test_security.py
from odoo.exceptions import AccessError
from odoo.tests import tagged
from .common import TestCommon


@tagged("post_install", "-at_install")
class TestSecurity(TestCommon):

    # ===== ACL: which OPERATIONS each role can perform =====

    def test_basic_user_can_create(self):
        """Basic user can create records."""
        record = self.env["{model}"].with_user(self.user_basic).create({
            "name": "User Record",
            "partner_id": self.partner.id,
        })
        self.assertTrue(record.id)

    def test_basic_user_cannot_unlink(self):
        """Basic user cannot delete records."""
        with self.assertRaises(AccessError):
            self.record.with_user(self.user_basic).unlink()

    def test_manager_can_unlink(self):
        """Manager can delete draft records."""
        self.record.with_user(self.user_manager).unlink()

    # ===== Record Rules: which RECORDS each role sees =====

    def test_basic_user_sees_own_records_only(self):
        """Basic user only sees records they created."""
        own_record = self.env["{model}"].with_user(self.user_basic).create({
            "name": "My Record",
            "partner_id": self.partner.id,
        })
        visible = self.env["{model}"].with_user(self.user_basic).search([])
        self.assertIn(own_record.id, visible.ids)
        # Record created by admin should NOT be visible
        self.assertNotIn(self.record.id, visible.ids)

    def test_validator_sees_all_records(self):
        """Validator can see all records."""
        visible = self.env["{model}"].with_user(self.user_validator).search([])
        self.assertIn(self.record.id, visible.ids)

    def test_multi_company_isolation(self):
        """User in company B cannot see records from company A."""
        company_b = self.env["res.company"].create({"name": "Company B"})
        user_b = self.env["res.users"].create({
            "name": "User B",
            "login": "userb@test.com",
            "company_id": company_b.id,
            "company_ids": [(6, 0, [company_b.id])],
            "groups_id": [(6, 0, [
                self.env.ref("{module_name}.group_user").id,
            ])],
        })
        visible = self.env["{model}"].with_user(user_b).search([])
        self.assertNotIn(self.record.id, visible.ids)

    # ===== Field-level: which FIELDS each role can access =====

    def test_basic_user_cannot_read_cost(self):
        """Basic user cannot read internal_cost field."""
        with self.assertRaises(AccessError):
            self.record.with_user(self.user_basic).read(["internal_cost"])

    def test_manager_can_read_cost(self):
        """Manager can read internal_cost field."""
        vals = self.record.with_user(self.user_manager).read(["internal_cost"])
        self.assertTrue(vals)

    def test_basic_user_cannot_write_priority(self):
        """Basic user cannot write priority field."""
        with self.assertRaises(AccessError):
            self.record.with_user(self.user_basic).write({"priority": "high"})

    def test_validator_can_write_priority(self):
        """Validator can write priority field."""
        self.record.with_user(self.user_validator).write({"priority": "high"})
        self.assertEqual(self.record.priority, "high")

    # ===== Actions: which ACTIONS each role can execute =====

    def test_basic_user_cannot_approve(self):
        """Basic user cannot execute approve action."""
        self.record.state = "pending"
        with self.assertRaises(AccessError):
            self.record.with_user(self.user_basic).action_approve()

    def test_validator_can_approve(self):
        """Validator can execute approve action."""
        self.record.state = "pending"
        self.record.with_user(self.user_validator).action_approve()
        self.assertEqual(self.record.state, "approved")
```

---

    ## Common Role Patterns by Module Type

    ### Tickets / Helpdesk Module

    | Group   | Permissions                                                  |
    | ------- | ------------------------------------------------------------ |
    | User    | Create own tickets, view/edit their own, add messages       |
    | Agent   | See all tickets, assign to self, change state/priority      |
    | Manager | Full CRUD, reassign, dashboard, configuration, delete      |

    ### Approvals Module

    | Group     | Permissions                                                  |
    | --------- | ------------------------------------------------------------ |
    | Requester | Create requests, view their own, cancel their own in draft  |
    | Approver  | See assigned requests, approve/reject                       |
    | Manager   | Full CRUD, configure workflows, reports, force approval     |

    ### Inventory / Warehouse Module

    | Group       | Permissions                                                        |
    | ----------- | ------------------------------------------------------------------ |
    | User        | View stock, create transfers, confirm own picking                  |
    | Responsible | Validate transfers, inventory adjustments, view reports           |
    | Manager     | Full CRUD, configure locations/rules, force transfers              |

    ### Accounting / Finance Module

    | Group        | Permissions                                                     |
    | ------------ | --------------------------------------------------------------- |
    | Invoice User | Create/edit draft invoices, view their own                      |
    | Accountant   | Validate invoices, reconcile, view reports, access journals    |
    | Advisor      | All of the above + accounting configuration + multi-company    |

---

    ## Common Mistakes to Avoid

```python
# ❌ DO NOT give public access to internal models
# This allows any anonymous user to read the model
access_model_public,model public,model_my_model,base.group_public,1,0,0,0

# ✅ At minimum, use group_user for internal models
access_model_user,model user,model_my_model,{module}.group_user,1,1,1,0

# ❌ DO NOT forget ACLs for child O2M models
# If the main model has ACLs but the line model does not, creating lines raises AccessError

# ❌ DO NOT use sudo() to "solve" permission problems
def action_confirm(self):
    self.sudo().write({"state": "confirmed"})  # ❌ Hides a security bug

# ✅ Fix the permissions correctly
def action_confirm(self):
    self.write({"state": "confirmed"})  # ✅ The user must have real permission

# ❌ DO NOT create overly restrictive record rules without a group
# Rules without a group apply to EVERYONE, including admin
<field name="domain_force">[('user_id', '=', user.id)]</field>
# ↑ This prevents ANY user from seeing other users' records, even admin

# ✅ Assign the restrictive rule to a specific group
<field name="groups" eval="[(4, ref('group_user'))]"/>
# And create another permissive rule for manager/admin
```
