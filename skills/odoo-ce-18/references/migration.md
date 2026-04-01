# Migrations Reference - Odoo v18 OCA

When a change in the module breaks the database schema, you MUST create migration scripts. Without them, Odoo will lose data or fail during the update.

## Migration Structure

```
{module_name}/
└── migrations/
    └── 18.0.{X}.{Y}.{Z}/       # MUST match the version in __manifest__
        ├── pre-migrate.py       # Runs BEFORE the ORM update
        ├── pre-migrate.sql      # Heavy SQL BEFORE the update (optional)
        ├── post-migrate.py      # Runs AFTER the ORM update
        ├── post-migrate.sql     # Heavy SQL AFTER the update (optional)
        └── end-migrate.py       # Runs at the very END (rare, optional)
```

**Execution order in Odoo when running `-u module_name`:**

1. `pre-migrate.py` / `pre-migrate.sql` -> The database has the OLD schema
2. ORM updates the schema (creates columns, removes those that no longer exist in the model)
3. Loads XML/CSV data (security, views, data)
4. `post-migrate.py` / `post-migrate.sql` -> The database has the NEW schema
5. `end-migrate.py` -> At the end of the update of ALL modules

## When to Use Each Type

### pre-migrate (BEFORE the ORM)

Use pre-migrate when you need to act on the OLD structure before the ORM destroys it:

- Rename fields/models/tables (the ORM would delete the old column and create a new empty one)
- Preserve data from fields that are going to disappear
- Remove SQL constraints that block the schema change
- Create temporary columns to move data

### post-migrate (AFTER the ORM)

Use post-migrate when the new schema already exists and you need to transform data:

- Convert old selection values to new ones
- Populate new fields with computed data
- Migrate data between models using the new ORM
- Recompute stored compute fields
- Clean orphaned data

### Direct SQL (.sql or cr.execute)

Use direct SQL when:

- The migration affects >100K records (the ORM is too slow)
- You need bulk UPDATE/INSERT operations that the ORM does not allow
- You need direct DROP/ALTER TABLE/CONSTRAINT operations
- The table has no ORM model available at that point in the migration

---

## Templates by Breaking Change Type

### 1. Rename Field

```python
# migrations/18.0.1.1.0/pre-migrate.py
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

import logging

from openupgradelib import openupgrade

_logger = logging.getLogger(__name__)


@openupgrade.migrate()
def migrate(env, version):
    _logger.info("Renaming field old_field -> new_field on module_model_name")
    openupgrade.rename_fields(
        env,
        [
            # (model_name, table_name, old_field, new_field)
            ("{module.model.name}", "{module_model_name}", "old_field", "new_field"),
        ],
    )
```

### 2. Rename Model / Table

```python
# migrations/18.0.2.0.0/pre-migrate.py
from openupgradelib import openupgrade


@openupgrade.migrate()
def migrate(env, version):
    openupgrade.rename_models(
        env,
        [("old.model.name", "new.model.name")],
    )
    openupgrade.rename_tables(
        env,
        [("old_model_name", "new_model_name")],
    )
    # Also rename associated XML IDs
    openupgrade.rename_xmlids(
        env,
        [
            ("{module}.old_xmlid", "{module}.new_xmlid"),
        ],
    )
```

### 3. Change Field Type (e.g. Char -> Many2one)

```python
# migrations/18.0.1.2.0/pre-migrate.py
import logging
from openupgradelib import openupgrade

_logger = logging.getLogger(__name__)

# Save the old value in a temporary column before the ORM removes it
_column_renames = {
    "{module_model_name}": [
        ("partner_name", None),  # None = openupgrade generates a temporary name
    ],
}


@openupgrade.migrate()
def migrate(env, version):
    openupgrade.rename_columns(env.cr, _column_renames)
    _logger.info("Saved old partner_name column as temporary")
```

