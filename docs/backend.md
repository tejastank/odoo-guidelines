# Odoo 18.0 — Backend Tips & Tricks (Expert)

This guide provides advanced backend techniques, ready-to-use code snippets, and best practices for Odoo 18.0 module development. Focus areas: models, fields, ORM patterns, performance, security, migration, testing, and packaging.

## Table of contents
- [Models & Fields Creation](#models--fields-creation)
- [Field Types & Advanced Usage](#field-types--advanced-usage)
- [Views Development Guidelines](#views-development-guidelines)
- [Wizards & Transient Models](#wizards--transient-models)
- [Security: CSV Files & Record Rules](#security-csv-files--record-rules)
- [Cron Jobs & Scheduled Actions](#cron-jobs--scheduled-actions)
- [Logging & Debugging](#logging--debugging)
- [ORM Patterns & Performance](#orm-patterns--performance)
- [Controllers & HTTP Routes](#controllers--http-routes)
- [Technical Architecture](#technical-architecture)
- [Data Migration & Upgrade](#data-migration--upgrade)
- [Testing Frameworks](#testing-frameworks)
- [Module Packaging](#module-packaging)

---

## Models & Fields Creation

### Complete Model Structure

```python
# models/project_task_advanced.py
from odoo import models, fields, api, _
from odoo.exceptions import ValidationError, UserError, AccessError
from datetime import datetime, timedelta
import logging

_logger = logging.getLogger(__name__)

class ProjectTaskAdvanced(models.Model):
    _name = 'project.task.advanced'
    _description = 'Advanced Project Task Management'
    _inherit = ['mail.thread', 'mail.activity.mixin', 'portal.mixin']
    _order = 'priority desc, sequence, date_deadline'
    _rec_name = 'name'
    _rec_names_search = ['name', 'code', 'description']
    _check_company_auto = True
    _mail_post_access = 'read'
    
    # Basic Information
    name = fields.Char(
        string='Task Name', 
        required=True, 
        tracking=True,
        translate=False,
        index=True,
        help="Enter the task name"
    )
    code = fields.Char(
        string='Task Code',
        required=True,
        copy=False,
        default=lambda self: _('New'),
        help="Unique task identifier"
    )
    description = fields.Html(
        string='Description',
        sanitize_attributes=False,
        translate=True
    )
    
    # Status and State Management
    state = fields.Selection([
        ('draft', 'Draft'),
        ('in_progress', 'In Progress'),
        ('review', 'Under Review'),
        ('done', 'Done'),
        ('cancelled', 'Cancelled')
    ], string='Status', default='draft', tracking=True, group_expand='_group_expand_states')
    
    # Relational Fields
    project_id = fields.Many2one(
        'project.project',
        string='Project',
        required=True,
        ondelete='cascade',
        index=True,
        domain="[('company_id', '=', company_id)]"
    )
    user_id = fields.Many2one(
        'res.users',
        string='Assigned to',
        tracking=True,
        domain="[('share', '=', False), ('company_ids', 'in', company_id)]"
    )
    partner_id = fields.Many2one(
        'res.partner',
        string='Customer',
        domain="[('is_company', '=', True)]"
    )
    tag_ids = fields.Many2many(
        'project.task.tag',
        'project_task_tag_rel',
        'task_id', 'tag_id',
        string='Tags'
    )
    
    # Date Fields
    date_deadline = fields.Datetime(
        string='Deadline',
        tracking=True,
        help="Task deadline"
    )
    date_start = fields.Datetime(
        string='Start Date',
        default=fields.Datetime.now
    )
    date_end = fields.Datetime(
        string='End Date',
        readonly=True,
        states={'done': [('readonly', False)]}
    )
    
    # Numeric Fields
    priority = fields.Selection([
        ('0', 'Low'),
        ('1', 'Normal'),
        ('2', 'High'),
        ('3', 'Urgent')
    ], default='1', string='Priority', tracking=True)
    
    sequence = fields.Integer(string='Sequence', default=10)
    progress = fields.Float(
        string='Progress (%)',
        compute='_compute_progress',
        store=True,
        group_operator='avg'
    )
    estimated_hours = fields.Float(string='Initially Planned Hours')
    effective_hours = fields.Float(
        string='Effective Hours',
        compute='_compute_effective_hours',
        store=True
    )
    
    # Boolean Fields
    active = fields.Boolean(default=True)
    is_milestone = fields.Boolean(string='Is Milestone')
    is_blocked = fields.Boolean(string='Is Blocked', tracking=True)
    
    # Company and Security
    company_id = fields.Many2one(
        'res.company',
        string='Company',
        required=True,
        default=lambda self: self.env.company
    )
    
    # Computed Fields
    task_count = fields.Integer(
        string='Subtask Count',
        compute='_compute_task_count'
    )
    is_deadline_exceeded = fields.Boolean(
        string='Deadline Exceeded',
        compute='_compute_deadline_status'
    )
    
    # SQL Constraints
    _sql_constraints = [
        ('check_dates', 'CHECK(date_start <= date_deadline)', 'Start date must be before deadline'),
        ('check_progress', 'CHECK(progress >= 0 AND progress <= 100)', 'Progress must be between 0 and 100'),
        ('unique_code', 'UNIQUE(code, company_id)', 'Task code must be unique per company'),
    ]
    
    # API Methods
    @api.model
    def create(self, vals):
        if vals.get('code', _('New')) == _('New'):
            vals['code'] = self.env['ir.sequence'].next_by_code('project.task.advanced') or _('New')
        return super().create(vals)
    
    def write(self, vals):
        if 'state' in vals and vals['state'] == 'done':
            vals['date_end'] = fields.Datetime.now()
        return super().write(vals)
    
    @api.depends('project_id.task_ids')
    def _compute_task_count(self):
        for task in self:
            task.task_count = len(task.project_id.task_ids)
    
    @api.depends('date_deadline')
    def _compute_deadline_status(self):
        for task in self:
            task.is_deadline_exceeded = (
                task.date_deadline and 
                task.date_deadline < fields.Datetime.now() and 
                task.state not in ['done', 'cancelled']
            )
    
    @api.depends('state', 'estimated_hours', 'effective_hours')
    def _compute_progress(self):
        for task in self:
            if task.state == 'done':
                task.progress = 100.0
            elif task.state == 'cancelled':
                task.progress = 0.0
            elif task.effective_hours and task.estimated_hours:
                task.progress = min(100.0, (task.effective_hours / task.estimated_hours) * 100)
            else:
                task.progress = 0.0
    
    @api.depends('timesheet_ids.unit_amount')
    def _compute_effective_hours(self):
        for task in self:
            task.effective_hours = sum(task.timesheet_ids.mapped('unit_amount'))
    
    # Constraints
    @api.constrains('date_start', 'date_deadline')
    def _check_dates(self):
        for task in self:
            if task.date_start and task.date_deadline and task.date_start > task.date_deadline:
                raise ValidationError(_('Start date cannot be after deadline'))
    
    @api.constrains('user_id', 'project_id')
    def _check_user_project_access(self):
        for task in self:
            if task.user_id and task.project_id:
                if not task.project_id.user_has_access(task.user_id):
                    raise ValidationError(_('Assigned user does not have access to this project'))
    
    # Onchange Methods
    @api.onchange('project_id')
    def _onchange_project_id(self):
        if self.project_id:
            self.partner_id = self.project_id.partner_id
            return {
                'domain': {
                    'user_id': [('id', 'in', self.project_id.member_ids.ids)]
                }
            }
    
    @api.onchange('is_milestone')
    def _onchange_is_milestone(self):
        if self.is_milestone:
            self.estimated_hours = 0
    
    # Business Methods
    def action_start(self):
        """Start the task"""
        for task in self:
            if task.state != 'draft':
                raise UserError(_('Only draft tasks can be started'))
            task.write({
                'state': 'in_progress',
                'date_start': fields.Datetime.now()
            })
            task.message_post(body=_('Task started'))
    
    def action_complete(self):
        """Complete the task"""
        for task in self:
            if task.state not in ['in_progress', 'review']:
                raise UserError(_('Only in-progress or review tasks can be completed'))
            task.write({
                'state': 'done',
                'date_end': fields.Datetime.now(),
                'progress': 100.0
            })
            task.message_post(body=_('Task completed'))
    
    def action_cancel(self):
        """Cancel the task"""
        for task in self:
            if task.state == 'done':
                raise UserError(_('Cannot cancel completed tasks'))
            task.write({'state': 'cancelled'})
            task.message_post(body=_('Task cancelled'))
    
    # Utility Methods
    @api.model
    def _group_expand_states(self, states, domain, order):
        return [key for key, val in type(self).state.selection]
    
    def get_portal_url(self):
        """Get portal URL for this task"""
        return f'/my/tasks/{self.id}'
    
    # Cron Methods
    @api.model
    def _cron_check_deadlines(self):
        """Check for overdue tasks and send notifications"""
        overdue_tasks = self.search([
            ('date_deadline', '<', fields.Datetime.now()),
            ('state', 'not in', ['done', 'cancelled'])
        ])
        
        for task in overdue_tasks:
            task.activity_schedule(
                'mail.mail_activity_data_todo',
                summary=_('Task Overdue'),
                note=_('Task %s is overdue') % task.name,
                user_id=task.user_id.id or task.create_uid.id
            )
```

### Model Inheritance Patterns

```python
# Extend existing model
class SaleOrder(models.Model):
    _inherit = 'sale.order'
    
    custom_field = fields.Char('Custom Field')
    
    def custom_method(self):
        return super().confirm()

# Multiple inheritance
class ProductAdvanced(models.Model):
    _name = 'product.advanced'
    _inherit = ['product.template', 'mail.thread']
    
# Delegation inheritance
class ProductBrand(models.Model):
    _name = 'product.brand'
    _inherits = {'res.partner': 'partner_id'}
    
    partner_id = fields.Many2one('res.partner', required=True, ondelete='cascade')
    brand_code = fields.Char('Brand Code')
```

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

## Field Types & Advanced Usage

### All Field Types with Examples

```python
class CompleteFieldExample(models.Model):
    _name = 'complete.field.example'
    _description = 'Complete Field Type Examples'
    
    # Basic Fields
    char_field = fields.Char(
        string='Text Field',
        size=100,
        required=True,
        translate=True,
        help="Character field with translation"
    )
    
    text_field = fields.Text(
        string='Long Text',
        translate=True
    )
    
    html_field = fields.Html(
        string='HTML Content',
        sanitize_tags=True,
        sanitize_attributes=False,
        strip_style=False
    )
    
    # Numeric Fields
    integer_field = fields.Integer(
        string='Integer',
        default=0,
        group_operator='sum'
    )
    
    float_field = fields.Float(
        string='Float',
        digits=(16, 2),
        group_operator='avg'
    )
    
    monetary_field = fields.Monetary(
        string='Price',
        currency_field='currency_id',
        tracking=True
    )
    
    currency_id = fields.Many2one(
        'res.currency',
        string='Currency',
        default=lambda self: self.env.company.currency_id
    )
    
    # Boolean Fields
    boolean_field = fields.Boolean(
        string='Active',
        default=True,
        help="Indicates if record is active"
    )
    
    # Selection Fields
    state = fields.Selection([
        ('draft', 'Draft'),
        ('confirmed', 'Confirmed'),
        ('done', 'Done'),
        ('cancelled', 'Cancelled')
    ], string='State', default='draft', tracking=True)
    
    priority = fields.Selection(
        selection='_get_priority_selection',
        string='Priority',
        default='normal'
    )
    
    @api.model
    def _get_priority_selection(self):
        return [
            ('low', _('Low')),
            ('normal', _('Normal')),
            ('high', _('High')),
            ('urgent', _('Urgent'))
        ]
    
    # Date and DateTime Fields
    date_field = fields.Date(
        string='Date',
        default=fields.Date.today,
        tracking=True
    )
    
    datetime_field = fields.Datetime(
        string='DateTime',
        default=fields.Datetime.now,
        tracking=True
    )
    
    # Binary Fields
    binary_field = fields.Binary(
        string='File',
        help="Upload any file"
    )
    
    image_field = fields.Image(
        string='Image',
        max_width=1920,
        max_height=1920
    )
    
    # Relational Fields
    many2one_field = fields.Many2one(
        'res.partner',
        string='Partner',
        ondelete='cascade',
        domain="[('is_company', '=', True)]",
        context="{'default_is_company': True}"
    )
    
    one2many_field = fields.One2many(
        'complete.field.line',
        'parent_id',
        string='Lines'
    )
    
    many2many_field = fields.Many2many(
        'res.partner.category',
        'complete_field_category_rel',
        'field_id', 'category_id',
        string='Categories'
    )
    
    # Reference Field
    reference_field = fields.Reference(
        selection='_get_reference_models',
        string='Reference'
    )
    
    @api.model
    def _get_reference_models(self):
        return [
            ('res.partner', 'Partner'),
            ('product.template', 'Product'),
            ('project.project', 'Project')
        ]
    
    # Computed Fields
    computed_field = fields.Char(
        string='Computed',
        compute='_compute_computed_field',
        store=True,
        readonly=False,  # Allow manual edit
        inverse='_inverse_computed_field'
    )
    
    related_field = fields.Char(
        string='Related Field',
        related='many2one_field.name',
        store=True,
        readonly=False
    )
    
    @api.depends('char_field', 'integer_field')
    def _compute_computed_field(self):
        for record in self:
            record.computed_field = f"{record.char_field} - {record.integer_field}"
    
    def _inverse_computed_field(self):
        for record in self:
            if record.computed_field:
                parts = record.computed_field.split(' - ')
                if len(parts) == 2:
                    record.char_field = parts[0]
                    try:
                        record.integer_field = int(parts[1])
                    except ValueError:
                        pass
    
    # Advanced Field Attributes
    readonly_field = fields.Char(
        string='Readonly Field',
        readonly=True,
        states={'draft': [('readonly', False)]}
    )
    
    company_dependent_field = fields.Char(
        string='Company Dependent',
        company_dependent=True
    )
    
    # JSON Field (Odoo 16+)
    json_field = fields.Json(
        string='JSON Data',
        default=dict
    )

class CompleteFieldLine(models.Model):
    _name = 'complete.field.line'
    _description = 'Complete Field Line'
    
    parent_id = fields.Many2one('complete.field.example', ondelete='cascade')
    name = fields.Char('Name')
    value = fields.Float('Value')
```

### Field Advanced Features

```python
class AdvancedFieldFeatures(models.Model):
    _name = 'advanced.field.features'
    
    # Tracking and History
    tracked_field = fields.Char(
        string='Tracked Field',
        tracking=True,  # Shows in chatter
        copy=False      # Don't copy on duplicate
    )
    
    # Groups and Security
    secret_field = fields.Char(
        string='Secret Field',
        groups='base.group_system'  # Only system users can see
    )
    
    # Performance Optimization
    indexed_field = fields.Char(
        string='Indexed Field',
        index=True  # Creates database index
    )
    
    # Search Features
    searchable_field = fields.Char(
        string='Searchable Field',
        search='_search_custom_field'
    )
    
    def _search_custom_field(self, operator, value):
        """Custom search method for field"""
        if operator == 'ilike':
            # Custom search logic
            domain = [
                '|',
                ('name', 'ilike', value),
                ('description', 'ilike', value)
            ]
            records = self.search(domain)
            return [('id', 'in', records.ids)]
        return []
    
    # Constraints and Validation
    email_field = fields.Char(
        string='Email',
        help="Valid email address",
    )
    
    @api.constrains('email_field')
    def _check_email(self):
        import re
        email_regex = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        for record in self:
            if record.email_field and not re.match(email_regex, record.email_field):
                raise ValidationError(_('Invalid email format'))
    
    # Default Value Functions
    sequence_field = fields.Char(
        string='Sequence',
        default=lambda self: self.env['ir.sequence'].next_by_code('advanced.field') or '/',
        copy=False
    )
    
    user_field = fields.Many2one(
        'res.users',
        string='Current User',
        default=lambda self: self.env.user
    )
    
    # Widget Specific Fields
    color_field = fields.Integer(
        string='Color',
        help="Select a color (widget=color)"
    )
    
    percentage_field = fields.Float(
        string='Percentage',
        help="Progress percentage (widget=percentage)"
    )
    
    phone_field = fields.Char(
        string='Phone',
        help="Phone number (widget=phone)"
    )
    
    url_field = fields.Char(
        string='Website',
        help="Website URL (widget=url)"
    )
```

---

## Views Development Guidelines

### Form Views - Complete Examples

```xml
<!-- views/project_task_views.xml -->
<odoo>
    <!-- Form View with Advanced Features -->
    <record id="view_project_task_advanced_form" model="ir.ui.view">
        <field name="name">project.task.advanced.form</field>
        <field name="model">project.task.advanced</field>
        <field name="arch" type="xml">
            <form string="Advanced Task">
                <!-- Header with Status Bar and Buttons -->
                <header>
                    <button name="action_start" type="object" string="Start" 
                            states="draft" class="btn-primary"/>
                    <button name="action_complete" type="object" string="Complete" 
                            states="in_progress,review" class="btn-primary"/>
                    <button name="action_cancel" type="object" string="Cancel" 
                            states="draft,in_progress" class="btn-secondary"/>
                    
                    <field name="state" widget="statusbar" statusbar_visible="draft,in_progress,review,done"/>
                </header>
                
                <!-- Alert Messages -->
                <div class="alert alert-warning" role="alert" 
                     attrs="{'invisible': [('is_deadline_exceeded', '=', False)]}">
                    <strong>Warning!</strong> This task is overdue.
                </div>
                
                <sheet>
                    <!-- Title Section -->
                    <div class="oe_title">
                        <label for="name" class="oe_edit_only" string="Task Name"/>
                        <h1>
                            <field name="name" placeholder="Task Name..." required="1"/>
                        </h1>
                        <label for="code" class="oe_edit_only" string="Reference"/>
                        <h3>
                            <field name="code" readonly="1"/>
                        </h3>
                    </div>
                    
                    <!-- Smart Buttons -->
                    <div class="oe_button_box" name="button_box">
                        <button name="%(action_project_task_subtasks)d" 
                                type="action" 
                                class="oe_stat_button" 
                                icon="fa-tasks"
                                context="{'default_parent_id': active_id}">
                            <field name="task_count" widget="statinfo" string="Subtasks"/>
                        </button>
                        
                        <button name="action_view_timesheet" 
                                type="object" 
                                class="oe_stat_button" 
                                icon="fa-clock-o"
                                attrs="{'invisible': [('effective_hours', '=', 0)]}">
                            <field name="effective_hours" widget="statinfo" string="Hours"/>
                        </button>
                    </div>
                    
                    <!-- Main Content -->
                    <group>
                        <group name="main_details">
                            <field name="project_id" options="{'no_create': True}"/>
                            <field name="user_id" widget="many2one_avatar_user"/>
                            <field name="partner_id" context="{'search_default_customer': 1}"/>
                            <field name="tag_ids" widget="many2many_tags" 
                                   options="{'color_field': 'color', 'no_create_edit': True}"/>
                            <field name="priority" widget="priority"/>
                        </group>
                        
                        <group name="dates_and_progress">
                            <field name="date_start"/>
                            <field name="date_deadline" attrs="{'required': [('state', '!=', 'draft')]}"/>
                            <field name="date_end" readonly="1"/>
                            <field name="progress" widget="percentage"/>
                            <field name="is_milestone"/>
                            <field name="is_blocked" attrs="{'invisible': [('state', 'in', ['done', 'cancelled'])]}"/>
                        </group>
                    </group>
                    
                    <!-- Hours Section -->
                    <group string="Time Tracking" name="time_tracking" 
                           attrs="{'invisible': [('is_milestone', '=', True)]}">
                        <group>
                            <field name="estimated_hours" widget="float_time"/>
                            <field name="effective_hours" widget="float_time"/>
                        </group>
                    </group>
                    
                    <!-- Description -->
                    <notebook>
                        <page string="Description" name="description">
                            <field name="description" widget="html" 
                                   options="{'collaborative': true, 'resizable': true}"/>
                        </page>
                        
                        <page string="Timesheets" name="timesheets" 
                              attrs="{'invisible': [('is_milestone', '=', True)]}">
                            <field name="timesheet_ids" context="{'default_project_id': project_id}">
                                <tree editable="bottom">
                                    <field name="date"/>
                                    <field name="user_id" required="1"/>
                                    <field name="name" required="1"/>
                                    <field name="unit_amount" widget="float_time" sum="Total"/>
                                </tree>
                            </field>
                        </page>
                        
                        <page string="Subtasks" name="subtasks">
                            <field name="child_ids">
                                <tree>
                                    <field name="name"/>
                                    <field name="user_id"/>
                                    <field name="date_deadline"/>
                                    <field name="state"/>
                                </tree>
                            </field>
                        </page>
                    </notebook>
                </sheet>
                
                <!-- Chatter -->
                <div class="oe_chatter">
                    <field name="message_follower_ids" groups="base.group_user"/>
                    <field name="activity_ids"/>
                    <field name="message_ids"/>
                </div>
            </form>
        </field>
    </record>
    
    <!-- Tree/List View -->
    <record id="view_project_task_advanced_tree" model="ir.ui.view">
        <field name="name">project.task.advanced.tree</field>
        <field name="model">project.task.advanced</field>
        <field name="arch" type="xml">
            <tree string="Tasks" 
                  decoration-muted="state=='cancelled'" 
                  decoration-danger="is_deadline_exceeded==True"
                  decoration-success="state=='done'"
                  sample="1">
                
                <!-- Searchable and sortable fields -->
                <field name="sequence" widget="handle"/>
                <field name="code"/>
                <field name="name"/>
                <field name="project_id"/>
                <field name="user_id" widget="many2one_avatar_user"/>
                <field name="date_deadline"/>
                <field name="priority" widget="priority"/>
                <field name="state" widget="badge" 
                       decoration-info="state=='draft'"
                       decoration-warning="state=='in_progress'"
                       decoration-success="state=='done'"/>
                <field name="progress" widget="progressbar"/>
                <field name="effective_hours" widget="float_time" sum="Total Hours"/>
                
                <!-- Invisible fields for decorations -->
                <field name="is_deadline_exceeded" invisible="1"/>
                
                <!-- Optional fields -->
                <field name="tag_ids" widget="many2many_tags" optional="hide"/>
                <field name="estimated_hours" widget="float_time" optional="hide"/>
                
                <!-- Control buttons -->
                <button name="action_start" type="object" 
                        icon="fa-play" title="Start Task"
                        attrs="{'invisible': [('state', '!=', 'draft')]}"/>
                <button name="action_complete" type="object" 
                        icon="fa-check" title="Complete Task"
                        attrs="{'invisible': [('state', 'not in', ['in_progress', 'review'])]}"/>
            </tree>
        </field>
    </record>
    
    <!-- Search View -->
    <record id="view_project_task_advanced_search" model="ir.ui.view">
        <field name="name">project.task.advanced.search</field>
        <field name="model">project.task.advanced</field>
        <field name="arch" type="xml">
            <search string="Search Tasks">
                <!-- Search fields -->
                <field name="name" string="Task" 
                       filter_domain="['|', ('name', 'ilike', self), ('code', 'ilike', self)]"/>
                <field name="project_id"/>
                <field name="user_id"/>
                <field name="partner_id"/>
                <field name="tag_ids"/>
                
                <!-- Filters -->
                <filter string="My Tasks" name="my_tasks" 
                        domain="[('user_id', '=', uid)]"/>
                <filter string="Unassigned" name="unassigned" 
                        domain="[('user_id', '=', False)]"/>
                <filter string="Overdue" name="overdue" 
                        domain="[('is_deadline_exceeded', '=', True)]"/>
                <filter string="Milestones" name="milestones" 
                        domain="[('is_milestone', '=', True)]"/>
                
                <separator/>
                
                <filter string="Draft" name="draft" domain="[('state', '=', 'draft')]"/>
                <filter string="In Progress" name="in_progress" domain="[('state', '=', 'in_progress')]"/>
                <filter string="Done" name="done" domain="[('state', '=', 'done')]"/>
                
                <separator/>
                
                <filter string="High Priority" name="high_priority" 
                        domain="[('priority', 'in', ['2', '3'])]"/>
                <filter string="This Week" name="this_week" 
                        domain="[('date_deadline', '&gt;=', (context_today() - datetime.timedelta(days=context_today().weekday())).strftime('%Y-%m-%d')),
                                ('date_deadline', '&lt;', (context_today() + datetime.timedelta(days=7-context_today().weekday())).strftime('%Y-%m-%d'))]"/>
                
                <!-- Group By -->
                <group expand="0" string="Group By">
                    <filter string="Project" name="group_project" context="{'group_by': 'project_id'}"/>
                    <filter string="Assigned User" name="group_user" context="{'group_by': 'user_id'}"/>
                    <filter string="State" name="group_state" context="{'group_by': 'state'}"/>
                    <filter string="Priority" name="group_priority" context="{'group_by': 'priority'}"/>
                    <filter string="Deadline" name="group_deadline" context="{'group_by': 'date_deadline:week'}"/>
                </group>
            </search>
        </field>
    </record>
    
    <!-- Kanban View -->
    <record id="view_project_task_advanced_kanban" model="ir.ui.view">
        <field name="name">project.task.advanced.kanban</field>
        <field name="model">project.task.advanced</field>
        <field name="arch" type="xml">
            <kanban default_group_by="state" 
                    class="o_kanban_small_column" 
                    on_create="quick_create"
                    quick_create_view="project_task_quick_create_form"
                    examples="1">
                
                <field name="color"/>
                <field name="name"/>
                <field name="user_id"/>
                <field name="project_id"/>
                <field name="date_deadline"/>
                <field name="priority"/>
                <field name="state"/>
                <field name="tag_ids"/>
                <field name="progress"/>
                <field name="effective_hours"/>
                <field name="estimated_hours"/>
                <field name="is_deadline_exceeded"/>
                
                <progressbar field="state" colors='{"done": "success", "in_progress": "warning", "cancelled": "danger"}'/>
                
                <templates>
                    <t t-name="kanban-box">
                        <div t-attf-class="oe_kanban_color_#{kanban_getcolor(record.color.raw_value)} oe_kanban_card oe_kanban_global_click">
                            <!-- Card Header -->
                            <div class="o_kanban_record_top">
                                <div class="o_kanban_record_headings">
                                    <strong class="o_kanban_record_title">
                                        <t t-esc="record.name.value"/>
                                    </strong>
                                    <small class="text-muted">
                                        <t t-esc="record.code.value"/>
                                    </small>
                                </div>
                                <div class="o_kanban_manage_button_section">
                                    <a class="o_kanban_manage_toggle_button" href="#" tabindex="-1">
                                        <i class="fa fa-ellipsis-v" role="img" aria-label="Manage" title="Manage"/>
                                    </a>
                                </div>
                            </div>
                            
                            <!-- Card Body -->
                            <div class="o_kanban_record_body">
                                <field name="tag_ids" widget="many2many_tags" options="{'color_field': 'color'}"/>
                                
                                <div class="o_kanban_record_bottom">
                                    <div class="oe_kanban_bottom_left">
                                        <field name="priority" widget="priority"/>
                                        <t t-if="record.date_deadline.raw_value">
                                            <span t-attf-class="#{record.is_deadline_exceeded.raw_value ? 'text-danger' : ''}">
                                                <i class="fa fa-clock-o" title="Deadline"/>
                                                <t t-esc="record.date_deadline.value"/>
                                            </span>
                                        </t>
                                    </div>
                                    <div class="oe_kanban_bottom_right">
                                        <field name="user_id" widget="many2one_avatar_user"/>
                                    </div>
                                </div>
                                
                                <!-- Progress Bar -->
                                <div class="o_kanban_record_progress">
                                    <field name="progress" widget="progressbar" title="Progress"/>
                                </div>
                            </div>
                            
                            <!-- Dropdown Menu -->
                            <div class="o_kanban_manage_button_section o_kanban_manage_view">
                                <div class="o_kanban_card_manage_pane dropdown-menu" role="menu">
                                    <a role="menuitem" type="edit" class="dropdown-item">Edit</a>
                                    <a role="menuitem" type="delete" class="dropdown-item">Delete</a>
                                    <div role="separator" class="dropdown-divider"/>
                                    <a role="menuitem" name="action_start" type="object" class="dropdown-item"
                                       t-if="record.state.raw_value == 'draft'">Start</a>
                                    <a role="menuitem" name="action_complete" type="object" class="dropdown-item"
                                       t-if="record.state.raw_value in ['in_progress', 'review']">Complete</a>
                                </div>
                            </div>
                        </div>
                    </t>
                </templates>
            </kanban>
        </field>
    </record>
    
    <!-- Calendar View -->
    <record id="view_project_task_advanced_calendar" model="ir.ui.view">
        <field name="name">project.task.advanced.calendar</field>
        <field name="model">project.task.advanced</field>
        <field name="arch" type="xml">
            <calendar string="Tasks Calendar" 
                      date_start="date_start" 
                      date_stop="date_deadline"
                      color="user_id"
                      event_open_popup="true"
                      quick_add="false"
                      mode="month">
                <field name="name"/>
                <field name="project_id"/>
                <field name="user_id" filters="1" invisible="1"/>
                <field name="state"/>
            </calendar>
        </field>
    </record>
    
    <!-- Graph/Chart View -->
    <record id="view_project_task_advanced_graph" model="ir.ui.view">
        <field name="name">project.task.advanced.graph</field>
        <field name="model">project.task.advanced</field>
        <field name="arch" type="xml">
            <graph string="Task Analysis" type="bar" stacked="True">
                <field name="project_id" type="row"/>
                <field name="state" type="col"/>
                <field name="effective_hours" type="measure"/>
            </graph>
        </field>
    </record>
    
    <!-- Pivot View -->
    <record id="view_project_task_advanced_pivot" model="ir.ui.view">
        <field name="name">project.task.advanced.pivot</field>
        <field name="model">project.task.advanced</field>
        <field name="arch" type="xml">
            <pivot string="Task Pivot Analysis" display_quantity="true">
                <field name="project_id" type="row"/>
                <field name="user_id" type="row"/>
                <field name="state" type="col"/>
                <field name="effective_hours" type="measure"/>
                <field name="estimated_hours" type="measure"/>
            </pivot>
        </field>
    </record>
</odoo>
```

### Action and Menu Configuration

```xml
<!-- Actions -->
<record id="action_project_task_advanced" model="ir.actions.act_window">
    <field name="name">Advanced Tasks</field>
    <field name="res_model">project.task.advanced</field>
    <field name="view_mode">kanban,tree,form,calendar,graph,pivot</field>
    <field name="context">{
        'search_default_my_tasks': 1,
        'search_default_group_state': 1
    }</field>
    <field name="help" type="html">
        <p class="o_view_nocontent_smiling_face">
            Create your first task!
        </p>
        <p>
            Track your project tasks efficiently with advanced features.
        </p>
    </field>
</record>

<!-- Menu Items -->
<menuitem id="menu_project_task_advanced"
          name="Advanced Tasks"
          parent="project.menu_project_management"
          action="action_project_task_advanced"
          sequence="10"/>

<!-- Server Actions -->
<record id="action_server_task_bulk_start" model="ir.actions.server">
    <field name="name">Bulk Start Tasks</field>
    <field name="model_id" ref="model_project_task_advanced"/>
    <field name="binding_model_id" ref="model_project_task_advanced"/>
    <field name="binding_view_types">list</field>
    <field name="state">code</field>
    <field name="code">
if records:
    for record in records.filtered(lambda r: r.state == 'draft'):
        record.action_start()
    </field>
</record>
```

---

## Wizards & Transient Models

### Complete Wizard Example

```python
# wizards/task_batch_update_wizard.py
from odoo import models, fields, api, _
from odoo.exceptions import ValidationError, UserError
from datetime import datetime, timedelta
import logging

_logger = logging.getLogger(__name__)

class TaskBatchUpdateWizard(models.TransientModel):
    _name = 'task.batch.update.wizard'
    _description = 'Batch Update Tasks Wizard'
    
    # Wizard Fields
    name = fields.Char(string='Wizard Name', default='Batch Update Tasks')
    update_type = fields.Selection([
        ('status', 'Update Status'),
        ('assignment', 'Update Assignment'),
        ('dates', 'Update Dates'),
        ('tags', 'Update Tags'),
        ('custom', 'Custom Update')
    ], string='Update Type', required=True, default='status')
    
    # Status Update Fields
    new_state = fields.Selection([
        ('draft', 'Draft'),
        ('in_progress', 'In Progress'),
        ('review', 'Under Review'),
        ('done', 'Done'),
        ('cancelled', 'Cancelled')
    ], string='New Status')
    
    # Assignment Fields
    new_user_id = fields.Many2one('res.users', string='Assign To')
    new_project_id = fields.Many2one('project.project', string='Move to Project')
    
    # Date Fields
    new_deadline = fields.Datetime(string='New Deadline')
    deadline_offset_days = fields.Integer(string='Deadline Offset (Days)', default=0)
    update_deadline_mode = fields.Selection([
        ('fixed', 'Fixed Date'),
        ('offset', 'Offset from Current')
    ], string='Deadline Mode', default='fixed')
    
    # Tags Fields
    tag_operation = fields.Selection([
        ('add', 'Add Tags'),
        ('remove', 'Remove Tags'),
        ('replace', 'Replace Tags')
    ], string='Tag Operation', default='add')
    tag_ids = fields.Many2many('project.task.tag', string='Tags')
    
    # Progress Fields
    new_progress = fields.Float(string='New Progress (%)', help="Set progress percentage")
    
    # Selection and Summary
    task_ids = fields.Many2many('project.task.advanced', string='Tasks to Update')
    task_count = fields.Integer(string='Task Count', compute='_compute_task_count')
    
    # Summary and Preview
    preview_changes = fields.Html(string='Preview Changes', compute='_compute_preview_changes')
    confirm_changes = fields.Boolean(string='I confirm these changes', default=False)
    
    # Results
    success_count = fields.Integer(string='Successfully Updated', readonly=True)
    error_count = fields.Integer(string='Errors', readonly=True)
    error_messages = fields.Text(string='Error Messages', readonly=True)
    
    @api.depends('task_ids')
    def _compute_task_count(self):
        for wizard in self:
            wizard.task_count = len(wizard.task_ids)
    
    @api.depends('update_type', 'new_state', 'new_user_id', 'new_deadline', 'tag_ids', 'task_ids')
    def _compute_preview_changes(self):
        for wizard in self:
            preview_html = "<h4>Preview of Changes:</h4><ul>"
            
            if wizard.update_type == 'status' and wizard.new_state:
                preview_html += f"<li>Status will be changed to: <strong>{dict(wizard._fields['new_state'].selection).get(wizard.new_state)}</strong></li>"
            
            if wizard.update_type == 'assignment':
                if wizard.new_user_id:
                    preview_html += f"<li>Assigned to: <strong>{wizard.new_user_id.name}</strong></li>"
                if wizard.new_project_id:
                    preview_html += f"<li>Moved to project: <strong>{wizard.new_project_id.name}</strong></li>"
            
            if wizard.update_type == 'dates' and wizard.new_deadline:
                preview_html += f"<li>Deadline will be set to: <strong>{wizard.new_deadline}</strong></li>"
            
            if wizard.update_type == 'tags' and wizard.tag_ids:
                tag_names = ', '.join(wizard.tag_ids.mapped('name'))
                preview_html += f"<li>Tags operation ({wizard.tag_operation}): <strong>{tag_names}</strong></li>"
            
            preview_html += f"<li>Number of tasks to update: <strong>{wizard.task_count}</strong></li>"
            preview_html += "</ul>"
            
            wizard.preview_changes = preview_html
    
    @api.model
    def default_get(self, fields_list):
        """Set default values from context"""
        result = super().default_get(fields_list)
        
        # Get selected tasks from context
        if self.env.context.get('active_model') == 'project.task.advanced':
            active_ids = self.env.context.get('active_ids', [])
            if active_ids:
                result['task_ids'] = [(6, 0, active_ids)]
        
        return result
    
    @api.onchange('update_type')
    def _onchange_update_type(self):
        """Clear fields when update type changes"""
        if self.update_type != 'status':
            self.new_state = False
        if self.update_type != 'assignment':
            self.new_user_id = False
            self.new_project_id = False
        if self.update_type != 'dates':
            self.new_deadline = False
            self.deadline_offset_days = 0
        if self.update_type != 'tags':
            self.tag_ids = [(5, 0, 0)]
    
    @api.onchange('update_deadline_mode')
    def _onchange_deadline_mode(self):
        """Clear deadline fields based on mode"""
        if self.update_deadline_mode == 'fixed':
            self.deadline_offset_days = 0
        else:
            self.new_deadline = False
    
    def action_preview(self):
        """Show preview of changes"""
        self.ensure_one()
        if not self.task_ids:
            raise ValidationError(_('Please select at least one task to update.'))
        
        return {
            'type': 'ir.actions.act_window',
            'res_model': 'task.batch.update.wizard',
            'res_id': self.id,
            'view_mode': 'form',
            'target': 'new',
            'context': {'show_preview': True}
        }
    
    def action_update_tasks(self):
        """Execute the batch update"""
        self.ensure_one()
        
        if not self.task_ids:
            raise ValidationError(_('Please select at least one task to update.'))
        
        if not self.confirm_changes:
            raise ValidationError(_('Please confirm the changes before proceeding.'))
        
        success_count = 0
        error_count = 0
        error_messages = []
        
        for task in self.task_ids:
            try:
                update_vals = self._prepare_update_values(task)
                if update_vals:
                    task.write(update_vals)
                    success_count += 1
                    
                    # Log the change
                    message = self._prepare_log_message(update_vals)
                    task.message_post(body=message)
                    
            except Exception as e:
                error_count += 1
                error_msg = f"Task {task.name}: {str(e)}"
                error_messages.append(error_msg)
                _logger.error(f"Batch update error: {error_msg}")
        
        # Update wizard with results
        self.write({
            'success_count': success_count,
            'error_count': error_count,
            'error_messages': '\n'.join(error_messages)
        })
        
        # Show results
        if error_count == 0:
            message = _('Successfully updated %d tasks.') % success_count
            message_type = 'success'
        else:
            message = _('Updated %d tasks with %d errors.') % (success_count, error_count)
            message_type = 'warning'
        
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'title': _('Batch Update Complete'),
                'message': message,
                'type': message_type,
                'sticky': True,
            }
        }
    
    def _prepare_update_values(self, task):
        """Prepare update values for a specific task"""
        self.ensure_one()
        vals = {}
        
        if self.update_type == 'status' and self.new_state:
            vals['state'] = self.new_state
        
        elif self.update_type == 'assignment':
            if self.new_user_id:
                vals['user_id'] = self.new_user_id.id
            if self.new_project_id:
                vals['project_id'] = self.new_project_id.id
        
        elif self.update_type == 'dates':
            if self.update_deadline_mode == 'fixed' and self.new_deadline:
                vals['date_deadline'] = self.new_deadline
            elif self.update_deadline_mode == 'offset' and self.deadline_offset_days:
                if task.date_deadline:
                    new_deadline = task.date_deadline + timedelta(days=self.deadline_offset_days)
                    vals['date_deadline'] = new_deadline
        
        elif self.update_type == 'tags' and self.tag_ids:
            if self.tag_operation == 'add':
                vals['tag_ids'] = [(4, tag.id) for tag in self.tag_ids]
            elif self.tag_operation == 'remove':
                vals['tag_ids'] = [(3, tag.id) for tag in self.tag_ids]
            elif self.tag_operation == 'replace':
                vals['tag_ids'] = [(6, 0, self.tag_ids.ids)]
        
        return vals
    
    def _prepare_log_message(self, update_vals):
        """Prepare log message for task update"""
        self.ensure_one()
        changes = []
        
        if 'state' in update_vals:
            state_label = dict(self._fields['new_state'].selection).get(update_vals['state'])
            changes.append(f"Status changed to {state_label}")
        
        if 'user_id' in update_vals:
            user = self.env['res.users'].browse(update_vals['user_id'])
            changes.append(f"Assigned to {user.name}")
        
        if 'project_id' in update_vals:
            project = self.env['project.project'].browse(update_vals['project_id'])
            changes.append(f"Moved to project {project.name}")
        
        if 'date_deadline' in update_vals:
            changes.append(f"Deadline changed to {update_vals['date_deadline']}")
        
        if 'tag_ids' in update_vals:
            changes.append(f"Tags updated ({self.tag_operation})")
        
        return f"Batch update: {', '.join(changes)}"
    
    def action_cancel(self):
        """Cancel wizard"""
        return {'type': 'ir.actions.act_window_close'}

# Advanced Wizard with Steps
class MultiStepWizard(models.TransientModel):
    _name = 'multi.step.wizard'
    _description = 'Multi-Step Wizard Example'
    
    # Step Management
    current_step = fields.Integer(string='Current Step', default=1)
    total_steps = fields.Integer(string='Total Steps', default=3)
    
    # Step 1: Basic Information
    step1_name = fields.Char(string='Name', required=True)
    step1_description = fields.Text(string='Description')
    
    # Step 2: Configuration
    step2_option1 = fields.Boolean(string='Option 1')
    step2_option2 = fields.Selection([
        ('a', 'Option A'),
        ('b', 'Option B'),
        ('c', 'Option C')
    ], string='Choose Option')
    
    # Step 3: Confirmation
    step3_confirmation = fields.Boolean(string='I confirm all settings')
    step3_summary = fields.Html(string='Summary', compute='_compute_summary')
    
    @api.depends('step1_name', 'step1_description', 'step2_option1', 'step2_option2')
    def _compute_summary(self):
        for wizard in self:
            summary = f"""
            <h4>Configuration Summary:</h4>
            <ul>
                <li>Name: {wizard.step1_name or 'Not set'}</li>
                <li>Description: {wizard.step1_description or 'Not set'}</li>
                <li>Option 1: {'Yes' if wizard.step2_option1 else 'No'}</li>
                <li>Option 2: {wizard.step2_option2 or 'Not selected'}</li>
            </ul>
            """
            wizard.step3_summary = summary
    
    def action_next_step(self):
        """Move to next step"""
        self.ensure_one()
        if self.current_step < self.total_steps:
            self.current_step += 1
        return self._reopen_wizard()
    
    def action_previous_step(self):
        """Move to previous step"""
        self.ensure_one()
        if self.current_step > 1:
            self.current_step -= 1
        return self._reopen_wizard()
    
    def action_finish(self):
        """Finish wizard and process data"""
        self.ensure_one()
        
        # Validate final step
        if not self.step3_confirmation:
            raise ValidationError(_('Please confirm the configuration.'))
        
        # Process the wizard data
        self._process_wizard_data()
        
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'title': _('Success'),
                'message': _('Configuration completed successfully.'),
                'type': 'success',
            }
        }
    
    def _reopen_wizard(self):
        """Reopen wizard with current state"""
        return {
            'type': 'ir.actions.act_window',
            'res_model': self._name,
            'res_id': self.id,
            'view_mode': 'form',
            'target': 'new',
        }
    
    def _process_wizard_data(self):
        """Process wizard data (implement business logic here)"""
        # Create or update records based on wizard data
        pass
```

### Wizard Views

```xml
<!-- views/wizard_views.xml -->
<odoo>
    <!-- Batch Update Wizard Form -->
    <record id="view_task_batch_update_wizard_form" model="ir.ui.view">
        <field name="name">task.batch.update.wizard.form</field>
        <field name="model">task.batch.update.wizard</field>
        <field name="arch" type="xml">
            <form string="Batch Update Tasks">
                <header>
                    <button name="action_preview" string="Preview Changes" 
                            type="object" class="btn-secondary"
                            attrs="{'invisible': [('task_count', '=', 0)]}"/>
                </header>
                
                <sheet>
                    <div class="oe_title">
                        <h1>
                            <field name="name" readonly="1"/>
                        </h1>
                    </div>
                    
                    <group>
                        <group name="selection">
                            <field name="task_count" readonly="1"/>
                            <field name="update_type" widget="radio"/>
                        </group>
                    </group>
                    
                    <!-- Update Type Specific Fields -->
                    <group string="Update Details" name="update_details">
                        <!-- Status Update -->
                        <group attrs="{'invisible': [('update_type', '!=', 'status')]}">
                            <field name="new_state" required="1" 
                                   attrs="{'required': [('update_type', '=', 'status')]}"/>
                        </group>
                        
                        <!-- Assignment Update -->
                        <group attrs="{'invisible': [('update_type', '!=', 'assignment')]}">
                            <field name="new_user_id"/>
                            <field name="new_project_id"/>
                        </group>
                        
                        <!-- Dates Update -->
                        <group attrs="{'invisible': [('update_type', '!=', 'dates')]}">
                            <field name="update_deadline_mode" widget="radio"/>
                            <field name="new_deadline" 
                                   attrs="{'invisible': [('update_deadline_mode', '!=', 'fixed')],
                                          'required': [('update_type', '=', 'dates'), ('update_deadline_mode', '=', 'fixed')]}"/>
                            <field name="deadline_offset_days" 
                                   attrs="{'invisible': [('update_deadline_mode', '!=', 'offset')],
                                          'required': [('update_type', '=', 'dates'), ('update_deadline_mode', '=', 'offset')]}"/>
                        </group>
                        
                        <!-- Tags Update -->
                        <group attrs="{'invisible': [('update_type', '!=', 'tags')]}">
                            <field name="tag_operation" widget="radio"/>
                            <field name="tag_ids" widget="many2many_tags" 
                                   attrs="{'required': [('update_type', '=', 'tags')]}"/>
                        </group>
                    </group>
                    
                    <!-- Preview Section -->
                    <group string="Preview" name="preview" 
                           attrs="{'invisible': [('preview_changes', '=', False)]}">
                        <field name="preview_changes" nolabel="1"/>
                        <field name="confirm_changes"/>
                    </group>
                    
                    <!-- Results Section -->
                    <group string="Results" name="results" 
                           attrs="{'invisible': [('success_count', '=', 0), ('error_count', '=', 0)]}">
                        <field name="success_count" readonly="1"/>
                        <field name="error_count" readonly="1"/>
                        <field name="error_messages" readonly="1" 
                               attrs="{'invisible': [('error_messages', '=', False)]}"/>
                    </group>
                    
                    <!-- Selected Tasks -->
                    <notebook>
                        <page string="Selected Tasks" name="tasks">
                            <field name="task_ids" nolabel="1">
                                <tree>
                                    <field name="name"/>
                                    <field name="project_id"/>
                                    <field name="user_id"/>
                                    <field name="state"/>
                                    <field name="date_deadline"/>
                                </tree>
                            </field>
                        </page>
                    </notebook>
                </sheet>
                
                <footer>
                    <button name="action_update_tasks" string="Update Tasks" 
                            type="object" class="btn-primary"
                            attrs="{'invisible': ['|', ('task_count', '=', 0), ('confirm_changes', '=', False)]}"/>
                    <button name="action_cancel" string="Cancel" 
                            type="object" class="btn-secondary"/>
                </footer>
            </form>
        </field>
    </record>
    
    <!-- Multi-Step Wizard Form -->
    <record id="view_multi_step_wizard_form" model="ir.ui.view">
        <field name="name">multi.step.wizard.form</field>
        <field name="model">multi.step.wizard</field>
        <field name="arch" type="xml">
            <form string="Multi-Step Configuration">
                <header>
                    <!-- Progress Bar -->
                    <div class="o_wizard_progressbar">
                        <div class="progress">
                            <div class="progress-bar" role="progressbar" 
                                 t-attf-style="width: #{(current_step / total_steps) * 100}%"/>
                        </div>
                        <div class="o_wizard_step_info">
                            Step <field name="current_step" readonly="1"/> of <field name="total_steps" readonly="1"/>
                        </div>
                    </div>
                </header>
                
                <sheet>
                    <!-- Step 1: Basic Information -->
                    <div attrs="{'invisible': [('current_step', '!=', 1)]}">
                        <h2>Step 1: Basic Information</h2>
                        <group>
                            <field name="step1_name"/>
                            <field name="step1_description"/>
                        </group>
                    </div>
                    
                    <!-- Step 2: Configuration -->
                    <div attrs="{'invisible': [('current_step', '!=', 2)]}">
                        <h2>Step 2: Configuration</h2>
                        <group>
                            <field name="step2_option1"/>
                            <field name="step2_option2"/>
                        </group>
                    </div>
                    
                    <!-- Step 3: Confirmation -->
                    <div attrs="{'invisible': [('current_step', '!=', 3)]}">
                        <h2>Step 3: Confirmation</h2>
                        <field name="step3_summary" nolabel="1"/>
                        <group>
                            <field name="step3_confirmation"/>
                        </group>
                    </div>
                </sheet>
                
                <footer>
                    <button name="action_previous_step" string="Previous" 
                            type="object" class="btn-secondary"
                            attrs="{'invisible': [('current_step', '=', 1)]}"/>
                    <button name="action_next_step" string="Next" 
                            type="object" class="btn-primary"
                            attrs="{'invisible': [('current_step', '=', total_steps)]}"/>
                    <button name="action_finish" string="Finish" 
                            type="object" class="btn-primary"
                            attrs="{'invisible': [('current_step', '!=', total_steps)]}"/>
                    <button name="action_cancel" string="Cancel" 
                            type="object" class="btn-secondary"/>
                </footer>
            </form>
        </field>
    </record>
    
    <!-- Wizard Actions -->
    <record id="action_task_batch_update_wizard" model="ir.actions.act_window">
        <field name="name">Batch Update Tasks</field>
        <field name="res_model">task.batch.update.wizard</field>
        <field name="view_mode">form</field>
        <field name="target">new</field>
        <field name="binding_model_id" ref="model_project_task_advanced"/>
        <field name="binding_view_types" ref="list"/>
    </record>
</odoo>
```

---

## Security: CSV Files & Record Rules

### Access Rights Configuration

```csv
# security/ir.model.access.csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_project_task_advanced_user,project.task.advanced.user,model_project_task_advanced,base.group_user,1,1,1,0
access_project_task_advanced_manager,project.task.advanced.manager,model_project_task_advanced,project.group_project_manager,1,1,1,1
access_project_task_advanced_portal,project.task.advanced.portal,model_project_task_advanced,base.group_portal,1,0,0,0
```

### Record Rules

```xml
<!-- security/security.xml -->
<odoo>
    <data noupdate="1">
        <!-- Users can only see their tasks -->
        <record id="rule_task_user_own" model="ir.rule">
            <field name="name">Task: User Own</field>
            <field name="model_id" ref="model_project_task_advanced"/>
            <field name="domain_force">[('user_id', '=', user.id)]</field>
            <field name="groups" eval="[(4, ref('base.group_user'))]"/>
        </record>
        
        <!-- Multi-company rule -->
        <record id="rule_task_company" model="ir.rule">
            <field name="name">Task: Multi-company</field>
            <field name="model_id" ref="model_project_task_advanced"/>
            <field name="domain_force">[('company_id', 'in', company_ids)]</field>
            <field name="global" eval="True"/>
        </record>
    </data>
</odoo>
```

---

## Cron Jobs & Scheduled Actions

### Complete Cron Implementation

```python
# models/task_cron.py
from odoo import models, api, fields, _
from datetime import datetime, timedelta
import logging

_logger = logging.getLogger(__name__)

class ProjectTaskAdvanced(models.Model):
    _inherit = 'project.task.advanced'
    
    @api.model
    def _cron_check_overdue_tasks(self):
        """Check for overdue tasks and send notifications"""
        try:
            overdue_tasks = self.search([
                ('date_deadline', '<', fields.Datetime.now()),
                ('state', 'not in', ['done', 'cancelled'])
            ])
            
            for task in overdue_tasks:
                self._send_overdue_notification(task)
                
            _logger.info(f"Processed {len(overdue_tasks)} overdue tasks")
            
        except Exception as e:
            _logger.error(f"Error in overdue task cron: {e}")
    
    @api.model
    def _cron_auto_assign_tasks(self):
        """Auto-assign unassigned tasks based on workload"""
        try:
            unassigned_tasks = self.search([
                ('user_id', '=', False),
                ('state', '=', 'draft'),
                ('project_id.auto_assign', '=', True)
            ])
            
            for task in unassigned_tasks:
                best_user = self._find_best_assignee(task)
                if best_user:
                    task.user_id = best_user
                    task.message_post(
                        body=f"Task auto-assigned to {best_user.name}"
                    )
            
            _logger.info(f"Auto-assigned {len(unassigned_tasks)} tasks")
            
        except Exception as e:
            _logger.error(f"Error in auto-assign cron: {e}")
    
    def _find_best_assignee(self, task):
        """Find best user to assign task based on workload"""
        project_members = task.project_id.member_ids
        if not project_members:
            return False
        
        # Calculate workload for each member
        workload = {}
        for user in project_members:
            active_tasks = self.search_count([
                ('user_id', '=', user.id),
                ('state', 'in', ['draft', 'in_progress'])
            ])
            workload[user.id] = active_tasks
        
        # Return user with minimum workload
        best_user_id = min(workload, key=workload.get)
        return self.env['res.users'].browse(best_user_id)
```

### Cron Job Configuration

```xml
<!-- data/cron_data.xml -->
<odoo>
    <data noupdate="1">
        
        <!-- Daily overdue task check -->
        <record id="cron_check_overdue_tasks" model="ir.cron">
            <field name="name">Check Overdue Tasks</field>
            <field name="model_id" ref="model_project_task_advanced"/>
            <field name="state">code</field>
            <field name="code">model._cron_check_overdue_tasks()</field>
            <field name="interval_number">1</field>
            <field name="interval_type">days</field>
            <field name="nextcall" eval="(DateTime.now() + timedelta(hours=9)).strftime('%Y-%m-%d %H:%M:%S')"/>
            <field name="priority">5</field>
            <field name="user_id" ref="base.user_root"/>
            <field name="active">True</field>
        </record>
        
        <!-- Hourly auto-assignment -->
        <record id="cron_auto_assign_tasks" model="ir.cron">
            <field name="name">Auto Assign Tasks</field>
            <field name="model_id" ref="model_project_task_advanced"/>
            <field name="state">code</field>
            <field name="code">model._cron_auto_assign_tasks()</field>
            <field name="interval_number">1</field>
            <field name="interval_type">hours</field>
            <field name="nextcall" eval="DateTime.now().strftime('%Y-%m-%d %H:%M:%S')"/>
            <field name="priority">10</field>
            <field name="user_id" ref="base.user_root"/>
            <field name="active">False</field>
        </record>
        
    </data>
</odoo>
```

---

## Logging & Debugging

### Comprehensive Logging Setup

```python
# models/logging_helper.py
import logging
import functools
from odoo import models, api

# Configure module logger
_logger = logging.getLogger(__name__)

def log_method_call(func):
    """Decorator to log method calls"""
    @functools.wraps(func)
    def wrapper(self, *args, **kwargs):
        _logger.debug(f"Calling {func.__name__} with args: {args}, kwargs: {kwargs}")
        try:
            result = func(self, *args, **kwargs)
            _logger.debug(f"{func.__name__} completed successfully")
            return result
        except Exception as e:
            _logger.error(f"Error in {func.__name__}: {e}")
            raise
    return wrapper

class ProjectTaskAdvanced(models.Model):
    _inherit = 'project.task.advanced'
    
    @log_method_call
    def create(self, vals):
        _logger.info(f"Creating new task: {vals.get('name', 'Unknown')}")
        return super().create(vals)
    
    @log_method_call
    def write(self, vals):
        for task in self:
            _logger.info(f"Updating task {task.name} with values: {vals}")
        return super().write(vals)
    
    @api.model
    def debug_user_access(self, task_id):
        """Debug method to check user access"""
        task = self.browse(task_id)
        user = self.env.user
        
        debug_info = {
            'user_id': user.id,
            'user_name': user.name,
            'user_groups': [g.name for g in user.groups_id],
            'task_id': task.id,
            'task_name': task.name,
            'task_user_id': task.user_id.id,
            'project_members': [u.name for u in task.project_id.member_ids],
        }
        
        _logger.debug(f"Debug user access: {debug_info}")
        return debug_info
```

### Performance Monitoring

```python
# models/performance_monitor.py
import time
import psutil
from odoo import models, api
import logging

_logger = logging.getLogger(__name__)

class PerformanceMonitor(models.AbstractModel):
    _name = 'performance.monitor'
    
    @api.model
    def monitor_method_performance(self, method_name, *args, **kwargs):
        """Monitor method performance"""
        start_time = time.time()
        start_memory = psutil.Process().memory_info().rss / 1024 / 1024  # MB
        
        try:
            # Execute method
            result = getattr(self, method_name)(*args, **kwargs)
            
            end_time = time.time()
            end_memory = psutil.Process().memory_info().rss / 1024 / 1024  # MB
            
            execution_time = end_time - start_time
            memory_used = end_memory - start_memory
            
            _logger.info(f"Performance - {method_name}: "
                        f"Time: {execution_time:.2f}s, "
                        f"Memory: {memory_used:.2f}MB")
            
            return result
            
        except Exception as e:
            _logger.error(f"Error monitoring {method_name}: {e}")
            raise

## ORM Patterns & Performance

### Advanced ORM Techniques

```python
from odoo import models, fields, api
from odoo.tools import config
from odoo.sql_db import LazyCursor

class ProjectTaskOptimized(models.Model):
    _name = 'project.task.optimized'
    _description = 'Optimized Project Task'
    _rec_name = 'name'
    _order = 'priority desc, date_deadline asc, id desc'
    
    # Computed fields with store=True for performance
    @api.depends('subtask_ids.is_closed')
    def _compute_subtask_count(self):
        """Optimized computation with domain conditions"""
        for task in self:
            task.open_subtask_count = len(task.subtask_ids.filtered('is_closed'))
    
    open_subtask_count = fields.Integer(
        string='Open Subtasks',
        compute='_compute_subtask_count',
        store=True,  # Store for performance
        help="Number of open subtasks"
    )
    
    # Index for performance
    _sql_constraints = [
        ('name_uniq', 'unique(name, project_id)', 'Task name must be unique per project'),
    ]
    
    def init(self):
        """Add database indexes for performance"""
        tools = self.env['ir.model.fields']
        # Composite index for common queries
        self._cr.execute("""
            CREATE INDEX IF NOT EXISTS task_project_status_idx 
            ON project_task_optimized (project_id, stage_id) 
            WHERE active = true
        """)
        
        # Partial index for urgent tasks
        self._cr.execute("""
            CREATE INDEX IF NOT EXISTS task_urgent_idx 
            ON project_task_optimized (date_deadline) 
            WHERE priority = '3' AND stage_id NOT IN (
                SELECT id FROM project_task_type WHERE fold = true
            )
        """)

class TaskPerformanceHelper(models.AbstractModel):
    """Performance optimization patterns"""
    _name = 'task.performance.helper'
    _description = 'Task Performance Helper'
    
    @api.model
    def bulk_create_optimized(self, task_data_list):
        """Optimized bulk creation"""
        # Validate data first
        validated_data = []
        for data in task_data_list:
            if self._validate_task_data(data):
                validated_data.append(data)
        
        # Bulk create with SQL
        if validated_data:
            # Use SQL for maximum performance
            query = """
                INSERT INTO project_task_optimized 
                (name, project_id, user_id, create_date, write_date, create_uid, write_uid)
                VALUES %s
            """
            values = [(
                data['name'], 
                data['project_id'], 
                data.get('user_id'),
                fields.Datetime.now(),
                fields.Datetime.now(),
                self.env.uid,
                self.env.uid
            ) for data in validated_data]
            
            self.env.cr.execute(query, values)
            self.env.invalidate_all()  # Clear cache
            
        return len(validated_data)
    
    @api.model
    def search_with_custom_domain(self, additional_domain=None):
        """Optimized search with proper domain usage"""
        base_domain = [
            ('active', '=', True),
            ('stage_id.fold', '=', False)
        ]
        
        if additional_domain:
            domain = base_domain + additional_domain
        else:
            domain = base_domain
            
        # Use read_group for aggregated data
        grouped_data = self.env['project.task.optimized'].read_group(
            domain=domain,
            fields=['project_id', 'priority'],
            groupby=['project_id', 'priority'],
            lazy=False
        )
        
        return grouped_data
    
    @api.model
    def batch_update_efficient(self, task_ids, update_values):
        """Efficient batch updates"""
        if not task_ids or not update_values:
            return False
            
        # Direct SQL update for performance
        set_clause = ', '.join([
            f"{key} = %s" for key in update_values.keys()
        ])
        
        query = f"""
            UPDATE project_task_optimized 
            SET {set_clause}, write_date = %s, write_uid = %s
            WHERE id = ANY(%s)
        """
        
        values = list(update_values.values()) + [
            fields.Datetime.now(),
            self.env.uid,
            task_ids
        ]
        
        self.env.cr.execute(query, values)
        
        # Invalidate cache for affected records
        self.env['project.task.optimized'].browse(task_ids).invalidate_cache()
        
        return True

    def _validate_task_data(self, data):
        """Validate task data before creation"""
        required_fields = ['name', 'project_id']
        return all(field in data and data[field] for field in required_fields)
```

### Performance Monitoring Patterns

```python
from odoo import models, fields, api
import time
import psutil
from functools import wraps

class PerformanceOptimization(models.AbstractModel):
    _name = 'performance.optimization'
    _description = 'Performance Optimization Patterns'
    
    @api.model
    def with_performance_cache(self, cache_key, compute_func, ttl=300):
        """Simple caching mechanism"""
        cache_store = getattr(self.env.registry, '_perf_cache', {})
        
        if cache_key in cache_store:
            cached_data, timestamp = cache_store[cache_key]
            if time.time() - timestamp < ttl:
                return cached_data
        
        # Compute new value
        result = compute_func()
        
        # Store in cache
        cache_store[cache_key] = (result, time.time())
        self.env.registry._perf_cache = cache_store
        
        return result
    
    @api.model
    def query_optimizer_analyze(self, model_name, domain=None):
        """Analyze query performance"""
        model = self.env[model_name]
        
        # Enable query logging
        self.env.cr.execute("SET log_statement = 'all'")
        
        start_time = time.time()
        
        # Execute search
        if domain:
            records = model.search(domain)
        else:
            records = model.search([])
        
        execution_time = time.time() - start_time
        
        # Get query plan for last query
        self.env.cr.execute("EXPLAIN ANALYZE SELECT * FROM %s" % model._table)
        query_plan = self.env.cr.fetchall()
        
        return {
            'record_count': len(records),
            'execution_time': execution_time,
            'query_plan': query_plan,
            'recommendations': self._get_performance_recommendations(
                execution_time, len(records)
            )
        }
    
    def _get_performance_recommendations(self, execution_time, record_count):
        """Generate performance recommendations"""
        recommendations = []
        
        if execution_time > 2.0:
            recommendations.append("Consider adding database indexes")
        
        if record_count > 10000:
            recommendations.append("Use pagination or limit search results")
        
        if execution_time > 5.0:
            recommendations.append("Consider caching for this query")
        
        return recommendations
```

## Controllers & HTTP Routes

### Advanced Controller Patterns

```python
from odoo import http, fields, _
from odoo.http import request, Response
from odoo.exceptions import AccessError, UserError, ValidationError
import json
import logging
from datetime import datetime
import io
import base64

_logger = logging.getLogger(__name__)

class TaskController(http.Controller):
    
    @http.route(['/api/v1/tasks'], type='json', auth='user', methods=['GET'])
    def get_tasks_api(self, **kwargs):
        """RESTful API endpoint for tasks"""
        try:
            # Parameter validation
            limit = kwargs.get('limit', 50)
            offset = kwargs.get('offset', 0)
            search_term = kwargs.get('search', '')
            
            # Security check
            if not request.env.user.has_group('project.group_project_user'):
                raise AccessError(_("Access denied for task API"))
            
            # Build domain
            domain = [('active', '=', True)]
            if search_term:
                domain.append(('name', 'ilike', search_term))
            
            # Execute search
            Task = request.env['project.task']
            tasks = Task.search(domain, limit=limit, offset=offset)
            total_count = Task.search_count(domain)
            
            # Prepare response
            task_data = []
            for task in tasks:
                task_data.append({
                    'id': task.id,
                    'name': task.name,
                    'description': task.description,
                    'stage': task.stage_id.name if task.stage_id else '',
                    'project': task.project_id.name if task.project_id else '',
                    'assignee': task.user_id.name if task.user_id else '',
                    'deadline': task.date_deadline.isoformat() if task.date_deadline else None,
                    'priority': task.priority,
                    'progress': task.progress,
                })
            
            return {
                'status': 'success',
                'data': task_data,
                'pagination': {
                    'total': total_count,
                    'limit': limit,
                    'offset': offset,
                    'has_more': (offset + limit) < total_count
                }
            }
            
        except AccessError as e:
            return {'status': 'error', 'message': str(e)}
        except Exception as e:
            _logger.error(f"Task API error: {e}")
            return {'status': 'error', 'message': 'Internal server error'}
    
    @http.route(['/api/v1/tasks'], type='json', auth='user', methods=['POST'])
    def create_task_api(self, **kwargs):
        """Create task via API"""
        try:
            # Validate required fields
            required_fields = ['name', 'project_id']
            for field in required_fields:
                if field not in kwargs:
                    raise ValidationError(_(f"Missing required field: {field}"))
            
            # Security check
            project_id = kwargs.get('project_id')
            project = request.env['project.project'].browse(project_id)
            if not project.exists():
                raise ValidationError(_("Project not found"))
            
            # Check project access
            if not project.check_access_rights('write', raise_exception=False):
                raise AccessError(_("No write access to project"))
            
            # Create task
            task_values = {
                'name': kwargs['name'],
                'description': kwargs.get('description', ''),
                'project_id': project_id,
                'user_id': kwargs.get('assignee_id'),
                'date_deadline': kwargs.get('deadline'),
                'priority': kwargs.get('priority', '0'),
            }
            
            task = request.env['project.task'].create(task_values)
            
            return {
                'status': 'success',
                'data': {
                    'id': task.id,
                    'name': task.name,
                    'message': _('Task created successfully')
                }
            }
            
        except ValidationError as e:
            return {'status': 'error', 'message': str(e)}
        except AccessError as e:
            return {'status': 'error', 'message': str(e)}
        except Exception as e:
            _logger.error(f"Task creation error: {e}")
            return {'status': 'error', 'message': 'Failed to create task'}

class TaskFileController(http.Controller):
    
    @http.route(['/api/v1/tasks/<int:task_id>/export'], 
                type='http', auth='user', methods=['GET'])
    def export_task_pdf(self, task_id, **kwargs):
        """Export task as PDF"""
        try:
            task = request.env['project.task'].browse(task_id)
            if not task.exists():
                return request.not_found()
            
            # Security check
            if not task.check_access_rights('read', raise_exception=False):
                return request.make_response(
                    'Access denied', 
                    status=403
                )
            
            # Generate PDF
            pdf_content = self._generate_task_pdf(task)
            
            # Prepare response
            filename = f"task_{task.id}_{task.name}.pdf"
            
            return request.make_response(
                pdf_content,
                headers=[
                    ('Content-Type', 'application/pdf'),
                    ('Content-Disposition', f'attachment; filename="{filename}"'),
                    ('Content-Length', len(pdf_content))
                ]
            )
            
        except Exception as e:
            _logger.error(f"PDF export error: {e}")
            return request.make_response('Export failed', status=500)
    
    def _generate_task_pdf(self, task):
        """Generate PDF content for task"""
        # Use Odoo's report engine
        report = request.env['ir.actions.report']._get_report_from_name(
            'project.action_report_project_task'
        )
        
        if report:
            pdf_content, _ = report.render_qweb_pdf([task.id])
            return pdf_content
        else:
            # Fallback: simple text content
            content = f"""
            Task Report
            ===========
            Name: {task.name}
            Project: {task.project_id.name}
            Description: {task.description}
            Status: {task.stage_id.name}
            """
            return content.encode('utf-8')

class WebhookController(http.Controller):
    
    @http.route(['/webhook/task/update'], 
                type='json', auth='none', methods=['POST'], csrf=False)
    def task_webhook(self, **kwargs):
        """Webhook endpoint for external integrations"""
        try:
            # Validate webhook signature
            signature = request.httprequest.headers.get('X-Webhook-Signature')
            if not self._validate_webhook_signature(signature, kwargs):
                return {'status': 'error', 'message': 'Invalid signature'}
            
            # Process webhook data
            event_type = kwargs.get('event_type')
            task_data = kwargs.get('data', {})
            
            if event_type == 'task.created':
                return self._handle_task_created(task_data)
            elif event_type == 'task.updated':
                return self._handle_task_updated(task_data)
            else:
                return {'status': 'error', 'message': 'Unknown event type'}
                
        except Exception as e:
            _logger.error(f"Webhook error: {e}")
            return {'status': 'error', 'message': 'Processing failed'}
    
    def _validate_webhook_signature(self, signature, data):
        """Validate webhook signature for security"""
        # Implement your signature validation logic
        # This is a simplified example
        import hmac
        import hashlib
        
        secret = request.env['ir.config_parameter'].sudo().get_param(
            'task.webhook.secret'
        )
        
        if not secret or not signature:
            return False
        
        expected_signature = hmac.new(
            secret.encode(),
            json.dumps(data).encode(),
            hashlib.sha256
        ).hexdigest()
        
        return hmac.compare_digest(signature, expected_signature)
    
    def _handle_task_created(self, data):
        """Handle external task creation"""
        # Process external task creation
        return {'status': 'success', 'message': 'Task created'}
    
    def _handle_task_updated(self, data):
        """Handle external task update"""
        # Process external task update
        return {'status': 'success', 'message': 'Task updated'}
```

## Technical Architecture

### Module Structure Best Practices

```
my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── project_task.py
│   ├── project_stage.py
│   └── res_partner.py
├── views/
│   ├── project_task_views.xml
│   ├── project_stage_views.xml
│   └── menus.xml
├── data/
│   ├── ir_cron_data.xml
│   ├── mail_template_data.xml
│   └── project_data.xml
├── demo/
│   └── project_demo.xml
├── security/
│   ├── ir.model.access.csv
│   └── project_security.xml
├── static/
│   ├── description/
│   │   ├── icon.png
│   │   └── index.html
│   └── src/
│       ├── css/
│       ├── js/
│       └── xml/
├── controllers/
│   ├── __init__.py
│   └── main.py
├── wizard/
│   ├── __init__.py
│   └── task_wizard.py
├── report/
│   ├── __init__.py
│   ├── project_report.py
│   └── project_report_views.xml
└── tests/
    ├── __init__.py
    ├── test_project_task.py
    └── test_controllers.py
```

### Manifest Configuration

```python
# __manifest__.py
{
    'name': 'Advanced Project Management',
    'version': '18.0.1.0.0',
    'category': 'Project',
    'summary': 'Advanced project management features',
    'description': """
        Advanced Project Management
        ===========================
        
        Features:
        - Enhanced task management
        - Advanced reporting
        - Custom workflows
        - Integration APIs
    """,
    'author': 'Your Company',
    'website': 'https://www.yourcompany.com',
    'license': 'LGPL-3',
    'depends': [
        'base',
        'project',
        'mail',
        'portal',
        'web',
    ],
    'external_dependencies': {
        'python': ['requests', 'pillow'],
        'bin': ['wkhtmltopdf'],
    },
    'data': [
        # Security
        'security/project_security.xml',
        'security/ir.model.access.csv',
        
        # Data
        'data/ir_cron_data.xml',
        'data/mail_template_data.xml',
        
        # Views
        'views/project_task_views.xml',
        'views/project_stage_views.xml',
        'views/menus.xml',
        
        # Reports
        'report/project_report_views.xml',
        
        # Wizards
        'wizard/task_wizard_views.xml',
    ],
    'demo': [
        'demo/project_demo.xml',
    ],
    'assets': {
        'web.assets_backend': [
            'my_module/static/src/css/project_style.css',
            'my_module/static/src/js/project_widget.js',
        ],
        'web.assets_frontend': [
            'my_module/static/src/css/portal_style.css',
        ],
    },
    'installable': True,
    'application': True,
    'auto_install': False,
    'post_init_hook': 'post_init_hook',
    'uninstall_hook': 'uninstall_hook',
}
```

### Hooks and Migration

```python
# __init__.py
from . import models
from . import controllers
from . import wizard
from . import report

def post_init_hook(cr, registry):
    """Post-installation hook"""
    from odoo import api, SUPERUSER_ID
    
    env = api.Environment(cr, SUPERUSER_ID, {})
    
    # Create default project stages
    stage_values = [
        {'name': 'New', 'sequence': 1, 'fold': False},
        {'name': 'In Progress', 'sequence': 2, 'fold': False},
        {'name': 'Testing', 'sequence': 3, 'fold': False},
        {'name': 'Done', 'sequence': 4, 'fold': True},
    ]
    
    for stage_data in stage_values:
        existing_stage = env['project.task.type'].search([
            ('name', '=', stage_data['name'])
        ], limit=1)
        
        if not existing_stage:
            env['project.task.type'].create(stage_data)
    
    # Set default configurations
    env['ir.config_parameter'].sudo().set_param(
        'project.default_task_deadline_days', '7'
    )

def uninstall_hook(cr, registry):
    """Pre-uninstallation hook"""
    from odoo import api, SUPERUSER_ID
    
    env = api.Environment(cr, SUPERUSER_ID, {})
    
    # Clean up custom configurations
    params_to_remove = [
        'project.default_task_deadline_days',
        'project.auto_assign_tasks',
    ]
    
    for param in params_to_remove:
        env['ir.config_parameter'].sudo().search([
            ('key', '=', param)
        ]).unlink()
```

---

## Data Migration & Upgrade

### Migration Scripts

```python
# migrations/18.0.1.1.0/pre-migration.py
import logging
from odoo.upgrade import util

_logger = logging.getLogger(__name__)

def migrate(cr, version):
    """Pre-migration script"""
    _logger.info("Starting pre-migration for version %s", version)
    
    # Backup critical data
    _backup_task_data(cr)
    
    # Rename columns if needed
    if util.column_exists(cr, 'project_task', 'old_field_name'):
        util.rename_column(cr, 'project_task', 'old_field_name', 'new_field_name')
    
    # Add new columns
    if not util.column_exists(cr, 'project_task', 'custom_priority'):
        cr.execute("""
            ALTER TABLE project_task 
            ADD COLUMN custom_priority INTEGER DEFAULT 1
        """)
    
    _logger.info("Pre-migration completed")

def _backup_task_data(cr):
    """Backup critical task data"""
    cr.execute("""
        CREATE TABLE IF NOT EXISTS project_task_backup AS 
        SELECT * FROM project_task WHERE write_date >= NOW() - INTERVAL '30 days'
    """)

# migrations/18.0.1.1.0/post-migration.py
import logging
from odoo import api, SUPERUSER_ID

_logger = logging.getLogger(__name__)

def migrate(cr, version):
    """Post-migration script"""
    _logger.info("Starting post-migration for version %s", version)
    
    env = api.Environment(cr, SUPERUSER_ID, {})
    
    # Update computed fields
    _recompute_task_fields(env)
    
    # Migrate data structure
    _migrate_task_priorities(env)
    
    # Clean up old data
    _cleanup_old_records(env)
    
    _logger.info("Post-migration completed")

def _recompute_task_fields(env):
    """Recompute stored fields after migration"""
    tasks = env['project.task'].search([])
    
    # Recompute in batches to avoid memory issues
    batch_size = 100
    for i in range(0, len(tasks), batch_size):
        batch = tasks[i:i + batch_size]
        batch._compute_subtask_count()
        
        # Commit every batch
        env.cr.commit()

def _migrate_task_priorities(env):
    """Migrate old priority system to new one"""
    # Map old priority values to new ones
    priority_mapping = {
        'low': '0',
        'normal': '1',
        'high': '2',
        'urgent': '3'
    }
    
    for old_val, new_val in priority_mapping.items():
        env.cr.execute("""
            UPDATE project_task 
            SET priority = %s 
            WHERE priority = %s
        """, (new_val, old_val))

def _cleanup_old_records(env):
    """Clean up obsolete records"""
    # Remove old configuration parameters
    old_params = env['ir.config_parameter'].search([
        ('key', 'like', 'old_module.%')
    ])
    old_params.unlink()
    
    # Archive old mail templates
    old_templates = env['mail.template'].search([
        ('name', 'like', 'Old Template%')
    ])
    old_templates.write({'active': False})
```

### Data Import/Export Utilities

```python
from odoo import models, fields, api
import csv
import io
import base64
import logging

_logger = logging.getLogger(__name__)

class TaskDataManager(models.TransientModel):
    _name = 'task.data.manager'
    _description = 'Task Data Import/Export Manager'
    
    operation = fields.Selection([
        ('import', 'Import'),
        ('export', 'Export')
    ], required=True)
    
    data_file = fields.Binary(string='Data File')
    filename = fields.Char(string='Filename')
    
    def action_process(self):
        """Process import or export"""
        if self.operation == 'import':
            return self._import_tasks()
        else:
            return self._export_tasks()
    
    def _import_tasks(self):
        """Import tasks from CSV"""
        if not self.data_file:
            raise UserError(_("Please upload a file"))
        
        # Decode file content
        file_content = base64.b64decode(self.data_file)
        csv_reader = csv.DictReader(io.StringIO(file_content.decode('utf-8')))
        
        created_tasks = []
        errors = []
        
        for row_num, row in enumerate(csv_reader, 1):
            try:
                task_data = self._prepare_task_data(row)
                task = self.env['project.task'].create(task_data)
                created_tasks.append(task)
                
            except Exception as e:
                errors.append(f"Row {row_num}: {str(e)}")
        
        # Show results
        message = f"Successfully imported {len(created_tasks)} tasks."
        if errors:
            message += f"\n\nErrors:\n" + "\n".join(errors[:10])  # Show first 10 errors
        
        return {
            'type': 'ir.actions.client',
            'tag': 'display_notification',
            'params': {
                'title': _('Import Results'),
                'message': message,
                'type': 'success' if not errors else 'warning',
                'sticky': True,
            }
        }
    
    def _prepare_task_data(self, row):
        """Prepare task data from CSV row"""
        # Map CSV columns to task fields
        data = {
            'name': row.get('name', '').strip(),
            'description': row.get('description', '').strip(),
        }
        
        # Handle project
        project_name = row.get('project', '').strip()
        if project_name:
            project = self.env['project.project'].search([
                ('name', '=', project_name)
            ], limit=1)
            if project:
                data['project_id'] = project.id
        
        # Handle user assignment
        user_email = row.get('assignee_email', '').strip()
        if user_email:
            user = self.env['res.users'].search([
                ('email', '=', user_email)
            ], limit=1)
            if user:
                data['user_id'] = user.id
        
        # Handle deadline
        deadline_str = row.get('deadline', '').strip()
        if deadline_str:
            try:
                data['date_deadline'] = fields.Datetime.to_datetime(deadline_str)
            except:
                pass  # Skip invalid dates
        
        return data
    
    def _export_tasks(self):
        """Export tasks to CSV"""
        # Get tasks to export
        domain = [('active', '=', True)]
        tasks = self.env['project.task'].search(domain)
        
        # Prepare CSV content
        output = io.StringIO()
        writer = csv.writer(output)
        
        # Write header
        headers = [
            'ID', 'Name', 'Description', 'Project', 'Assignee', 
            'Assignee Email', 'Stage', 'Priority', 'Deadline', 'Created Date'
        ]
        writer.writerow(headers)
        
        # Write data
        for task in tasks:
            row = [
                task.id,
                task.name,
                task.description or '',
                task.project_id.name if task.project_id else '',
                task.user_id.name if task.user_id else '',
                task.user_id.email if task.user_id else '',
                task.stage_id.name if task.stage_id else '',
                task.priority,
                task.date_deadline.strftime('%Y-%m-%d') if task.date_deadline else '',
                task.create_date.strftime('%Y-%m-%d %H:%M:%S') if task.create_date else '',
            ]
            writer.writerow(row)
        
        # Prepare file
        csv_content = output.getvalue().encode('utf-8')
        filename = f"tasks_export_{fields.Date.today()}.csv"
        
        # Create attachment
        attachment = self.env['ir.attachment'].create({
            'name': filename,
            'type': 'binary',
            'datas': base64.b64encode(csv_content),
            'res_model': self._name,
            'res_id': self.id,
        })
        
        return {
            'type': 'ir.actions.act_url',
            'url': f'/web/content/{attachment.id}?download=true',
            'target': 'self',
        }
```
## Testing Frameworks

### Unit Testing Best Practices

```python
from odoo.tests.common import TransactionCase, Form
from odoo.exceptions import ValidationError, AccessError
from unittest.mock import patch, Mock
import logging

_logger = logging.getLogger(__name__)

class TestProjectTask(TransactionCase):
    
    def setUp(self):
        super().setUp()
        
        # Create test data
        self.project = self.env['project.project'].create({
            'name': 'Test Project',
            'description': 'Test project for unit tests'
        })
        
        self.user = self.env['res.users'].create({
            'name': 'Test User',
            'login': 'testuser',
            'email': 'test@example.com',
            'groups_id': [(6, 0, [
                self.env.ref('project.group_project_user').id
            ])]
        })
        
        self.task_values = {
            'name': 'Test Task',
            'description': 'Test task description',
            'project_id': self.project.id,
            'user_id': self.user.id,
        }
    
    def test_task_creation_basic(self):
        """Test basic task creation"""
        task = self.env['project.task'].create(self.task_values)
        
        self.assertTrue(task.exists())
        self.assertEqual(task.name, 'Test Task')
        self.assertEqual(task.project_id, self.project)
        self.assertEqual(task.user_id, self.user)
        self.assertTrue(task.active)
    
    def test_task_creation_with_form(self):
        """Test task creation using Form helper"""
        with Form(self.env['project.task']) as task_form:
            task_form.name = 'Form Test Task'
            task_form.project_id = self.project
            task_form.user_id = self.user
            task_form.description = 'Created with form'
            
            task = task_form.save()
        
        self.assertEqual(task.name, 'Form Test Task')
        self.assertEqual(task.description, 'Created with form')
    
    def test_task_validation_constraints(self):
        """Test task validation and constraints"""
        # Test name required
        with self.assertRaises(ValidationError):
            self.env['project.task'].create({
                'project_id': self.project.id
            })
        
        # Test unique constraint (if exists)
        task1 = self.env['project.task'].create(self.task_values)
        
        with self.assertRaises(ValidationError):
            self.env['project.task'].create(self.task_values)  # Duplicate name
    
    def test_task_computed_fields(self):
        """Test computed fields calculation"""
        task = self.env['project.task'].create(self.task_values)
        
        # Create subtasks
        subtask1 = self.env['project.task'].create({
            'name': 'Subtask 1',
            'parent_id': task.id,
            'project_id': self.project.id,
        })
        
        subtask2 = self.env['project.task'].create({
            'name': 'Subtask 2',
            'parent_id': task.id,
            'project_id': self.project.id,
        })
        
        # Test subtask count computation
        task._compute_subtask_count()
        self.assertEqual(task.subtask_count, 2)
    
    def test_task_security_rules(self):
        """Test security access rules"""
        # Create task as admin
        task = self.env['project.task'].create(self.task_values)
        
        # Test access as different user
        limited_user = self.env['res.users'].create({
            'name': 'Limited User',
            'login': 'limited',
            'email': 'limited@example.com',
            'groups_id': [(6, 0, [
                self.env.ref('base.group_user').id  # Basic user only
            ])]
        })
        
        # Switch to limited user
        task_as_limited = task.with_user(limited_user)
        
        # Test read access
        with self.assertRaises(AccessError):
            task_as_limited.read(['name'])
        
        # Test write access
        with self.assertRaises(AccessError):
            task_as_limited.write({'name': 'Updated name'})
    
    def test_task_workflow_methods(self):
        """Test task workflow methods"""
        task = self.env['project.task'].create(self.task_values)
        
        # Create stages
        stage_new = self.env['project.task.type'].create({
            'name': 'New',
            'sequence': 1,
        })
        stage_done = self.env['project.task.type'].create({
            'name': 'Done',
            'sequence': 10,
            'fold': True,
        })
        
        task.stage_id = stage_new
        
        # Test stage progression
        task.action_set_done()
        self.assertEqual(task.stage_id, stage_done)
        
        # Test task archiving
        task.action_archive()
        self.assertFalse(task.active)
    
    @patch('odoo.addons.project.models.project_task.ProjectTask._send_notification')
    def test_task_notifications(self, mock_send):
        """Test task notification sending"""
        mock_send.return_value = True
        
        task = self.env['project.task'].create(self.task_values)
        
        # Trigger notification
        task.with_context(notify_followers=True).write({
            'description': 'Updated description'
        })
        
        # Verify notification was sent
        mock_send.assert_called_once()
    
    def test_task_performance_bulk_operations(self):
        """Test bulk operations performance"""
        import time
        
        # Test bulk creation
        start_time = time.time()
        
        task_data = []
        for i in range(100):
            task_data.append({
                'name': f'Bulk Task {i}',
                'project_id': self.project.id,
            })
        
        tasks = self.env['project.task'].create(task_data)
        
        creation_time = time.time() - start_time
        
        # Should complete in reasonable time
        self.assertLess(creation_time, 5.0, "Bulk creation took too long")
        self.assertEqual(len(tasks), 100)
        
        # Test bulk update
        start_time = time.time()
        
        tasks.write({'description': 'Bulk updated'})
        
        update_time = time.time() - start_time
        self.assertLess(update_time, 2.0, "Bulk update took too long")

class TestProjectTaskIntegration(TransactionCase):
    """Integration tests for project tasks"""
    
    def setUp(self):
        super().setUp()
        self.project = self.env['project.project'].create({
            'name': 'Integration Test Project'
        })
    
    def test_task_mail_integration(self):
        """Test task mail integration"""
        # Create task with followers
        task = self.env['project.task'].create({
            'name': 'Mail Test Task',
            'project_id': self.project.id,
        })
        
        # Add follower
        task.message_subscribe([self.env.user.partner_id.id])
        
        # Post message
        task.message_post(
            body='Test message',
            message_type='comment',
            subtype_xmlid='mail.mt_comment'
        )
        
        # Verify message was posted
        messages = task.message_ids.filtered(
            lambda m: m.body and 'Test message' in m.body
        )
        self.assertTrue(messages)
    
    def test_task_portal_access(self):
        """Test task portal access"""
        # Create portal user
        portal_user = self.env['res.users'].create({
            'name': 'Portal User',
            'login': 'portal_user',
            'email': 'portal@example.com',
            'groups_id': [(6, 0, [
                self.env.ref('base.group_portal').id
            ])]
        })
        
        # Create task
        task = self.env['project.task'].create({
            'name': 'Portal Task',
            'project_id': self.project.id,
            'user_id': portal_user.id,
        })
        
        # Test portal access
        task_as_portal = task.with_user(portal_user)
        
        # Portal user should be able to read their own tasks
        self.assertTrue(task_as_portal.check_access_rights('read', False))
        
        # But not create new ones
        self.assertFalse(task_as_portal.check_access_rights('create', False))
```

### Integration Testing

```python
from odoo.tests.common import HttpCase, tagged
from odoo.tests import Form
import json

@tagged('post_install', '-at_install')
class TestTaskController(HttpCase):
    
    def setUp(self):
        super().setUp()
        
        self.project = self.env['project.project'].create({
            'name': 'API Test Project'
        })
        
        self.admin_user = self.env.ref('base.user_admin')
    
    def test_task_api_get(self):
        """Test task API GET endpoint"""
        # Create test tasks
        for i in range(5):
            self.env['project.task'].create({
                'name': f'API Task {i}',
                'project_id': self.project.id,
            })
        
        # Authenticate
        self.authenticate('admin', 'admin')
        
        # Make API request
        response = self.url_open(
            '/api/v1/tasks',
            data=json.dumps({
                'params': {
                    'limit': 3,
                    'offset': 0,
                }
            }),
            headers={'Content-Type': 'application/json'}
        )
        
        self.assertEqual(response.status_code, 200)
        
        data = response.json()
        self.assertEqual(data['status'], 'success')
        self.assertEqual(len(data['data']), 3)
        self.assertEqual(data['pagination']['total'], 5)
    
    def test_task_api_create(self):
        """Test task API POST endpoint"""
        self.authenticate('admin', 'admin')
        
        task_data = {
            'name': 'API Created Task',
            'description': 'Created via API',
            'project_id': self.project.id,
            'priority': '2',
        }
        
        response = self.url_open(
            '/api/v1/tasks',
            data=json.dumps({'params': task_data}),
            headers={'Content-Type': 'application/json'}
        )
        
        self.assertEqual(response.status_code, 200)
        
        data = response.json()
        self.assertEqual(data['status'], 'success')
        
        # Verify task was created
        task = self.env['project.task'].browse(data['data']['id'])
        self.assertEqual(task.name, 'API Created Task')
        self.assertEqual(task.project_id, self.project)
    
    def test_task_export_pdf(self):
        """Test task PDF export"""
        task = self.env['project.task'].create({
            'name': 'Export Test Task',
            'project_id': self.project.id,
            'description': 'Task for PDF export testing',
        })
        
        self.authenticate('admin', 'admin')
        
        response = self.url_open(f'/api/v1/tasks/{task.id}/export')
        
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.headers['Content-Type'], 'application/pdf')

class TestTaskWizard(TransactionCase):
    """Test task wizards"""
    
    def test_batch_update_wizard(self):
        """Test batch update wizard"""
        # Create test tasks
        tasks = self.env['project.task'].create([
            {'name': 'Task 1', 'priority': '0'},
            {'name': 'Task 2', 'priority': '0'},
            {'name': 'Task 3', 'priority': '0'},
        ])
        
        # Create wizard
        wizard = self.env['task.batch.update.wizard'].create({
            'task_ids': [(6, 0, tasks.ids)],
            'update_priority': True,
            'new_priority': '2',
        })
        
        # Execute wizard
        wizard.action_update_tasks()
        
        # Verify updates
        for task in tasks:
            self.assertEqual(task.priority, '2')
```

## Module Packaging

### Development Environment Setup

```bash
#!/bin/bash
# setup_dev_env.sh - Development environment setup script

# Create virtual environment
python3 -m venv odoo-dev
source odoo-dev/bin/activate

# Install Odoo dependencies
pip install -r requirements.txt

# Development tools
pip install \
    pre-commit \
    black \
    isort \
    flake8 \
    pylint \
    pytest \
    coverage

# Setup pre-commit hooks
pre-commit install

# Create development configuration
cat > odoo.conf << EOF
[options]
admin_passwd = admin
db_host = localhost
db_port = 5432
db_user = odoo
db_password = odoo
addons_path = addons,custom_addons
logfile = odoo.log
log_level = debug
dev_mode = reload,qweb,werkzeug,xml
EOF

echo "Development environment setup complete!"
```

### Code Quality Configuration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-xml
      - id: check-json

  - repo: https://github.com/psf/black
    rev: 22.10.0
    hooks:
      - id: black
        language_version: python3

  - repo: https://github.com/pycqa/isort
    rev: 5.10.1
    hooks:
      - id: isort
        args: ["--profile", "black"]

  - repo: https://github.com/pycqa/flake8
    rev: 5.0.4
    hooks:
      - id: flake8
        args: [--max-line-length=88, --extend-ignore=E203,W503]

  - repo: local
    hooks:
      - id: pylint
        name: pylint
        entry: pylint
        language: system
        types: [python]
        args: [--rcfile=.pylintrc]
```

```ini
# .pylintrc
[MASTER]
load-plugins=pylint_odoo

[ODOO]
license_headers=LGPL-3

[MESSAGES CONTROL]
disable=missing-docstring,
        too-few-public-methods,
        too-many-ancestors,
        import-error,
        pointless-statement

[FORMAT]
max-line-length=88
```

### CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: odoo
          POSTGRES_USER: odoo
          POSTGRES_DB: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v3
      with:
        python-version: '3.9'

    - name: Cache pip
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install coverage pytest

    - name: Lint with flake8
      run: |
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=88 --statistics

    - name: Test with pytest
      run: |
        coverage run -m pytest
        coverage xml

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        
    - name: Run Odoo tests
      run: |
        odoo-bin -i my_module -d test_db --test-enable --stop-after-init
      env:
        HOST: localhost
        PORT: 5432
        USER: odoo
        PASSWORD: odoo
```

### Production Deployment

```dockerfile
# Dockerfile
FROM odoo:18.0

USER root

# Install additional dependencies
RUN apt-get update && apt-get install -y \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Copy custom addons
COPY ./custom_addons /mnt/extra-addons/

# Install Python dependencies
COPY requirements.txt /tmp/
RUN pip3 install -r /tmp/requirements.txt

# Set permissions
RUN chown -R odoo:odoo /mnt/extra-addons

USER odoo
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_PASSWORD=odoo
      - POSTGRES_USER=odoo
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  odoo:
    build: .
    depends_on:
      - db
    ports:
      - "8069:8069"
    environment:
      - HOST=db
      - USER=odoo
      - PASSWORD=odoo
    volumes:
      - odoo_data:/var/lib/odoo
      - ./config:/etc/odoo
      - ./logs:/var/log/odoo
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - odoo
    restart: unless-stopped

volumes:
  postgres_data:
  odoo_data:
```

### Module Documentation Template

```markdown
# Module Name

## Overview
Brief description of what the module does.

## Features
- Feature 1
- Feature 2
- Feature 3

## Installation
1. Copy module to addons directory
2. Update apps list
3. Install module

## Configuration
1. Go to Settings > Technical > Parameters
2. Set parameter `module.setting` to desired value

## Usage
### For End Users
Step-by-step guide for end users

### For Developers
API documentation and extension points

## Dependencies
- base
- project
- mail

## Technical Details
### Models
- `custom.model`: Description
- `another.model`: Description

### Views
- Form views
- List views
- Custom views

### Security
- Access rights
- Record rules

## Testing
```bash
# Run tests
odoo-bin -i module_name -d test_db --test-enable --stop-after-init
```

## Changelog
### Version 1.0.0
- Initial release
- Basic functionality

### Version 1.1.0
- Added feature X
- Fixed bug Y

## License
LGPL-3

## Authors
- Author Name <email@example.com>

## Support
For support, please contact: support@company.com
```

---

## Expert Tips & Best Practices Summary

### Performance Optimization
1. **Database Indexes**: Add strategic indexes for frequent queries
2. **Batch Operations**: Use bulk operations for large datasets
3. **Lazy Loading**: Implement proper field loading strategies
4. **Cache Management**: Use appropriate caching mechanisms
5. **Query Optimization**: Monitor and optimize database queries

### Security Guidelines
1. **Access Rights**: Define granular access controls
2. **Record Rules**: Implement row-level security
3. **Input Validation**: Validate all user inputs
4. **SQL Injection**: Use ORM methods, avoid raw SQL
5. **Cross-Site Scripting**: Sanitize HTML content

### Code Quality Standards
1. **Documentation**: Comprehensive docstrings and comments
2. **Testing**: Unit tests with >80% coverage
3. **Type Hints**: Use Python type annotations
4. **Error Handling**: Proper exception handling
5. **Code Review**: Mandatory peer reviews

### Deployment Best Practices
1. **Environment Separation**: Dev/Test/Prod environments
2. **Configuration Management**: Externalized configuration
3. **Monitoring**: Application and infrastructure monitoring
4. **Backup Strategy**: Regular automated backups
5. **Security Updates**: Regular security patching

### Development Workflow
1. **Version Control**: Git with proper branching strategy
2. **CI/CD Pipeline**: Automated testing and deployment
3. **Code Formatting**: Consistent code style
4. **Documentation**: Keep docs up-to-date
5. **Migration Scripts**: Proper upgrade procedures

This comprehensive guide covers all aspects of expert-level Odoo 18.0 backend development. Each section provides production-ready code examples and follows Odoo best practices for building robust, scalable applications.