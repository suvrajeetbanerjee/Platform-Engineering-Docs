# Platform-Engineering-Docs

#### Platform Engineering using K8s for building Internal Developer Platforms

*Architecture* 

![architecture](https://raw.githubusercontent.com/suvrajeetbanerjee/Platform-Engineering-Docs/refs/heads/main/architecture/architecture.svg)


***

# Host360 Cloud Platform — Codebase 

> **Platform:** Host360 Cloud Control Panel  
> **Stack:** Go (Billing Engine) + NestJS (Admin Backend) + Next.js (Admin & Customer Frontends)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Top-Level Directory Structure](#2-top-level-directory-structure)
3. [Billing Engine — `billing-cron-go`](#3-billing-engine--billing-cron-go)
4. [Admin Portal Backend — `admin-cc-portal-backend`](#4-admin-portal-backend--admin-cc-portal-backend)
5. [Admin Portal Frontend — `admin-cc-portal-frontend`](#5-admin-portal-frontend--admin-cc-portal-frontend)
6. [Cloud Console Frontend — `cloud-console-frontend`](#6-cloud-console-frontend--cloud-console-frontend)
7. [Billing Lifecycle Service — `billing-lifecycle-service`](#7-billing-lifecycle-service--billing-lifecycle-service)
8. [Testing Access — `testing-access`](#8-testing-access--testing-access)
11. [Architecture & Data Flow](#11-architecture--data-flow)
12. [Technology Stack Summary](#12-technology-stack-summary)
13. [Deployment & Infrastructure](#13-deployment--infrastructure)
14. [Key Architectural Patterns](#14-key-architectural-patterns)

---

## 1. Project Overview

The **Host360 Cloud Platform** is an enterprise-grade cloud management system designed to orchestrate OpenStack-based infrastructure. It comprises four primary subsystems:

| Subsystem | Language/Framework | Purpose |
|---|---|---|
| **Billing Engine** | Go 1.21+ (Golang) | High-performance billing cron, resource tracking, invoice generation, suspension enforcement |
| **Admin Portal Backend** | NestJS 11 (TypeScript) | REST API for admin operations: accounts, IAM, billing, payments, KYC, OpenStack orchestration |
| **Admin Portal Frontend** | Next.js 15 + React 19 | Admin dashboard for managing customers, billing, infrastructure, and system configuration |
| **Cloud Console Frontend** | Next.js 15 + React 19 | Customer-facing cloud console for managing instances, volumes, networking, S3, backups |

The platform integrates with:
- **OpenStack** (Nova, Cinder, Neutron, Glance, Octavia, Barbican, Keystone, RadosGW)
- **Payment Gateways** (Razorpay, Stripe)
- **Communication** (Twilio SMS, Zepto Mail)
- **KYC** (Digio)
- **Monitoring** (Grafana)
- **Storage** (AWS S3 / S3-compatible)
- **Databases** (MySQL/MariaDB, MongoDB, Redis/Valkey)

---

## 2. Top-Level Directory Structure

```
CB-AP/
├── .opencode/                      # OpenCode AI configuration
├── admin-cc-portal-backend/        # NestJS admin API (~1200+ files)
├── admin-cc-portal-frontend/       # Next.js admin dashboard (~350+ files)
├── AGENTS.md                       # OpenCode agent instructions + graphify rules
├── billing-cron-go/                # Go billing engine (~50+ files)
├── billing-lifecycle-service/      # Placeholder (empty scaffold)
├── cloud-console-frontend/         # Next.js customer console (~2000+ files)
├── graphify-out/                   # Knowledge graph output
└── testing-access/                 # GitLab test/sandbox repo
```

---

## 3. Billing Engine — `billing-cron-go`

### 3.1 Overview

A high-performance rewrite of a legacy Python billing system. Achieves **10x speed improvement** and **90% memory reduction** over its predecessor. Handles multi-region OpenStack resource tracking, usage computation, invoice generation, and account suspension.

### 3.2 Directory Structure

```
billing-cron-go/
├── main.go                        # Application entry point
├── go.mod / go.sum                # Go module dependencies
├── Makefile                       # Build, lint, test automation
├── Dockerfile                     # Multi-stage build (~20MB binary)
├── docker-compose.yml             # Local dev orchestration
├── .env.example                   # Environment variable template
├── config/
│   └── config.go                  # Centralized config loader (env vars)
├── pkg/
│   ├── cronjobs/
│   │   ├── cron.go                # Cron scheduler (robfig/cron/v3)
│   │   ├── monthly_cron.go        # Monthly billing cycle
│   │   ├── hourly_cron.go         # Hourly usage sync
│   │   ├── daily_cron.go          # Daily reconciliation
│   │   └── suspension_cron.go     # Account suspension enforcement
│   ├── database/
│   │   ├── mongodb.go             # MongoDB client (billing records)
│   │   ├── mysql.go               # MySQL connection manager
│   │   └── redis.go               # Redis caching layer
│   ├── models/
│   │   ├── instance.go            # Instance billing model
│   │   ├── volume.go              # Volume billing model
│   │   ├── floating_ip.go         # Floating IP billing model
│   │   ├── snapshot.go            # Snapshot billing model
│   │   ├── bandwidth.go           # Bandwidth usage model
│   │   ├── invoice.go             # Invoice data model
│   │   ├── account.go             # Account/pricing plan model
│   │   └── usage.go               # Usage record model
│   ├── services/
│   │   ├── instance_service.go    # Instance usage computation
│   │   ├── volume_service.go      # Volume usage computation
│   │   ├── network_service.go     # Network resource computation
│   │   ├── pricing_service.go     # Pricing calculation engine
│   │   ├── invoice_service.go     # Invoice generation logic
│   │   ├── suspension_service.go  # Account suspension logic
│   │   └── audit_service.go       # Audit logging
│   ├── openstack/
│   │   ├── client.go              # OpenStack API client
│   │   ├── nova.go                # Nova (compute) queries
│   │   ├── cinder.go              # Cinder (block storage) queries
│   │   └── neutron.go             # Neutron (networking) queries
│   └── utils/
│       ├── logger.go              # Structured JSON logging (logrus)
│       ├── rotator.go             # Log rotation (lumberjack)
│       ├── retry.go               # Retry logic with backoff
│       └── pagination.go          # API pagination helper
└── scripts/
    ├── seed_pricing.go            # Pricing data seeder
    └── migrate_legacy.go          # Legacy data migration
```

### 3.3 Architecture Details

#### Entry Point (`main.go`)
- Reads configuration from environment variables via `config/config.go`
- Initializes connections: MongoDB (billing store), MySQL (OpenStack + Portal), Redis (cache)
- Registers all cron jobs with `robfig/cron/v3` scheduler
- Starts HTTP health check server on configurable port

#### Configuration (`config/config.go`)
- All configuration via environment variables
- Structured `Config` struct loaded at startup via `os.Environ()` + custom parser
- Supports per-region configuration arrays

#### Cron Jobs (`pkg/cronjobs/`)
| Job | Schedule | Description |
|---|---|---|
| **Hourly Cron** | `@every 1h` | Syncs running resource states from OpenStack to MongoDB; computes hourly usage snapshots |
| **Daily Cron** | `@daily` | Daily reconciliation of usage records against OpenStack; generates daily audit summaries |
| **Monthly Cron** | `0 0 1 * *` | End-of-month billing cycle: finalizes usage, generates invoices, applies credits/discounts |
| **Suspension Cron** | `*/30 * * * *` | Checks overdue accounts every 30 min; triggers suspension workflow for accounts past due |

#### Database Layer (`pkg/database/`)
- **MongoDB**: Primary billing store (usage records, monthly summaries, audit logs). Collections: `instances-{region}`, `volumes-{region}`, `usage-records`, `monthly-bills`, `audit-logs`
- **MySQL**: Two separate data sources:
  - *OpenStack databases* (Nova `instances` table, Cinder `volumes` table, Neutron `floatingips` table) — read-only queries for live resource state
  - *Portal database* — account details, pricing plans, billing config
- **Redis**: Caching layer for pricing plans (reduces DB load on hot paths)

#### Service Layer (`pkg/services/`)
- **Instance Service**: Queries Nova for running VMs, calculates runtime duration, applies pricing tier
- **Volume Service**: Queries Cinder for volume attachments, computes storage-hours
- **Network Service**: Queries Neutron for floating IPs, computes allocation duration
- **Pricing Service**: Loads pricing plans from Redis cache or portal DB; computes cost per resource
- **Invoice Service**: Aggregates monthly usage records, generates invoice line items, posts to portal DB
- **Suspension Service**: Marks accounts as suspended in portal DB; can trigger OpenStack API calls to stop resources
- **Audit Service**: Writes structured audit records to MongoDB for compliance

#### OpenStack Integration (`pkg/openstack/`)
- Custom client built on `gophercloud` library
- Service discovery via Keystone catalog
- Region-scoped authentication with per-region credentials
- Pagination-aware list queries for large resource sets

### 3.4 Key Dependencies (from `go.mod`)
- `github.com/robfig/cron/v3` — Cron scheduling
- `github.com/sirupsen/logrus` — Structured logging
- `github.com/natefinch/lumberjack` — Log rotation
- `github.com/gophercloud/gophercloud` — OpenStack SDK
- `go.mongodb.org/mongo-driver` — MongoDB driver
- `github.com/go-sql-driver/mysql` — MySQL driver
- `github.com/redis/go-redis/v9` — Redis client
- `github.com/spf13/viper` — Configuration management

### 3.5 Performance Characteristics
- **Memory**: ~15-25MB resident set size (vs ~200MB+ Python predecessor)
- **Throughput**: Processes 10k+ instances per minute per region
- **Concurrency**: Goroutine-per-region parallel processing
- **Startup**: Cold start in <500ms

---

## 4. Admin Portal Backend — `admin-cc-portal-backend`

### 4.1 Overview

A modular **NestJS 11** enterprise REST API powering the Host360 admin portal. Manages accounts, billing, IAM, payments, KYC, OpenStack orchestration, and system configuration. ~1,200+ TypeScript files across 40+ feature modules.

### 4.2 Directory Structure (High-Level)

```
admin-cc-portal-backend/
├── src/
│   ├── main.ts                        # Application bootstrap
│   ├── app.module.ts                  # Root module (imports all features)
│   ├── core/                          # Cross-cutting infrastructure
│   │   ├── database/                  # TypeORM + Mongoose + Redis configs
│   │   ├── guards/                    # Auth, permissions, suspension guards
│   │   ├── interceptors/              # Response wrapper, logging, audit
│   │   ├── filters/                   # Global exception filter
│   │   ├── decorators/                # @Public, @CurrentUser, @Permissions
│   │   ├── pipes/                     # Validation pipes
│   │   └── enums/                     # Permissions, statuses, constants
│   ├── modules/                       # 40+ feature modules
│   │   ├── auth/                      # JWT, 2FA, local strategies, sessions
│   │   ├── accounts/                  # Account CRUD, closure, ownership
│   │   ├── admin/                     # Admin users, roles, permissions
│   │   ├── iam/                       # IAM roles, policies, API keys
│   │   ├── wallet/                    # Wallet transactions, balance
│   │   ├── subscriptions/             # Subscription lifecycle, renewal
│   │   ├── invoices/                  # Invoice generation, PDF, GST
│   │   ├── payment/                   # Razorpay integration
│   │   ├── openstack/                 # OpenStack orchestration service
│   │   ├── instances/                 # Cloud instance management
│   │   ├── volumes/                   # Block storage management
│   │   ├── vpcs/                      # VPC/subnet/port management
│   │   ├── routers/                   # Router management
│   │   ├── floatingips/               # Floating IP management
│   │   ├── security-group/            # Security group management
│   │   ├── loadbalancers/             # Octavia LB management
│   │   ├── bucket/                    # S3 bucket/object management
│   │   ├── key-pair/                  # SSH key pair management
│   │   ├── custom-images/             # Custom OS image management
│   │   ├── ssl-certificates/          # SSL via Barbican
│   │   ├── snapshot-center/           # Scheduled/manual snapshots
│   │   ├── backups/                   # Backup management
│   │   ├── pdf-generator/             # Handlebars PDF + S3 upload
│   │   ├── digio-kyc/                 # KYC verification (Digio)
│   │   ├── notifications/             # Email notifications
│   │   ├── billing-dashboard/         # Billing dashboard data
│   │   ├── metrics/                   # Grafana metrics proxy
│   │   ├── marketplace/               # Marketplace management
│   │   ├── category/                  # Compute categories + GPU details
│   │   ├── regions/                   # Region config, credentials
│   │   ├── contacts/                  # Contact CRUD, project invites
│   │   ├── otp/                       # OTP generation/verification
│   │   └── profile-settings/          # Personal settings, password
│   └── integrations/                  # 3rd-party integration modules
│       ├── email/                     # Zepto Mail
│       ├── sms/                       # Twilio SMS
│       ├── razorpay/                  # Payment gateway
│       ├── exchange-rate/             # Currency conversion
│       └── recaptcha/                 # Google reCAPTCHA
├── docs/                              # Design documents (2FA, RBAC, IAM, etc.)
├── Dockerfile                         # Multi-stage build (node:20-alpine)
└── package.json                       # NestJS 11, TypeORM, Mongoose
```

### 4.3 Core Infrastructure

#### Database Configuration (`src/core/database/`)
- **TypeORM** — Dual connection setup:
  - `default` (write connection) — Portal database
  - `READ_CONNECTION` (read replica / secondary) — Read-only queries
- **Mongoose** — Dual connection:
  - Default — Main MongoDB (activity logs, audit logs)
  - `BILLING_CONNECTION` — MongoDB billing store
- **Redis** — `CacheModule` via `cache-manager` for session caching, account status caching, rate limiting

#### Authentication & Authorization
- **JWT** — Passport.js JWT strategy with access/refresh token pairs
- **2FA** — Time-based OTP via `otplib`, backup codes, recovery flow
- **Multi-strategy auth** — Local (email/password), IAM token, admin SSO (`loginAsOwner`)
- **Permission system** — Two parallel hierarchies:
  - `AdminPermissions` / `AdminRole` — Staff/admin access control
  - `IamPermissions` — End-user cloud IAM (projects, roles, policies)
- **Guards**:
  - `AccountSuspensionGuard` — Caches account status (60s Redis TTL); allows limited POST paths (wallet, payment, invoice, 2FA) for suspended/MFD accounts; blocks all other writes
  - `PermissionsGuard` / `IamGuard` — Route-level permission enforcement
  - `ThrottlerGuard` — Rate limiting

#### Global Middleware Pipeline
1. **LoggingInterceptor** — Logs method, URL, duration for every request
2. **ResponseInterceptor** — Wraps all responses in `{ success, data, message, meta }`
3. **HttpExceptionFilter** — Catches all exceptions, normalizes error format with `reason`, `code`, `details`, `errorCode`
4. **AuditInterceptor** — Tracks entity changes to MongoDB `audit_logs` collection

### 4.4 Key Feature Modules

#### Auth Module (`auth/`)
- Login/Logout with session management
- 2FA enrollment + verification (QR code via `otplib`)
- Password reset flow with OTP
- Token refresh with HTTP-only cookies
- Session tracking in MongoDB

#### Accounts Module (`accounts/`)
- Full account lifecycle: create, suspend, close, mark-for-deletion
- Account closure flow with ownership verification
- Project-based multi-tenancy
- Contact management + project invites
- Quota management per project

#### IAM Module (`iam/`)
- IAM role definitions with granular permissions
- Policy documents (JSON-based permission statements)
- API key generation for programmatic access
- Project-level role assignments
- Group-based permission management

#### Invoices Module (`invoices/`)
- Monthly invoice generation (triggered by cron or on-demand)
- Pro-rated billing for mid-cycle changes
- GST calculation (Indian tax compliance)
- PDF generation via Handlebars templates + Puppeteer
- Invoice PDFs uploaded to S3 with signed URLs
- TDS refund claim management
- Statement view with HTML rendering

#### Payment Module (`payment/`)
- Razorpay order creation and verification
- Payment event handling via webhooks
- Exchange rate support (USD → INR via external API)
- Payment history and reconciliation
- Refund processing

#### Wallet Module (`wallet/`)
- Credit/debit transactions
- Balance monitoring and alerts
- Email notification on low balance
- Wallet-based deduction for postpaid accounts

#### OpenStack Service (`openstack/`)
- Centralized OpenStack API client with:
  - Keystone authentication + service catalog caching
  - Per-region credential management
  - Dynamic microversion headers per service URL
  - Service-specific submodules for: Nova, Cinder, Neutron, Glance, Octavia, Barbican, Keystone, RadosGW

### 4.5 API Design

- **Swagger** docs at `/api/docs` (auto-generated via `@nestjs/swagger`)
- **Global prefix** configured in `main.ts`
- **Class-Validator** DTOs with whitelisting/transformation via `ValidationPipe`
- **Serialization** via `ClassSerializerInterceptor` (entity → DTO transformation)
- **Versioning**: URI-based versioning strategy

### 4.6 Key Dependencies (from `package.json`)
- `@nestjs/core` 11.x, `@nestjs/platform-express` 11.x
- `@nestjs/typeorm`, `typeorm`, `mysql2`
- `@nestjs/mongoose`, `mongoose`
- `@nestjs/passport`, `@nestjs/jwt`, `passport`, `passport-jwt`
- `otplib` — 2FA TOTP
- `@nestjs/throttler` — Rate limiting
- `@nestjs/bull`, `bull` — Background job queue (Redis-backed)
- `razorpay` — Payment gateway SDK
- `twilio` — SMS integration
- `@aws-sdk/client-s3` — S3 object storage
- `puppeteer`, `handlebars` — PDF generation
- `ejs` — Email templating

---

## 5. Admin Portal Frontend — `admin-cc-portal-frontend`

### 5.1 Overview

A feature-rich **Next.js 15** admin dashboard for the Host360 platform. Built with React 19, Tailwind CSS v4, and shadcn/ui components. ~350+ TypeScript/TSX files.

### 5.2 Directory Structure

```
admin-cc-portal-frontend/
├── src/
│   ├── app/                              # Next.js App Router
│   │   ├── layout.tsx                    # Root layout (providers, fonts)
│   │   ├── page.tsx                      # Auth redirect (/sign-in or /dashboard)
│   │   ├── sign-in/                      # Multi-step login (email → password → 2FA)
│   │   ├── signup/                       # Admin signup with invite
│   │   ├── forgot-password/              # 3-step password reset
│   │   ├── 2fa/                          # 2FA setup
│   │   ├── admin/accept-invite/          # Invite validation
│   │   └── dashboard/                    # Protected dashboard (layout + pages)
│   │       ├── layout.tsx                # Sidebar + navbar + auth guard
│   │       ├── page.tsx                  # Dashboard overview
│   │       ├── accounts/                 # Customer account management
│   │       ├── billing/                  # Invoice management + statements + TDS
│   │       ├── plan-management/          # Plans, categories, marketplace, volumes
│   │       ├── activity-logs/            # Audit log viewer
│   │       ├── profile/                  # Admin profile
│   │       ├── system-configuration/     # Tabbed: Users, Regions, Integrations, Defaults, GST
│   │       ├── marketplace/              # (redirect to plan-management)
│   │       ├── reports/                  # Reporting placeholder
│   │       └── ...                       # Distributor, Partner accounts
│   ├── api/                              # 18 API service modules
│   │   ├── accountService.api.ts         # Account CRUD + lifecycle
│   │   ├── accountPlansService.api.ts    # Pricing plan management
│   │   ├── invoiceApi.ts                 # Invoice CRUD + PDF download
│   │   ├── regionApi.ts                  # Region configuration
│   │   ├── usermanagement.api.ts         # Admin users + roles
│   │   ├── marketplaceApi.ts             # Marketplace images
│   │   ├── categoryApi.ts                # Compute categories
│   │   └── ...                           # 10 more API modules
│   ├── components/
│   │   ├── ui/                           # 30+ reusable components (shadcn/ui + custom)
│   │   │   ├── Sidebar/ + SidebarWrapper/ # Navigation sidebar
│   │   │   ├── Table/                    # Data table with sorting/filtering
│   │   │   ├── Modal/                    # Reusable modal dialog
│   │   │   ├── Pagination/               # Paginated data display
│   │   │   ├── PermissionBoundary/       # Conditional render by permission
│   │   │   └── ...                       # Button, Input, Select, Toast, etc.
│   │   ├── (Management components)/       # 30+ page-level components
│   │   ├── PlanManagement/               # Tabbed plan admin UI
│   │   ├── DefaultSettings/              # System defaults forms
│   │   └── pages/Login/                  # Login form + 2FA
│   ├── contexts/
│   │   └── AuthContext.tsx               # Auth state (user, tokens, permissions)
│   ├── hooks/
│   │   └── usePermissions.ts             # Permission checking hook
│   └── lib/
│       ├── axios.ts                      # Axios instance with interceptors
│       ├── utils.ts                      # cn(), formatCurrency(), validation
│       ├── permissions.ts                # Module-permission mapping
│       ├── services/
│       │   ├── authService.ts            # Auth API calls + token management
│       │   └── activityTracker.ts        # Idle detection + token refresh
│       └── constants/
├── Dockerfile                            # multi-stage build (node:20-alpine, standalone)
├── docker-compose.yml
├── next.config.ts                        # output: standalone
├── tailwind.config.ts
└── package.json                          # Next.js 15.5, React 19.1, MUI 7, Radix UI
```

### 5.3 Routes & Pages

| Route | Type | Description |
|---|---|---|
| `/` | Page | Auth-aware redirect |
| `/sign-in` | Page | 3-step login: email → password → 2FA |
| `/signup` | Page | Signup with invite token |
| `/forgot-password` | Page | 3-step password reset with OTP |
| `/2fa` | Page | 2FA setup (QR code + backup codes) |
| `/admin/accept-invite` | Page | Invite token validation |
| `/dashboard` | Layout | Protected shell (sidebar + navbar) |
| `/dashboard/accounts` | Page | Customer accounts table + CRUD |
| `/dashboard/billing` | Page | Invoice list + management |
| `/dashboard/billing/accounts/[id]` | Page | Per-account invoices |
| `/dashboard/billing/statement/[id]` | Page | Invoice statement (HTML iframe) |
| `/dashboard/billing/tds-refund-claims` | Page | TDS refund management |
| `/dashboard/plan-management` | Page | Tabbed: Plans, Categories, Marketplace, Volumes |
| `/dashboard/activity-logs` | Page | Audit log viewer with filters |
| `/dashboard/profile` | Page | Admin profile editing |
| `/dashboard/system-configuration` | Layout | Tabbed system config |
| → `/user-management` | Page | Admin users, roles, permissions |
| → `/region-management` | Page | Region CRUD + endpoint config |
| → `/integrations` | Page | S3 + Grafana config |
| → `/default-settings` | Page | Global defaults |
| → `/gst-approval` | Page | GST document approval |

### 5.4 Authentication Flow

1. User enters email → POST `/auth/login` → partial token returned
2. If 2FA enabled: user enters OTP → POST `/auth/2fa/verify` → full access token
3. If no 2FA: direct full token on password submission
4. Token stored in `localStorage` + HTTP-only cookie for refresh
5. `ActivityTracker` monitors idle time (15 min warning → 20 min auto-logout)
6. Proactive token refresh before expiry via activity tracker intervals

### 5.5 State Management

- **AuthContext** — User session, permissions, authentication status
- **TanStack React Query** — Server state caching (imported in dependencies)
- **Component-local state** — `useState` for form state, modals, UI controls
- **Activity Tracker** — Singleton module with module-level state for idle/timer tracking

### 5.6 UI Component Library

Built on **shadcn/ui** (New York style) with Radix UI primitives:
- Sidebar with collapsible sections + permission-based filtering
- Data tables with sorting, column selection, pagination
- Modal dialogs for CRUD operations
- Toast notifications for success/error feedback
- Permission boundary wrapper for conditional rendering
- Custom tooltips, multi-select, accordion, tabs

### 5.7 Key Dependencies
- `next` 15.5, `react` 19.1, `react-dom` 19.1
- `@tanstack/react-query` 5.x — Server state
- `tailwindcss` 4.x, `tw-animate-css` — Styling
- `@radix-ui/*` — Headless UI primitives
- `@mui/material` 7.x — MUI components
- `lucide-react` — Icons
- `recharts`, `chart.js` — Charts
- `axios` — HTTP client
- `react-hot-toast` — Notifications
- `@stripe/react-stripe-js`, `@stripe/stripe-js` — Stripe integration

---

## 6. Cloud Console Frontend — `cloud-console-frontend`

### 6.1 Overview

The **customer-facing cloud console** — a comprehensive Next.js 15 dashboard where end users manage their cloud infrastructure. ~2,000+ files making it the largest component in the monorepo.

### 6.2 Directory Structure

```
cloud-console-frontend/
├── src/
│   ├── app/                              # Next.js App Router
│   │   ├── layout.tsx + providers.tsx     # Root layout + context providers
│   │   ├── page.tsx                      # Landing / redirect
│   │   ├── (auth)/                       # Login, signup, 2FA, forgot password
│   │   │   ├── login/
│   │   │   ├── signup/
│   │   │   ├── 2fa-verify/
│   │   │   └── forgot-password/
│   │   ├── (dashboard)/                  # Customer dashboard
│   │   │   ├── layout.tsx                # Dashboard shell (sidebar + topbar)
│   │   │   ├── overview/                 # Resource overview dashboard
│   │   │   ├── instances/                # VM instance management
│   │   │   ├── volumes/                  # Block storage management
│   │   │   ├── vpc/                      # VPC management
│   │   │   ├── subnet/                   # Subnet management
│   │   │   ├── virtual-routers/          # Router management
│   │   │   ├── securitygroup/            # Security groups
│   │   │   ├── floatingip/               # Floating IPs (elastic IP)
│   │   │   ├── loadbalancer/             # Load balancers
│   │   │   ├── s3-storage/               # S3 object storage
│   │   │   ├── backup-center/            # Backup management
│   │   │   ├── snapshot-center/          # Snapshot management
│   │   │   ├── custom-images/            # Custom OS images
│   │   │   ├── key-pair/                 # SSH key pairs
│   │   │   ├── billing/                  # Billing & invoices
│   │   │   ├── iam/                      # IAM & API keys
│   │   │   ├── audit-logs/               # Activity logs
│   │   │   ├── settings/                 # Account settings
│   │   │   └── kyc/                      # KYC verification
│   ├── api/                              # 62 API endpoint modules
│   ├── components/
│   │   ├── ui/                           # 54 UI component directories
│   │   ├── pages/                        # 41 page components (per route)
│   │   ├── Modals/                       # 141 modal components
│   │   └── Providers/                    # App providers (query, auth, inactivity)
│   ├── contexts/                         # 7 React contexts
│   │   ├── AuthContext.tsx               # Authentication state
│   │   ├── KycContext.tsx                # KYC status
│   │   ├── NetworkingActionsContext.tsx  # Networking action tracking
│   │   ├── ParentProjectContext.tsx      # Parent project context
│   │   ├── ProjectRegionContext.tsx      # Active project/region
│   │   └── UserProfileContext.tsx        # User profile data
│   ├── hooks/                            # 19 custom hooks
│   ├── lib/                              # Utilities (axios, helpers, S3 SDK)
│   ├── schema/                           # Zod validation schemas
│   ├── configs/                          # API config, app config, constants
│   ├── assets/                           # SVGs, images, icons
│   └── middleware.ts                     # S3 credential injection middleware
├── Dockerfile                            # Multi-stage Next.js standalone build
├── docker-compose.yml
├── next.config.ts
├── tailwind.config.ts
└── package.json
```

### 6.3 Feature Modules

#### Compute
- **Instances**: Create, start, stop, reboot, resize, snapshot; flavor selection; network attachment; console access
- **Key Pairs**: Import/generate SSH keys; associate with instances

#### Storage
- **Volumes**: Create, attach, detach, delete, resize, snapshot
- **Snapshots**: Manual and scheduled; restores from snapshot
- **Backups**: Backup policies, backup vaults, restore workflows
- **Custom Images**: Upload ISO, create from snapshot, manage image visibility

#### Networking
- **VPC**: Create/delete VPCs with CIDR configuration
- **Subnets**: Subnet creation within VPCs
- **Virtual Routers**: Router management, interface attachment, static routes
- **Floating IPs**: Allocate/associate/disassociate elastic IPs
- **Security Groups**: Rule-based traffic filtering (ingress/egress)
- **Load Balancers**: Octavia-based LB with listeners, pools, L7 policies/rules
- **Inbound/Outbound**: Traffic rules and NAT configurations

#### Object Storage (S3)
- Bucket CRUD with policy management
- Object upload/download with presigned URLs
- S3 user/subuser management via RGW admin API
- Lifecycle policies and versioning

#### Infrastructure
- **Monitoring**: Grafana dashboard integration
- **Health Monitoring**: Resource health checks and alerts
- **Observability**: Metrics and logging overview

#### Business Operations
- **Billing**: Invoice view, payment management, wallet transactions
- **KYC**: Document upload, Digio-based verification
- **IAM**: Role management, API keys, group membership
- **Settings**: Profile, password, 2FA, notifications
- **Order Project**: Project ordering workflow

### 6.4 Authentication & Middleware

- Similar multi-step auth flow as admin portal (JWT + 2FA)
- **S3 Credential Injection Middleware** (`middleware.ts`): Intercepts S3 API requests at edge, injects RGW credentials from request headers
- **AuthContext**: Manages user session, project context, region selection
- **InactivityProvider**: Idle session timeout with countdown

### 6.5 UI Components

- **54 reusable UI components**: Accordion, Button, Card, Checkbox, DataTable, Dialog, Dropdown, Form, Input, Modal, NavigationMenu, Popover, Select, Sheet, Skeleton, Table, Tabs, Toast, Toggle, Tooltip, etc.
- **141 modal components**: One per action (CreateInstanceModal, AttachVolumeModal, EditSecurityGroupModal, etc.)
- **41 page components**: Full-page layouts for each route
- Styling: Tailwind CSS v4 + CSS variables + custom theme tokens
- Icons: `lucide-react` SVGs

### 6.6 State Management

- **7 React Contexts** for global state (auth, KYC, networking, project, region, profile)
- **TanStack React Query** for server state caching
- **Zod schemas** for client-side validation
- **Custom hooks** (19 total) for domain-specific logic

### 6.7 API Layer (62 modules)

Organized by domain:
- Auth, accounts, billing, instances, volumes, VPC, subnets, routers, security groups, floating IPs, load balancers, S3, IAM, KYC, snapshots, backups, custom images, key pairs, notifications, settings, marketplace, regions, metadata, monitoring, alerts, orders

---

## 7. Billing Lifecycle Service — `billing-lifecycle-service`

### 7.1 Status

**Empty scaffold / placeholder project.** Contains only:
- `.gitignore` (Go template from GitHub)
- `README.md` (single-line title: `# billing-lifecycle-service`)
- Git repository with one initial commit (`dea5830` — "Initial commit" by Soniya, Apr 17 2026)

### 7.2 Notes

- No source code, no configuration, no dependencies
- The `.gitignore` suggests an intended Go-based implementation
- Remote origin exists (internal GitLab), but no code has been pushed beyond the scaffold

---

## 8. Testing Access — `testing-access`

### 8.1 Overview

A **GitLab sandbox repository** cloned from `https://product-gitlab.host360.ai/core-platform/applications/testing-access.git`. Used for testing Git workflows and access permissions.

### 8.2 Files

| File | Content |
|---|---|
| `README.md` | GitLab template README (93 lines of boilerplate instructions) |
| `test.md` | `Hi, This is test md file` |
| `test.ts` | `testing push request is working fine with github desktop` |
| `ptsxt.md` | `yes working?` |

### 8.3 Git History

- **Clone**: by `suvrajeet.banerjee` from remote
- **Remote branches tracked**: `main`, `access-allow`, `anoop-testing`, `satyam`
- Working tree clean, no local commits


---

## 11. Architecture & Data Flow

### 11.1 System Architecture Diagram (Text)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CUSTOMER (Browser)                          │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐    │
│  │ Cloud Console        │  │ Admin Portal                    │    │
│  │ (cloud-console-      │  │ (admin-cc-portal-frontend)      │    │
│  │  frontend)           │  │  Next.js 15 + React 19          │    │
│  │  Next.js 15 + React 19│  │  Port 3000 (dev)               │    │
│  │  Port 3000 (dev)     │  │                                  │    │
│  └──────────┬───────────┘  └──────────────┬───────────────────┘    │
└─────────────┼──────────────────────────────┼────────────────────────┘
              │                              │
              │  HTTPS (REST + JWT)          │  HTTPS (REST + JWT + 2FA)
              ▼                              ▼
┌─────────────────────────────┐  ┌──────────────────────────────────┐
│   Admin Portal Backend      │  │  Billing Engine                  │
│   (admin-cc-portal-backend) │  │  (billing-cron-go)               │
│   NestJS 11 + TypeScript    │  │  Go 1.21+                        │
│   Port 3000                 │  │  Port 8080 (health)              │
│                             │  │                                  │
│   ┌─────────────────────┐   │  │  ┌──────────────────────────┐    │
│   │ Auth (JWT + 2FA)   │   │  │  │ Cron Scheduler           │    │
│   │ IAM (RBAC)         │   │  │  │ robfig/cron/v3           │    │
│   │ Permissions Guard  │   │  │  │  ┌─────────────────┐     │    │
│   └─────────────────────┘   │  │  │  │ Hourly Cron     │     │    │
│                             │  │  │  │ Daily Cron      │     │    │
│   ┌─────────────────────┐   │  │  │  │ Monthly Cron    │     │    │
│   │ OpenStack Service   │───┼──┼──┼──┤ Suspension Cron │     │    │
│   │ (gophercloud)       │   │  │  │  └─────────────────┘     │    │
│   └─────────────────────┘   │  │  └──────────────────────────┘    │
│                             │  │                                  │
│   ┌─────────────────────┐   │  │  ┌──────────────────────────┐    │
│   │ Payment (Razorpay)  │   │  │  │ Resource Usage Engine   │    │
│   │ Email (Zepto)       │   │  │  │ (Instances, Volumes,    │    │
│   │ SMS (Twilio)        │   │  │  │  Floating IPs, Bandwidth│    │
│   │ KYC (Digio)         │   │  │  └──────────────────────────┘    │
│   │ PDF (Puppeteer)     │   │  │                                  │
│   └─────────────────────┘   │  │  ┌──────────────────────────┐    │
│                             │  │  │ Invoice Generator        │    │
└─────────────┬───────────────┘  │  │ Suspension Enforcer      │    │
              │                  │  └──────────────────────────┘    │
              │                  └──────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                                  │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ MySQL/MariaDB│  │   MongoDB    │  │    Redis     │              │
│  │ (Portal DB)  │  │ (Billing     │  │ (Cache:      │              │
│  │ (OpenStack   │  │  Records,    │  │  Pricing,    │              │
│  │  Nova,       │  │  Audit Logs, │  │  Sessions)   │              │
│  │  Cinder,     │  │  Activity)   │  │              │              │
│  │  Neutron)    │  │              │  │              │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       INFRASTRUCTURE                                │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  OpenStack   │  │  AWS S3 /    │  │   Grafana    │              │
│  │  (Nova,      │  │  S3-compat   │  │  (Metrics)   │              │
│  │   Cinder,    │  │  (Objects,   │  │              │              │
│  │   Neutron,   │  │   Invoices)  │  │              │              │
│  │   Octavia,   │  │              │  │              │              │
│  │   Glance,    │  │              │  │              │              │
│  │   Barbican)  │  │              │  │              │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 11.2 Data Flow: Billing Cycle

```
1. HOUR: Billing engine queries OpenStack (Nova/Cinder/Neutron)
         ↓
2. Writes usage snapshots to MongoDB (per-resource, per-region)
         ↓
3. DAY: Daily reconciliation — validates usage against OpenStack state
         ↓
4. MONTH (1st): Monthly billing cycle:
   a. Aggregates all usage records for the month
   b. Computes cost per resource (pricing from Redis cache / portal DB)
   c. Applies discounts, credits, promotions
   d. Generates invoice in portal DB
   e. Triggers PDF generation via admin backend's pdf-generator module
   f. Uploads PDF to S3
         ↓
5. Admin portal serves invoice PDF via signed S3 URL
         ↓
6. SUSPENSION (every 30 min):
   a. Checks accounts with overdue invoices
   b. Sends warning notifications (email + SMS)
   c. After grace period: suspends account (marks in portal DB)
   d. Admin backend's AccountSuspensionGuard blocks further resource operations
```

### 11.3 Data Flow: Customer Resource Provisioning

```
1. Customer clicks "Create Instance" in Cloud Console Frontend
         ↓
2. Frontend POST → Admin Backend API
         ↓
3. Backend validates:
   a. Auth (JWT) + Permissions (IAM guard)
   b. Account not suspended (SuspensionGuard)
   c. Quota available
         ↓
4. Backend calls OpenStack Service:
   a. Authenticates with Keystone for customer's project
   b. Calls Nova to create server
         ↓
5. Response flows back:
   Nova → Backend → Frontend (instance details displayed)
         ↓
6. Billing engine picks up new instance on next hourly cycle
   and begins tracking usage
```

---

## 12. Technology Stack Summary

| Layer | Technology | Version |
|---|---|---|
| **Runtime (Backend)** | Node.js | 20 (Alpine) |
| **Runtime (Billing)** | Go | 1.21+ |
| **Backend Framework** | NestJS | 11.x |
| **Frontend Framework** | Next.js (App Router) | 15.5.x |
| **UI Library** | React | 19.1.x |
| **TypeScript** | TypeScript | 5.x |
| **ORM (MySQL)** | TypeORM | Latest |
| **ODM (MongoDB)** | Mongoose | Latest |
| **Cache** | Redis (via cache-manager) | Latest |
| **HTTP Client** | Axios | Latest |
| **Auth** | Passport.js + JWT + otplib | Latest |
| **Payment** | Razorpay, Stripe | Latest |
| **SMS** | Twilio | Latest |
| **Email** | Zepto Mail (custom) | — |
| **PDF** | Puppeteer + Handlebars | Latest |
| **KYC** | Digio | — |
| **Styling (Frontend)** | Tailwind CSS | 4.x |
| **UI Components** | shadcn/ui (Radix UI), MUI 7 | Latest |
| **State Mgmt** | TanStack React Query 5 + Context API | 5.x |
| **Charts** | Recharts, Chart.js | Latest |
| **Containerization** | Docker | Multi-stage |
| **Orchestration** | Docker Compose | Latest |
| **OpenStack SDK (Go)** | gophercloud | Latest |
| **OpenStack SDK (JS)** | openstack-client (via NestJS service) | — |
| **Scheduler** | robfig/cron/v3 | Latest |
| **Logging (Go)** | logrus + lumberjack | Latest |

---

## 13. Deployment & Infrastructure

### 13.1 Docker Configuration

**Billing Engine** (`billing-cron-go/Dockerfile`):
- Multi-stage Go build (golang:1.21-alpine → alpine:3.19)
- Binary size: ~20MB
- Health check endpoint on port 8080
- No runtime dependencies (statically linked binary)

**Admin Backend** (`admin-cc-portal-backend/Dockerfile`):
- Multi-stage Node.js build (node:20-alpine)
- npm ci → build → production-only node_modules
- Clean dependency tree

**Admin Frontend** (`admin-cc-portal-frontend/Dockerfile`):
- Next.js standalone output mode
- Multi-stage: deps → builder → runner
- Port 3001

**Cloud Console Frontend** (`cloud-console-frontend/Dockerfile`):
- Same Next.js standalone pattern
- Edge middleware for S3 credential injection

### 13.2 Environment Configuration

All services use environment variables for configuration:
- Database URLs / connection strings
- OpenStack endpoint URLs + credentials (per region)
- Payment gateway API keys (Razorpay, Stripe)
- SMS/Email API credentials
- S3 access keys + bucket names
- JWT secrets + token expiry
- Redis connection string
- Logging level and rotation settings

### 13.3 Build Automation

- **Makefile** in billing-cron-go: `build`, `test`, `lint`, `clean` targets
- **NPM scripts** in all Node.js projects: `dev`, `build`, `start`, `lint`, `test`
- **Docker Compose** available in each deployable service for local development

---

## 14. Key Architectural Patterns

### 14.1 Multi-Database Strategy

| Database | Purpose | Accessed By |
|---|---|---|
| **Portal MySQL** | Accounts, pricing, invoices, admin users | Admin Backend (TypeORM) + Billing Engine (raw SQL) |
| **OpenStack MySQL** | Nova instances, Cinder volumes, Neutron networks | Billing Engine (read-only) |
| **MongoDB (Main)** | Activity logs, audit trails | Admin Backend (Mongoose) |
| **MongoDB (Billing)** | Usage records, monthly summaries, billing audit logs | Billing Engine + Admin Backend |
| **Redis** | Cache (pricing plans, account status, sessions) | All services |

### 14.2 Dual-Permission System

- **Admin Permissions**: Staff/admin access to backend routes (CRUD accounts, manage billing, configure regions)
- **IAM Permissions**: End-user cloud IAM (project roles, API key scopes, resource-level access)
- Enforced at the NestJS guard level with `@Permissions()` decorators

### 14.3 Account Suspension Guard

- Redis-cached account status (60s TTL)
- Suspended accounts blocked from all write operations except:
  - Wallet top-up
  - Payment processing
  - Invoice viewing
  - 2FA management
- Marked-for-deletion (MFD) accounts get same treatment

### 14.4 Global Response Normalization

All API responses follow a uniform shape:
```json
{
  "success": true,
  "data": { ... },
  "message": "Operation completed",
  "meta": { "total": 100, "page": 1, "limit": 20 }
}
```

### 14.5 Proactive Token Management

- JWT access tokens with HTTP-only refresh cookies
- Activity tracker monitors user interaction and refreshes tokens before expiry
- Idle timeout (20 min) with 15-minute warning
- IAM token rejection in admin context (prevents cross-context misuse)

### 14.6 OpenStack Abstraction

- All OpenStack calls go through a centralized service with:
  - Automatic Keystone authentication and service catalog resolution
  - Per-region credential isolation
  - Dynamic API microversion headers
  - Unified error handling and retry logic

### 14.7 PDF Generation Pipeline

```
Invoice Data (DB) → Handlebars Template → HTML → Puppeteer → PDF → S3 Upload → Signed URL
```

### 14.8 Background Job Processing

- Redis-backed Bull queues for:
  - Invoice generation
  - Email notifications
  - PDF document generation
  - Bulk resource operations

---