```python
# migrations/18.0.1.2.0/post-migrate.py
import logging
from openupgradelib import openupgrade

_logger = logging.getLogger(__name__)


@openupgrade.migrate()
def migrate(env, version):
    # The temporary column is named {old_name}{openupgrade suffix}
    old_column = openupgrade.get_legacy_name("partner_name")

    env.cr.execute(f"""
        UPDATE {module_model_name} AS m
        SET partner_id = p.id
        FROM res_partner AS p
        WHERE p.name = m.{old_column}
    """)
    _logger.info("Migrated partner_name (text) -> partner_id (M2O)")

    # Clean up the temporary column
    openupgrade.drop_columns(
        env.cr,
        [("{module_model_name}", old_column)],
    )
```

### 4. Change Selection Values

```python
# migrations/18.0.1.0.1/post-migrate.py
import logging

_logger = logging.getLogger(__name__)


def migrate(cr, version):
    if not version:
        return
    _logger.info("Migrating state values: open -> confirmed, closed -> done")
    cr.execute("""
        UPDATE {module_model_name}
        SET state = CASE
            WHEN state = 'open' THEN 'confirmed'
            WHEN state = 'closed' THEN 'done'
            ELSE state
        END
        WHERE state IN ('open', 'closed')
    """)
    _logger.info("Updated %d rows", cr.rowcount)
```

### 5. Heavy SQL - Mass Migration (>100K records)

```sql
-- migrations/18.0.2.0.0/pre-migrate.sql
-- Copyright 2025 {Company}
-- License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

-- Remove the constraint that blocks the schema change
ALTER TABLE {module_model_name}
    DROP CONSTRAINT IF EXISTS {module_model_name}_name_company_uniq;

-- Save data before the ORM removes the column
ALTER TABLE {module_model_name}
    ADD COLUMN IF NOT EXISTS old_code VARCHAR;
UPDATE {module_model_name} SET old_code = code WHERE code IS NOT NULL;
```

```sql
-- migrations/18.0.2.0.0/post-migrate.sql
-- Copyright 2025 {Company}
-- License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

-- Mass migration: move data between tables
INSERT INTO {new_table} (name, partner_id, amount, company_id, create_uid, create_date)
SELECT
    old.name,
    old.partner_id,
    old.total_amount,
    old.company_id,
    old.create_uid,
    old.create_date
FROM {old_table} AS old
WHERE old.active = TRUE
ON CONFLICT (name, company_id) DO NOTHING;

-- Populate the new field from existing data
UPDATE {module_model_name}
SET new_reference = CONCAT('REF-', LPAD(id::TEXT, 6, '0'))
WHERE new_reference IS NULL;

-- Recreate the constraint with the new definition
ALTER TABLE {module_model_name}
    ADD CONSTRAINT {module_model_name}_name_company_uniq
    UNIQUE (name, company_id);

-- Clean up the temporary column
ALTER TABLE {module_model_name}
    DROP COLUMN IF EXISTS old_code;
```

### 6. Merge Two Models into One

```python
# migrations/18.0.2.0.0/pre-migrate.py
from openupgradelib import openupgrade


@openupgrade.migrate()
def migrate(env, version):
    # Create a temporary column to store the origin
    env.cr.execute("""
        ALTER TABLE {target_table}
            ADD COLUMN IF NOT EXISTS _migrated_from VARCHAR DEFAULT NULL
    """)
    # Copy data from the model that will disappear
    env.cr.execute("""
        INSERT INTO {target_table}
            (name, partner_id, amount, company_id, state, _migrated_from,
             create_uid, create_date, write_uid, write_date)
        SELECT
            s.name, s.partner_id, s.total, s.company_id, 'draft', 'old_model',
            s.create_uid, s.create_date, s.write_uid, s.write_date
        FROM {source_table} AS s
    """)
```

```python
# migrations/18.0.2.0.0/post-migrate.py
from openupgradelib import openupgrade


@openupgrade.migrate()
def migrate(env, version):
    # Clean up the temporary column
    env.cr.execute("""
        ALTER TABLE {target_table} DROP COLUMN IF EXISTS _migrated_from
    """)
    # Recompute stored fields
    records = env["{target.model}"].search([])
    records._compute_total()
```

