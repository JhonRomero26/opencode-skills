---
name: odoo-ce-18
description: >
  Complete scaffolding for Odoo v18 modules with OCA standards.
  Generates a certifiable module structure: models, views, security, tests, README.
  Trigger: When the user asks to create an Odoo module, add a model, create a view,
  write tests, migrate a module, or handle any Odoo 18 development task.
  Also when they mention OCA, ERP, __manifest__, TransactionCase, HttpCase,
  ir.model.access, OWL, controller, wizard, or simply "Odoo".
  ALWAYS use this skill for any Odoo code — even small modifications.
license: AGPL-3.0
metadata:
  author: gentleman-programming
  version: "1.1"
---

## Agent Role

You are a senior Odoo v18 developer certified by OCA. When the user asks to create or modify a module, you generate ALL required files at once, ready to install. Never generate partial files or ask for confirmation file by file.

## References (read before generating)

| File                       | When to read                                                |
| -------------------------- | ----------------------------------------------------------- |
| `references/templates.md`  | ALWAYS — contains the base templates for each file          |
| `references/testing.md`    | ALWAYS — mandatory OCA test patterns                        |
| `references/views.md`      | When creating or modifying XML views                        |
| `references/security.md`   | When creating security rules and access controls            |
| `references/owl.md`        | When creating JS frontend components                        |
| `references/migrations.md` | ALWAYS when there are version changes with breaking changes |
| `references/advanced.md`   | Wizards, controllers, reports, cron, server actions         |
| `references/localization.md`| ALWAYS when creating/modifying localization (l10n) modules (e.g. EC, PE) |

---

## Scaffolding Flow (New Module)

When the user asks for a new module:

### Step 1 — Capture Requirements (max 3 questions)

Infer from the user's message:

- Technical module name (`snake_case`, singular)
- Required models (name, main fields, relationships)
- Features (workflow/states, reports, wizards, portal)

If critical information is missing, ask ONCE and be specific. Do not ask unnecessary confirmations.

### Step 2 — Generate Complete Structure

Create ALL these files in order. Do not omit any:

```
{module_name}/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── {model_name}.py          # One file per model
├── views/
│   └── {model_name}_views.xml   # Form + List + Search + Action + Menu
├── security/
│   ├── {module_name}_security.xml  # Groups + Record Rules
│   └── ir.model.access.csv         # ACLs
├── data/
│   └── (if there are sequences, cron jobs, master data)
├── demo/
│   └── {model_name}_demo.xml    # Demo data ALWAYS
├── tests/
│   ├── __init__.py
│   ├── common.py                # Shared setup
│   └── test_{model_name}.py     # COMPLETE tests
├── i18n/                        # Translations
│   ├── {module_name}.pot        # Base template (generate via CLI)
│   └── es.po                    # specific languages (e.g. Spanish)
├── static/
│   └── description/
│       └── icon.png             # (indicate that it is missing)
├── migrations/                  # IF there are breaking changes
│   └── 18.0.X.Y.Z/
│       ├── pre-migrate.py       # Before ORM update
│       ├── post-migrate.py      # After ORM update
│       └── pre-migrate.sql      # Heavy SQL (optional)
└── readme/
    ├── DESCRIPTION.md
    ├── USAGE.md
    ├── ROADMAP.md               # (Optional)
    └── CONTRIBUTORS.md
```

---

## Existing Module Modification Flow

When the user asks to MODIFY a module that is already installed in production:

### Step M1 — Analyze the Change

Before touching code, classify each requested change:

```
Does the change affect the DB schema?
├── NO (only Python logic, views, security)
│   └── Modify files + bump version Z (patch)
│       "18.0.1.0.0" → "18.0.1.0.1"
│
├── YES — Additive (new field, new model)
│   └── Modify files + bump version Y (minor)
│       "18.0.1.0.0" → "18.0.1.1.0"
│       Does NOT require migration script (ORM auto-creates)
│
└── YES — Destructive (rename, remove, change type)
    └── Modify files + bump version Y/X + GENERATE MIGRATION
        "18.0.1.0.0" → "18.0.1.1.0" (minor) or "18.0.2.0.0" (major)
        MANDATORY: generate migrations/{new_version}/
```

### Step M2 — Automatically Generate Migration

If the change is destructive, generate ALONGSIDE the modified files:

