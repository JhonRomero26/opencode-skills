# Localization (l10n) Development in Odoo

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
The golden rule for multi-company localizations: **never show a country-specific field to a user operating under another country**.

In Odoo, use the `company_country_code` field (or similar, depending on the model, e.g. `country_code` on `res.company`) directly in the `invisible` attribute. Do not use complex domain rules or attrs.

```xml
<!-- ✅ Correct: Hide field if the active company's country is not XX (e.g. EC or PE) -->
<field name="l10n_xx_field_name" invisible="company_country_code != 'XX'"/>

<!-- For menus, views, or pages (Notebooks) -->
<page string="Local Withholdings" invisible="country_code != 'XX'">
    <!-- ... -->
</page>

<!-- In Action domains -->
<record id="action_l10n_xx_taxes" model="ir.actions.act_window">
    ...
    <field name="domain">[('company_id.account_fiscal_country_id.code', '=', 'XX')]</field>
</record>
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

Odoo 18 completely abandoned the old XML-based `account.chart.template` model for installing accounting data. Everything is now managed via Python classes inheriting from `account.chart.template`.

If you are creating or extending a localization (`l10n_xx`), you must implement a model like this:

```python
from odoo import models
from odoo.addons.account.models.chart_template import template

class AccountChartTemplate(models.AbstractModel):
    _inherit = 'account.chart.template'

    @template('xx')
    def _get_xx_template_data(self):
        return {
            'code_digits': 6,
            'property_account_receivable_id': 'xx_account_1210',
            'property_account_payable_id': 'xx_account_2110',
            'property_account_expense_categ_id': 'xx_account_6110',
            'property_account_income_categ_id': 'xx_account_7110',
            'name': 'Standard Chart of Accounts (XX)',
        }

    @template('xx', 'res.company')
    def _get_xx_res_company(self):
        return {
            self.env.company.id: {
                'account_fiscal_country_id': 'base.xx',
                'bank_account_code_prefix': '111',
                'cash_account_code_prefix': '112',
                'transfer_account_code_prefix': '113',
            },
        }

    @template('xx', 'account.account')
    def _get_xx_account_account(self):
        # Return a dictionary of accounts: {'xml_id': {'name': 'Account Name', 'code': '1010', 'account_type': 'asset_receivable'}}
        pass

    @template('xx', 'account.tax')
    def _get_xx_account_tax(self):
        # Return a dictionary of taxes
        pass
```

### Key rules for the new template engine:
1. **Dynamic Loading**: Use the `@template('country_code')` decorator. Odoo will automatically discover and load these dictionaries when installing the chart of accounts for that country.
2. **Account Types**: Always use standard Odoo 18 account types (e.g. `asset_receivable`, `liability_payable`, `income`, `expense`).
3. **XML IDs**: Even though data is in Python, use string keys as XML IDs (e.g., `'xx_account_1210'`). These are resolved dynamically.
4. **Fiscal Tags**: Use `account.account.tag` provided by the country code to group accounts in dynamic balance and profit reports, rather than hardcoding account codes.

---

## 10. EDI (Electronic Data Interchange) Standards

When building e-invoicing for a localization (e.g., `l10n_xx_edi`), do not re-invent the wheel. Odoo provides a powerful base framework for electronic documents:

1. **Inherit `account.edi.format`**: Create a new record in `account.edi.format` with your country's standard (e.g., Factura-e).
2. **Override `_create_invoice_edi_xml`**: Instead of manually building strings or using raw lxml, use Odoo's QWeb templating system to generate the XML payload from a view (`ir.ui.view`).
3. **Attachments**: Always use `ir.attachment` linked to the `account.move` with the generated XML, and mark it with the specific EDI type.

## 11. No Magic Numbers (Constants)

In Odoo development, especially in localizations (where tax rates, fiscal codes, and regimes are abundant), **hardcoding primitive data or "magic numbers" directly into your logic is strictly prohibited**. 

Primitive data must always be stored in named constants so that they are easily identifiable, grouped, and reusable. This avoids "ghost" numbers scattered throughout the code.

**Where to place them:**
If the constants are used in a single file, define them at the top of the `.py` file (after imports). If they are shared across multiple models, create a `const.py` file and import them from there.

```python
# ✅ GOOD EXAMPLE (Pattern applies to ANY localization or module):
# Grouping primitive values (rates, codes, statuses) into descriptive constants.

L10N_XX_VAT_RATES = {
    5: 5.0,
    2: 12.0,
    10: 13.0,
    0: 0.0,
}

L10N_XX_VAT_TAX_NOT_ZERO_GROUPS = ('vat05', 'vat12', 'vat13')
L10N_XX_VAT_TAX_ZERO_GROUPS = ('zero_vat', 'not_charged_vat', 'exempt_vat')
L10N_XX_VAT_TAX_GROUPS = tuple(L10N_XX_VAT_TAX_NOT_ZERO_GROUPS + L10N_XX_VAT_TAX_ZERO_GROUPS)

L10N_XX_WITHHOLD_VAT_CODES = {
    0.0: 7,   # 0% vat withhold
    10.0: 9,  # 10% vat withhold
    100.0: 3, # 100% vat withhold
}

L10N_XX_WTH_FOREIGN_GENERAL_REGIME_CODES = ['402', '403', '404', '405']
L10N_XX_WTH_FOREIGN_NOT_SUBJECT_WITHHOLD_CODES = ['412', '423']
L10N_XX_WTH_FOREIGN_SUBJECT_WITHHOLD_CODES = list(set(L10N_XX_WTH_FOREIGN_GENERAL_REGIME_CODES) - set(L10N_XX_WTH_FOREIGN_NOT_SUBJECT_WITHHOLD_CODES))

L10N_XX_WITHHOLD_FOREIGN_REGIME = [
    ('01', '(01) General Regime'), 
    ('02', '(02) Fiscal Paradise'), 
    ('03', '(03) Preferential Tax Regime')
]
```

**Why this is mandatory:**
Using dictionaries, tuples, and lists like the pattern above allows you to do `if code in L10N_XX_WTH_FOREIGN_SUBJECT_WITHHOLD_CODES` rather than having unreadable strings of `if code in ['402', '403', ...]`. This applies not just to localizations, but to **any Odoo module** dealing with statuses, static mappings, or external codes.
