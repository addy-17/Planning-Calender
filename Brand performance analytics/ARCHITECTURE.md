# BrandIQ — Enterprise Architecture v2.0

## Architecture Document

---

# Table of Contents

1. [System Overview](#1-system-overview)
2. [Data Model & Schema](#2-data-model--schema)
3. [Folder Structure](#3-folder-structure)
4. [API Endpoints](#4-api-endpoints)
5. [Authentication Flow](#5-authentication-flow)
6. [Frontend Architecture](#6-frontend-architecture)
7. [Backend Architecture](#7-backend-architecture)
8. [File Upload & Processing Pipeline](#8-file-upload--processing-pipeline)
9. [Component Hierarchy](#9-component-hierarchy)
10. [Routing Design](#10-routing-design)
11. [State Management](#11-state-management)
12. [AI Summary Architecture](#12-ai-summary-architecture)
13. [PDF Report Architecture](#13-pdf-report-architecture)
14. [Design System](#14-design-system)

---

# 1. System Overview

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Client Browser                               │
│              Next.js 15 (App Router + SSR/CSR Hybrid)                │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ HTTPS
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         Go Backend (Gin)                              │
│                         Port 8080                                    │
│                                                                       │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐     │
│  │ Auth    │  │ Upload   │  │ Analysis │  │ Report           │     │
│  │ Handler │  │ Handler  │  │ Handler  │  │ Generator        │     │
│  └────┬────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘     │
│       │            │             │                   │               │
│  ┌────▼────────────▼─────────────▼───────────────────▼──────────┐   │
│  │                    Service Layer                               │   │
│  │  AuthService | UploadService | AnalysisService | ReportService │   │
│  └────────────────────────┬──────────────────────────────────────┘   │
│                           │                                          │
│  ┌────────────────────────▼──────────────────────────────────────┐   │
│  │                    Repository Layer (GORM)                      │   │
│  │  UserRepo | BillRepo | ProductRepo | BrandRepo | ...          │   │
│  └────────────────────────┬──────────────────────────────────────┘   │
└───────────────────────────┼──────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌──────────────┐ ┌───────────┐ ┌──────────────┐
    │  PostgreSQL   │ │   Redis   │ │  File System  │
    │  (Primary DB) │ │  (Cache)  │ │ (Uploaded     │
    │               │ │           │ │  Excel files) │
    └──────────────┘ └───────────┘ └──────────────┘
```

## Input Data Model

### File 1: POS Bill Register (Excel/CSV)
| Column | Description |
|--------|-------------|
| Bill No | Invoice number |
| Bill Date | Transaction date |
| Item Code | SKU (join key with Product Master) |
| Item Name | Product description |
| Quantity | Units sold |
| Rate | Selling price per unit |
| Amount | Gross amount (qty × rate) |
| Discount % | Discount percentage |
| Discount Amount | Discount value |
| Net Amount | Final amount after discount |
| Payment Mode | Cash/Card/UPI/Wallet/Gift Card |
| Customer Name | (optional) |
| Customer Mobile | (optional) |

### File 2: Product Master (Excel/CSV)
| Column | Description |
|--------|-------------|
| Item Code | SKU (join key with POS) |
| Item Name | Product description |
| Brand | Brand name |
| Department | Menswear/Women's/Kids/etc |
| Category | Product category |
| Sub Category | Product subcategory |
| MRP | Maximum Retail Price |
| Cost Price | (optional) |
| Size | Size variant |
| Color | Color variant |
| Vendor | Supplier name |
| HSN Code | Tax classification |

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **No Inventory Dependency** | All metrics from bills + products | Business requirement — no stock report exists |
| **Weekly Cadence** | Monday analysis | Owner reviews weekends, not daily |
| **Data Join** | Item Code | Primary key between both files |
| **Backend** | Go + Gin | High throughput for Excel parsing + aggregation |
| **ORM** | GORM | Auto-migration, PostgreSQL support |
| **Frontend** | Next.js 15 App Router | SSR + RSC for performance |
| **State** | Zustand + TanStack Query | UI state + Server state separation |
| **Charts** | Apache ECharts | Heavy dashboard rendering |
| **Reports** | Go Chromedp + Excelize | Server-side PDF/XLSX generation |
| **AI Summary** | Template-based NLG | LLM-ready architecture for future Gemini/OpenAI |

---

# 2. Data Model & Schema

## Entity Relationship

```
users (1)──(N) upload_jobs
brands (1)──(N) products
departments (1)──(N) products
categories (1)──(N) products
vendors (1)──(N) products
products (1)──(N) bill_items
bills (1)──(N) bill_items
upload_jobs (1)──(N) audit_logs
```

## PostgreSQL Tables

### `users`
```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(20) NOT NULL DEFAULT 'analyst'
                    CHECK (role IN ('admin', 'manager', 'analyst', 'viewer')),
    avatar_url      TEXT,
    is_active       BOOLEAN DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    refresh_token   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `brands`
```sql
CREATE TABLE brands (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    code            VARCHAR(50) GENERATED ALWAYS AS (UPPER(LEFT(name, 3))) STORED,
    is_active       BOOLEAN DEFAULT true,
    first_seen_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `departments`
```sql
CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `categories`
```sql
CREATE TABLE categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    department_id   UUID REFERENCES departments(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `vendors`
```sql
CREATE TABLE vendors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    contact_info    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `products`
```sql
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_code       VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(500) NOT NULL,
    brand_id        UUID NOT NULL REFERENCES brands(id),
    department_id   UUID REFERENCES departments(id),
    category_id     UUID REFERENCES categories(id),
    vendor_id       UUID REFERENCES vendors(id),
    mrp             DECIMAL(12,2) DEFAULT 0,
    cost_price      DECIMAL(12,2) DEFAULT 0,
    size            VARCHAR(50),
    color           VARCHAR(50),
    hsn_code        VARCHAR(20),
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_item_code ON products(item_code);
CREATE INDEX idx_products_brand_id ON products(brand_id);
CREATE INDEX idx_products_department_id ON products(department_id);
CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_vendor_id ON products(vendor_id);
```

### `bills`
```sql
CREATE TABLE bills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bill_no         VARCHAR(100) NOT NULL,
    bill_date       DATE NOT NULL,
    payment_mode    VARCHAR(50),
    customer_name   VARCHAR(255),
    customer_mobile VARCHAR(20),
    total_amount    DECIMAL(14,2) DEFAULT 0,
    total_discount  DECIMAL(14,2) DEFAULT 0,
    net_amount      DECIMAL(14,2) DEFAULT 0,
    item_count      INTEGER DEFAULT 0,
    upload_job_id   UUID REFERENCES upload_jobs(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(bill_no, bill_date)
);

CREATE INDEX idx_bills_date ON bills(bill_date);
CREATE INDEX idx_bills_payment_mode ON bills(payment_mode);
CREATE INDEX idx_bills_upload_job ON bills(upload_job_id);
```

### `bill_items`
```sql
CREATE TABLE bill_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bill_id         UUID NOT NULL REFERENCES bills(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(id),
    quantity        INTEGER NOT NULL DEFAULT 1,
    rate            DECIMAL(12,2) DEFAULT 0,
    amount          DECIMAL(14,2) DEFAULT 0,
    discount_pct    DECIMAL(5,2) DEFAULT 0,
    discount_amount DECIMAL(14,2) DEFAULT 0,
    net_amount      DECIMAL(14,2) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bill_items_bill_id ON bill_items(bill_id);
CREATE INDEX idx_bill_items_product_id ON bill_items(product_id);
```

### `upload_jobs`
```sql
CREATE TABLE upload_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    file_type       VARCHAR(20) NOT NULL CHECK (file_type IN ('pos_bills', 'product_master')),
    file_name       VARCHAR(500) NOT NULL,
    file_path       TEXT,
    file_size       BIGINT,
    total_rows      INTEGER DEFAULT 0,
    processed_rows  INTEGER DEFAULT 0,
    error_rows      INTEGER DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'uploaded'
                    CHECK (status IN ('uploaded', 'validating', 'processing', 'completed', 'failed')),
    error_log       JSONB DEFAULT '[]',
    week_start_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_upload_jobs_status ON upload_jobs(status);
CREATE INDEX idx_upload_jobs_week ON upload_jobs(week_start_date);
```

### `weekly_summaries`
```sql
CREATE TABLE weekly_summaries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    week_start_date     DATE NOT NULL UNIQUE,
    total_revenue       DECIMAL(16,2) DEFAULT 0,
    total_bills         INTEGER DEFAULT 0,
    total_units_sold    INTEGER DEFAULT 0,
    avg_bill_value      DECIMAL(12,2) DEFAULT 0,
    avg_discount_pct    DECIMAL(5,2) DEFAULT 0,
    top_brand_id        UUID REFERENCES brands(id),
    top_vendor_id       UUID REFERENCES vendors(id),
    top_department_id   UUID REFERENCES departments(id),
    top_category_id     UUID REFERENCES categories(id),
    top_product_id      UUID REFERENCES products(id),
    fastest_growing_brand_id UUID REFERENCES brands(id),
    revenue_trend       JSONB DEFAULT '[]',
    ai_summary          TEXT,
    is_current          BOOLEAN DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `brand_weekly_metrics`
```sql
CREATE TABLE brand_weekly_metrics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_id            UUID NOT NULL REFERENCES brands(id),
    week_start_date     DATE NOT NULL,
    revenue             DECIMAL(14,2) DEFAULT 0,
    units_sold          INTEGER DEFAULT 0,
    bills_count         INTEGER DEFAULT 0,
    avg_selling_price   DECIMAL(12,2) DEFAULT 0,
    avg_discount_pct    DECIMAL(5,2) DEFAULT 0,
    revenue_contribution DECIMAL(5,2) DEFAULT 0,
    growth_pct          DECIMAL(8,2) DEFAULT 0,
    rank                INTEGER,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(brand_id, week_start_date)
);

CREATE INDEX idx_brand_metrics_week ON brand_weekly_metrics(week_start_date);
CREATE INDEX idx_brand_metrics_rank ON brand_weekly_metrics(week_start_date, rank);
```

### `product_weekly_metrics`
```sql
CREATE TABLE product_weekly_metrics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id          UUID NOT NULL REFERENCES products(id),
    brand_id            UUID NOT NULL REFERENCES brands(id),
    week_start_date     DATE NOT NULL,
    revenue             DECIMAL(14,2) DEFAULT 0,
    units_sold          INTEGER DEFAULT 0,
    bills_count         INTEGER DEFAULT 0,
    avg_selling_price   DECIMAL(12,2) DEFAULT 0,
    avg_discount_pct    DECIMAL(5,2) DEFAULT 0,
    rank                INTEGER,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(product_id, week_start_date)
);

CREATE INDEX idx_product_metrics_week ON product_weekly_metrics(week_start_date);
```

### `audit_logs`
```sql
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50),
    resource_id     UUID,
    details         JSONB DEFAULT '{}',
    ip_address      VARCHAR(45),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Redis Schema
```
# Cache
brandiq:cache:{resource}:{week}                 → JSON (TTL: 1hr)

# Rate limiting
brandiq:ratelimit:{user_id}:{endpoint}          → counter (TTL: sliding window)

# Session
brandiq:session:{user_id}                       → JSON (TTL: 7d)

# Upload progress
brandiq:upload:{job_id}:progress                → JSON (real-time)

# Auth lockout
brandiq:lockout:{email}                         → counter (TTL: 15min)
```

---

# 3. Folder Structure

## Frontend (`frontend/`)
```
frontend/
├── public/
│   ├── static/
│   │   ├── fonts/
│   │   └── images/
│   └── manifests/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   │   └── page.tsx
│   │   │   └── forgot-password/
│   │   │       └── page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── dashboard/
│   │   │   │   ├── page.tsx
│   │   │   │   └── loading.tsx
│   │   │   ├── brands/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [brandId]/
│   │   │   │       └── page.tsx
│   │   │   ├── products/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [productId]/
│   │   │   │       └── page.tsx
│   │   │   ├── departments/
│   │   │   │   └── page.tsx
│   │   │   ├── categories/
│   │   │   │   └── page.tsx
│   │   │   ├── vendors/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [vendorId]/
│   │   │   │       └── page.tsx
│   │   │   ├── discounts/
│   │   │   │   └── page.tsx
│   │   │   ├── payments/
│   │   │   │   └── page.tsx
│   │   │   ├── timeline/
│   │   │   │   └── page.tsx
│   │   │   ├── ai-summary/
│   │   │   │   └── page.tsx
│   │   │   ├── reports/
│   │   │   │   └── page.tsx
│   │   │   ├── upload/
│   │   │   │   └── page.tsx
│   │   │   └── settings/
│   │   │       └── page.tsx
│   │   ├── layout.tsx
│   │   ├── not-found.tsx
│   │   ├── error.tsx
│   │   └── global-error.tsx
│   ├── components/
│   │   ├── ui/                       # shadcn/ui primitives
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── dropdown-menu.tsx
│   │   │   ├── select.tsx
│   │   │   ├── table.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── input.tsx
│   │   │   ├── form.tsx
│   │   │   ├── tooltip.tsx
│   │   │   ├── skeleton.tsx
│   │   │   ├── progress.tsx
│   │   │   ├── toast.tsx
│   │   │   ├── separator.tsx
│   │   │   └── chart.tsx
│   │   ├── layout/
│   │   │   ├── sidebar.tsx
│   │   │   ├── header.tsx
│   │   │   ├── mobile-nav.tsx
│   │   │   ├── breadcrumb.tsx
│   │   │   └── user-nav.tsx
│   │   ├── dashboard/
│   │   │   ├── kpi-card.tsx
│   │   │   ├── kpi-grid.tsx
│   │   │   ├── revenue-trend-chart.tsx
│   │   │   ├── top-brands-table.tsx
│   │   │   ├── ai-summary-card.tsx
│   │   │   └── upload-banner.tsx
│   │   ├── brands/
│   │   │   ├── brand-ranking-table.tsx
│   │   │   ├── brand-detail-header.tsx
│   │   │   ├── brand-metrics-grid.tsx
│   │   │   ├── brand-trend-chart.tsx
│   │   │   ├── brand-top-products.tsx
│   │   │   └── brand-distributions.tsx
│   │   ├── products/
│   │   │   ├── product-table.tsx
│   │   │   ├── product-detail-header.tsx
│   │   │   └── product-ranking.tsx
│   │   ├── departments/
│   │   │   ├── department-comparison-table.tsx
│   │   │   └── department-chart.tsx
│   │   ├── categories/
│   │   │   ├── category-table.tsx
│   │   │   └── category-breakdown.tsx
│   │   ├── vendors/
│   │   │   ├── vendor-ranking-table.tsx
│   │   │   └── vendor-products.tsx
│   │   ├── discounts/
│   │   │   ├── discount-metrics.tsx
│   │   │   ├── discount-scatter-chart.tsx
│   │   │   └── discount-brand-table.tsx
│   │   ├── payments/
│   │   │   ├── payment-distribution.tsx
│   │   │   └── payment-pie-chart.tsx
│   │   ├── timeline/
│   │   │   ├── daily-trend-chart.tsx
│   │   │   └── weekday-performance.tsx
│   │   ├── ai-summary/
│   │   │   ├── summary-report.tsx
│   │   │   └── recommendation-list.tsx
│   │   ├── upload/
│   │   │   ├── file-dropzone.tsx
│   │   │   ├── upload-progress.tsx
│   │   │   └── upload-history.tsx
│   │   ├── reports/
│   │   │   ├── report-generator.tsx
│   │   │   └── report-history.tsx
│   │   └── shared/
│   │       ├── glass-card.tsx
│   │       ├── loading-spinner.tsx
│   │       ├── error-state.tsx
│   │       ├── empty-state.tsx
│   │       ├── confirm-dialog.tsx
│   │       ├── search-input.tsx
│   │       └── filter-bar.tsx
│   ├── hooks/
│   │   ├── use-auth.ts
│   │   ├── use-debounce.ts
│   │   ├── use-media-query.ts
│   │   └── use-websocket.ts
│   ├── stores/
│   │   ├── auth-store.ts
│   │   ├── ui-store.ts
│   │   └── filter-store.ts
│   ├── lib/
│   │   ├── api-client.ts
│   │   ├── query-keys.ts
│   │   ├── query-provider.tsx
│   │   ├── auth-provider.tsx
│   │   ├── theme-provider.tsx
│   │   ├── utils.ts
│   │   └── validators/
│   │       ├── auth.ts
│   │       └── upload.ts
│   ├── types/
│   │   ├── api.ts
│   │   ├── brand.ts
│   │   ├── product.ts
│   │   ├── bill.ts
│   │   ├── dashboard.ts
│   │   ├── department.ts
│   │   ├── category.ts
│   │   ├── vendor.ts
│   │   ├── discount.ts
│   │   ├── payment.ts
│   │   ├── report.ts
│   │   └── user.ts
│   ├── config/
│   │   ├── navigation.ts
│   │   ├── site.ts
│   │   └── charts.ts
│   └── styles/
│       └── globals.css
├── .env.local
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── components.json
└── package.json
```

## Backend (`backend/`)
```
backend/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── config/
│   │   ├── config.go
│   │   └── config.yaml
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── cors.go
│   │   ├── ratelimit.go
│   │   ├── logger.go
│   │   └── recovery.go
│   ├── handler/
│   │   ├── auth_handler.go
│   │   ├── upload_handler.go
│   │   ├── dashboard_handler.go
│   │   ├── brand_handler.go
│   │   ├── product_handler.go
│   │   ├── department_handler.go
│   │   ├── category_handler.go
│   │   ├── vendor_handler.go
│   │   ├── discount_handler.go
│   │   ├── payment_handler.go
│   │   ├── timeline_handler.go
│   │   ├── summary_handler.go
│   │   ├── report_handler.go
│   │   ├── search_handler.go
│   │   ├── user_handler.go
│   │   └── health_handler.go
│   ├── service/
│   │   ├── auth_service.go
│   │   ├── upload_service.go
│   │   ├── dashboard_service.go
│   │   ├── brand_service.go
│   │   ├── product_service.go
│   │   ├── department_service.go
│   │   ├── category_service.go
│   │   ├── vendor_service.go
│   │   ├── discount_service.go
│   │   ├── payment_service.go
│   │   ├── timeline_service.go
│   │   ├── summary_service.go
│   │   ├── report_service.go
│   │   ├── search_service.go
│   │   └── analysis_service.go
│   ├── repository/
│   │   ├── user_repo.go
│   │   ├── brand_repo.go
│   │   ├── product_repo.go
│   │   ├── bill_repo.go
│   │   ├── bill_item_repo.go
│   │   ├── department_repo.go
│   │   ├── category_repo.go
│   │   ├── vendor_repo.go
│   │   ├── upload_repo.go
│   │   └── weekly_summary_repo.go
│   ├── model/
│   │   ├── user.go
│   │   ├── brand.go
│   │   ├── product.go
│   │   ├── bill.go
│   │   ├── bill_item.go
│   │   ├── department.go
│   │   ├── category.go
│   │   ├── vendor.go
│   │   ├── upload_job.go
│   │   ├── weekly_summary.go
│   │   ├── brand_weekly_metric.go
│   │   ├── product_weekly_metric.go
│   │   └── audit_log.go
│   ├── dto/
│   │   ├── auth_dto.go
│   │   ├── dashboard_dto.go
│   │   ├── brand_dto.go
│   │   ├── product_dto.go
│   │   ├── department_dto.go
│   │   ├── category_dto.go
│   │   ├── vendor_dto.go
│   │   ├── discount_dto.go
│   │   ├── payment_dto.go
│   │   ├── timeline_dto.go
│   │   ├── summary_dto.go
│   │   ├── report_dto.go
│   │   ├── upload_dto.go
│   │   └── common.go
│   ├── parser/
│   │   ├── pos_parser.go           # POS Bill Register parser
│   │   ├── product_parser.go       # Product Master parser
│   │   └── parser_test.go
│   ├── report/
│   │   ├── pdf_generator.go
│   │   └── templates/
│   │       ├── executive.html
│   │       ├── brand_report.html
│   │       └── styles.css
│   ├── scheduler/
│   │   ├── scheduler.go
│   │   └── weekly_analysis.go
│   └── router/
│       └── router.go
├── pkg/
│   ├── jwt/
│   │   ├── jwt.go
│   │   └── jwt_test.go
│   ├── password/
│   │   └── bcrypt.go
│   ├── excel/
│   │   └── excel.go
│   ├── response/
│   │   └── response.go
│   └── pagination/
│       └── pagination.go
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_brands.sql
│   ├── 003_create_departments.sql
│   ├── 004_create_categories.sql
│   ├── 005_create_vendors.sql
│   ├── 006_create_products.sql
│   ├── 007_create_bills.sql
│   ├── 008_create_bill_items.sql
│   ├── 009_create_upload_jobs.sql
│   ├── 010_create_weekly_summaries.sql
│   ├── 011_create_brand_metrics.sql
│   ├── 012_create_product_metrics.sql
│   └── 013_create_audit_logs.sql
├── scripts/
│   ├── seed.go
│   └── mock-data-generator.go
├── Dockerfile
├── docker-compose.yml
├── go.mod
├── Makefile
└── .env.example
```

---

# 4. API Endpoints

## Base URL: `/api/v1`

### Auth
| Method | Path | Description |
|--------|------|-------------|
| POST | `/auth/login` | Login with email + password |
| POST | `/auth/refresh` | Refresh access token |
| POST | `/auth/logout` | Invalidate session |
| GET | `/auth/me` | Current user profile |

### Upload
| Method | Path | Description |
|--------|------|-------------|
| POST | `/upload/pos` | Upload POS Bill Register (multipart) |
| POST | `/upload/products` | Upload Product Master (multipart) |
| GET | `/upload/jobs` | List upload history |
| GET | `/upload/jobs/:id` | Upload job status + errors |
| POST | `/upload/process` | Trigger analysis on uploaded files |

### Dashboard
| Method | Path | Description |
|--------|------|-------------|
| GET | `/dashboard/executive?week=2026-06-22` | Executive dashboard data |
| GET | `/dashboard/kpi?week=2026-06-22` | KPI cards data |

### Brands
| Method | Path | Description |
|--------|------|-------------|
| GET | `/brands?week=2026-06-22&page=1&per_page=20` | Brand ranking (all) |
| GET | `/brands/:id?week=2026-06-22` | Brand detail |
| GET | `/brands/:id/trends?weeks=12` | Brand weekly trend |
| GET | `/brands/:id/top-products?week=2026-06-22&limit=10` | Brand top/worst products |
| GET | `/brands/:id/distributions?week=2026-06-22` | Category/Dept/Color/Size/Payment distribution |
| GET | `/brands/top?week=2026-06-22&limit=10` | Top N brands |
| GET | `/brands/bottom?week=2026-06-22&limit=10` | Bottom N brands |

### Products
| Method | Path | Description |
|--------|------|-------------|
| GET | `/products?week=2026-06-22&page=1&per_page=50&sort=revenue` | Product listing |
| GET | `/products/:id?week=2026-06-22` | Product detail |
| GET | `/products/top?week=2026-06-22&limit=20` | Top products |
| GET | `/products/worst?week=2026-06-22&limit=20` | Worst products |

### Departments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/departments?week=2026-06-22` | Department comparison |
| GET | `/departments/:id?week=2026-06-22` | Department detail |

### Categories
| Method | Path | Description |
|--------|------|-------------|
| GET | `/categories?week=2026-06-22` | Category analytics |
| GET | `/categories/:id?week=2026-06-22` | Category detail |

### Vendors
| Method | Path | Description |
|--------|------|-------------|
| GET | `/vendors?week=2026-06-22&page=1&per_page=20` | Vendor ranking |
| GET | `/vendors/:id?week=2026-06-22` | Vendor detail |

### Discounts
| Method | Path | Description |
|--------|------|-------------|
| GET | `/discounts/summary?week=2026-06-22` | Discount analytics |
| GET | `/discounts/brands?week=2026-06-22` | Brand-wise discount |
| GET | `/discounts/scatter?week=2026-06-22` | Revenue vs Discount data |

### Payments
| Method | Path | Description |
|--------|------|-------------|
| GET | `/payments/distribution?week=2026-06-22` | Payment mode distribution |

### Timeline
| Method | Path | Description |
|--------|------|-------------|
| GET | `/timeline/daily?week=2026-06-22` | Daily revenue/units/bills |
| GET | `/timeline/weekday?week=2026-06-22` | Weekday vs weekend |

### AI Summary
| Method | Path | Description |
|--------|------|-------------|
| GET | `/summary?week=2026-06-22` | AI-generated business summary |
| POST | `/summary/regenerate?week=2026-06-22` | Force regenerate summary |

### Reports
| Method | Path | Description |
|--------|------|-------------|
| POST | `/reports/generate` | Generate PDF report |
| GET | `/reports?page=1&per_page=20` | Report history |
| GET | `/reports/:id/download` | Download report file |

### Search
| Method | Path | Description |
|--------|------|-------------|
| GET | `/search?q=query&type=brand,product,department,category,bill` | Global search |

### Admin
| Method | Path | Description |
|--------|------|-------------|
| POST | `/admin/run-analysis` | Force weekly analysis |
| GET | `/admin/health` | System health check |

---

# 5. Authentication Flow

## JWT Token Strategy

```
Access Token:  JWT (RS256), 15 min expiry
Refresh Token: Opaque 32-byte hex, 7 day expiry, stored in Redis

Login Flow:
  1. POST /auth/login { email, password }
  2. Server validates (bcrypt compare)
  3. Generates access_token (15m) + refresh_token (7d)
  4. Stores refresh_token hash in Redis
  5. Returns { access_token, refresh_token, user }

Token Refresh:
  1. Axios interceptor detects 401
  2. POST /auth/refresh { refresh_token }
  3. Server validates against Redis
  4. Rotates: invalidates old, generates new pair
  5. Retries original request

Logout:
  1. POST /auth/logout
  2. Remove refresh_token from Redis
  3. Client clears state
```

## RBAC Matrix

| Feature | admin | manager | analyst | viewer |
|---------|-------|---------|---------|--------|
| Dashboard | R | R | R | R |
| Brands | R | R | R | R |
| Products | R | R | R | R |
| Upload | W | W | W | - |
| Reports | W | W | W | R |
| AI Summary | R | R | R | R |
| Settings | W | R | - | - |
| Users | W | - | - | - |

---

# 6. Frontend Architecture

## Layer Architecture
```
┌────────────────────────────────────────┐
│           Pages (App Router)            │
│     Server Components by default        │
├────────────────────────────────────────┤
│       Feature Components (pages/)        │
│   Dashboard, Brands, Products, etc.     │
├────────────────────────────────────────┤
│        Shared Components (components/)   │
│   GlassCard, DataTable, Charts, KPI     │
├────────────────────────────────────────┤
│         UI Primitives (shadcn/ui)        │
│   Button, Card, Dialog, Table, Tabs     │
├────────────────────────────────────────┤
│            State Layer                   │
│  ┌──────────┐  ┌──────────┐             │
│  │ Zustand   │  │ TanStack │             │
│  │ Stores    │  │ Query    │             │
│  └──────────┘  └──────────┘             │
├────────────────────────────────────────┤
│          Service/API Layer (lib/)        │
│    api-client.ts, query-keys.ts         │
├────────────────────────────────────────┤
│              Backend API                 │
└────────────────────────────────────────┘
```

## Component Type Strategy

| Component Type | When to Use | Example |
|----------------|-------------|---------|
| **Server** | Static content, initial data fetch | Dashboard page shell |
| **Client** | Interactivity, charts, forms, state | KPI cards, tables, filters |
| **Suspense** | Section-level loading states | Chart loading skeleton |

## Chart Configuration

| Page | Charts |
|------|--------|
| Executive Dashboard | Line (revenue trend), Bar (top brands) |
| Brand Detail | Line (weekly trend), Pie (distributions), Bar (top products) |
| Discounts | Scatter (revenue vs discount) |
| Payments | Pie (payment distribution) |
| Timeline | Area (daily), Bar (weekday) |
| Departments | Grouped bar (comparison) |
| Categories | Bar (contribution) |

---

# 7. Backend Architecture

## Layered Pattern
```
Router → Middleware → Handler → Service → Repository → DB
                           │
                           └→ Parser (Excel/CSV)
                           └→ Report Generator
                           └→ AI Client
```

## Middleware Pipeline
```
Request → Recovery → CORS → Logger → Rate Limiter → Auth → Handler → Response
```

## Standardized Response
```json
// Success
{ "data": { ... }, "meta": { "page": 1, "per_page": 20, "total": 150 } }

// Error
{ "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [] } }
```

## Error Codes
| Code | HTTP | Description |
|------|------|-------------|
| VALIDATION_ERROR | 400 | Invalid input |
| UNAUTHORIZED | 401 | Invalid/missing token |
| FORBIDDEN | 403 | Insufficient role |
| NOT_FOUND | 404 | Resource not found |
| CONFLICT | 409 | Duplicate |
| RATE_LIMITED | 429 | Too many requests |
| INTERNAL_ERROR | 500 | Server error |

---

# 8. File Upload & Processing Pipeline

## Upload Flow

```
User uploads POS Bill Register (.xlsx/.csv)
         │
         ▼
┌──────────────────┐
│  File Received    │
│  - Validate MIME  │
│  - Size check     │
│  - Store to disk  │
│  - Create Job     │
└──────┬───────────┘
       ▼
┌──────────────────┐
│  Parse File       │
│  - Detect columns │
│  - Map headers    │
│  - Validate rows  │
│  - Error tracking │
└──────┬───────────┘
       ▼
┌──────────────────┐
│  Process Bills    │
│  - Upsert bills   │
│  - Upsert items   │
│  - Link products  │
│  - Track progress │
└──────┬───────────┘
       ▼
User uploads Product Master (.xlsx/.csv)
         │
         ▼
┌──────────────────┐
│  Parse Products   │
│  - Detect columns │
│  - Upsert brands  │
│  - Upsert depts   │
│  - Upsert cats    │
│  - Upsert vendors │
│  - Upsert products│
└──────┬───────────┘
       ▼
┌──────────────────┐
│  Join on Item Code│
│  - Link bill_items│
│    to products    │
│  - Flag orphans   │
│  - Report errors  │
└──────┬───────────┘
       ▼
┌──────────────────┐
│  Run Analysis     │
│  - Aggregate      │
│  - Compute metrics│
│  - Generate AI    │
│    summary        │
│  - Cache results  │
└──────┬───────────┘
       ▼
┌──────────────────┐
│  Notify User      │
│  - WebSocket      │
│  - Update UI      │
└──────────────────┘
```

## Parser Architecture (Go)

```go
// Adapter interface for different file sources
type FileParser interface {
    Parse(reader io.Reader) ([]RawRecord, []ParseError)
    Validate(record RawRecord) error
}

// POS Parser
type POSParser struct {
    columnMapping map[string]string  // flexible column mapping
}

// Product Parser
type ProductParser struct {
    columnMapping map[string]string
}

// Column mapping config (user-configurable)
type ColumnMapping struct {
    BillNo      string `json:"bill_no"`
    BillDate    string `json:"bill_date"`
    ItemCode    string `json:"item_code"`
    Quantity    string `json:"quantity"`
    Rate        string `json:"rate"`
    Amount      string `json:"amount"`
    Discount    string `json:"discount"`
    NetAmount   string `json:"net_amount"`
    PaymentMode string `json:"payment_mode"`
}
```

---

# 9. Component Hierarchy

```
<RootLayout>
  └─ <Providers>                           // QueryClient, ThemeProvider, AuthProvider
      ├─ <AuthLayout>                      // (auth)/layout.tsx
      │   ├─ <LoginPage>
      │   └─ <ForgotPasswordPage>
      │
      └─ <DashboardLayout>                 // (dashboard)/layout.tsx
          ├─ <Sidebar>
          │   ├─ <SidebarLogo>
          │   ├─ <SidebarItem>[]
          │   └─ <SidebarUserSection>
          │
          ├─ <Header>
          │   ├─ <Breadcrumb>
          │   ├─ <SearchInput>             // Global search
          │   ├─ <WeekSelector>            // Week picker
          │   └─ <UserNav>
          │
          ├─ <ExecutiveDashboard>
          │   ├─ <UploadBanner>            // If no data for current week
          │   ├─ <KPIGrid>
          │   │   └─ <KpiCard>[]
          │   ├─ <RevenueTrendChart>
          │   ├─ <TopBrandsTable>
          │   └─ <AISummaryCard>
          │
          ├─ <BrandRankingPage>
          │   ├─ <BrandRankingTable>
          │   └─ <BrandComparisonChart>
          │
          ├─ <BrandDetailPage>
          │   ├─ <BrandDetailHeader>
          │   ├─ <BrandMetricsGrid>
          │   │   └─ <KpiCard>[]
          │   ├─ <BrandTrendChart>
          │   ├─ <BrandTopProducts>
          │   └─ <BrandDistributions>
          │       ├─ <CategoryPieChart>
          │       ├─ <DepartmentPieChart>
          │       ├─ <ColorPieChart>
          │       ├─ <SizePieChart>
          │       └─ <PaymentPieChart>
          │
          ├─ <ProductAnalyticsPage>
          │   ├─ <FilterBar>
          │   └─ <ProductTable>
          │
          ├─ <DepartmentAnalyticsPage>
          │   ├─ <DepartmentComparisonTable>
          │   └─ <DepartmentChart>
          │
          ├─ <CategoryAnalyticsPage>
          │   ├─ <CategoryTable>
          │   └─ <CategoryBreakdown>
          │
          ├─ <VendorAnalyticsPage>
          │   ├─ <VendorRankingTable>
          │   └─ <VendorDetail>
          │
          ├─ <DiscountAnalyticsPage>
          │   ├─ <DiscountMetrics>
          │   ├─ <DiscountBrandTable>
          │   └─ <DiscountScatterChart>
          │
          ├─ <PaymentAnalyticsPage>
          │   └─ <PaymentDistribution>
          │
          ├─ <TimelinePage>
          │   ├─ <DailyTrendChart>
          │   └─ <WeekdayPerformance>
          │
          ├─ <AISummaryPage>
          │   ├─ <SummaryReport>
          │   └─ <RecommendationList>
          │
          ├─ <ReportsPage>
          │   ├─ <ReportGenerator>
          │   └─ <ReportHistory>
          │
          ├─ <UploadPage>
          │   ├─ <FileDropzone>
          │   ├─ <UploadProgress>
          │   └─ <UploadHistory>
          │
          └─ <SettingsPage>
```

---

# 10. Routing Design

| Path | Page | Layout | Auth |
|------|------|--------|------|
| `/login` | Login | Auth | No |
| `/` | Redirect → `/dashboard` | Dashboard | Yes |
| `/dashboard` | Executive Dashboard | Dashboard | Yes |
| `/brands` | Brand Ranking | Dashboard | Yes |
| `/brands/:brandId` | Brand Detail | Dashboard | Yes |
| `/products` | Product Analytics | Dashboard | Yes |
| `/departments` | Department Analytics | Dashboard | Yes |
| `/categories` | Category Analytics | Dashboard | Yes |
| `/vendors` | Vendor Analytics | Dashboard | Yes |
| `/discounts` | Discount Analytics | Dashboard | Yes |
| `/payments` | Payment Analytics | Dashboard | Yes |
| `/timeline` | Sales Timeline | Dashboard | Yes |
| `/ai-summary` | AI Business Summary | Dashboard | Yes |
| `/reports` | Reports | Dashboard | Yes |
| `/upload` | Data Upload | Dashboard | Yes |
| `/settings` | Settings | Dashboard | Yes |

---

# 11. State Management

## Zustand Stores

### `auth-store.ts`
```typescript
interface AuthState {
  user: User | null
  accessToken: string | null
  isAuthenticated: boolean
  login: (email: string, password: string) => Promise<void>
  logout: () => void
  refreshToken: () => Promise<void>
  setUser: (user: User) => void
}
```

### `ui-store.ts`
```typescript
interface UIState {
  sidebarCollapsed: boolean
  theme: 'light' | 'dark' | 'system'
  activeModal: string | null
  toggleSidebar: () => void
  setTheme: (theme: string) => void
}
```

### `filter-store.ts`
```typescript
interface FilterState {
  weekStartDate: string
  selectedBrandId: string | null
  selectedVendorId: string | null
  selectedDepartment: string | null
  selectedCategory: string | null
  selectedPaymentMode: string | null
  priceRange: [number, number] | null
  discountRange: [number, number] | null
  setWeekStartDate: (date: string) => void
  setSelectedBrand: (id: string | null) => void
  setSelectedVendor: (id: string | null) => void
  setSelectedDepartment: (dept: string | null) => void
  setSelectedCategory: (cat: string | null) => void
  setPriceRange: (range: [number, number] | null) => void
  setDiscountRange: (range: [number, number] | null) => void
  resetFilters: () => void
}
```

## TanStack Query Keys

```typescript
export const queryKeys = {
  dashboard: {
    executive: (week: string) => ['dashboard', 'executive', week],
    kpis: (week: string) => ['dashboard', 'kpis', week],
  },
  brands: {
    all: (week: string, page: number) => ['brands', week, page],
    detail: (id: string, week: string) => ['brands', id, week],
    trends: (id: string, weeks: number) => ['brands', id, 'trends', weeks],
    topProducts: (id: string, week: string) => ['brands', id, 'top-products', week],
    distributions: (id: string, week: string) => ['brands', id, 'distributions', week],
    top: (week: string) => ['brands', 'top', week],
    bottom: (week: string) => ['brands', 'bottom', week],
  },
  products: {
    all: (filters: ProductFilters) => ['products', filters],
    detail: (id: string, week: string) => ['products', id, week],
    top: (week: string) => ['products', 'top', week],
    worst: (week: string) => ['products', 'worst', week],
  },
  departments: {
    all: (week: string) => ['departments', week],
    detail: (id: string, week: string) => ['departments', id, week],
  },
  categories: {
    all: (week: string) => ['categories', week],
  },
  vendors: {
    all: (week: string) => ['vendors', week],
    detail: (id: string, week: string) => ['vendors', id, week],
  },
  discounts: {
    summary: (week: string) => ['discounts', 'summary', week],
    brands: (week: string) => ['discounts', 'brands', week],
    scatter: (week: string) => ['discounts', 'scatter', week],
  },
  payments: {
    distribution: (week: string) => ['payments', 'distribution', week],
  },
  timeline: {
    daily: (week: string) => ['timeline', 'daily', week],
    weekday: (week: string) => ['timeline', 'weekday', week],
  },
  summary: {
    get: (week: string) => ['summary', week],
  },
  reports: {
    all: () => ['reports'],
    download: (id: string) => ['reports', id],
  },
  uploads: {
    jobs: () => ['uploads', 'jobs'],
    job: (id: string) => ['uploads', 'jobs', id],
  },
}
```

## Cache Strategy
```
- Dashboard data: 5 min stale time (weekly data is static within week)
- Brand/Product lists: 5 min
- Trends: 10 min
- Upload jobs: 30 sec (polling for progress)
- AI Summary: 1 hr (regenerated weekly, cached)
```

---

# 12. AI Summary Architecture

## Template-Based NLG (LLM-Ready)

The summary is generated by a Go service using template injection with conditional logic.

```go
type AISummary struct {
    WeekStartDate    string
    TotalRevenue     float64
    RevenueChange    float64
    TotalBills       int
    TotalUnits       int
    AvgBillValue     float64
    AvgDiscount      float64
    TopBrand         BrandSummary
    FastestBrand     BrandSummary
    TopDepartment    DepartmentSummary
    TopCategory      CategorySummary
    TopProduct       ProductSummary
    TopVendor        VendorSummary
    DecliningBrands  []BrandSummary
    Recommendations  []Recommendation
}
```

### Template Variables
```
{{.TotalRevenue}}         — Formatted revenue
{{.RevenueChange}}        — WoW change %
{{.TopBrand.Name}}        — Best performing brand
{{.TopBrand.Revenue}}     — Brand revenue
{{.TopBrand.Contribution}} — Brand contribution %

Sentences are constructed by evaluating metrics:
- If revenue growth > 10%: "Revenue increased strongly"
- If revenue growth < -10%: "Revenue declined significantly"
- If top brand contribution > 25%: "X contributed Y% of total sales"
```

### LLM-Ready Architecture
```
Current: Template-based NLG in Go (no external dependencies)
Future:   POST to OpenAI/Gemini with metrics JSON → return natural text

The interface is designed so the AI provider can be swapped:
  type AIGenerator interface {
      GenerateSummary(metrics DashboardMetrics) (string, error)
  }
  Implementation: TemplateGenerator | OpenAIGenerator | GeminiGenerator
```

---

# 13. PDF Report Architecture

```
Report Request
      │
      ▼
┌──────────────────┐
│  Collect Data     │
│  - Dashboard      │
│  - Brands         │
│  - Departments    │
│  - Categories     │
│  - Vendors        │
│  - Top Products   │
│  - AI Summary     │
└──────┬───────────┘
       ▼
┌──────────────────┐
│  Render HTML      │
│  - template.Parse │
│  - Inject data    │
│  - Include CSS    │
│  - Generate       │
│    inline charts  │
└──────┬───────────┘
       ▼
┌──────────────────┐
│  Convert to PDF   │
│  (Chromedp)       │
│  - Headless       │
│  - A4 portrait    │
│  - Print CSS      │
└──────┬───────────┘
       ▼
┌──────────────────┐
│  Store + Return   │
│  - Save to disk   │
│  - Record in DB   │
│  - Return URL     │
└──────────────────┘
```

---

# 14. Design System

## Color Palette (Dark Mode Default)

```css
@theme {
  --color-background: 222.2 84% 4.9%;
  --color-foreground: 210 40% 98%;
  --color-card: 222.2 84% 4.9% / 0.6;
  --color-card-foreground: 210 40% 98%;
  --color-primary: 221.2 83.2% 53.3%;
  --color-primary-foreground: 210 40% 98%;
  --color-secondary: 217.2 32.6% 17.5%;
  --color-muted: 217.2 32.6% 17.5%;
  --color-accent: 217.2 32.6% 17.5%;
  --color-destructive: 0 62.8% 30.6%;
  --color-border: 217.2 32.6% 17.5%;
  --color-input: 217.2 32.6% 17.5%;
  --color-ring: 224.3 76.3% 48%;
  --color-success: 142.1 76.2% 36.3%;
  --color-warning: 38 92% 50%;
  --color-critical: 0 72% 51%;
}
```

## Glassmorphism
```css
.glass-card {
  background: hsl(var(--color-card));
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border: 1px solid hsl(var(--color-border) / 0.3);
  border-radius: 12px;
  box-shadow: 0 4px 24px hsl(0 0% 0% / 0.1);
}
```

## Typography
```
Font: Inter (body), JetBrains Mono (data)
h1: text-3xl font-bold tracking-tight
h2: text-2xl font-semibold
h3: text-xl font-semibold
body: text-sm
data: font-mono text-base
small: text-xs
```

## Animation (Framer Motion)
```
Page transitions: opacity 0.3s + y offset 8px
KPI cards: staggerChildren 0.05s, y 20px → 0
Glass cards: hover lift (y -2px, shadow increase)
Charts: ECharts built-in animation (800ms)
Sidebar: width transition 0.3s ease
Notifications: slide-in from right
```

---

*End of Architecture Document v2.0*