1. Increment `version` in `__manifest__.py`
2. Create directory `migrations/{new_version}/`
3. Generate `pre-migrate.py` if you need to act BEFORE the ORM (rename, save data)
4. Generate `post-migrate.py` if you need to act AFTER the ORM (transform data)
5. Generate `pre-migrate.sql` if there is heavy SQL or DROP CONSTRAINT
6. Add migration tests in `tests/test_migration.py`
7. Document in `readme/HISTORY.md`

**Read `references/migrations.md` for complete templates of each scenario.**

### Step M3 — Full Modification Example

User asks: _"Rename the field `client_name` to `customer_id` (Many2one to res.partner) in the model `sale.custom.order`"_

Files to generate/modify:

```
sale_custom_order/
├── __manifest__.py                    # version: "18.0.1.0.0" → "18.0.1.1.0"
├── models/
│   └── sale_custom_order.py           # Remove client_name, add customer_id
├── views/
│   └── sale_custom_order_views.xml    # Replace client_name → customer_id
├── migrations/
│   └── 18.0.1.1.0/
│       ├── pre-migrate.py             # Save client_name in temporary column
│       └── post-migrate.py            # Look up partner by name, assign customer_id
├── tests/
│   ├── test_sale_custom_order.py      # Update existing tests
│   └── test_migration.py             # Test that verifies the migration
└── readme/
    └── HISTORY.rst                    # Document breaking change
```

---

## Step 3 — Verify Quality

Before delivering, verify mentally:

- [ ] Every `.py` and `.xml` file has an AGPL-3 copyright header
- [ ] `__manifest__.py` has version `18.0.1.0.0` and `"Mass EC"` in author
- [ ] All models have `_description`
- [ ] Fields are ordered: primitives → relational → computed
- [ ] Methods follow OCA order (see section below)
- [ ] `ir.model.access.csv` covers ALL models (including TransientModel)
- [ ] Multi-company record rules if there is `company_id`
- [ ] Tests cover: CRUD, constraints, actions, **security per role**, Form simulation
- [ ] **Security tests**: each role verifies allowed AND denied access
- [ ] `invisible` (NOT `attrs`) in v18 views
- [ ] Demo data is realistic
- [ ] No `# -*- coding: utf-8 -*-`
- [ ] No `import *`
- [ ] No `self.env.cr.commit()`
- [ ] If there are breaking changes → `migrations/` with pre/post-migrate + SQL if heavy
- [ ] Version in `__manifest__` incremented correctly (bump Z, Y, or X)
- [ ] Execute pre-commit and fix any issues

---

## Mandatory OCA Standards

### Copyright Header — EACH file

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).
```

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- Copyright 2025 {Company}
     License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl). -->
```

### **manifest**.py

```python
{
    "name": "Human-Readable Module Name",
    "summary": "One-line description",
    "version": "18.0.1.0.0",
    "category": "Category",
    "website": "https://github.com/mass-ec/{repo}",
    "author": "{Company}, Mass EC",
    "maintainers": ["{github_user}"],
    "license": "AGPL-3",
    "application": False,
    "installable": True,
    "depends": ["base"],
    "data": [
        "security/{module}_security.xml",
        "security/ir.model.access.csv",
        "views/{model}_views.xml",
    ],
    "demo": ["demo/{model}_demo.xml"],
    "development_status": "Alpha",
}
```

### Method Order in Model (MANDATORY)

1. `_name`, `_description`, `_inherit`, `_order`, `_rec_name`
2. `_default_*` methods
3. Fields: Boolean → Char/Text → Integer/Float → Date → Selection → Many2one → One2many → Many2many → Computed
4. `_sql_constraints`
5. `@api.constrains` (`_check_*`)
6. `_compute_*`
7. `_onchange_*`
8. CRUD: `create`, `write`, `unlink`, `copy`
9. `action_*` (public methods only; XML buttons must never target `_private` methods)
10. `_private_methods`

### OCA Python Style

```python
# ✅ Ordered imports: odoo → odoo.exceptions → third parties
from odoo import api, fields, models, _
from odoo.exceptions import UserError, ValidationError

# ✅ Recordset API
confirmed = records.filtered(lambda r: r.state == "confirmed")
names = records.mapped("name")

# ✅ User-translatable strings
raise UserError(_("Cannot delete record in state '%s'.") % record.state)

# ✅ Logger (no f-strings, no _())
_logger.info("Processing %d records", len(records))

# ❌ NEVER
# browse() in loops — use search() + mapped()
# Hardcoded IDs — use self.env.ref("module.xml_id")
# self.env.cr.commit()
# raw SQL for simple lookups
# any / import *
```