### 7. Move a Field from One Model to Another

```python
# migrations/18.0.1.3.0/pre-migrate.py
def migrate(cr, version):
    if not version:
        return
    # Create the column in the target and copy data via JOIN
    cr.execute("""
        ALTER TABLE {dest_table}
            ADD COLUMN IF NOT EXISTS {field_name} VARCHAR;

        UPDATE {dest_table} AS d
        SET {field_name} = s.{field_name}
        FROM {source_table} AS s
        WHERE s.id = d.{fk_to_source};
    """)
```

### 8. Remove Constraint Before a Schema Change

```sql
-- migrations/18.0.1.1.0/pre-migrate.sql
-- Remove blocking constraints
ALTER TABLE {table} DROP CONSTRAINT IF EXISTS {table}_{constraint_name};

-- Remove blocking indexes
DROP INDEX IF EXISTS {index_name};
```

---

## Without openupgradelib (native migration)

If `openupgradelib` is not available, use `cr` directly:

```python
# migrations/18.0.1.1.0/pre-migrate.py
import logging

_logger = logging.getLogger(__name__)


def migrate(cr, version):
    """Native signature: receives cursor, not env."""
    if not version:
        return
    _logger.info("Renaming column old_field -> new_field")
    cr.execute("""
        ALTER TABLE {module_model_name}
        RENAME COLUMN old_field TO new_field
    """)
```

```python
# migrations/18.0.1.1.0/post-migrate.py
import logging

_logger = logging.getLogger(__name__)


def migrate(cr, version):
    if not version:
        return
    cr.execute("""
        UPDATE {module_model_name}
        SET state = 'confirmed'
        WHERE state = 'open'
    """)
    _logger.info("Migrated %d rows", cr.rowcount)
```

**Signature difference:**
- With `openupgradelib`: `def migrate(env, version)` -> you have access to the ORM (`env["model"]`)
- Without `openupgradelib`: `def migrate(cr, version)` -> SQL cursor only (`cr.execute()`)
- `.sql` files: executed automatically by Odoo, they do not need a `migrate()` function

---

## Migration Checklist

Before delivering a change with a migration, verify:

- [ ] The version in `__manifest__.py` was incremented
- [ ] `migrations/X.Y.Z/` matches the new version EXACTLY
- [ ] `pre-migrate` works on the OLD schema (does not use new models)
- [ ] `post-migrate` works on the NEW schema (can use the new ORM)
- [ ] SQL uses `IF EXISTS` / `IF NOT EXISTS` to be idempotent
- [ ] Logger explains what is migrated and how many records
- [ ] It was tested on a copy of the production database
- [ ] If the SQL is heavy, `.sql` is used instead of `cr.execute()` in `.py`
- [ ] No `env.cr.commit()` - Odoo manages the transaction
- [ ] The change was documented in `readme/HISTORY.rst` or the changelog

---

## Most Used openupgradelib Helpers

```python
from openupgradelib import openupgrade

# Rename fields (columns)
openupgrade.rename_fields(env, [
    ("model.name", "table_name", "old_field", "new_field"),
])

# Rename models (tables + ir.model.data)
openupgrade.rename_models(env, [
    ("old.model", "new.model"),
])

# Rename tables directly
openupgrade.rename_tables(env, [
    ("old_table", "new_table"),
])

# Rename columns (low level)
openupgrade.rename_columns(env.cr, {
    "table_name": [("old_col", "new_col")],
})

# Save a column with a temporary name (to migrate field type)
openupgrade.rename_columns(env.cr, {
    "table_name": [("field_to_change", None)],  # None = auto-generated name
})
# Recover the temporary name
legacy_name = openupgrade.get_legacy_name("field_to_change")

# Rename XML IDs
openupgrade.rename_xmlids(env, [
    ("module.old_id", "module.new_id"),
])

# Remove columns
openupgrade.drop_columns(env.cr, [
    ("table_name", "column_name"),
])

# Check whether a column exists
openupgrade.column_exists(env.cr, "table_name", "column_name")

# Check whether a table exists
openupgrade.table_exists(env.cr, "table_name")

# Logging helper
openupgrade.logged_query(env.cr, """
    UPDATE table SET field = 'value' WHERE condition
""")
# Automatically logs the SQL and cr.rowcount

# Recompute stored fields (post-migrate)
openupgrade.recompute_fields(env, "model.name", ["field1", "field2"])
```

