# Odoo 18.0 — Backend Tips & Tricks (Expert)

This guide provides advanced backend techniques, ready-to-use code snippets, and best practices for Odoo 18.0 module development. Focus areas: models, fields, ORM patterns, performance, security, migration, testing, and packaging.

## Table of contents
- Models & Fields
- Create / Update patterns
- Compute / Inverse / Onchange
- SQL Constraints & `_constraints`
- Access control and record rules
- Performance & scaling
- Data migration & upgrade hints
- Tests & CI
- Packaging and manifest

---

## Models & Fields

- Prefer explicit `_name` for new business models and inherit for extension.
- Use `model_name._inherit = ["mail.thread", "mail.activity.mixin"]` to add chatter features.

Example: simple product extension

```python
# models/product_extension.py
from odoo import models, fields, api

class ProductTemplate(models.Model):
    _inherit = 'product.template'

    engineering_cost = fields.Monetary(string='Engineering Cost')
    total_cost = fields.Monetary(string='Total Cost', compute='_compute_total_cost', store=True)

    @api.depends('engineering_cost', 'standard_price')
    def _compute_total_cost(self):
        for rec in self:
            rec.total_cost = (rec.engineering_cost or 0.0) + (rec.standard_price or 0.0)
```

Tips:
- Use `store=True` for frequently searched computed fields to improve searchability.
- Use `currency_field='currency_id'` on Monetary fields if needed.

---

## Create / Update Patterns

Batch create with `create` using lists to reduce RPC chattiness.

```python
# create many records efficiently
products_vals = [{ 'name': 'A', 'list_price': 10 }, { 'name': 'B', 'list_price': 20 }]
self.env['product.product'].create(products_vals)
```

Use `sudo()` carefully: only when you intentionally bypass access rules for system operations. Prefer explicit `with_context(force_company=company.id)` for multi-company operations.

---

## Compute / Inverse / Onchange

- Use `@api.depends` for compute fields.
- Prefer `@api.onchange` for UI-only reactive changes (not stored).
- Implement inverse functions when users need to edit computed fields.

Example inverse:

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    margin_percentage = fields.Float(string='Margin %', compute='_compute_margin', inverse='_inverse_margin')

    @api.depends('amount_total', 'amount_untaxed')
    def _compute_margin(self):
        for o in self:
            if o.amount_total:
                o.margin_percentage = 100.0 * (o.amount_total - o.amount_untaxed) / (o.amount_total or 1)

    def _inverse_margin(self):
        # inverse example: adjust amount_total proportionally (demo only, not advisable in real invoices)
        for o in self:
            o.amount_total = o.amount_untaxed * (1 + (o.margin_percentage or 0.0) / 100.0)
```

---

## SQL Constraints & Python constraints

- Use `sql_constraints` for database-level uniqueness and speed.
- Use `@api.constrains` for Python logic that requires ORM / relationships.

```python
_sql_constraints = [
    ('code_uniq', 'UNIQUE(code)', 'Code must be unique.'),
]

@api.constrains('start_date', 'end_date')
def _check_dates(self):
    for r in self:
        if r.end_date and r.start_date and r.end_date < r.start_date:
            raise ValidationError('Invalid date range')
```

---

## Access Control and Record Rules

- Keep `security/ir.model.access.csv` minimal and explicit per group.
- Use record rules to scope portal users, salespersons, or company-specific records.

Example `ir.model.access.csv` line:

```
access_my_model_user,my.model.user,model_my_model,base.group_user,1,1,0,0
```

Record rule example:

```xml
<record id="rule_my_model_user_company" model="ir.rule">
    <field name="name">My Model: company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">['|',('company_id','=',user.company_id.id),('company_id','=',False)]</field>
    <field name="groups" eval="[(4, ref('base.group_user'))]"/>
</record>
```

Testing access changes is essential; use `self.env['res.users'].create([...])` fixtures in tests or `with_user(...)
` in code context.

---

## Performance & Scaling

Key strategies:
- Avoid searching within loops. Use one search with domain and map results.
- Use `read` and `read_group` for aggregated queries and to retrieve only required fields.
- Use `mapped` and `filtered` on recordsets rather than manual loops where possible.
- Use `prefetch` provided by Odoo automatically; ensure you access fields in batch to let prefetch work.
- For large imports, use SQL COPY or `cr.copy_expert` and disable expensive ORM features.

Example: efficient summation

```python
# Bad: repeated domain search
for partner in partners:
    partner.total_invoiced = self.env['account.move'].search_count([('partner_id','=',partner.id)])

# Good: group read
data = self.env['account.move'].read_group([('partner_id','in', partners.ids)], ['partner_id'], ['partner_id'])
result = {d['partner_id'][0]: d['partner_id_count'] for d in data}
for partner in partners:
    partner.total_invoiced = result.get(partner.id, 0)
```

- Use background jobs for heavy processing; use `ir.cron` or `queue_job` module (recommended for retry and monitoring).

---

## Data Migration & Upgrade Hints

- For structural changes, provide `pre_init_hook` / `post_init_hook` in `__manifest__.py` where needed.
- Use `rename` operations carefully; add both old and new field until migration complete.
- Keep migration scripts idempotent.

Example post-init hook in `__manifest__.py`:

```python
# __manifest__.py
{
    'post_init_hook': 'post_init_hook',
}

# __init__.py
from . import migration

# migration.py

def post_init_hook(cr, registry):
    env = api.Environment(cr, SUPERUSER_ID, {})
    # migration tasks
```

---

## Tests & Continuous Integration

- Write unit tests for business logic in `tests/test_*.py` using `TransactionCase` and `HttpCase` for controllers.
- Use `title` fixtures and `record.create` with deterministic data.
- Run tests via `odoo-bin --test-enable -i module_name` in CI.

Basic test example:

```python
from odoo.tests.common import TransactionCase

class TestMyModule(TransactionCase):
    def test_compute_field(self):
        p = self.env['product.template'].create({'name': 'T1', 'standard_price': 5})
        self.assertEqual(p.total_cost, 5)
```

---

## Packaging and Manifest Best Practices

- Provide metadata: summary, version, category, website, author.
- Group your assets under `assets` key in `__manifest__.py`.
- Split JS/CSS depending on backend vs frontend vs website.

Example manifest assets:

```python
'assets': {
    'web.assets_backend': [
        'my_module/static/src/js/my_widget.js',
        'my_module/static/src/scss/backend.scss',
    ],
    'web.assets_frontend': [
        'my_module/static/src/js/frontend.js',
    ],
}
```

---

This file should act as a quick reference. See the other files in this folder for frontend, theme and portal-specific advanced tips and code snippets.