### v18 Views — Critical Changes

```xml
<!-- ✅ v18: direct invisible -->
<button name="action_confirm" invisible="state != 'draft'"/>
<field name="amount" readonly="state != 'draft'"/>
<field name="partner_id" required="state == 'confirmed'"/>

<!-- ❌ OBSOLETE in v18 -->
<button attrs="{'invisible': [('state', '!=', 'draft')]}"/>

<!-- ✅ v18: <list> replaces <tree> -->
<list multi_edit="1">...</list>

<!-- ✅ v18: simplified chatter -->
<chatter/>
```

### XML Button Rule

- Buttons with `type="object"` must call **public** model methods.
- Use `action_*` names for anything triggered from XML views.
- Never use `_action_*` or any other private method name as a button target.
- If a view needs a private helper internally, wrap it in a public `action_*` method.

### Mandatory Minimum Security

For EACH model:

1. `ir.model.access.csv` — at minimum a user group (CRUD without unlink) + manager (full CRUD)
2. If it has `company_id` → multi-company record rule
3. If it is sensitive → record rules by group

### Mandatory Tests (per model)

Each model MUST have tests that cover:

1. **Creation** with correct default values
2. **Constraints** — each `@api.constrains` and `_sql_constraints`
3. **Compute** — each computed field
4. **Actions** — each `action_*` (happy path + error path)
5. **CRUD guards** — `unlink` blocked in non-draft states, etc.
6. **Security per role** — FOR EACH defined group:
   - Base user: can create/read their own, CANNOT delete or see others'
   - Validator/agent (if exists): can read all, CANNOT delete
   - Manager: full CRUD
   - Record rules: user from company A CANNOT see records from company B
   - Fields with `groups=`: base user CANNOT write sensitive fields
7. **Form simulation** — onchanges fire correctly
8. **Multi-company** — isolation between companies (if applicable)

Standard tag: `@tagged("post_install", "-at_install")`

### Migrations — When to Generate Scripts

When modifying an existing module, evaluate if the change breaks the database. If it DOES break → generate migration scripts MANDATORILY along with the change. Read `references/migrations.md` for complete templates.

**Changes that REQUIRE migration:**

| Change                                 | Script type                                                 |
| -------------------------------------- | ----------------------------------------------------------- |
| Rename field                           | `pre-migrate.py` (before ORM drops the old column)          |
| Rename model / table                   | `pre-migrate.py`                                            |
| Remove field with valuable data        | `pre-migrate.py` (save data first)                          |
| Change field type (Char→Many2one, etc) | `pre-migrate.py` + `post-migrate.py`                        |
| Change Selection values                | `post-migrate.py`                                           |
| Merge/split models                     | `pre-migrate.sql` + `post-migrate.py`                       |
| Change inheritance structure           | `pre-migrate.py`                                            |
| Migrate massive data (>100K records)   | `pre-migrate.sql` (direct SQL, no ORM)                      |
| Change constraints (unique, check)     | `pre-migrate.sql` (DROP constraint first)                   |
| Move field from one model to another   | `pre-migrate.py` (temp column) + `post-migrate.py`          |
| Compute non-stored → stored            | `post-migrate.py` (force recompute)                         |
| Many2one → Many2many (or vice versa)   | `pre-migrate.py` (save old) + `post-migrate.py` (build rel) |

**Changes that DO NOT require migration:**

- Add new field (ORM creates it automatically)
- Add new model
- Change views/menus/actions (reloaded on update)
- Change Python logic (no schema change)
- Add security group / record rule

**Version rule**: When creating a migration, ALWAYS increment the version in `__manifest__.py` and the `migrations/` directory must match exactly with the new version.

```
# Example: renamed field
__manifest__.py  →  "version": "18.0.1.1.0"  (was 18.0.1.0.0)
migrations/18.0.1.1.0/
├── pre-migrate.py    # Rename SQL column before ORM acts
└── post-migrate.py   # (optional) Post-update cleanup
```

### Translations (i18n)

Always consider translations for modules with user-facing strings or views. Generate the `i18n/` directory.
- **Generate via CLI**: Instead of writing `.pot`/`.po` files manually, suggest or run the Odoo CLI command to export translations when requested.
  ```bash
  odoo-bin -c path/to/odoo.conf -d <dbname> --i18n-export=path/to/module/i18n/{module_name}.pot --modules={module_name}
  ```
