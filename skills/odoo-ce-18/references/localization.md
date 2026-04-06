# Localization (l10n) Development in Odoo 18

Designing a localization module (`l10n_XX`) requires a flawless architecture dictated by Odoo S.A. and OCA standards. In multi-company and multi-country environments, **several companies share the same database but are governed by different tax regulations** (e.g., SRI in Ecuador, SUNAT in Peru, AFIP in Argentina).

**The golden rule is: your code must not be invasive and must coexist peacefully with other localizations.**

Below are the mandatory architectural principles and code patterns for localizations in Odoo 18.

---

## 1. Naming Conventions and Dependencies (OCA Standard)

Everything in the module must be strictly prefixed to avoid clashing with other modules.

*   **Module Names:** `l10n_xx_feature` (e.g., `l10n_ec_edi`, `l10n_pe_pos`).
*   **Fields and Methods:** Always use the `l10n_xx_` prefix. (e.g., `l10n_ec_authorization_number`, `_l10n_ec_compute_taxes()`).
*   **XML IDs:** Always prefixed with the country code (e.g., `id="l10n_pe_action_invoice"`).
*   **Dependencies:** If your module extends the accounting or fiscal functionality of a country, it MUST depend (in `__manifest__.py`) on the base localization module provided by Odoo for that country (`l10n_ec`, `l10n_pe`, etc.).

---

## 2. Demo Data Reusability

When Odoo installs the base module of a country (e.g., `l10n_ec`), **it creates a preconfigured demo company** (e.g., `base.demo_company_ec`).
Any satellite or extension module (`l10n_ec_edi`, `l10n_ec_hr`) **MUST NEVER CREATE NEW DEMO COMPANIES**. It must use the existing base demo company to inject its data.

```xml
<!-- ✅ Correct: Reuse the demo company created by the base l10n_xx module -->
<record id="demo_invoice_1" model="account.move">
    <field name="move_type">out_invoice</field>
    <field name="company_id" ref="l10n_ec.demo_company_ec"/> <!-- or l10n_pe.demo_company_pe -->
    <field name="partner_id" ref="base.res_partner_2"/>
    ...
</record>
```

---

## 3. Translations (i18n) in Localizations

Localization modules must strictly respect user languages. Even if the module is for Ecuador (Spanish), the database could be configured in English, and the menus must not break.

1.  **Generate the `i18n/` folder**: Every localization module must export its `.pot` file and its respective `.po` files (e.g., `es_EC.po`, `es_PE.po`, `es.po`).
2.  **Translating Data in XML/CSV**:
    *   If you create tax groups (`account.tax.group`) or journals in your templates/data, ensure the translations are loaded.
    *   In Python, if you need to raise an error to the user, always use the `_()` function:
        ```python
        from odoo import _
        from odoo.exceptions import UserError

        raise UserError(_("The SRI authorization number is missing."))
        ```
3.  **Updating Existing Records**: If your module injects fields or menus into standard models (like `res.partner`), the translation from the `.po` file will be automatically applied to the interface when the language is updated.

---

## 4. Country-Specific UI and Logic

It is not enough to filter by company (`company_id`). You must verify the **fiscal country** of the active company.
If the company is from Ecuador, show the SRI menus; if it's from Peru, show SUNAT.

### In XML Views (UI)
Use the `account_fiscal_country_id` or `country_code` field to hide elements:

```xml
<!-- ✅ Correct in v18: Hide field if the company's country is not XX (e.g. EC or PE) -->
<field name="l10n_xx_field_name" invisible="company_country_code != 'XX'"/>

<!-- For menus, views, or pages (Notebooks) -->
<page string="Local Withholdings" invisible="country_code != 'XX'">
    <!-- ... -->
</page>
```

### In Python (Logic)
Always verify the country before executing tax, withholding, or payroll calculations.

```python
def _compute_l10n_xx_taxes(self):
    for record in self:
        # ✅ Always evaluate in the context of the record's company
        if record.company_id.account_fiscal_country_id.code != 'XX':
            continue
        # Exclusive logic for localization XX...
```

---

## 5. Clear Separation of Data, Menus, and Security

Avoid at all costs adding global menus that clutter the interface of users from other countries.

*   **Menus and Actions:** Condition them using domains (`domain="[('company_id.country_id.code', '=', 'XX')]"`).
*   **Security Groups:** Define access groups (`res.groups`) specific to the localization (e.g., `group_l10n_xx_edi`) and use them in your menus. A user from Peru should not see the menus nor belong to the groups of Ecuador.
*   **Record Rules:** Use access rules that combine the current company and the country.

---

## 6. Modular and Non-Invasive Design (Extensibility)

Localization modules **extend, they do not overwrite**.
*   **Never** use `replace` in an XPath unless strictly necessary. Use `position="after"` or `position="before"`.
*   If you need to change the behavior of a core function (e.g., `account.move.action_post()`), use `super()` and **condition** your additional logic to your country.

```python
def action_post(self):
    res = super().action_post()
    # ✅ Filter and iterate ONLY over the records that belong to localization XX
    for move in self.filtered(lambda m: m.company_id.account_fiscal_country_id.code == 'XX'):
        move._l10n_xx_generate_xml_report()
    return res
```

---

## 7. Multi-Company Context in Calculations

Odoo evaluates the environment based on `self.env.company` or the record's `company_id`.
*   When calculating withholdings, payroll, or taxes, the code must iterate over the records and base its rules on `record.company_id.country_id`.
*   **Never** assume the base currency is fixed. Use Odoo's currency conversion methods (`currency_id._convert()`) if the module operates in an environment that supports multiple currencies.

---

## 8. Smart Defaults

When you install a module like `l10n_xx`, it should attempt to preconfigure default values (country, timezone) for the active company, reducing human error.

```python
class ResCompany(models.Model):
    _inherit = 'res.company'

    @api.model
    def _get_default_country(self):
        # ✅ Force the localization's country if creating/installing under this context
        country_xx = self.env.ref('base.xx', raise_if_not_found=False)
        return country_xx if country_xx else super()._get_default_country()

    country_id = fields.Many2one(default=_get_default_country)
    
class ResPartner(models.Model):
    _inherit = 'res.partner'
    
    # ✅ Force the partner's country based on the active company (if the company is XX)
    @api.model
    def default_get(self, fields_list):
        res = super().default_get(fields_list)
        if 'country_id' not in res and self.env.company.country_id.code == 'XX':
            res['country_id'] = self.env.ref('base.xx').id
        return res
```

---

## 9. Chart of Accounts and Accounting Engine (Odoo 18)

The handling of accounting templates in Odoo 18 has evolved. The definition of the Chart of Accounts and base Taxes is no longer managed with pure XML `account.chart.template` records (as in v14).
*   In Odoo 18, tax, account, and journal data is usually loaded dynamically through the localization engine (Python and JSON/XML templates).
*   If you are going to expand accounting, study how template loading works in the `account` module of v18 (usually through classes that inherit from `account.chart.template` in Python and return the structured data `_get_xx_reports()`).
*   Use fiscal tags (`account.account.tag`) provided by the country code so that dynamic balance and profit reports group accounts correctly without hardcoding IDs.