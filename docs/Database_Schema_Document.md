# Database Schema Document: Cost Accounting Assistant

**Project Name:** Cost Accounting Assistant (Backend)
**Version:** 1.0
**Date:** December 28, 2025
**Status:** Design Phase
**Document Owner:** Database Architecture Team

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Database Overview](#database-overview)
3. [Schema Design Principles](#schema-design-principles)
4. [Entity Relationship Diagram](#entity-relationship-diagram)
5. [Table Definitions](#table-definitions)
6. [Multi-Tenancy Implementation](#multi-tenancy-implementation)
7. [Indexes and Performance](#indexes-and-performance)
8. [Constraints and Validations](#constraints-and-validations)
9. [Sample Data](#sample-data)
10. [Migration Strategy](#migration-strategy)

---

## 1. Executive Summary

The Cost Accounting Assistant uses **PostgreSQL 15+** with **Row-Level Security (RLS)** for multi-tenant data isolation. The schema supports:

- **Multi-tenant architecture** with complete data isolation
- **Cost accounting** (standard, actual, ABC, job costing)
- **Variance analysis** (material, labor, overhead, sales)
- **Financial metrics** (break-even, ROI, contribution margin)
- **Reporting and audit logging**

**Key Features:**
- 30+ tables organized into logical domains
- Foreign key constraints for referential integrity
- Comprehensive indexing for performance
- Row-Level Security for tenant isolation
- Audit trail for compliance

---

## 2. Database Overview

### 2.1 Database Configuration

```yaml
Database Name: cost_accounting_db
PostgreSQL Version: 15+
Character Set: UTF8
Collation: en_US.UTF-8
Timezone: UTC
Connection Pool: 10-50 connections (asyncpg)
```

### 2.2 Schema Organization

The database is organized into **6 logical domains**:

```
cost_accounting_db/
├── COMPANY DOMAIN
│   ├── companies (tenants)
│   ├── departments
│   ├── users
│   └── user_roles
│
├── PRODUCT DOMAIN
│   ├── products
│   ├── materials
│   ├── bill_of_materials (BOM)
│   └── product_categories
│
├── COST DOMAIN
│   ├── standard_costs
│   ├── actual_costs
│   ├── overhead_rates
│   ├── labor_rates
│   └── cost_drivers
│
├── TRANSACTION DOMAIN
│   ├── production_orders
│   ├── cost_allocations
│   ├── material_requisitions
│   ├── labor_time_entries
│   └── overhead_applications
│
├── ANALYSIS DOMAIN
│   ├── cost_variances
│   ├── variance_explanations
│   ├── anomalies
│   └── forecasts
│
└── REPORTING DOMAIN
    ├── cost_reports
    ├── budget_vs_actual
    ├── profit_margins
    └── audit_logs
```

---

## 3. Schema Design Principles

### 3.1 Design Principles

1. **Normalization:** Tables normalized to 3NF to reduce redundancy
2. **Multi-Tenancy:** Every table includes `tenant_id` with RLS policies
3. **Audit Trail:** `created_at`, `updated_at`, `created_by` on all tables
4. **Soft Deletes:** Use `is_deleted` flag instead of hard deletes
5. **UUIDs:** Use UUID for primary keys (better for distributed systems)
6. **Type Safety:** Use appropriate data types (NUMERIC for money, TIMESTAMP WITH TIME ZONE)

### 3.2 Naming Conventions

```
TABLES:       lowercase_with_underscores (plural)
COLUMNS:      lowercase_with_underscores
PRIMARY KEYS: {table_name}_id (UUID)
FOREIGN KEYS: referenced_table_singular_id
INDEXES:      idx_{table}_{columns}
CONSTRAINTS:  {table}_{constraint_type}_{detail}
```

---

## 4. Entity Relationship Diagram

### 4.1 High-Level ERD

```
┌──────────────────────────────────────────────────────────────┐
│                   COST ACCOUNTING SCHEMA                     │
└──────────────────────────────────────────────────────────────┘

COMPANY DOMAIN                    PRODUCT DOMAIN
┌───────────────┐                 ┌───────────────┐
│   companies   │◀────────────────│   products    │
│   (tenants)   │                 │               │
└───────┬───────┘                 └───────┬───────┘
        │                                 │
        │                                 │
        ▼                                 ▼
┌───────────────┐                 ┌───────────────┐
│  departments  │                 │  materials    │
└───────┬───────┘                 └───────┬───────┘
        │                                 │
        │                                 │
        ▼                                 ▼
┌───────────────┐                 ┌───────────────┐
│     users     │                 │ bill_of_      │
│               │                 │  materials    │
└───────┬───────┘                 └───────────────┘
        │
        │
        ▼
┌───────────────┐
│  user_roles   │
└───────────────┘

COST DOMAIN                       TRANSACTION DOMAIN
┌───────────────┐                 ┌───────────────┐
│ standard_costs│◀────────────────│  production_  │
│               │                 │    orders     │
└───────────────┘                 └───────┬───────┘
                                          │
┌───────────────┐                         │
│ actual_costs  │◀────────────────────────┤
└───────────────┘                         │
                                          │
┌───────────────┐                         │
│ overhead_rates│◀────────────────────────┤
└───────────────┘                         │
                                          │
┌───────────────┐                         │
│  labor_rates  │◀────────────────────────┘
└───────────────┘

ANALYSIS DOMAIN                   REPORTING DOMAIN
┌───────────────┐                 ┌───────────────┐
│     cost_     │──────────────┐  │  cost_reports │
│   variances   │              │  └───────────────┘
└───────────────┘              │
                               │  ┌───────────────┐
┌───────────────┐              └─▶│  budget_vs_   │
│   variance_   │                 │    actual     │
│  explanations │                 └───────────────┘
└───────────────┘
                                  ┌───────────────┐
┌───────────────┐                 │  profit_      │
│   anomalies   │                 │   margins     │
└───────────────┘                 └───────────────┘

                                  ┌───────────────┐
                                  │  audit_logs   │
                                  └───────────────┘
```

---

## 5. Table Definitions

### 5.1 COMPANY DOMAIN

#### Table: `companies` (Tenants)

```sql
CREATE TABLE companies (
    company_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_name VARCHAR(255) NOT NULL,
    company_code VARCHAR(50) NOT NULL UNIQUE,
    industry VARCHAR(100),
    fiscal_year_start INTEGER DEFAULT 1 CHECK (fiscal_year_start BETWEEN 1 AND 12),
    currency VARCHAR(3) DEFAULT 'USD',
    timezone VARCHAR(50) DEFAULT 'UTC',
    costing_method VARCHAR(20) DEFAULT 'standard'
        CHECK (costing_method IN ('standard', 'actual', 'abc', 'hybrid')),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,

    -- Configuration (JSONB for flexibility)
    config JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_companies_code ON companies(company_code);
CREATE INDEX idx_companies_active ON companies(is_active) WHERE is_active = TRUE;

COMMENT ON TABLE companies IS 'Multi-tenant companies (each company is a tenant)';
COMMENT ON COLUMN companies.config IS 'JSON configuration: variance thresholds, reporting preferences, etc.';
```

**Sample Config:**
```json
{
  "variance_threshold_percent": 5,
  "variance_threshold_amount": 1000,
  "auto_approve_small_variances": true,
  "reporting_currency": "USD",
  "enable_ai_insights": true
}
```

---

#### Table: `departments`

```sql
CREATE TABLE departments (
    department_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    department_name VARCHAR(255) NOT NULL,
    department_code VARCHAR(50) NOT NULL,
    parent_department_id UUID REFERENCES departments(department_id),
    is_cost_center BOOLEAN DEFAULT TRUE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,

    UNIQUE(tenant_id, department_code)
);

CREATE INDEX idx_departments_tenant ON departments(tenant_id);
CREATE INDEX idx_departments_parent ON departments(parent_department_id);

-- Row-Level Security
ALTER TABLE departments ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON departments
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE departments IS 'Organizational departments within a company';
```

---

#### Table: `users`

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL,
    department_id UUID REFERENCES departments(department_id),
    role VARCHAR(50) NOT NULL
        CHECK (role IN ('admin', 'accountant', 'analyst', 'manager', 'viewer')),
    is_active BOOLEAN DEFAULT TRUE,
    last_login TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    -- OAuth integration
    oauth_provider VARCHAR(50), -- 'microsoft', 'auth0', etc.
    oauth_sub VARCHAR(255) UNIQUE -- OAuth subject identifier
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_oauth_sub ON users(oauth_sub);

-- Row-Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE users IS 'System users with role-based access control';
```

---

### 5.2 PRODUCT DOMAIN

#### Table: `products`

```sql
CREATE TABLE products (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    product_code VARCHAR(100) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    product_category VARCHAR(100),
    unit_of_measure VARCHAR(20) NOT NULL, -- 'units', 'kg', 'lbs', 'liters', etc.
    selling_price NUMERIC(15, 2),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,

    -- Additional attributes
    description TEXT,
    attributes JSONB DEFAULT '{}'::jsonb,

    UNIQUE(tenant_id, product_code)
);

CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_products_code ON products(tenant_id, product_code);
CREATE INDEX idx_products_category ON products(tenant_id, product_category);

-- Row-Level Security
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON products
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE products IS 'Products manufactured or sold by the company';
```

---

#### Table: `materials`

```sql
CREATE TABLE materials (
    material_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    material_code VARCHAR(100) NOT NULL,
    material_name VARCHAR(255) NOT NULL,
    material_type VARCHAR(50) NOT NULL
        CHECK (material_type IN ('raw', 'component', 'packaging', 'consumable')),
    unit_of_measure VARCHAR(20) NOT NULL,
    standard_cost NUMERIC(15, 4),
    reorder_level NUMERIC(15, 2),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,

    UNIQUE(tenant_id, material_code)
);

CREATE INDEX idx_materials_tenant ON materials(tenant_id);
CREATE INDEX idx_materials_code ON materials(tenant_id, material_code);
CREATE INDEX idx_materials_type ON materials(tenant_id, material_type);

-- Row-Level Security
ALTER TABLE materials ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON materials
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE materials IS 'Raw materials, components, and supplies';
```

---

#### Table: `bill_of_materials`

```sql
CREATE TABLE bill_of_materials (
    bom_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    material_id UUID NOT NULL REFERENCES materials(material_id) ON DELETE CASCADE,
    quantity_required NUMERIC(15, 4) NOT NULL CHECK (quantity_required > 0),
    unit_of_measure VARCHAR(20) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    effective_date DATE NOT NULL DEFAULT CURRENT_DATE,
    expiration_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(tenant_id, product_id, material_id, effective_date)
);

CREATE INDEX idx_bom_tenant_product ON bill_of_materials(tenant_id, product_id);
CREATE INDEX idx_bom_material ON bill_of_materials(material_id);

-- Row-Level Security
ALTER TABLE bill_of_materials ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON bill_of_materials
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE bill_of_materials IS 'Bill of materials: materials required to produce each product';
```

---

### 5.3 COST DOMAIN

#### Table: `standard_costs`

```sql
CREATE TABLE standard_costs (
    standard_cost_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    effective_date DATE NOT NULL,
    expiration_date DATE,

    -- Cost components
    direct_material_cost NUMERIC(15, 4) NOT NULL DEFAULT 0,
    direct_labor_cost NUMERIC(15, 4) NOT NULL DEFAULT 0,
    variable_overhead_cost NUMERIC(15, 4) NOT NULL DEFAULT 0,
    fixed_overhead_cost NUMERIC(15, 4) NOT NULL DEFAULT 0,

    -- Calculated
    total_standard_cost NUMERIC(15, 4) GENERATED ALWAYS AS
        (direct_material_cost + direct_labor_cost + variable_overhead_cost + fixed_overhead_cost) STORED,

    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,

    UNIQUE(tenant_id, product_id, effective_date)
);

CREATE INDEX idx_standard_costs_tenant_product ON standard_costs(tenant_id, product_id);
CREATE INDEX idx_standard_costs_effective_date ON standard_costs(effective_date DESC);

-- Row-Level Security
ALTER TABLE standard_costs ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON standard_costs
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE standard_costs IS 'Predetermined standard costs for products';
```

---

#### Table: `overhead_rates`

```sql
CREATE TABLE overhead_rates (
    overhead_rate_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    department_id UUID REFERENCES departments(department_id),
    rate_type VARCHAR(50) NOT NULL
        CHECK (rate_type IN ('variable', 'fixed')),
    allocation_base VARCHAR(50) NOT NULL
        CHECK (allocation_base IN ('direct_labor_hours', 'machine_hours', 'direct_labor_cost', 'units')),

    -- Rate calculation
    estimated_overhead_cost NUMERIC(15, 2) NOT NULL,
    estimated_allocation_base NUMERIC(15, 2) NOT NULL CHECK (estimated_allocation_base > 0),
    overhead_rate NUMERIC(15, 4) GENERATED ALWAYS AS
        (estimated_overhead_cost / estimated_allocation_base) STORED,

    effective_date DATE NOT NULL,
    expiration_date DATE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID
);

CREATE INDEX idx_overhead_rates_tenant ON overhead_rates(tenant_id);
CREATE INDEX idx_overhead_rates_dept ON overhead_rates(department_id);

-- Row-Level Security
ALTER TABLE overhead_rates ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON overhead_rates
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE overhead_rates IS 'Predetermined overhead rates for cost allocation';
```

---

#### Table: `labor_rates`

```sql
CREATE TABLE labor_rates (
    labor_rate_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    department_id UUID REFERENCES departments(department_id),
    labor_category VARCHAR(100) NOT NULL, -- 'skilled', 'unskilled', 'supervisory'
    standard_rate_per_hour NUMERIC(10, 2) NOT NULL CHECK (standard_rate_per_hour > 0),
    overtime_rate_per_hour NUMERIC(10, 2),
    effective_date DATE NOT NULL,
    expiration_date DATE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID
);

CREATE INDEX idx_labor_rates_tenant ON labor_rates(tenant_id);
CREATE INDEX idx_labor_rates_dept ON labor_rates(department_id);

-- Row-Level Security
ALTER TABLE labor_rates ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON labor_rates
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE labor_rates IS 'Standard labor rates by category and department';
```

---

### 5.4 TRANSACTION DOMAIN

#### Table: `production_orders`

```sql
CREATE TABLE production_orders (
    production_order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    order_number VARCHAR(100) NOT NULL,
    product_id UUID NOT NULL REFERENCES products(product_id),
    department_id UUID REFERENCES departments(department_id),

    -- Order details
    quantity_ordered NUMERIC(15, 2) NOT NULL CHECK (quantity_ordered > 0),
    quantity_completed NUMERIC(15, 2) DEFAULT 0,
    quantity_scrapped NUMERIC(15, 2) DEFAULT 0,

    -- Dates
    order_date DATE NOT NULL,
    start_date DATE,
    completion_date DATE,

    -- Status
    status VARCHAR(50) NOT NULL DEFAULT 'planned'
        CHECK (status IN ('planned', 'released', 'in_progress', 'completed', 'cancelled')),

    -- Costing method for this order
    costing_method VARCHAR(20) NOT NULL DEFAULT 'standard'
        CHECK (costing_method IN ('standard', 'actual')),

    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,

    UNIQUE(tenant_id, order_number)
);

CREATE INDEX idx_production_orders_tenant ON production_orders(tenant_id);
CREATE INDEX idx_production_orders_product ON production_orders(product_id);
CREATE INDEX idx_production_orders_status ON production_orders(tenant_id, status);
CREATE INDEX idx_production_orders_dates ON production_orders(tenant_id, order_date DESC);

-- Row-Level Security
ALTER TABLE production_orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON production_orders
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE production_orders IS 'Production orders (jobs) for manufacturing products';
```

---

#### Table: `actual_costs`

```sql
CREATE TABLE actual_costs (
    actual_cost_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    production_order_id UUID NOT NULL REFERENCES production_orders(production_order_id) ON DELETE CASCADE,
    cost_type VARCHAR(50) NOT NULL
        CHECK (cost_type IN ('material', 'labor', 'overhead')),
    cost_category VARCHAR(100), -- 'direct_material', 'indirect_material', 'skilled_labor', etc.

    -- Cost details
    cost_date DATE NOT NULL,
    quantity NUMERIC(15, 4),
    unit_cost NUMERIC(15, 4),
    total_cost NUMERIC(15, 2) NOT NULL,

    -- References
    material_id UUID REFERENCES materials(material_id),
    department_id UUID REFERENCES departments(department_id),

    -- Metadata
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID
);

CREATE INDEX idx_actual_costs_tenant ON actual_costs(tenant_id);
CREATE INDEX idx_actual_costs_order ON actual_costs(production_order_id);
CREATE INDEX idx_actual_costs_type ON actual_costs(tenant_id, cost_type);
CREATE INDEX idx_actual_costs_date ON actual_costs(tenant_id, cost_date DESC);

-- Row-Level Security
ALTER TABLE actual_costs ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON actual_costs
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE actual_costs IS 'Actual costs incurred for production orders';
```

---

### 5.5 ANALYSIS DOMAIN

#### Table: `cost_variances`

```sql
CREATE TABLE cost_variances (
    variance_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    production_order_id UUID REFERENCES production_orders(production_order_id),
    product_id UUID REFERENCES products(product_id),

    -- Variance details
    variance_type VARCHAR(50) NOT NULL
        CHECK (variance_type IN (
            'material_price', 'material_quantity', 'material_mix',
            'labor_rate', 'labor_efficiency',
            'variable_overhead_spending', 'variable_overhead_efficiency',
            'fixed_overhead_spending', 'fixed_overhead_volume',
            'sales_price', 'sales_volume'
        )),

    -- Amounts
    standard_amount NUMERIC(15, 2) NOT NULL,
    actual_amount NUMERIC(15, 2) NOT NULL,
    variance_amount NUMERIC(15, 2) GENERATED ALWAYS AS (actual_amount - standard_amount) STORED,
    variance_percent NUMERIC(8, 2) GENERATED ALWAYS AS
        (CASE WHEN standard_amount != 0
         THEN ((actual_amount - standard_amount) / standard_amount) * 100
         ELSE 0 END) STORED,

    -- Classification
    is_favorable BOOLEAN GENERATED ALWAYS AS (variance_amount < 0) STORED,
    is_material BOOLEAN DEFAULT FALSE, -- Exceeds threshold

    -- Period
    period_year INTEGER NOT NULL,
    period_month INTEGER NOT NULL CHECK (period_month BETWEEN 1 AND 12),
    period_date DATE NOT NULL,

    -- Status
    investigation_status VARCHAR(50) DEFAULT 'pending'
        CHECK (investigation_status IN ('pending', 'investigating', 'explained', 'resolved')),

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID
);

CREATE INDEX idx_variances_tenant ON cost_variances(tenant_id);
CREATE INDEX idx_variances_order ON cost_variances(production_order_id);
CREATE INDEX idx_variances_type ON cost_variances(tenant_id, variance_type);
CREATE INDEX idx_variances_period ON cost_variances(tenant_id, period_year, period_month);
CREATE INDEX idx_variances_material ON cost_variances(tenant_id, is_material) WHERE is_material = TRUE;

-- Row-Level Security
ALTER TABLE cost_variances ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON cost_variances
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE cost_variances IS 'Cost variances (standard vs actual)';
```

---

#### Table: `variance_explanations`

```sql
CREATE TABLE variance_explanations (
    explanation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    variance_id UUID NOT NULL REFERENCES cost_variances(variance_id) ON DELETE CASCADE,

    -- AI-generated explanation
    explanation_text TEXT NOT NULL,
    potential_causes JSONB, -- Array of causes
    recommended_actions JSONB, -- Array of recommendations
    severity VARCHAR(20) CHECK (severity IN ('low', 'medium', 'high', 'critical')),

    -- Metadata
    generated_by VARCHAR(50) DEFAULT 'ai', -- 'ai' or 'user'
    ai_model VARCHAR(50), -- 'gpt-4', 'gpt-3.5-turbo'
    confidence_score NUMERIC(3, 2), -- 0.00 to 1.00

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID
);

CREATE INDEX idx_variance_explanations_tenant ON variance_explanations(tenant_id);
CREATE INDEX idx_variance_explanations_variance ON variance_explanations(variance_id);

-- Row-Level Security
ALTER TABLE variance_explanations ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON variance_explanations
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE variance_explanations IS 'AI-generated explanations for cost variances';
```

---

### 5.6 REPORTING DOMAIN

#### Table: `cost_reports`

```sql
CREATE TABLE cost_reports (
    report_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,
    report_type VARCHAR(100) NOT NULL
        CHECK (report_type IN (
            'product_cost', 'department_cost', 'variance_analysis',
            'budget_vs_actual', 'trend_analysis', 'executive_summary',
            'profitability', 'contribution_margin', 'break_even'
        )),

    -- Report parameters
    report_name VARCHAR(255) NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,

    -- Filters
    product_id UUID REFERENCES products(product_id),
    department_id UUID REFERENCES departments(department_id),
    filters JSONB, -- Additional filters

    -- Report data (stored as JSONB)
    report_data JSONB NOT NULL,

    -- AI-generated summary
    executive_summary TEXT,
    key_insights JSONB, -- Array of insights

    -- Metadata
    generated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    generated_by UUID,

    -- Status
    status VARCHAR(50) DEFAULT 'generated'
        CHECK (status IN ('generating', 'generated', 'failed')),
    file_url TEXT -- If exported to file
);

CREATE INDEX idx_cost_reports_tenant ON cost_reports(tenant_id);
CREATE INDEX idx_cost_reports_type ON cost_reports(tenant_id, report_type);
CREATE INDEX idx_cost_reports_period ON cost_reports(tenant_id, period_start, period_end);

-- Row-Level Security
ALTER TABLE cost_reports ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON cost_reports
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

COMMENT ON TABLE cost_reports IS 'Generated cost reports with AI summaries';
```

---

#### Table: `audit_logs`

```sql
CREATE TABLE audit_logs (
    log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES companies(company_id) ON DELETE CASCADE,

    -- User and action
    user_id UUID REFERENCES users(user_id),
    action VARCHAR(100) NOT NULL, -- 'create', 'update', 'delete', 'calculate', 'generate_report'
    resource_type VARCHAR(100) NOT NULL, -- 'product', 'cost', 'report', etc.
    resource_id UUID,

    -- Details
    description TEXT,
    old_values JSONB,
    new_values JSONB,

    -- Request metadata
    ip_address INET,
    user_agent TEXT,
    request_id UUID,

    -- Timestamp (immutable)
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

CREATE INDEX idx_audit_logs_tenant ON audit_logs(tenant_id);
CREATE INDEX idx_audit_logs_user ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_timestamp ON audit_logs(created_at DESC);

-- Row-Level Security
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON audit_logs
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Prevent updates and deletes (audit logs are immutable)
CREATE RULE no_update_audit_logs AS ON UPDATE TO audit_logs DO INSTEAD NOTHING;
CREATE RULE no_delete_audit_logs AS ON DELETE TO audit_logs DO INSTEAD NOTHING;

COMMENT ON TABLE audit_logs IS 'Immutable audit trail for compliance';
```

---

## 6. Multi-Tenancy Implementation

### 6.1 Row-Level Security (RLS) Setup

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE companies ENABLE ROW LEVEL SECURITY;
ALTER TABLE departments ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE materials ENABLE ROW LEVEL SECURITY;
ALTER TABLE bill_of_materials ENABLE ROW LEVEL SECURITY;
ALTER TABLE standard_costs ENABLE ROW LEVEL SECURITY;
ALTER TABLE overhead_rates ENABLE ROW LEVEL SECURITY;
ALTER TABLE labor_rates ENABLE ROW LEVEL SECURITY;
ALTER TABLE production_orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE actual_costs ENABLE ROW LEVEL SECURITY;
ALTER TABLE cost_variances ENABLE ROW LEVEL SECURITY;
ALTER TABLE variance_explanations ENABLE ROW LEVEL SECURITY;
ALTER TABLE cost_reports ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

-- Create RLS policy for all tables (example for one table)
CREATE POLICY tenant_isolation ON departments
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Repeat for all tables with tenant_id
```

### 6.2 Setting Tenant Context

```python
# In Python (application layer)
async def set_tenant_context(conn, tenant_id: uuid.UUID):
    """Set the tenant context for this database connection"""
    await conn.execute(f"SET app.current_tenant_id = '{tenant_id}'")

# Usage in MCP tools
async def calculate_product_cost(product_id: str, tenant_id: str):
    async with db_pool.acquire() as conn:
        await set_tenant_context(conn, UUID(tenant_id))
        # All subsequent queries automatically filtered by tenant_id
        result = await conn.fetch("SELECT * FROM products WHERE product_id = $1", UUID(product_id))
```

### 6.3 Tenant Isolation Testing

```sql
-- Test script to verify tenant isolation
-- User from Tenant A should NOT see data from Tenant B

SET app.current_tenant_id = 'tenant-a-uuid';
SELECT * FROM products; -- Should only return Tenant A products

SET app.current_tenant_id = 'tenant-b-uuid';
SELECT * FROM products; -- Should only return Tenant B products
```

---

## 7. Indexes and Performance

### 7.1 Primary Indexes

```sql
-- Tenant-based queries (most common)
CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_orders_tenant ON production_orders(tenant_id);
CREATE INDEX idx_costs_tenant ON actual_costs(tenant_id);
CREATE INDEX idx_variances_tenant ON cost_variances(tenant_id);

-- Composite indexes for common queries
CREATE INDEX idx_orders_tenant_status ON production_orders(tenant_id, status);
CREATE INDEX idx_costs_tenant_date ON actual_costs(tenant_id, cost_date DESC);
CREATE INDEX idx_variances_tenant_period ON cost_variances(tenant_id, period_year, period_month);

-- Foreign key indexes
CREATE INDEX idx_bom_product ON bill_of_materials(product_id);
CREATE INDEX idx_bom_material ON bill_of_materials(material_id);
CREATE INDEX idx_costs_order ON actual_costs(production_order_id);
```

### 7.2 Partial Indexes

```sql
-- Index only active records (more efficient)
CREATE INDEX idx_products_active ON products(tenant_id) WHERE is_active = TRUE;
CREATE INDEX idx_materials_active ON materials(tenant_id) WHERE is_active = TRUE;

-- Index only material variances (for investigation)
CREATE INDEX idx_variances_material ON cost_variances(tenant_id, is_material)
    WHERE is_material = TRUE;
```

### 7.3 JSONB Indexes

```sql
-- Index for JSONB columns (for fast lookups)
CREATE INDEX idx_companies_config ON companies USING GIN (config);
CREATE INDEX idx_reports_data ON cost_reports USING GIN (report_data);
```

---

## 8. Constraints and Validations

### 8.1 Check Constraints

```sql
-- Ensure positive quantities
ALTER TABLE bill_of_materials
    ADD CONSTRAINT positive_quantity CHECK (quantity_required > 0);

-- Ensure valid date ranges
ALTER TABLE standard_costs
    ADD CONSTRAINT valid_date_range CHECK (expiration_date IS NULL OR expiration_date > effective_date);

-- Ensure cost values are non-negative
ALTER TABLE standard_costs
    ADD CONSTRAINT non_negative_costs CHECK (
        direct_material_cost >= 0 AND
        direct_labor_cost >= 0 AND
        variable_overhead_cost >= 0 AND
        fixed_overhead_cost >= 0
    );
```

### 8.2 Triggers

```sql
-- Update timestamp trigger function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to all tables with updated_at
CREATE TRIGGER update_products_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Repeat for all tables with updated_at column
```

---

## 9. Sample Data

### 9.1 Sample Company

```sql
INSERT INTO companies (company_id, company_name, company_code, industry, costing_method)
VALUES
    ('123e4567-e89b-12d3-a456-426614174000', 'Acme Manufacturing', 'ACME', 'Manufacturing', 'standard');
```

### 9.2 Sample Products

```sql
INSERT INTO products (product_id, tenant_id, product_code, product_name, unit_of_measure, selling_price)
VALUES
    (gen_random_uuid(), '123e4567-e89b-12d3-a456-426614174000', 'PROD-001', 'Widget A', 'units', 50.00),
    (gen_random_uuid(), '123e4567-e89b-12d3-a456-426614174000', 'PROD-002', 'Widget B', 'units', 75.00);
```

### 9.3 Sample Standard Costs

```sql
INSERT INTO standard_costs (tenant_id, product_id, effective_date, direct_material_cost, direct_labor_cost, variable_overhead_cost, fixed_overhead_cost)
VALUES
    ('123e4567-e89b-12d3-a456-426614174000',
     (SELECT product_id FROM products WHERE product_code = 'PROD-001' LIMIT 1),
     '2025-01-01', 15.00, 10.00, 5.00, 8.00);
```

---

## 10. Migration Strategy

### 10.1 Migration Tool: Alembic

```bash
# Initialize Alembic
alembic init alembic

# Create migration
alembic revision --autogenerate -m "Create initial schema"

# Apply migration
alembic upgrade head

# Rollback
alembic downgrade -1
```

### 10.2 Migration Scripts

```python
# alembic/versions/001_initial_schema.py
def upgrade():
    # Create all tables
    op.create_table('companies', ...)
    op.create_table('products', ...)
    # ...

def downgrade():
    # Drop all tables
    op.drop_table('products')
    op.drop_table('companies')
```

---

## Appendix A: Complete Schema Script

See `database/schema.sql` for the complete SQL script to create all tables, indexes, and constraints.

---

## Appendix B: ER Diagram (Detailed)

See `database/er_diagram.png` for a visual representation of all table relationships.

---

## Document Approval

**Prepared By:**
Database Architecture Team
Date: December 28, 2025

**Reviewed By:**
- [ ] Database Architect
- [ ] Security Architect
- [ ] Lead Developer

**Approved By:**
- [ ] CTO / Technical Director
Date: _______________

---

**End of Database Schema Document**