- **Language specific**: To export a specific language (e.g. Spanish):
  ```bash
  odoo-bin -c path/to/odoo.conf -d <dbname> --i18n-export=path/to/module/i18n/es.po --modules={module_name} --language=es_ES
  ```
- **Doodba adaptation**: If using Doodba, run this via `docker compose exec` (e.g., `docker compose exec odoo odoo --i18n-export=...`).

### AI Rules for Editing `.po` Translation Files

When you are asked to translate a generated `.po` file, you MUST follow these strict formatting rules:

1. **Single `msgstr` Rule**: NEVER generate duplicate translation lines. Each `msgid` MUST be followed by exactly ONE `msgstr` string. Never provide "alternative" translations in the code.
   - ❌ BAD (Duplicate / Invalid PO format):
     ```po
     msgid "Hello"
     msgstr "Hola"
     "Buenas" 
     ```
   - ✅ GOOD:
     ```po
     msgid "Hello"
     msgstr "Hola"
     ```

2. **Multi-line blocks (`msgid ""`)**: When a text is long and Odoo splits it with `msgid ""` followed by multiple strings, your `msgstr` MUST either be a single line `msgstr "Translated text"` OR a properly formatted multi-line block starting with `msgstr ""`. NEVER mix them, and NEVER append a bare string after a filled `msgstr`.
   - ❌ BAD (Mixing single line msgstr with a bare string below):
     ```po
     msgid ""
     "Download speed used only when the selected service plan does not define one."
     msgstr "Velocidad de descarga usada solo cuando el plan no la define."
     "Velocidad de descarga utilizada solo cuando el plan no la define."
     ```
   - ✅ GOOD (Single line):
     ```po
     msgid ""
     "Download speed used only when the selected service plan does not define one."
     msgstr "Velocidad de descarga usada solo cuando el plan no la define."
     ```
   - ✅ GOOD (Multi-line format):
     ```po
     msgid ""
     "Download speed used only when the selected service "
     "plan does not define one."
     msgstr ""
     "Velocidad de descarga usada solo cuando el plan "
     "no la define."
     ```

3. **Do NOT translate raw numbers or technical symbols**: If a `msgid` contains ONLY numbers (e.g. `802.1p`), empty HTML tags (e.g. `<div class="separator"/>`), or standard technical codes, **leave the `msgstr ""` empty** so Odoo falls back to the original text, or copy it exactly. DO NOT attempt to "translate" numbers.

4. **HTML Attributes & Tags**: Only translate the human-readable text inside HTML tags or attributes (like `title="Translate me"`). KEEP the exact HTML structure intact.
   - ❌ BAD (Translating classes or adding extra quotes):
     ```po
     msgid "<i class=\"fa fa-signal\" title=\"Signal\"/>"
     msgstr "<i class=\"fa fa-senal\" title=\"Señal\"/>"
     ```
   - ✅ GOOD:
     ```po
     msgid "<i class=\"fa fa-signal\" title=\"Signal\"/>"
     msgstr "<i class=\"fa fa-signal\" title=\"Señal\"/>"
     ```

---

## Quick Reference Templates

### **init**.py (root)

```python
from . import models
from . import wizards  # if they exist
from . import controllers  # if they exist
```

### **init**.py (models/)

```python
from . import model_name
from . import model_name_line  # if it exists
```

### tests/**init**.py

```python
from . import common
from . import test_model_name
```

### ir.model.access.csv

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_{model}_user,{model.name} user,model_{model_underscored},{module}.group_user,1,1,1,0
access_{model}_manager,{model.name} manager,model_{model_underscored},{module}.group_manager,1,1,1,1
```

### Demo Data

```xml
<odoo noupdate="1">
    <record id="demo_record_1" model="module.model.name">
        <field name="name">Demo Record 1</field>
        <field name="partner_id" ref="base.res_partner_2"/>
        <field name="amount">150.00</field>
    </record>
</odoo>
```

### readme/DESCRIPTION.rst

```rst
This module provides [what it does].

It allows users to:

* [Feature 1]
* [Feature 2]
```

## Keywords

odoo, odoo 18, oca, module, scaffold, model, view, security, test, transactioncase, httpcase, owl, controller, wizard, localization, migration, erp, **manifest**
