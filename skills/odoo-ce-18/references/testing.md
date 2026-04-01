# Testing Reference — Odoo v18 CE

When generating a module, ALWAYS create these test files. Tests are MANDATORY to pass OCA CI.

## tests/common.py — Shared Setup

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

from odoo.tests.common import TransactionCase


class {TestCommonClass}(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.env = cls.env(
            context=dict(cls.env.context, tracking_disable=True)
        )
        # --- Test users ---
        cls.group_user = cls.env.ref("{module_name}.group_user")
        cls.group_manager = cls.env.ref("{module_name}.group_manager")
        cls.user = cls.env["res.users"].create(
            {
                "name": "Test User",
                "login": "test_user_{module}@test.com",
                "groups_id": [(6, 0, [cls.group_user.id])],
            }
        )
        cls.manager = cls.env["res.users"].create(
            {
                "name": "Test Manager",
                "login": "test_manager_{module}@test.com",
                "groups_id": [(6, 0, [cls.group_manager.id])],
            }
        )
        # --- Test data ---
        cls.partner = cls.env["res.partner"].create(
            {
                "name": "Test Partner",
                "email": "partner@test.com",
            }
        )
        cls.record = cls.env["{module.model.name}"].create(
            {
                "name": "Test Record",
                "partner_id": cls.partner.id,
            }
        )
```

## tests/test\_{model_name}.py — Complete Tests

```python
# Copyright 2025 {Company}
# License AGPL-3.0 or later (https://www.gnu.org/licenses/agpl).

from odoo.exceptions import UserError, ValidationError
from odoo.tests import Form, tagged

from .common import {TestCommonClass}


@tagged("post_install", "-at_install")
class Test{ModelClass}(TestCommonClass):

    # ==================== CRUD ====================

    def test_create_defaults(self):
        """Test record creation sets correct defaults."""
        record = self.env["{module.model.name}"].create(
            {
                "name": "New Record",
                "partner_id": self.partner.id,
            }
        )
        self.assertEqual(record.state, "draft")
        self.assertTrue(record.active)
        self.assertEqual(record.company_id, self.env.company)

    def test_create_sequence(self):
        """Test auto-sequence when name is empty."""
        record = self.env["{module.model.name}"].create(
            {"partner_id": self.partner.id}
        )
        self.assertTrue(record.name)

    def test_unlink_draft(self):
        """Test draft records can be deleted."""
        record_id = self.record.id
        self.record.unlink()
        self.assertFalse(
            self.env["{module.model.name}"].search([("id", "=", record_id)])
        )

    def test_unlink_non_draft_raises(self):
        """Test non-draft records cannot be deleted."""
        self.record.action_confirm()
        with self.assertRaises(UserError):
            self.record.unlink()

    # ==================== CONSTRAINTS ====================

    def test_sql_constraint_unique_name(self):
        """Test unique name per company constraint."""
        with self.assertRaises(Exception):
            self.env["{module.model.name}"].create(
                {
                    "name": self.record.name,
                    "partner_id": self.partner.id,
                }
            )

    def test_check_name_min_length(self):
        """Test name minimum length constraint."""
        with self.assertRaises(ValidationError):
            self.env["{module.model.name}"].create(
                {
                    "name": "AB",
                    "partner_id": self.partner.id,
                }
            )

    # ==================== COMPUTE ====================

    def test_compute_total(self):
        """Test total computed from lines."""
        self.env["{module.model.line}"].create(
            {
                "parent_id": self.record.id,
                "name": "Line 1",
                "quantity": 2.0,
                "price_unit": 50.0,
            }
        )
        self.env["{module.model.line}"].create(
            {
                "parent_id": self.record.id,
                "name": "Line 2",
                "quantity": 1.0,
                "price_unit": 25.0,
            }
        )
        self.assertEqual(self.record.total, 125.0)

    def test_compute_line_subtotal(self):
        """Test line subtotal = qty * price."""
        line = self.env["{module.model.line}"].create(
            {
                "parent_id": self.record.id,
                "name": "Line",
                "quantity": 3.0,
                "price_unit": 10.0,
            }
        )
        self.assertEqual(line.subtotal, 30.0)

    # ==================== WORKFLOW / ACTIONS ====================

    def test_action_confirm(self):
        """Test confirm from draft to confirmed."""
        self.record.action_confirm()
        self.assertEqual(self.record.state, "confirmed")

    def test_action_confirm_non_draft_raises(self):
        """Test confirm raises if not draft."""
        self.record.action_confirm()
        with self.assertRaises(UserError):
            self.record.action_confirm()

    def test_action_done(self):
        """Test done from confirmed."""
        self.record.action_confirm()
        self.record.action_done()
        self.assertEqual(self.record.state, "done")

    def test_action_done_skips_non_confirmed(self):
        """Test done only affects confirmed records."""
        self.record.action_done()  # draft → should not change
        self.assertEqual(self.record.state, "draft")

    def test_action_cancel(self):
        """Test cancel action."""
        self.record.action_cancel()
        self.assertEqual(self.record.state, "cancelled")

    def test_action_draft_from_cancelled(self):
        """Test reset to draft from cancelled."""
        self.record.action_cancel()
        self.record.action_draft()
        self.assertEqual(self.record.state, "draft")

    def test_full_workflow(self):
        """Test complete lifecycle: draft → confirmed → done."""
        self.assertEqual(self.record.state, "draft")
        self.record.action_confirm()
        self.assertEqual(self.record.state, "confirmed")
        self.record.action_done()
        self.assertEqual(self.record.state, "done")

    # ==================== FORM SIMULATION ====================

    def test_form_onchange_partner(self):
        """Test onchange partner sets name via Form."""
        form = Form(self.env["{module.model.name}"])
        form.partner_id = self.partner
        self.assertEqual(form.name, self.partner.name)

    def test_form_create_with_lines(self):
        """Test record creation with O2M lines via Form."""
        form = Form(self.env["{module.model.name}"])
        form.name = "Form Test"
        form.partner_id = self.partner
        with form.line_ids.new() as line:
            line.name = "Test Line"
            line.quantity = 5.0
            line.price_unit = 20.0
        record = form.save()
        self.assertEqual(len(record.line_ids), 1)
        self.assertEqual(record.total, 100.0)

    # ==================== SECURITY ====================

    def test_user_read(self):
        """Test user can read records."""
        records = self.env["{module.model.name}"].with_user(
            self.user
        ).search([])
        self.assertIn(self.record.id, records.ids)

    def test_user_cannot_unlink(self):
        """Test user cannot delete records (no unlink permission)."""
        with self.assertRaises(Exception):
            self.record.with_user(self.user).unlink()

    def test_manager_can_unlink(self):
        """Test manager can delete draft records."""
        self.record.with_user(self.manager).unlink()

    def test_multi_company_isolation(self):
        """Test records isolated between companies."""
        company2 = self.env["res.company"].create({"name": "Test Co 2"})
        user2 = self.env["res.users"].create(
            {
                "name": "User Co2",
                "login": "user_co2_{module}@test.com",
                "company_id": company2.id,
                "company_ids": [(6, 0, [company2.id])],
                "groups_id": [(6, 0, [self.group_user.id])],
            }
        )
        visible = self.env["{module.model.name}"].with_user(user2).search([])
        self.assertNotIn(self.record.id, visible.ids)
```

## Wizard Tests

```python
@tagged("post_install", "-at_install")
class Test{WizardClass}(TestCommonClass):
    def test_wizard_default_get(self):
        """Test wizard loads active records."""
        wizard = (
            self.env["{module.model.wizard}"]
            .with_context(active_ids=self.record.ids)
            .create({})
        )
        self.assertIn(self.record.id, wizard.line_ids.ids)

    def test_wizard_apply(self):
        """Test wizard applies changes."""
        wizard = (
            self.env["{module.model.wizard}"]
            .with_context(active_ids=self.record.ids)
            .create({"date": "2025-06-01"})
        )
        wizard.action_apply()
        self.assertEqual(str(self.record.date), "2025-06-01")
```

## HttpCase for UI Tours

```python
from odoo.tests import HttpCase, tagged


@tagged("-at_install", "post_install")
class Test{Module}Tour(HttpCase):
    def test_tour_create(self):
        """Test UI flow for creating a record."""
        self.start_tour(
            "/odoo/{module-slug}",
            "{module_name}_tour",
            login="admin",
        )
```

## Mocking

```python
from unittest.mock import patch

def test_external_api(self):
    """Test external API call with mock."""
    with patch(
        "odoo.addons.{module_name}.models.{model}.requests.post"
    ) as mock_post:
        mock_post.return_value.status_code = 200
        mock_post.return_value.json.return_value = {"ok": True}
        result = self.record._call_external_api()
        self.assertTrue(result)
        mock_post.assert_called_once()
```

## Run Tests

```bash
# All tests for the module
odoo-bin -d testdb -i {module_name} --test-enable --stop-after-init \
    --test-tags /{module_name}

# Specific class
odoo-bin -d testdb --test-tags /{module_name}:Test{ModelClass} --stop-after-init

# Specific method
odoo-bin -d testdb --test-tags /{module_name}:Test{ModelClass}.test_action_confirm \
    --stop-after-init

# With coverage
coverage run odoo-bin -d testdb -i {module_name} --test-enable --stop-after-init
coverage report --include="*{module_name}*"
coverage html --include="*{module_name}*"
```
