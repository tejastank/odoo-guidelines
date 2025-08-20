# Odoo 18.0 — Portal Tips & Tricks (Expert)

This document covers portal customizations, access control, templates, controllers, and UX patterns for Odoo 18.0.

## Table of contents
- Portal access model
- Controllers and routes
- Templates and widgets
- Security and performance
- Examples

---

## Portal Access Model

- Portal users are `res.users` with `portal` group. Use `portal` and `portal_public` groups appropriately.
- Use `sudo()` carefully; ensure correct record rules are in place.

Example record rule for portal users to access their orders:

```xml
<record model="ir.rule" id="portal_sale_order_rule">
    <field name="name">Portal: sale orders</field>
    <field name="model_id" ref="sale.model_sale_order"/>
    <field name="domain_force">[('partner_id','in', user.partner_id.ids)]</field>
    <field name="groups" eval="[(4, ref('portal.group_portal'))]"/>
</record>
```

---

## Controllers and Routes

- Use controllers for portal pages. Use `@route` with `auth='user'` or `auth='public'` accordingly.
- Prefer `route` return HTML via `request.render` for website pages.

Example controller:

```python
from odoo import http
from odoo.http import request

class PortalOrderController(http.Controller):
    @http.route(['/my/orders'], type='http', auth='user', website=True)
    def my_orders(self):
        orders = request.env['sale.order'].search([('partner_id', '=', request.env.user.partner_id.id)])
        return request.render('my_module.portal_my_orders', {'orders': orders})
```

---

## Templates & Widgets

- Keep templates minimal and use JS for interactive parts.
- Use `data-oe-id` and `t-att` to bind dynamic attributes.

---

## Security & Performance

- Limit recordsets in portal pages to avoid heavy loads; paginate results.
- Cache unauthenticated pages using `http.Cache` headers for public pages.

---

## Examples

- Invoice download link for portal users
- Portal forms that create `sale.order` records with proper `create` access

This file contains practical portal patterns and snippets. Combine with backend rules and frontend components for a full solution.