---

## Additional Scenarios

### 9. Convert a non-stored compute field -> stored

When a `compute` field becomes `store=True`, Odoo does NOT calculate it retroactively. You need to force a recompute.

```python
# migrations/18.0.1.1.0/post-migrate.py
import logging
from openupgradelib import openupgrade

_logger = logging.getLogger(__name__)


@openupgrade.migrate()
def migrate(env, version):
    _logger.info("Recomputing stored field 'total' on module.model.name")
    openupgrade.recompute_fields(
        env, "module.model.name", ["total"]
    )
```

Without `openupgradelib`:

```python
def migrate(cr, version):
    if not version:
        return
    # Mark records as "need recompute"
    cr.execute("""
        UPDATE module_model_name SET total = NULL
    """)
    # Odoo will recompute them automatically on the next access
```

### 10. Change Inheritance (_inherit -> _inherits or vice versa)

```python
# migrations/18.0.2.0.0/pre-migrate.py
import logging

_logger = logging.getLogger(__name__)


def migrate(cr, version):
    if not version:
        return
    # Before changing from _inherit to a standalone model with _inherits:
    # Create the new table BEFORE the ORM tries to do so
    cr.execute("""
        CREATE TABLE IF NOT EXISTS library_book (
            id SERIAL PRIMARY KEY,
            product_id INTEGER REFERENCES product_product(id) ON DELETE CASCADE,
            isbn VARCHAR,
            create_uid INTEGER,
            create_date TIMESTAMP DEFAULT NOW(),
            write_uid INTEGER,
            write_date TIMESTAMP DEFAULT NOW()
        )
    """)
    # Move data from the inherited model to the new one
    cr.execute("""
        INSERT INTO library_book (product_id, isbn, create_uid, create_date)
        SELECT pp.id, pp.x_isbn, pp.create_uid, pp.create_date
        FROM product_product AS pp
        WHERE pp.x_isbn IS NOT NULL
    """)
    _logger.info("Migrated %d records to library_book", cr.rowcount)
```

### 11. Migration Between Major Odoo Versions (e.g. v17 -> v18)

For a major version migration, use the corresponding `openupgrade` branch:

```python
# migrations/18.0.1.0.0/pre-migrate.py
from openupgradelib import openupgrade

_column_renames = {
    "module_model_name": [
        # Columns that were renamed in v18
        ("old_v17_field", "new_v18_field"),
    ],
}

_field_renames = [
    # (model, table, old_field, new_field)
    ("module.model", "module_model", "old_field", "new_field"),
]


@openupgrade.migrate()
def migrate(env, version):
    openupgrade.rename_columns(env.cr, _column_renames)
    openupgrade.rename_fields(env, _field_renames)
    # v18-specific changes:
    # - attrs -> direct invisible/readonly/required (XML only, not DB)
    # - SavepointCase -> TransactionCase (tests only, not DB)
    # - <tree> -> <list> (XML only, not DB)
```

### 12. Add a Value to an Existing Selection + Migrate Data

