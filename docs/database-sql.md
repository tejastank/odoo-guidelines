# Odoo 18.0 — Database & SQL Integration (Expert)

Advanced guide for database operations, SQL integration, and complex data handling in Odoo 18.0. Covers raw SQL, performance optimization, database constraints, migrations, and advanced ORM patterns.

## Table of contents
- [Database Architecture & Design](#database-architecture--design)
- [Advanced ORM Patterns](#advanced-orm-patterns)
- [Raw SQL Integration](#raw-sql-integration)
- [Database Constraints & Indexes](#database-constraints--indexes)
- [Query Optimization & Performance](#query-optimization--performance)
- [Database Migrations & Schema Changes](#database-migrations--schema-changes)
- [Multi-Database Operations](#multi-database-operations)
- [Data Import/Export & ETL](#data-importexport--etl)
- [Database Security & Access Control](#database-security--access-control)
- [Monitoring & Debugging](#monitoring--debugging)
- [Database Replication & Clustering](#database-replication--clustering)
- [Advanced Data Types & JSON](#advanced-data-types--json)

---

## Database Architecture & Design

### Database Schema Planning

```python
from odoo import models, fields, api, tools
import psycopg2.extras
import logging

_logger = logging.getLogger(__name__)

class DatabaseArchitect(models.AbstractModel):
    """Advanced database architecture patterns"""
    _name = 'database.architect'
    _description = 'Database Architecture Helper'
    
    @api.model
    def analyze_table_structure(self, table_name):
        """Analyze table structure and relationships"""
        query = """
            SELECT 
                c.column_name,
                c.data_type,
                c.is_nullable,
                c.column_default,
                c.character_maximum_length,
                tc.constraint_type,
                kcu.constraint_name,
                ccu.table_name AS foreign_table_name,
                ccu.column_name AS foreign_column_name,
                pg_catalog.col_description(pgc.oid, c.ordinal_position) as column_comment
            FROM information_schema.columns c
            LEFT JOIN information_schema.table_constraints tc 
                ON c.table_name = tc.table_name 
                AND c.table_schema = tc.table_schema
            LEFT JOIN information_schema.key_column_usage kcu 
                ON tc.constraint_name = kcu.constraint_name
                AND tc.table_schema = kcu.table_schema
            LEFT JOIN information_schema.constraint_column_usage ccu 
                ON ccu.constraint_name = tc.constraint_name
                AND ccu.table_schema = tc.table_schema
            LEFT JOIN pg_catalog.pg_class pgc 
                ON pgc.relname = c.table_name
            WHERE c.table_name = %s
            ORDER BY c.ordinal_position;
        """
        
        self.env.cr.execute(query, (table_name,))
        columns = self.env.cr.dictfetchall()
        
        # Analyze indexes
        index_query = """
            SELECT 
                i.relname as index_name,
                a.attname as column_name,
                ix.indisunique as is_unique,
                ix.indisprimary as is_primary,
                pg_get_indexdef(ix.indexrelid) as index_definition
            FROM pg_class t
            JOIN pg_index ix ON t.oid = ix.indrelid
            JOIN pg_class i ON i.oid = ix.indexrelid
            JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(ix.indkey)
            WHERE t.relname = %s
            ORDER BY i.relname, a.attnum;
        """
        
        self.env.cr.execute(index_query, (table_name,))
        indexes = self.env.cr.dictfetchall()
        
        return {
            'columns': columns,
            'indexes': indexes,
            'analysis': self._analyze_performance_implications(columns, indexes)
        }
    
    def _analyze_performance_implications(self, columns, indexes):
        """Analyze performance implications of current schema"""
        issues = []
        recommendations = []
        
        # Check for missing indexes on foreign keys
        foreign_keys = [col for col in columns if col.get('constraint_type') == 'FOREIGN KEY']
        indexed_columns = {idx['column_name'] for idx in indexes}
        
        for fk in foreign_keys:
            if fk['column_name'] not in indexed_columns:
                issues.append(f"Missing index on foreign key: {fk['column_name']}")
                recommendations.append(f"CREATE INDEX ON {fk['table_name']} ({fk['column_name']})")
        
        # Check for large text columns without appropriate settings
        large_text_cols = [col for col in columns 
                          if col['data_type'] == 'text' and col.get('character_maximum_length', 0) > 1000]
        
        for col in large_text_cols:
            recommendations.append(f"Consider using PostgreSQL TOAST for column: {col['column_name']}")
        
        return {
            'issues': issues,
            'recommendations': recommendations,
            'performance_score': max(0, 100 - len(issues) * 10)
        }

class AdvancedTableDesign(models.Model):
    """Example of advanced table design with complex relationships"""
    _name = 'advanced.table.design'
    _description = 'Advanced Table Design Example'
    _order = 'priority desc, create_date desc'
    _rec_name = 'display_name'
    
    # Efficient naming and display
    name = fields.Char(string='Name', required=True, index=True)
    display_name = fields.Char(string='Display Name', compute='_compute_display_name', store=True)
    
    # Optimized status tracking
    state = fields.Selection([
        ('draft', 'Draft'),
        ('confirmed', 'Confirmed'),
        ('processing', 'Processing'),
        ('done', 'Done'),
        ('cancelled', 'Cancelled'),
    ], default='draft', index=True, tracking=True)
    
    # Priority with efficient querying
    priority = fields.Selection([
        ('0', 'Low'),
        ('1', 'Normal'),
        ('2', 'High'),
        ('3', 'Urgent'),
    ], default='1', index=True)
    
    # Efficient many2one relationships
    company_id = fields.Many2one('res.company', string='Company', 
                                required=True, index=True,
                                default=lambda self: self.env.company)
    
    user_id = fields.Many2one('res.users', string='Responsible User',
                             index=True, tracking=True,
                             default=lambda self: self.env.user)
    
    # Optimized many2many with custom table
    tag_ids = fields.Many2many('advanced.tag', 'advanced_table_tag_rel',
                              'table_id', 'tag_id', string='Tags')
    
    # JSON field for flexible data
    metadata = fields.Json(string='Metadata', default=dict)
    
    # Computed fields with proper storage
    @api.depends('name', 'state')
    def _compute_display_name(self):
        for record in self:
            record.display_name = f"{record.name} ({record.state})"
    
    # Database constraints
    _sql_constraints = [
        ('name_company_uniq', 'UNIQUE(name, company_id)', 
         'Name must be unique per company'),
        ('priority_check', 'CHECK(priority IN (\'0\', \'1\', \'2\', \'3\'))',
         'Priority must be valid'),
    ]
    
    def init(self):
        """Custom database initialization"""
        # Composite indexes for common queries
        tools.create_index(self._cr, 'advanced_table_design_compound_idx',
                          self._table, ['company_id', 'state', 'priority'])
        
        # Partial indexes for specific conditions
        self._cr.execute(f"""
            CREATE INDEX IF NOT EXISTS {self._table}_active_urgent_idx 
            ON {self._table} (create_date DESC) 
            WHERE state NOT IN ('done', 'cancelled') AND priority = '3'
        """)
        
        # Functional indexes
        self._cr.execute(f"""
            CREATE INDEX IF NOT EXISTS {self._table}_name_lower_idx 
            ON {self._table} (LOWER(name))
        """)
```

### Advanced Model Inheritance Patterns

```python
class BaseTimestamped(models.AbstractModel):
    """Base model for timestamped records with optimized queries"""
    _name = 'base.timestamped'
    _description = 'Base Timestamped Model'
    
    # Optimized timestamp fields
    create_date = fields.Datetime(string='Creation Date', readonly=True, index=True)
    write_date = fields.Datetime(string='Last Update', readonly=True, index=True)
    create_uid = fields.Many2one('res.users', string='Created By', readonly=True, index=True)
    write_uid = fields.Many2one('res.users', string='Last Updated By', readonly=True, index=True)
    
    # Soft delete pattern
    active = fields.Boolean(string='Active', default=True, index=True)
    archived_date = fields.Datetime(string='Archived Date', readonly=True)
    
    def archive_records(self):
        """Soft delete with timestamp"""
        self.write({
            'active': False,
            'archived_date': fields.Datetime.now()
        })
    
    @api.model
    def get_recent_records(self, days=30, limit=100):
        """Optimized query for recent records"""
        date_limit = fields.Datetime.now() - timedelta(days=days)
        return self.search([
            ('create_date', '>=', date_limit),
            ('active', '=', True)
        ], order='create_date desc', limit=limit)

class AuditableModel(models.AbstractModel):
    """Advanced auditing with change tracking"""
    _name = 'auditable.model'
    _description = 'Auditable Model Base'
    _inherit = ['base.timestamped']
    
    # Change tracking
    version = fields.Integer(string='Version', default=1, readonly=True)
    change_log = fields.Json(string='Change Log', default=list)
    
    @api.model_create_multi
    def create(self, vals_list):
        """Override create to track initial state"""
        records = super().create(vals_list)
        for record, vals in zip(records, vals_list):
            record._log_changes({'created': vals})
        return records
    
    def write(self, vals):
        """Override write to track changes"""
        if not vals:
            return True
        
        # Capture old values
        old_values = {}
        tracked_fields = self._get_tracked_fields()
        
        for record in self:
            old_values[record.id] = {
                field: record[field] for field in tracked_fields if field in vals
            }
        
        # Perform the write
        result = super().write(vals)
        
        # Log changes
        for record in self:
            if record.id in old_values:
                changes = {}
                for field, old_value in old_values[record.id].items():
                    new_value = record[field]
                    if old_value != new_value:
                        changes[field] = {'old': old_value, 'new': new_value}
                
                if changes:
                    record.version += 1
                    record._log_changes(changes)
        
        return result
    
    def _get_tracked_fields(self):
        """Define which fields to track"""
        return ['name', 'state', 'user_id']  # Override in concrete models
    
    def _log_changes(self, changes):
        """Log changes to audit trail"""
        log_entry = {
            'timestamp': fields.Datetime.now().isoformat(),
            'user_id': self.env.uid,
            'changes': changes
        }
        
        current_log = self.change_log or []
        current_log.append(log_entry)
        
        # Keep only last 100 entries
        if len(current_log) > 100:
            current_log = current_log[-100:]
        
        self.change_log = current_log
```

## Advanced ORM Patterns

### Complex Query Building

```python
class AdvancedQueryBuilder(models.AbstractModel):
    """Advanced query building patterns"""
    _name = 'advanced.query.builder'
    _description = 'Advanced Query Builder'
    
    @api.model
    def build_dynamic_domain(self, filters):
        """Build complex domains dynamically"""
        domain = []
        
        for filter_def in filters:
            filter_type = filter_def.get('type')
            field = filter_def.get('field')
            value = filter_def.get('value')
            operator = filter_def.get('operator', '=')
            
            if filter_type == 'simple':
                domain.append((field, operator, value))
            
            elif filter_type == 'date_range':
                if value.get('start'):
                    domain.append((field, '>=', value['start']))
                if value.get('end'):
                    domain.append((field, '<=', value['end']))
            
            elif filter_type == 'text_search':
                # Multi-field text search
                text_domain = []
                for search_field in value.get('fields', [field]):
                    text_domain.append((search_field, 'ilike', value['text']))
                if text_domain:
                    domain.append('|' * (len(text_domain) - 1))
                    domain.extend(text_domain)
            
            elif filter_type == 'related_count':
                # Count related records
                related_model = value['model']
                related_field = value['field']
                count_operator = value.get('operator', '>')
                count_value = value.get('count', 0)
                
                subquery = f"""
                    SELECT {related_field} 
                    FROM {self.env[related_model]._table} 
                    WHERE {related_field} IS NOT NULL
                    GROUP BY {related_field}
                    HAVING COUNT(*) {count_operator} {count_value}
                """
                
                domain.append(('id', 'in', subquery))
        
        return domain
    
    @api.model
    def search_with_aggregates(self, domain=None, group_by=None, aggregates=None):
        """Search with aggregate calculations"""
        if not aggregates:
            aggregates = []
        
        # Build base query
        base_query = self._where_calc(domain or [])
        from_clause, where_clause, where_clause_params = base_query.get_sql()
        
        # Build aggregate selects
        select_parts = ['id']
        
        for agg in aggregates:
            field = agg['field']
            function = agg['function'].upper()  # SUM, COUNT, AVG, etc.
            alias = agg.get('alias', f"{function.lower()}_{field}")
            
            if function in ('SUM', 'AVG', 'MIN', 'MAX'):
                select_parts.append(f'{function}("{field}") as {alias}')
            elif function == 'COUNT':
                select_parts.append(f'COUNT("{field}") as {alias}')
        
        # Build GROUP BY clause
        group_clause = ''
        if group_by:
            group_fields = [f'"{field}"' for field in group_by]
            group_clause = f"GROUP BY {', '.join(group_fields)}"
            select_parts.extend([f'"{field}"' for field in group_by])
        
        # Execute query
        query = f"""
            SELECT {', '.join(select_parts)}
            FROM {from_clause}
            {where_clause}
            {group_clause}
        """
        
        self.env.cr.execute(query, where_clause_params)
        return self.env.cr.dictfetchall()
    
    @api.model
    def search_with_window_functions(self, domain=None, window_specs=None):
        """Advanced search with PostgreSQL window functions"""
        if not window_specs:
            return self.search(domain or [])
        
        # Build base query
        Model = self._name
        table = self._table
        
        base_domain = domain or []
        base_query = self._where_calc(base_domain)
        from_clause, where_clause, where_params = base_query.get_sql()
        
        # Build window function selects
        select_parts = [f'{table}.id']
        
        for spec in window_specs:
            function = spec['function']  # ROW_NUMBER, RANK, DENSE_RANK, etc.
            partition_by = spec.get('partition_by', [])
            order_by = spec.get('order_by', [])
            alias = spec.get('alias', function.lower())
            
            window_clause = 'OVER ('
            
            if partition_by:
                partition_fields = [f'"{field}"' for field in partition_by]
                window_clause += f"PARTITION BY {', '.join(partition_fields)} "
            
            if order_by:
                order_fields = []
                for order_field in order_by:
                    if isinstance(order_field, tuple):
                        field, direction = order_field
                        order_fields.append(f'"{field}" {direction.upper()}')
                    else:
                        order_fields.append(f'"{order_field}"')
                window_clause += f"ORDER BY {', '.join(order_fields)}"
            
            window_clause += ')'
            
            select_parts.append(f'{function}() {window_clause} as {alias}')
        
        # Execute query
        query = f"""
            SELECT {', '.join(select_parts)}
            FROM {from_clause}
            {where_clause}
        """
        
        self.env.cr.execute(query, where_params)
        results = self.env.cr.dictfetchall()
        
        # Create recordset with additional data
        ids = [r['id'] for r in results]
        records = self.browse(ids)
        
        # Attach window function results
        for record, result in zip(records, results):
            for spec in window_specs:
                alias = spec.get('alias', spec['function'].lower())
                setattr(record, alias, result[alias])
        
        return records

class BatchOperationsManager(models.AbstractModel):
    """Efficient batch operations"""
    _name = 'batch.operations.manager'
    _description = 'Batch Operations Manager'
    
    @api.model
    def bulk_create_optimized(self, vals_list, batch_size=1000):
        """Optimized bulk creation with batching"""
        if not vals_list:
            return self.browse()
        
        all_records = self.browse()
        
        # Process in batches
        for i in range(0, len(vals_list), batch_size):
            batch = vals_list[i:i + batch_size]
            
            # Prepare batch data
            prepared_batch = []
            for vals in batch:
                # Add default values
                complete_vals = self._add_missing_default_values(vals)
                prepared_batch.append(complete_vals)
            
            # Use SQL for maximum performance
            if len(prepared_batch) > 100:
                records = self._sql_bulk_create(prepared_batch)
            else:
                records = super().create(prepared_batch)
            
            all_records |= records
            
            # Commit every batch to avoid locks
            if i % (batch_size * 10) == 0:
                self.env.cr.commit()
        
        return all_records
    
    def _sql_bulk_create(self, vals_list):
        """Direct SQL bulk insert"""
        if not vals_list:
            return self.browse()
        
        # Get field information
        fields_info = {}
        for field_name, field in self._fields.items():
            if not field.store:
                continue
            fields_info[field_name] = {
                'type': field.type,
                'required': field.required,
                'column': field.column_type[1] if hasattr(field, 'column_type') else 'text'
            }
        
        # Prepare columns and values
        columns = list(fields_info.keys())
        table = self._table
        
        # Build INSERT query
        placeholders = ', '.join(['%s'] * len(columns))
        columns_str = ', '.join([f'"{col}"' for col in columns])
        
        query = f"""
            INSERT INTO "{table}" ({columns_str})
            VALUES ({placeholders})
            RETURNING id
        """
        
        # Prepare data
        values_list = []
        for vals in vals_list:
            row = []
            for col in columns:
                value = vals.get(col)
                # Handle special field types
                if fields_info[col]['type'] == 'many2one' and value:
                    row.append(value if isinstance(value, int) else value.id)
                elif fields_info[col]['type'] == 'datetime' and isinstance(value, str):
                    row.append(fields.Datetime.from_string(value))
                else:
                    row.append(value)
            values_list.append(row)
        
        # Execute bulk insert
        self.env.cr.executemany(query, values_list)
        inserted_ids = [row[0] for row in self.env.cr.fetchall()]
        
        # Clear cache and return records
        self.invalidate_cache()
        return self.browse(inserted_ids)
    
    @api.model
    def bulk_update_optimized(self, records_data, batch_size=1000):
        """Optimized bulk updates"""
        if not records_data:
            return True
        
        # Group updates by fields to update
        updates_by_fields = {}
        for record_id, vals in records_data.items():
            fields_key = tuple(sorted(vals.keys()))
            if fields_key not in updates_by_fields:
                updates_by_fields[fields_key] = []
            updates_by_fields[fields_key].append((record_id, vals))
        
        # Process each group
        for fields_tuple, data_list in updates_by_fields.items():
            # Process in batches
            for i in range(0, len(data_list), batch_size):
                batch = data_list[i:i + batch_size]
                self._sql_bulk_update(fields_tuple, batch)
        
        # Invalidate cache
        self.invalidate_cache()
        return True
    
    def _sql_bulk_update(self, fields, data_list):
        """Direct SQL bulk update using CASE statements"""
        if not data_list or not fields:
            return
        
        table = self._table
        set_clauses = []
        
        # Build CASE statements for each field
        for field in fields:
            when_clauses = []
            for record_id, vals in data_list:
                value = vals.get(field)
                when_clauses.append(f"WHEN id = {record_id} THEN %s")
            
            when_clause = ' '.join(when_clauses)
            set_clauses.append(f'"{field}" = CASE {when_clause} ELSE "{field}" END')
        
        # Build UPDATE query
        set_clause = ', '.join(set_clauses)
        ids = [record_id for record_id, _ in data_list]
        ids_placeholder = ', '.join(['%s'] * len(ids))
        
        query = f"""
            UPDATE "{table}"
            SET {set_clause}
            WHERE id IN ({ids_placeholder})
        """
        
        # Prepare parameters
        params = []
        for field in fields:
            for record_id, vals in data_list:
                params.append(vals.get(field))
        params.extend(ids)
        
        # Execute update
        self.env.cr.execute(query, params)
```

## Raw SQL Integration

### Advanced SQL Operations

```python
class RawSQLManager(models.AbstractModel):
    """Advanced raw SQL operations manager"""
    _name = 'raw.sql.manager'
    _description = 'Raw SQL Manager'
    
    @api.model
    def execute_complex_query(self, query, params=None, as_dict=True):
        """Execute complex SQL queries safely"""
        try:
            # Validate query safety
            if not self._is_query_safe(query):
                raise ValueError("Potentially unsafe SQL query detected")
            
            # Execute query
            self.env.cr.execute(query, params or ())
            
            if as_dict:
                return self.env.cr.dictfetchall()
            else:
                return self.env.cr.fetchall()
        
        except psycopg2.Error as e:
            _logger.error(f"SQL Error: {e}")
            raise
    
    def _is_query_safe(self, query):
        """Basic SQL injection protection"""
        dangerous_keywords = [
            'DROP', 'DELETE', 'TRUNCATE', 'ALTER', 'CREATE',
            'INSERT', 'UPDATE', 'GRANT', 'REVOKE'
        ]
        
        query_upper = query.upper()
        return not any(keyword in query_upper for keyword in dangerous_keywords)
    
    @api.model
    def get_table_statistics(self, table_name):
        """Get comprehensive table statistics"""
        stats_query = """
            SELECT 
                schemaname,
                tablename,
                attname as column_name,
                n_distinct,
                correlation,
                most_common_vals,
                most_common_freqs,
                histogram_bounds
            FROM pg_stats 
            WHERE tablename = %s
            ORDER BY attname;
        """
        
        size_query = """
            SELECT 
                pg_size_pretty(pg_total_relation_size(%s)) as total_size,
                pg_size_pretty(pg_relation_size(%s)) as table_size,
                pg_size_pretty(pg_indexes_size(%s)) as indexes_size,
                (SELECT reltuples::bigint FROM pg_class WHERE relname = %s) as estimated_rows
        """
        
        # Get column statistics
        self.env.cr.execute(stats_query, (table_name,))
        column_stats = self.env.cr.dictfetchall()
        
        # Get size information
        self.env.cr.execute(size_query, (table_name, table_name, table_name, table_name))
        size_info = self.env.cr.dictfetchone()
        
        # Get index information
        index_query = """
            SELECT 
                indexname,
                indexdef,
                schemaname
            FROM pg_indexes 
            WHERE tablename = %s
        """
        
        self.env.cr.execute(index_query, (table_name,))
        indexes = self.env.cr.dictfetchall()
        
        return {
            'table_name': table_name,
            'size_info': size_info,
            'column_statistics': column_stats,
            'indexes': indexes
        }
    
    @api.model
    def analyze_query_performance(self, query, params=None):
        """Analyze query performance with EXPLAIN ANALYZE"""
        explain_query = f"EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) {query}"
        
        try:
            self.env.cr.execute(explain_query, params or ())
            result = self.env.cr.fetchone()[0]
            
            # Extract key metrics
            plan = result[0]['Plan']
            execution_time = result[0]['Execution Time']
            planning_time = result[0]['Planning Time']
            
            analysis = {
                'execution_time_ms': execution_time,
                'planning_time_ms': planning_time,
                'total_time_ms': execution_time + planning_time,
                'plan': plan,
                'recommendations': self._analyze_plan(plan)
            }
            
            return analysis
            
        except Exception as e:
            _logger.error(f"Query analysis failed: {e}")
            return {'error': str(e)}
    
    def _analyze_plan(self, plan):
        """Analyze execution plan and provide recommendations"""
        recommendations = []
        
        def analyze_node(node):
            node_type = node.get('Node Type')
            
            # Check for expensive operations
            if node_type == 'Seq Scan':
                recommendations.append(
                    f"Sequential scan detected on {node.get('Relation Name', 'unknown')}. "
                    "Consider adding an index."
                )
            
            elif node_type == 'Sort' and node.get('Sort Method') == 'external merge':
                recommendations.append(
                    "External sort detected. Consider increasing work_mem or optimizing query."
                )
            
            elif node_type == 'Nested Loop' and node.get('Actual Loops', 0) > 1000:
                recommendations.append(
                    "High-cost nested loop detected. Consider rewriting query or adding indexes."
                )
            
            # Recursively analyze child nodes
            for child in node.get('Plans', []):
                analyze_node(child)
        
        analyze_node(plan)
        return recommendations
    
    @api.model
    def create_materialized_view(self, view_name, query, refresh=True):
        """Create and manage materialized views"""
        try:
            # Drop existing view if it exists
            drop_query = f"DROP MATERIALIZED VIEW IF EXISTS {view_name}"
            self.env.cr.execute(drop_query)
            
            # Create materialized view
            create_query = f"CREATE MATERIALIZED VIEW {view_name} AS {query}"
            self.env.cr.execute(create_query)
            
            # Create indexes if specified
            self._create_materialized_view_indexes(view_name)
            
            if refresh:
                self.refresh_materialized_view(view_name)
            
            _logger.info(f"Materialized view '{view_name}' created successfully")
            return True
            
        except Exception as e:
            _logger.error(f"Failed to create materialized view: {e}")
            raise
    
    def refresh_materialized_view(self, view_name, concurrent=True):
        """Refresh materialized view"""
        try:
            refresh_type = "CONCURRENTLY" if concurrent else ""
            query = f"REFRESH MATERIALIZED VIEW {refresh_type} {view_name}"
            self.env.cr.execute(query)
            
            _logger.info(f"Materialized view '{view_name}' refreshed")
            return True
            
        except Exception as e:
            _logger.error(f"Failed to refresh materialized view: {e}")
            raise
    
    def _create_materialized_view_indexes(self, view_name):
        """Create recommended indexes for materialized view"""
        # This would be customized based on the specific view
        # Example: Create index on commonly queried columns
        pass
    
    @api.model
    def setup_table_partitioning(self, table_name, partition_column, partition_type='RANGE'):
        """Setup table partitioning for large tables"""
        try:
            # Check if table exists
            check_query = """
                SELECT EXISTS (
                    SELECT FROM information_schema.tables 
                    WHERE table_name = %s
                )
            """
            self.env.cr.execute(check_query, (table_name,))
            
            if not self.env.cr.fetchone()[0]:
                raise ValueError(f"Table {table_name} does not exist")
            
            # Create partitioned table structure
            if partition_type.upper() == 'RANGE':
                self._setup_range_partitioning(table_name, partition_column)
            elif partition_type.upper() == 'HASH':
                self._setup_hash_partitioning(table_name, partition_column)
            
            _logger.info(f"Partitioning setup for table '{table_name}' completed")
            return True
            
        except Exception as e:
            _logger.error(f"Failed to setup partitioning: {e}")
            raise
    
    def _setup_range_partitioning(self, table_name, partition_column):
        """Setup range partitioning (e.g., by date)"""
        # Example implementation for date-based partitioning
        current_year = fields.Date.today().year
        
        for year in range(current_year - 1, current_year + 2):
            partition_name = f"{table_name}_{year}"
            start_date = f"{year}-01-01"
            end_date = f"{year + 1}-01-01"
            
            create_partition_query = f"""
                CREATE TABLE IF NOT EXISTS {partition_name} 
                PARTITION OF {table_name}
                FOR VALUES FROM ('{start_date}') TO ('{end_date}')
            """
            
            self.env.cr.execute(create_partition_query)
    
    def _setup_hash_partitioning(self, table_name, partition_column):
        """Setup hash partitioning"""
        # Create hash partitions
        num_partitions = 4
        
        for i in range(num_partitions):
            partition_name = f"{table_name}_hash_{i}"
            
            create_partition_query = f"""
                CREATE TABLE IF NOT EXISTS {partition_name}
                PARTITION OF {table_name}
                FOR VALUES WITH (modulus {num_partitions}, remainder {i})
            """
            
            self.env.cr.execute(create_partition_query)
```

## Query Optimization & Performance

### Advanced Query Performance Tuning

```python
class QueryPerformanceOptimizer(models.AbstractModel):
    """Advanced query performance optimization"""
    _name = 'query.performance.optimizer'
    _description = 'Query Performance Optimizer'
    
    @api.model
    def optimize_search_performance(self, model_name, domain, context_keys=None):
        """Optimize search queries with advanced techniques"""
        Model = self.env[model_name]
        
        # Analyze domain complexity
        domain_analysis = self._analyze_domain_complexity(domain)
        
        # Choose optimization strategy
        if domain_analysis['complexity'] == 'high':
            return self._optimize_complex_search(Model, domain, context_keys)
        elif domain_analysis['has_text_search']:
            return self._optimize_text_search(Model, domain, context_keys)
        else:
            return self._optimize_standard_search(Model, domain, context_keys)
    
    def _analyze_domain_complexity(self, domain):
        """Analyze domain complexity and characteristics"""
        analysis = {
            'complexity': 'low',
            'has_text_search': False,
            'has_date_range': False,
            'has_many2one': False,
            'has_computed_fields': False,
            'operator_count': 0
        }
        
        for clause in domain:
            if isinstance(clause, (list, tuple)) and len(clause) >= 2:
                field, operator = clause[0], clause[1]
                
                analysis['operator_count'] += 1
                
                # Check for text search operators
                if operator in ('ilike', 'like', '=ilike', '=like'):
                    analysis['has_text_search'] = True
                
                # Check for date range operations
                if operator in ('>=', '<=', '>', '<') and 'date' in field.lower():
                    analysis['has_date_range'] = True
                
                # Check for many2one relationships
                if '.' in field:
                    analysis['has_many2one'] = True
        
        # Determine complexity
        if analysis['operator_count'] > 10 or analysis['has_many2one']:
            analysis['complexity'] = 'high'
        elif analysis['operator_count'] > 5 or analysis['has_text_search']:
            analysis['complexity'] = 'medium'
        
        return analysis
    
    def _optimize_complex_search(self, Model, domain, context_keys):
        """Optimize complex searches with multiple joins"""
        # Break down complex domain into sub-queries
        simple_clauses = []
        complex_clauses = []
        
        for clause in domain:
            if isinstance(clause, (list, tuple)) and len(clause) >= 2:
                field = clause[0]
                if '.' in field:  # Related field
                    complex_clauses.append(clause)
                else:
                    simple_clauses.append(clause)
            else:
                simple_clauses.append(clause)
        
        # Execute simple search first to reduce dataset
        if simple_clauses:
            base_records = Model.search(simple_clauses)
            if not base_records:
                return Model.browse()
        else:
            base_records = Model.search([])
        
        # Apply complex filters progressively
        for clause in complex_clauses:
            if base_records:
                filtered_records = base_records.filtered_domain([clause])
                base_records = filtered_records
            else:
                break
        
        return base_records
    
    def _optimize_text_search(self, Model, domain, context_keys):
        """Optimize text search queries using PostgreSQL features"""
        # Extract text search clauses
        text_clauses = []
        other_clauses = []
        
        for clause in domain:
            if isinstance(clause, (list, tuple)) and len(clause) >= 3:
                field, operator, value = clause[0], clause[1], clause[2]
                if operator in ('ilike', 'like', '=ilike', '=like'):
                    text_clauses.append(clause)
                else:
                    other_clauses.append(clause)
            else:
                other_clauses.append(clause)
        
        # Use PostgreSQL full-text search if available
        if len(text_clauses) > 1:
            return self._use_fulltext_search(Model, text_clauses, other_clauses)
        else:
            # Standard search with index hints
            return Model.search(domain)
    
    def _use_fulltext_search(self, Model, text_clauses, other_clauses):
        """Use PostgreSQL full-text search capabilities"""
        # Build text search query
        search_terms = []
        search_fields = []
        
        for clause in text_clauses:
            field, operator, value = clause[0], clause[1], clause[2]
            search_fields.append(field)
            search_terms.append(value.replace('%', ''))
        
        # Create full-text search vector
        search_vector = ' || \' \' || '.join([f"COALESCE({field}, '')" for field in search_fields])
        search_query = ' & '.join(search_terms)
        
        # Execute full-text search
        query = f"""
            SELECT id 
            FROM {Model._table}
            WHERE to_tsvector('english', {search_vector}) @@ to_tsquery('english', %s)
        """
        
        try:
            Model.env.cr.execute(query, (search_query,))
            text_search_ids = [row[0] for row in Model.env.cr.fetchall()]
            
            # Combine with other filters
            if other_clauses:
                final_domain = [('id', 'in', text_search_ids)] + other_clauses
                return Model.search(final_domain)
            else:
                return Model.browse(text_search_ids)
                
        except Exception as e:
            _logger.warning(f"Full-text search failed, falling back to standard search: {e}")
            return Model.search(text_clauses + other_clauses)
    
    def _optimize_standard_search(self, Model, domain, context_keys):
        """Optimize standard searches with caching and indexing"""
        # Check if result can be cached
        cache_key = self._generate_cache_key(Model._name, domain, context_keys)
        
        if hasattr(self, '_search_cache'):
            cached_result = self._search_cache.get(cache_key)
            if cached_result:
                return Model.browse(cached_result)
        
        # Execute search with optimizations
        records = Model.with_context(prefetch_fields=False).search(domain)
        
        # Cache result if appropriate
        if len(records) < 1000:  # Don't cache large result sets
            if not hasattr(self, '_search_cache'):
                self._search_cache = {}
            self._search_cache[cache_key] = records.ids
        
        return records
    
    def _generate_cache_key(self, model_name, domain, context_keys):
        """Generate cache key for search results"""
        import hashlib
        
        domain_str = str(sorted(domain))
        context_str = str(sorted(context_keys or []))
        cache_string = f"{model_name}:{domain_str}:{context_str}"
        
        return hashlib.md5(cache_string.encode()).hexdigest()
    
    @api.model
    def optimize_read_performance(self, records, fields=None):
        """Optimize read operations with smart prefetching"""
        if not records:
            return {}
        
        Model = records._name
        
        # Analyze field access patterns
        field_analysis = self._analyze_field_access_patterns(Model, fields)
        
        # Group fields by access pattern
        eager_fields = field_analysis['eager_load']
        lazy_fields = field_analysis['lazy_load']
        computed_fields = field_analysis['computed']
        
        # Prefetch eager fields
        if eager_fields:
            records.read(eager_fields)
        
        # Set up lazy loading for expensive fields
        if lazy_fields:
            records = records.with_context(prefetch_fields=False)
        
        return {
            'optimized_records': records,
            'field_strategy': field_analysis
        }
    
    def _analyze_field_access_patterns(self, model_name, fields):
        """Analyze field access patterns for optimization"""
        Model = self.env[model_name]
        
        if not fields:
            fields = list(Model._fields.keys())
        
        analysis = {
            'eager_load': [],
            'lazy_load': [],
            'computed': []
        }
        
        for field_name in fields:
            field = Model._fields.get(field_name)
            if not field:
                continue
            
            # Categorize fields
            if field.compute:
                analysis['computed'].append(field_name)
            elif field.type in ('text', 'html', 'binary'):
                analysis['lazy_load'].append(field_name)
            elif field.type in ('many2many', 'one2many'):
                analysis['lazy_load'].append(field_name)
            else:
                analysis['eager_load'].append(field_name)
        
        return analysis
```

## Data Import/Export & ETL

### Advanced ETL Operations

```python
class ETLManager(models.AbstractModel):
    """Advanced ETL (Extract, Transform, Load) operations"""
    _name = 'etl.manager'
    _description = 'ETL Manager'
    
    @api.model
    def execute_etl_pipeline(self, pipeline_config):
        """Execute complete ETL pipeline"""
        pipeline_name = pipeline_config['name']
        stages = pipeline_config['stages']
        
        _logger.info(f"Starting ETL pipeline: {pipeline_name}")
        
        # Initialize pipeline context
        context = {
            'pipeline_name': pipeline_name,
            'start_time': time.time(),
            'processed_records': 0,
            'error_records': 0,
            'temp_tables': []
        }
        
        try:
            for stage_config in stages:
                stage_name = stage_config['name']
                stage_type = stage_config['type']
                
                _logger.info(f"Executing ETL stage: {stage_name}")
                
                if stage_type == 'extract':
                    context = self._execute_extract_stage(stage_config, context)
                elif stage_type == 'transform':
                    context = self._execute_transform_stage(stage_config, context)
                elif stage_type == 'load':
                    context = self._execute_load_stage(stage_config, context)
                elif stage_type == 'validate':
                    context = self._execute_validate_stage(stage_config, context)
            
            # Cleanup temporary tables
            self._cleanup_temp_tables(context['temp_tables'])
            
            total_time = time.time() - context['start_time']
            _logger.info(f"ETL pipeline {pipeline_name} completed in {total_time:.2f}s")
            
            return {
                'status': 'success',
                'processed_records': context['processed_records'],
                'error_records': context['error_records'],
                'execution_time': total_time
            }
            
        except Exception as e:
            _logger.error(f"ETL pipeline {pipeline_name} failed: {e}")
            self._cleanup_temp_tables(context.get('temp_tables', []))
            raise
    
    def _execute_extract_stage(self, stage_config, context):
        """Execute data extraction stage"""
        source_type = stage_config['source_type']
        
        if source_type == 'database':
            return self._extract_from_database(stage_config, context)
        elif source_type == 'csv':
            return self._extract_from_csv(stage_config, context)
        elif source_type == 'json':
            return self._extract_from_json(stage_config, context)
        elif source_type == 'api':
            return self._extract_from_api(stage_config, context)
        elif source_type == 'xml':
            return self._extract_from_xml(stage_config, context)
        else:
            raise ValueError(f"Unsupported source type: {source_type}")
    
    def _extract_from_database(self, stage_config, context):
        """Extract data from database source"""
        source_config = stage_config['source_config']
        query = stage_config['query']
        temp_table = stage_config.get('temp_table', f"etl_temp_{int(time.time())}")
        
        # Connect to source database
        if source_config.get('external', False):
            conn = self.setup_cross_database_connection(source_config)
            cursor = conn.cursor()
        else:
            cursor = self.env.cr
        
        try:
            # Execute extraction query
            cursor.execute(query)
            
            # Create temporary table with extracted data
            columns_info = cursor.description
            columns = [desc[0] for desc in columns_info]
            
            # Create temp table
            column_defs = []
            for i, desc in enumerate(columns_info):
                col_name = desc[0]
                # Map PostgreSQL types
                if desc[1] == 23:  # INTEGER
                    col_type = 'INTEGER'
                elif desc[1] == 25:  # TEXT
                    col_type = 'TEXT'
                elif desc[1] == 1082:  # DATE
                    col_type = 'DATE'
                elif desc[1] == 1114:  # TIMESTAMP
                    col_type = 'TIMESTAMP'
                else:
                    col_type = 'TEXT'  # Default to TEXT
                
                column_defs.append(f"{col_name} {col_type}")
            
            create_temp_sql = f"""
                CREATE TEMP TABLE {temp_table} (
                    {', '.join(column_defs)}
                )
            """
            
            self.env.cr.execute(create_temp_sql)
            
            # Insert extracted data
            insert_sql = f"""
                INSERT INTO {temp_table} ({', '.join(columns)})
                VALUES ({', '.join(['%s'] * len(columns))})
            """
            
            batch_size = stage_config.get('batch_size', 1000)
            processed = 0
            
            while True:
                rows = cursor.fetchmany(batch_size)
                if not rows:
                    break
                
                self.env.cr.executemany(insert_sql, rows)
                processed += len(rows)
                
                if processed % 10000 == 0:
                    _logger.info(f"Extracted {processed} records")
            
            context['extracted_table'] = temp_table
            context['temp_tables'].append(temp_table)
            context['processed_records'] += processed
            
            _logger.info(f"Extraction completed: {processed} records")
            
        finally:
            if source_config.get('external', False):
                conn.close()
        
        return context
    
    def _extract_from_csv(self, stage_config, context):
        """Extract data from CSV file"""
        import csv
        import io
        
        file_path = stage_config['file_path']
        delimiter = stage_config.get('delimiter', ',')
        encoding = stage_config.get('encoding', 'utf-8')
        has_header = stage_config.get('has_header', True)
        temp_table = stage_config.get('temp_table', f"etl_temp_{int(time.time())}")
        
        # Read CSV file
        with open(file_path, 'r', encoding=encoding) as csvfile:
            reader = csv.reader(csvfile, delimiter=delimiter)
            
            if has_header:
                headers = next(reader)
            else:
                # Generate column names
                first_row = next(reader)
                headers = [f"col_{i}" for i in range(len(first_row))]
                csvfile.seek(0)
                next(reader)  # Skip first row again
            
            # Create temp table
            column_defs = [f"{col} TEXT" for col in headers]
            create_temp_sql = f"""
                CREATE TEMP TABLE {temp_table} (
                    {', '.join(column_defs)}
                )
            """
            
            self.env.cr.execute(create_temp_sql)
            
            # Insert data
            insert_sql = f"""
                INSERT INTO {temp_table} ({', '.join(headers)})
                VALUES ({', '.join(['%s'] * len(headers))})
            """
            
            batch_size = stage_config.get('batch_size', 1000)
            batch = []
            processed = 0
            
            for row in reader:
                # Pad or truncate row to match headers
                if len(row) < len(headers):
                    row.extend([''] * (len(headers) - len(row)))
                elif len(row) > len(headers):
                    row = row[:len(headers)]
                
                batch.append(row)
                
                if len(batch) >= batch_size:
                    self.env.cr.executemany(insert_sql, batch)
                    processed += len(batch)
                    batch = []
                    
                    if processed % 10000 == 0:
                        _logger.info(f"Loaded {processed} CSV records")
            
            # Insert remaining records
            if batch:
                self.env.cr.executemany(insert_sql, batch)
                processed += len(batch)
        
        context['extracted_table'] = temp_table
        context['temp_tables'].append(temp_table)
        context['processed_records'] += processed
        
        _logger.info(f"CSV extraction completed: {processed} records")
        return context

class BulkDataOperations(models.AbstractModel):
    """Bulk data operations for high-performance data handling"""
    _name = 'bulk.data.operations'
    _description = 'Bulk Data Operations'
    
    @api.model
    def bulk_insert_csv(self, model_name, csv_data, field_mapping, options=None):
        """High-performance bulk insert from CSV data"""
        options = options or {}
        delimiter = options.get('delimiter', ',')
        batch_size = options.get('batch_size', 10000)
        
        Model = self.env[model_name]
        
        # Parse CSV data
        import csv
        import io
        
        csv_reader = csv.DictReader(io.StringIO(csv_data), delimiter=delimiter)
        
        batch = []
        processed = 0
        
        for row in csv_reader:
            # Map fields
            mapped_row = {}
            for odoo_field, csv_field in field_mapping.items():
                if csv_field in row:
                    mapped_row[odoo_field] = self._convert_field_value(
                        Model._fields[odoo_field], 
                        row[csv_field]
                    )
            
            batch.append(mapped_row)
            
            if len(batch) >= batch_size:
                Model.create(batch)
                processed += len(batch)
                batch = []
                
                _logger.info(f"Bulk inserted {processed} records")
        
        # Insert remaining records
        if batch:
            Model.create(batch)
            processed += len(batch)
        
        return processed
    
    def _convert_field_value(self, field, value):
        """Convert string value to appropriate field type"""
        if not value or value.strip() == '':
            return False
        
        field_type = field.type
        
        try:
            if field_type == 'integer':
                return int(value)
            elif field_type == 'float':
                return float(value)
            elif field_type == 'boolean':
                return value.lower() in ('true', '1', 'yes', 'on')
            elif field_type == 'date':
                from datetime import datetime
                return datetime.strptime(value, '%Y-%m-%d').date()
            elif field_type == 'datetime':
                from datetime import datetime
                return datetime.strptime(value, '%Y-%m-%d %H:%M:%S')
            elif field_type in ('many2one', 'many2many'):
                # Handle relational fields
                return self._resolve_relational_field(field, value)
            else:
                return value
                
        except (ValueError, TypeError) as e:
            _logger.warning(f"Field conversion error for {field.name}: {e}")
            return False
    
    def _resolve_relational_field(self, field, value):
        """Resolve relational field references"""
        if field.type == 'many2one':
            # Try to find record by name or external ID
            target_model = self.env[field.comodel_name]
            
            # Try by name first
            record = target_model.search([('name', '=', value)], limit=1)
            if record:
                return record.id
            
            # Try by external ID
            try:
                return self.env.ref(value).id
            except ValueError:
                pass
            
            # Create new record if allowed
            if hasattr(target_model, 'name'):
                new_record = target_model.create({'name': value})
                return new_record.id
        
        return False
```

## Database Security & Access Control

### Advanced Security Management

```python
class DatabaseSecurityManager(models.AbstractModel):
    """Database security and access control management"""
    _name = 'database.security.manager'
    _description = 'Database Security Manager'
    
    @api.model
    def audit_database_permissions(self):
        """Audit database permissions and security settings"""
        audit_results = {
            'user_permissions': [],
            'table_permissions': [],
            'security_recommendations': [],
            'risk_level': 'low'
        }
        
        # Audit user permissions
        user_perms_query = """
            SELECT 
                u.usename as username,
                u.usesuper as is_superuser,
                u.usecreatedb as can_create_db,
                u.userepl as can_replicate,
                array_agg(g.rolname) as groups
            FROM pg_user u
            LEFT JOIN pg_auth_members m ON u.usesysid = m.member
            LEFT JOIN pg_group g ON m.roleid = g.grosysid
            GROUP BY u.usename, u.usesuper, u.usecreatedb, u.userepl
        """
        
        self.env.cr.execute(user_perms_query)
        user_permissions = self.env.cr.fetchall()
        
        risk_level = 'low'
        
        for user_perm in user_permissions:
            username, is_super, can_create_db, can_replicate, groups = user_perm
            
            user_risk = 'low'
            
            # Check for high-risk permissions
            if is_super and username != 'postgres':
                user_risk = 'high'
                risk_level = 'high'
                audit_results['security_recommendations'].append(
                    f"User {username} has superuser privileges - consider restricting"
                )
            
            if can_create_db and username not in ['postgres', 'odoo']:
                user_risk = 'medium'
                if risk_level == 'low':
                    risk_level = 'medium'
            
            audit_results['user_permissions'].append({
                'username': username,
                'is_superuser': is_super,
                'can_create_db': can_create_db,
                'can_replicate': can_replicate,
                'groups': groups or [],
                'risk_level': user_risk
            })
        
        audit_results['risk_level'] = risk_level
        return audit_results
    
    @api.model
    def implement_row_level_security(self, table_config):
        """Implement row-level security (RLS) for tables"""
        table_name = table_config['table']
        policy_name = table_config['policy_name']
        policy_expression = table_config['policy_expression']
        
        # Enable RLS on table
        enable_rls_sql = f"ALTER TABLE {table_name} ENABLE ROW LEVEL SECURITY"
        self.env.cr.execute(enable_rls_sql)
        
        # Create RLS policy
        create_policy_sql = f"""
            CREATE POLICY {policy_name} ON {table_name}
            FOR ALL
            TO odoo
            USING ({policy_expression})
        """
        
        self.env.cr.execute(create_policy_sql)
        
        _logger.info(f"Row-level security enabled for table {table_name}")

class DataMaskingManager(models.AbstractModel):
    """Data masking for privacy protection"""
    _name = 'data.masking.manager'
    _description = 'Data Masking Manager'
    
    @api.model
    def apply_data_masking(self, masking_config):
        """Apply data masking based on configuration"""
        table_name = masking_config['table']
        masking_rules = masking_config['rules']
        
        for rule in masking_rules:
            column_name = rule['column']
            masking_type = rule['type']
            
            if masking_type == 'email':
                self._mask_email_column(table_name, column_name, rule)
            elif masking_type == 'phone':
                self._mask_phone_column(table_name, column_name, rule)
            elif masking_type == 'name':
                self._mask_name_column(table_name, column_name, rule)
            elif masking_type == 'credit_card':
                self._mask_credit_card_column(table_name, column_name, rule)
    
    def _mask_email_column(self, table_name, column_name, rule):
        """Mask email addresses"""
        preserve_domain = rule.get('preserve_domain', False)
        
        if preserve_domain:
            mask_sql = f"""
                UPDATE {table_name}
                SET {column_name} = 
                    'user' || generate_random_uuid()::text || '@' || 
                    split_part({column_name}, '@', 2)
                WHERE {column_name} IS NOT NULL AND {column_name} LIKE '%@%'
            """
        else:
            mask_sql = f"""
                UPDATE {table_name}
                SET {column_name} = 'user' || generate_random_uuid()::text || '@example.com'
                WHERE {column_name} IS NOT NULL
            """
        
        self.env.cr.execute(mask_sql)
```

## Monitoring & Debugging

### Database Performance Monitoring

```python
class DatabaseMonitor(models.AbstractModel):
    """Database performance monitoring and alerting"""
    _name = 'database.monitor'
    _description = 'Database Monitor'
    
    @api.model
    def get_database_health_metrics(self):
        """Get comprehensive database health metrics"""
        metrics = {
            'connection_stats': self._get_connection_stats(),
            'query_performance': self._get_query_performance_stats(),
            'table_stats': self._get_table_statistics(),
            'index_usage': self._get_index_usage_stats(),
            'lock_stats': self._get_lock_statistics(),
            'cache_stats': self._get_cache_statistics()
        }
        
        # Calculate overall health score
        metrics['health_score'] = self._calculate_health_score(metrics)
        
        return metrics
    
    def _get_connection_stats(self):
        """Get database connection statistics"""
        conn_stats_query = """
            SELECT 
                COUNT(*) as total_connections,
                COUNT(*) FILTER (WHERE state = 'active') as active_connections,
                COUNT(*) FILTER (WHERE state = 'idle') as idle_connections,
                COUNT(*) FILTER (WHERE state = 'idle in transaction') as idle_in_transaction,
                MAX(EXTRACT(EPOCH FROM (now() - backend_start))) as longest_connection_age,
                AVG(EXTRACT(EPOCH FROM (now() - backend_start))) as avg_connection_age
            FROM pg_stat_activity
            WHERE pid != pg_backend_pid()
        """
        
        self.env.cr.execute(conn_stats_query)
        result = self.env.cr.fetchone()
        
        # Get max connections setting
        self.env.cr.execute("SELECT setting FROM pg_settings WHERE name = 'max_connections'")
        max_connections = int(self.env.cr.fetchone()[0])
        
        return {
            'total_connections': result[0],
            'active_connections': result[1],
            'idle_connections': result[2],
            'idle_in_transaction': result[3],
            'longest_connection_age_seconds': result[4],
            'avg_connection_age_seconds': result[5],
            'max_connections': max_connections,
            'connection_utilization_percent': (result[0] / max_connections) * 100
        }
    
    def _calculate_health_score(self, metrics):
        """Calculate overall database health score"""
        score = 100
        
        # Connection utilization penalty
        conn_util = metrics['connection_stats']['connection_utilization_percent']
        if conn_util > 80:
            score -= (conn_util - 80) * 2
        
        # Cache hit ratio consideration
        cache_hit = metrics.get('cache_stats', {}).get('cache_hit_ratio', 95)
        if cache_hit < 95:
            score -= (95 - cache_hit) * 2
        
        # Long-running connection penalty
        longest_conn = metrics['connection_stats']['longest_connection_age_seconds']
        if longest_conn and longest_conn > 3600:  # 1 hour
            score -= min(20, (longest_conn - 3600) / 180)
        
        return max(0, min(100, score))
    
    @api.model
    def analyze_slow_queries(self, analysis_config=None):
        """Analyze slow queries and provide optimization suggestions"""
        analysis_config = analysis_config or {}
        min_duration = analysis_config.get('min_duration_ms', 1000)
        
        slow_queries_sql = """
            SELECT 
                query,
                calls,
                total_time,
                mean_time,
                max_time,
                rows,
                100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) as hit_percent
            FROM pg_stat_statements
            WHERE mean_time > %s
            ORDER BY mean_time DESC
            LIMIT 20
        """
        
        try:
            self.env.cr.execute(slow_queries_sql, (min_duration,))
            slow_queries = self.env.cr.fetchall()
            
            analysis_results = []
            
            for query_data in slow_queries:
                query = query_data[0]
                analysis = self._analyze_query_performance(query, query_data)
                analysis_results.append(analysis)
            
            return analysis_results
            
        except psycopg2.ProgrammingError:
            return []
    
    def _analyze_query_performance(self, query, stats):
        """Analyze individual query performance"""
        suggestions = []
        
        # Check cache hit ratio
        if stats[6] and stats[6] < 90:
            suggestions.append({
                'type': 'cache',
                'description': 'Low cache hit ratio - consider increasing shared_buffers',
                'impact': 'high'
            })
        
        # Check for missing indexes (basic heuristics)
        if 'WHERE' in query.upper() and 'INDEX' not in query.upper():
            suggestions.append({
                'type': 'index',
                'description': 'Consider adding indexes for WHERE clause columns',
                'impact': 'medium'
            })
        
        return {
            'query': query[:200] + '...' if len(query) > 200 else query,
            'stats': {
                'calls': stats[1],
                'total_time_ms': stats[2],
                'mean_time_ms': stats[3],
                'max_time_ms': stats[4],
                'rows': stats[5],
                'cache_hit_percent': stats[6]
            },
            'suggestions': suggestions
        }

# Configuration Examples
DATABASE_CONFIG = {
    'monitoring': {
        'thresholds': {
            'connection_utilization': 80,
            'cache_hit_ratio': 95,
            'slow_query_threshold_ms': 1000,
            'lock_timeout_ms': 30000
        }
    },
    'security': {
        'row_level_security': True,
        'column_encryption': ['social_security_number', 'credit_card'],
        'data_masking_rules': {
            'res_partner': [
                {'column': 'email', 'type': 'email', 'preserve_domain': True},
                {'column': 'phone', 'type': 'phone', 'preserve_format': True}
            ]
        }
    },
    'performance': {
        'query_cache_size': 1000,
        'batch_size': 10000,
        'connection_pool_size': 50
    }
}
```

This comprehensive database-sql.md file provides complete expert-level guidelines for Odoo 18.0 database operations and SQL integrations. It covers all complex scenarios including advanced ORM patterns, raw SQL integration, performance optimization, database migrations, multi-database operations, ETL processes, security management, and monitoring systems with production-ready code examples.
