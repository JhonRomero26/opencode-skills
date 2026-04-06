# Desarrollo de Localizaciones (l10n) en Odoo 18

El diseño de un módulo de localización (`l10n_XX`) exige una arquitectura impecable dictada por Odoo S.A. y OCA. En entornos multiempresa y multipaís, **varias compañías comparten la misma base de datos pero se rigen por normativas fiscales diferentes** (ej. SRI en Ecuador, SUNAT en Perú, AFIP en Argentina).

**La regla de oro es: tu código no debe ser invasivo y debe coexistir pacíficamente con otras localizaciones.**

A continuación, los principios arquitectónicos y patrones de código obligatorios para localizaciones en Odoo 18.

---

## 1. Convenciones de Nombres y Dependencias (OCA Standard)

Todo en el módulo debe estar fuertemente prefijado para no colisionar con otros módulos.

*   **Nombres de Módulos:** `l10n_xx_funcionalidad` (ej. `l10n_ec_edi`, `l10n_pe_pos`).
*   **Campos y Métodos:** Siempre usar el prefijo `l10n_xx_`. (ej. `l10n_ec_authorization_number`, `_l10n_ec_compute_taxes()`).
*   **XML IDs:** Siempre prefijados con el código del país (ej. `id="l10n_pe_action_invoice"`).
*   **Dependencias:** Si tu módulo extiende la funcionalidad contable o fiscal de un país, DEBE depender (en `__manifest__.py`) del módulo base de localización de ese país provisto por Odoo (`l10n_ec`, `l10n_pe`, etc.).

---

## 2. Reutilización de Data Demo (Demo Data Reusability)

Cuando Odoo instala el módulo base de un país (ej. `l10n_ec`), **crea una compañía demo preconfigurada** (ej. `base.demo_company_ec`).
Cualquier módulo satélite o de extensión (`l10n_ec_edi`, `l10n_ec_hr`) **NUNCA DEBE CREAR NUEVAS COMPAÑÍAS DEMO**. Debe utilizar la compañía base existente para inyectar su data.

```xml
<!-- ✅ Correcto: Reutilizar la compañía demo que creó el módulo base l10n_xx -->
<record id="demo_invoice_1" model="account.move">
    <field name="move_type">out_invoice</field>
    <field name="company_id" ref="l10n_ec.demo_company_ec"/> <!-- o l10n_pe.demo_company_pe -->
    <field name="partner_id" ref="base.res_partner_2"/>
    ...
</record>
```

---

## 3. Traducciones (i18n) en Localizaciones

Los módulos de localización deben respetar estrictamente los idiomas de los usuarios. Incluso si el módulo es para Ecuador (español), la base de datos podría estar configurada en inglés y los menús no pueden romperse.

1.  **Generación de la carpeta `i18n/`**: Todo módulo de localización debe exportar su `.pot` y sus respectivos archivos `.po` (ej. `es_EC.po`, `es_PE.po`, `es.po`).
2.  **Traducción de Datos en XML/CSV**:
    *   Si creás grupos de impuestos (`account.tax.group`) o diarios en tus templates/datos, asegurate de cargar las traducciones.
    *   En Python, si necesitás lanzar un error al usuario, utilizá siempre la función `_()`:
        ```python
        from odoo import _
        from odoo.exceptions import UserError

        raise UserError(_("The SRI authorization number is missing."))
        ```
3.  **Actualización de Registros Existentes**: Si tu módulo inyecta campos o menús a modelos estándar (como `res.partner`), la traducción del `.po` se aplicará automáticamente a la interfaz cuando se actualice el idioma.

---

## 4. Condicionar UI y Lógica por País (Country-Specific Logic)

No basta con filtrar por empresa (`company_id`). Debés verificar el **país fiscal** de la empresa activa.
Si la empresa es de Ecuador, se muestran los menús del SRI; si es de Perú, los de SUNAT.

### En Vistas XML (UI)
Usá el campo `account_fiscal_country_id` o `country_code` para ocultar elementos:

```xml
<!-- ✅ Correcto en v18: Ocultar campo si el país de la compañía no es XX (ej. EC o PE) -->
<field name="l10n_xx_field_name" invisible="company_country_code != 'XX'"/>

<!-- Para menús, vistas, o páginas (Notebooks) -->
<page string="Retenciones Locales" invisible="country_code != 'XX'">
    <!-- ... -->
</page>
```