```python
# migrations/18.0.1.1.0/post-migrate.py
import logging

_logger = logging.getLogger(__name__)


def migrate(cr, version):
    if not version:
        return
    # Migrate old values to the new value
    cr.execute("""
        UPDATE module_model_name
        SET priority = 'urgent'
        WHERE priority = 'critical'
    """)
    _logger.info("Migrated %d 'critical' -> 'urgent'", cr.rowcount)
    
    # Populate the new field with a default value for existing records
    cr.execute("""
        UPDATE module_model_name
        SET new_selection_field = 'default_value'
        WHERE new_selection_field IS NULL
    """)
    _logger.info("Set default for %d records", cr.rowcount)
```

### 13. Migrate Many2one -> Many2many (or vice versa)

```python
# migrations/18.0.1.2.0/pre-migrate.py
from openupgradelib import openupgrade


@openupgrade.migrate()
def migrate(env, version):
    # Save the M2O value before the ORM removes it
    openupgrade.rename_columns(env.cr, {
        "module_model_name": [("tag_id", None)],  # temporary column
    })
```

```python
# migrations/18.0.1.2.0/post-migrate.py
from openupgradelib import openupgrade


@openupgrade.migrate()
def migrate(env, version):
    old_column = openupgrade.get_legacy_name("tag_id")
    
    # Create M2M relations from the old M2O value
    env.cr.execute(f"""
        INSERT INTO module_model_name_module_model_tag_rel
            (module_model_name_id, module_model_tag_id)
        SELECT id, {old_column}
        FROM module_model_name
        WHERE {old_column} IS NOT NULL
        ON CONFLICT DO NOTHING
    """)
    
    # Clean up
    openupgrade.drop_columns(env.cr, [
        ("module_model_name", old_column),
    ])
```

---

## Migration Tests

Create `tests/test_migration.py` to verify that migrated data is correct:

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

from odoo.tests import tagged

from .common import TestCommon


@tagged("post_install", "-at_install")
class TestMigration(TestCommon):

    def test_renamed_field_data_preserved(self):
        """Test that data survived the field rename migration."""
        # Verify that the new field has data
        records_with_data = self.env["{module.model.name}"].search(
            [("new_field", "!=", False)]
        )
        self.assertTrue(
            records_with_data,
            "Migration failed: new_field has no data after rename"
        )

    def test_selection_values_migrated(self):
        """Test that old selection values were migrated."""
        # No record should remain with the old value
        old_records = self.env["{module.model.name}"].search(
            [("state", "=", "old_value")]
        )
        self.assertFalse(
            old_records,
            "Migration failed: found records with old state 'old_value'"
        )

    def test_m2o_to_m2m_migration(self):
        """Test that M2O data was correctly moved to M2M."""
        for record in self.env["{module.model.name}"].search([]):
            if record.tag_ids:
                # Each record that had a tag_id now has at least
                # that tag in tag_ids
                self.assertTrue(len(record.tag_ids) >= 1)

    def test_computed_fields_recomputed(self):
        """Test that stored computed fields are correct after migration."""
        for record in self.env["{module.model.name}"].search([], limit=10):
            expected = sum(record.line_ids.mapped("subtotal"))
            self.assertEqual(
                record.total, expected,
                f"Record {record.id}: total={record.total}, expected={expected}"
            )
```

---

## Versioning Pattern for Multiple Migrations

When a module goes through several consecutive migrations, each one needs its own directory:

```
migrations/
├── 18.0.1.0.1/       # Bugfix: fix constraint
│   └── pre-migrate.sql
├── 18.0.1.1.0/       # Feature: rename field
│   ├── pre-migrate.py
│   └── post-migrate.py
├── 18.0.1.2.0/       # Feature: M2O -> M2M
│   ├── pre-migrate.py
│   └── post-migrate.py
└── 18.0.2.0.0/       # Breaking: merge models
    ├── pre-migrate.sql
    ├── pre-migrate.py
    └── post-migrate.py
```

Odoo runs ALL pending migrations in version order. If a user goes from `18.0.1.0.0` to `18.0.2.0.0`, the 4 migrations are executed in sequence.

**Do not group** migrations from different versions into a single directory. Each version bump = its own directory.