### En Python (Lógica)
Siempre verificá el país antes de ejecutar cálculos de impuestos, retenciones o nómina.

```python
def _compute_l10n_xx_taxes(self):
    for record in self:
        # ✅ Evaluar siempre en el contexto de la compañía del registro
        if record.company_id.account_fiscal_country_id.code != 'XX':
            continue
        # Lógica exclusiva para la localización XX...
```

---

## 5. Separación Clara de Datos, Menús y Seguridad

Evitá a toda costa agregar menús globales que ensucien la interfaz de usuarios de otros países.

*   **Menús y Acciones:** Condicionalos mediante dominios (`domain="[('company_id.country_id.code', '=', 'XX')]"`).
*   **Grupos de Seguridad:** Definí grupos de acceso (`res.groups`) específicos de la localización (ej. `group_l10n_xx_edi`) y usalos en tus menús. Un usuario de Perú no debe ver los menús ni pertenecer a los grupos de Ecuador.
*   **Record Rules:** Usá reglas de acceso que combinen la compañía actual y el país.

---

## 6. Diseño Modular y No Invasivo (Extensibilidad)

Los módulos de localización **extienden, no sobreescriben**.
*   **Nunca** uses `replace` en un XPath a menos que sea estrictamente necesario. Usá `position="after"` o `position="before"`.
*   Si necesitás cambiar un comportamiento de una función core (ej. `account.move.action_post()`), usá `super()` y **condicioná** tu lógica adicional a tu país.

```python
def action_post(self):
    res = super().action_post()
    # ✅ Filtrar e iterar SOLO sobre los registros que pertenecen a la localización XX
    for move in self.filtered(lambda m: m.company_id.account_fiscal_country_id.code == 'XX'):
        move._l10n_xx_generate_xml_report()
    return res
```

---

## 7. Contexto Multiempresa en Cálculos (Multi-Company)

Odoo evalúa el entorno en base a `self.env.company` o al `company_id` del registro.
*   Al calcular retenciones, nómina (roles de pago) o impuestos, el código debe iterar sobre los registros y basar sus reglas en `record.company_id.country_id`.
*   **Nunca** asumas que la moneda base es fija. Utilizá los métodos de conversión de moneda de Odoo (`currency_id._convert()`) si el módulo opera en un entorno que soporta múltiples monedas.

---

## 8. Smart Defaults: Valores por Defecto Inteligentes

Cuando instalás un módulo como `l10n_xx`, debe intentar preconfigurar los valores por defecto (país, zona horaria) para la compañía activa, reduciendo el margen de error humano.

```python
class ResCompany(models.Model):
    _inherit = 'res.company'

    @api.model
    def _get_default_country(self):
        # ✅ Forzar el país de la localización si estamos creando/instalando bajo este contexto
        country_xx = self.env.ref('base.xx', raise_if_not_found=False)
        return country_xx if country_xx else super()._get_default_country()

    country_id = fields.Many2one(default=_get_default_country)
    
class ResPartner(models.Model):
    _inherit = 'res.partner'
    
    # ✅ Forzar el país del partner en base a la compañía activa (si la empresa es XX)
    @api.model
    def default_get(self, fields_list):
        res = super().default_get(fields_list)
        if 'country_id' not in res and self.env.company.country_id.code == 'XX':
            res['country_id'] = self.env.ref('base.xx').id
        return res
```

---

## 9. Plan de Cuentas y Motor Contable (Odoo 18 Engine)

El manejo de plantillas contables en Odoo 18 evolucionó. La definición del Plan de Cuentas (Chart of Accounts) y los Impuestos base ya no se gestiona con registros `account.chart.template` en XML puro (como en v14).
*   En Odoo 18, la data de impuestos, cuentas y diarios suele ser cargada dinámicamente mediante el motor de localización (Python y JSON/XML templates).
*   Si vas a expandir contabilidad, estudiá cómo funciona la carga de templates en el módulo `account` de v18 (usualmente mediante clases que heredan de `account.chart.template` en Python y devuelven la data estructurada `_get_xx_reports()`).
*   Utilizá etiquetas fiscales (`account.account.tag`) provistas por el código país para que los reportes de balance y ganancias dinámicos agrupen las cuentas correctamente sin hardcodear IDs.