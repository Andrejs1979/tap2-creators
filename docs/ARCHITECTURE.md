# Tap2 Creators - Technical Architecture

## Overview

This document describes the technical architecture for the Tap2 Creators platform, including system components, technology choices, data models, and implementation patterns.

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Technology Stack](#technology-stack)
3. [Component Architecture](#component-architecture)
4. [Data Model](#data-model)
5. [API Design](#api-design)
6. [Authentication & Authorization](#authentication--authorization)
7. [Payment Processing](#payment-processing)
8. [Payout System](#payout-system)
9. [Content Delivery](#content-delivery)
10. [Infrastructure](#infrastructure)
11. [Security](#security)
12. [Monitoring & Observability](#monitoring--observability)

---

## System Architecture

### High-Level Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │  Creator Portal  │  │   Fan Platform   │  │   Public Pages   │          │
│  │   (Next.js)      │  │    (React)       │  │   (Static)       │          │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                             API GATEWAY                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Cloudflare Workers / API Routes (Next.js)                          │   │
│  │  - Rate Limiting  - Auth Validation  - Request Routing              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   Core Services  │  │  Payment Service │  │  Content Service │
│  (API Routes)    │  │   (Stripe)       │  │    (R2/Workers)  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DATA LAYER                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │   PostgreSQL     │  │      Redis       │  │   Cloudflare R2  │          │
│  │   (Primary DB)   │  │    (Cache)       │  │   (Files/Assets) │          │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EXTERNAL SERVICES                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  Stripe  │  FlowPay  │  SendGrid  │  Plaid  │  Vercel  │  Sentry          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

### Frontend

| Component | Technology | Rationale |
|-----------|------------|-----------|
| Creator Portal | Next.js 15 (App Router) | SSR, SEO, API routes |
| Fan Platform | React 19 + PWA | Offline support, app-like experience |
| Styling | Tailwind CSS | Rapid development, small bundle |
| State Management | Zustand | Simple, lightweight, no boilerplate |
| Forms | React Hook Form + Zod | Type-safe validation |
| Data Fetching | SWR / React Query | Caching, revalidation, optimistic updates |
| Components | Radix UI + shadcn/ui | Accessible, customizable primitives |

### Backend

| Component | Technology | Rationale |
|-----------|------------|-----------|
| API Runtime | Node.js 20 LTS | Long-term support, performance |
| Web Framework | Next.js API Routes | Full-stack in one repo |
| ORM | Drizzle ORM | Type-safe, lightweight, great DX |
| Database | PostgreSQL 16 | Relational data, transactions, JSONB |
| Cache | Redis (Upstash) | Session storage, rate limiting |
| Queue | PostgreSQL + pg-boss | Simple job queue, no additional infra |
| File Storage | Cloudflare R2 | S3-compatible, egress-free |
| Email | SendGrid / Resend | Transactional email, webhooks |

### Infrastructure

| Component | Technology | Rationale |
|-----------|------------|-----------|
| Hosting | Vercel | Zero-config deployments, preview URLs |
| CDN | Cloudflare | Global edge, DDoS protection |
| Monitoring | Sentry + Vercel Analytics | Error tracking, performance |
| Logging | Datadog / Loki | Structured logs, aggregation |
| Secret Management | Vercel Env Variables / Doppler | Secure credential storage |

---

## Component Architecture

### Frontend Architecture

```
tap2-creators/
├── apps/
│   ├── creator-portal/          # Next.js app for creators
│   │   ├── app/
│   │   │   ├── (auth)/          # Auth routes (login, signup)
│   │   │   ├── (dashboard)/     # Protected dashboard
│   │   │   │   ├── page.tsx     # Dashboard home
│   │   │   │   ├── products/    # Product management
│   │   │   │   ├── subscriptions/
│   │   │   │   ├── payouts/     # Payout management
│   │   │   │   └── analytics/   # Analytics dashboard
│   │   │   └── api/             # API routes
│   │   ├── components/
│   │   │   ├── ui/              # shadcn/ui components
│   │   │   ├── forms/           # Form components
│   │   │   ├── charts/          # Analytics charts
│   │   │   └── layout/          # Layout components
│   │   └── lib/
│   │       ├── db.ts            # Database client
│   │       ├── auth.ts          # Auth utilities
│   │       ├── stripe.ts        # Stripe client
│   │       └── utils.ts         # Shared utilities
│   │
│   └── fan-platform/            # React PWA for fans
│       ├── app/
│       │   ├── discover/        # Creator discovery
│       │   ├── @username/       # Creator profiles
│       │   ├── library/         # Purchased products
│       │   └── settings/        # Account settings
│       └── components/
│
├── packages/
│   ├── ui/                      # Shared UI components
│   ├── config/                  # Shared config (ESLint, TS)
│   ├── types/                   # Shared TypeScript types
│   └── api-client/              # API client wrapper
│
└── services/
    ├── auth-service/            # Authentication logic
    ├── payment-service/         # Payment processing
    ├── payout-service/          # Payout engine
    └── notification-service/    # Email/push notifications
```

### Backend Architecture

```
API Routes (Next.js)
│
├── /api/v1/auth/
│   ├── register
│   ├── login
│   ├── logout
│   └── me
│
├── /api/v1/creators/
│   ├── profile
│   ├── products
│   ├── subscriptions
│   ├── payouts
│   └── analytics
│
├── /api/v1/fans/
│   ├── tips
│   ├── subscriptions
│   ├── purchases
│   └── library
│
├── /api/v1/webhooks/
│   ├── stripe
│   └── payout
│
└── /api/v1/public/
    ├── creators
    └── discovery
```

---

## Data Model

### Entity Relationship Diagram

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   Creator   │───1:N─│   Product   │───1:N─│  Purchase   │
├─────────────┤       ├─────────────┤       ├─────────────┤
│ id          │       │ id          │       │ id          │
│ userId      │       │ creatorId   │       │ fanId       │
│ slug        │       │ title       │       │ productId   │
│ displayName │       │ description │       │ amount      │
│ bio         │       │ price       │       │ status      │
│ avatarUrl   │       │ fileUrls    │       └─────────────┘
│ bannerUrl   │       │ tierId      │
│ bankAccounts│       │ published   │
└─────────────┘       └─────────────┘
       │
       │
       │1:N
┌─────────────┐       ┌─────────────┐
│   Tier      │───1:N─│ Subscription│
├─────────────┤       ├─────────────┤
│ id          │       │ id          │
│ creatorId   │       │ fanId       │
│ name        │       │ tierId      │
│ price       │       │ status      │
│ description │       │ currentPeriodEnd
│ benefits    │       └─────────────┘
│ sortOrder   │
└─────────────┘

┌─────────────┐       ┌─────────────┐
│     User    │───1:1─│  Account    │
├─────────────┤       ├─────────────┤
│ id          │       │ id          │
│ email       │       │ userId      │
│ passwordHash│       │ provider    │
│ role        │       │ providerAccountId
│ createdAt   │       └─────────────┘
└─────────────┘
       │1:N
┌─────────────┐
│     Tip     │
├─────────────┤
│ id          │
│ creatorId   │
│ fanId       │
│ amount      │
│ message     │
│ anonymous   │
│ status      │
└─────────────┘
```

### Schema Definitions

#### Users & Authentication

```sql
-- Users table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255),
  role VARCHAR(50) NOT NULL, -- 'creator' | 'fan' | 'admin'
  email_verified BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Accounts table (for OAuth)
CREATE TABLE accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  provider VARCHAR(50) NOT NULL, -- 'google' | 'email'
  provider_account_id VARCHAR(255) NOT NULL,
  refresh_token TEXT,
  access_token TEXT,
  expires_at TIMESTAMPTZ,
  UNIQUE(provider, provider_account_id)
);

-- Sessions table
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  token VARCHAR(255) UNIQUE NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2FA table
CREATE TABLE two_factor_auth (
  user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  secret VARCHAR(255),
  backup_codes TEXT[],
  enabled BOOLEAN DEFAULT FALSE
);
```

#### Creators

```sql
CREATE TABLE creators (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  slug VARCHAR(100) UNIQUE NOT NULL,
  display_name VARCHAR(255) NOT NULL,
  bio TEXT,
  avatar_url TEXT,
  banner_url TEXT,
  social_links JSONB, -- { twitter, youtube, instagram, tiktok }
  theme_colors JSONB, -- { primary, secondary, accent }
  published BOOLEAN DEFAULT FALSE,
  auto_payout_enabled BOOLEAN DEFAULT FALSE,
  default_bank_account_id UUID,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Bank accounts for payouts
CREATE TABLE bank_accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id UUID REFERENCES creators(id) ON DELETE CASCADE,
  country_code VARCHAR(2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  account_number_last_4 VARCHAR(4),
  routing_number VARCHAR(9),
  account_holder_name VARCHAR(255),
  plaid_account_id VARCHAR(255),
  stripe_bank_account_id VARCHAR(255),
  verification_status VARCHAR(50) DEFAULT 'pending',
  is_default BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### Fans

```sql
CREATE TABLE fans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  display_name VARCHAR(255),
  avatar_url TEXT,
  notification_preferences JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Saved payment methods
CREATE TABLE payment_methods (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  fan_id UUID REFERENCES fans(id) ON DELETE CASCADE,
  stripe_payment_method_id VARCHAR(255) NOT NULL,
  is_default BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### Products

```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id UUID REFERENCES creators(id) ON DELETE CASCADE,
  title VARCHAR(255) NOT NULL,
  description TEXT,
  type VARCHAR(50) NOT NULL, -- 'download' | 'course' | 'unlockable'
  price_cents INTEGER NOT NULL,
  cover_image_url TEXT,
  file_urls JSONB, -- Array of file URLs
  tier_id UUID REFERENCES tiers(id), -- Optional: gate by tier
  published BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Product purchases
CREATE TABLE purchases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID REFERENCES products(id),
  fan_id UUID REFERENCES fans(id),
  amount_cents INTEGER NOT NULL,
  stripe_payment_intent_id VARCHAR(255),
  status VARCHAR(50) DEFAULT 'pending', -- 'pending' | 'completed' | 'refunded'
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### Subscriptions & Tiers

```sql
-- Subscription tiers
CREATE TABLE tiers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id UUID REFERENCES creators(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price_cents INTEGER NOT NULL,
  image_url TEXT,
  benefits TEXT[],
  sort_order INTEGER DEFAULT 0,
  published BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Active subscriptions
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  fan_id UUID REFERENCES fans(id) ON DELETE CASCADE,
  tier_id UUID REFERENCES tiers(id),
  creator_id UUID REFERENCES creators(id),
  stripe_subscription_id VARCHAR(255) UNIQUE,
  status VARCHAR(50) DEFAULT 'active', -- 'active' | 'cancelled' | 'past_due'
  current_period_start TIMESTAMPTZ,
  current_period_end TIMESTAMPTZ,
  cancel_at_period_end BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### Tips

```sql
CREATE TABLE tips (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id UUID REFERENCES creators(id),
  fan_id UUID REFERENCES fans(id),
  amount_cents INTEGER NOT NULL,
  message TEXT,
  anonymous BOOLEAN DEFAULT FALSE,
  stripe_payment_intent_id VARCHAR(255),
  status VARCHAR(50) DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### Payouts

```sql
CREATE TABLE payouts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id UUID REFERENCES creators(id),
  bank_account_id UUID REFERENCES bank_accounts(id),
  amount_cents INTEGER NOT NULL,
  currency VARCHAR(3) DEFAULT 'USD',
  status VARCHAR(50) DEFAULT 'pending', -- 'pending' | 'processing' | 'completed' | 'failed'
  type VARCHAR(50) DEFAULT 'instant', -- 'instant' | 'daily'
  flowpay_transfer_id VARCHAR(255),
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);

-- Creator balances (denormalized for performance)
CREATE TABLE creator_balances (
  creator_id UUID PRIMARY KEY REFERENCES creators(id),
  available_cents INTEGER DEFAULT 0,
  pending_cents INTEGER DEFAULT 0,
  lifetime_cents INTEGER DEFAULT 0,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### Analytics (Time-series data)

```sql
-- Daily metrics for analytics
CREATE TABLE daily_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id UUID REFERENCES creators(id),
  date DATE NOT NULL,
  revenue_cents INTEGER DEFAULT 0,
  tips_count INTEGER DEFAULT 0,
  tips_amount_cents INTEGER DEFAULT 0,
  new_subscriptions INTEGER DEFAULT 0,
  subscriptions_cents INTEGER DEFAULT 0,
  purchases_count INTEGER DEFAULT 0,
  purchases_cents INTEGER DEFAULT 0,
  UNIQUE(creator_id, date)
);
```

---

## API Design

### RESTful Conventions

| Method | Pattern | Description |
|--------|---------|-------------|
| GET | `/api/v1/resource` | List resources (paginated) |
| GET | `/api/v1/resource/:id` | Get single resource |
| POST | `/api/v1/resource` | Create resource |
| PUT | `/api/v1/resource/:id` | Update resource (full) |
| PATCH | `/api/v1/resource/:id` | Update resource (partial) |
| DELETE | `/api/v1/resource/:id` | Delete resource |

### Response Format

```typescript
// Success response
interface SuccessResponse<T> {
  data: T;
  meta?: {
    pagination?: {
      page: number;
      limit: number;
      total: number;
      totalPages: number;
    };
  };
}

// Error response
interface ErrorResponse {
  error: {
    code: string;
    message: string;
    details?: Record<string, string[]>;
  };
}
```

### API Endpoints

#### Authentication

```typescript
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh
GET    /api/v1/auth/me
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
POST   /api/v1/auth/verify-email
POST   /api/v1/auth/enable-2fa
POST   /api/v1/auth/verify-2fa
```

#### Creator Profile

```typescript
GET    /api/v1/creators/profile           // Get own profile
PUT    /api/v1/creators/profile           // Update profile
GET    /api/v1/creators/:slug             // Public profile (fans)
POST   /api/v1/creators/slug/validate     // Check slug availability
```

#### Products

```typescript
GET    /api/v1/creators/products          // List products
POST   /api/v1/creators/products          // Create product
GET    /api/v1/creators/products/:id      // Get product
PUT    /api/v1/creators/products/:id      // Update product
DELETE /api/v1/creators/products/:id      // Delete product
POST   /api/v1/creators/products/:id/publish
POST   /api/v1/creators/products/:id/unpublish
```

#### Tiers & Subscriptions

```typescript
// Tiers
GET    /api/v1/creators/tiers
POST   /api/v1/creators/tiers
PUT    /api/v1/creators/tiers/:id
DELETE /api/v1/creators/tiers/:id

// Subscriptions (creator view)
GET    /api/v1/creators/subscriptions
GET    /api/v1/creators/subscribers
GET    /api/v1/creators/subscribers/:id
```

#### Payouts

```typescript
GET    /api/v1/creators/payouts           // List payouts
POST   /api/v1/creators/payouts           // Request payout
GET    /api/v1/creators/payouts/:id       // Get payout details
GET    /api/v1/creators/balance           // Current balance
POST   /api/v1/creators/balance/withdraw  // Instant cash out
PUT    /api/v1/creators/payouts/settings  // Auto-payout settings
```

#### Analytics

```typescript
GET    /api/v1/creators/analytics/overview
GET    /api/v1/creators/analytics/revenue
GET    /api/v1/creators/analytics/subscribers
GET    /api/v1/creators/analytics/products
```

#### Bank Accounts

```typescript
GET    /api/v1/creators/bank-accounts
POST   /api/v1/creators/bank-accounts
DELETE /api/v1/creators/bank-accounts/:id
PUT    /api/v1/creators/bank-accounts/:id/default
POST   /api/v1/creators/bank-accounts/verify
POST   /api/v1/creators/bank-accounts/plaid/link
```

#### Fan Actions

```typescript
// Tips
POST   /api/v1/fans/tips                  // Create tip
GET    /api/v1/fans/tips/history          // Tip history

// Subscriptions
POST   /api/v1/fans/subscriptions         // Subscribe
PUT    /api/v1/fans/subscriptions/:id     // Update/cancel
GET    /api/v1/fans/subscriptions         // My subscriptions

// Purchases
POST   /api/v1/fans/purchases             // Buy product
GET    /api/v1/fans/library               // Purchased products

// Payment methods
GET    /api/v1/fans/payment-methods
POST   /api/v1/fans/payment-methods
PUT    /api/v1/fans/payment-methods/:id/default
DELETE /api/v1/fans/payment-methods/:id
```

#### Public Endpoints

```typescript
GET    /api/v1/public/creators            // Discover creators
GET    /api/v1/public/creators/:slug      // Creator profile
GET    /api/v1/public/creators/:slug/products
GET    /api/v1/public/creators/:slug/tiers
```

#### Webhooks

```typescript
POST   /api/v1/webhooks/stripe            // Stripe webhooks
POST   /api/v1/webhooks/payout            // Payout provider webhooks
```

---

## Authentication & Authorization

### JWT-based Authentication

```typescript
// Access token structure
interface JWTPayload {
  sub: string;      // User ID
  role: string;     // 'creator' | 'fan' | 'admin'
  type: string;     // 'access' | 'refresh'
  iat: number;      // Issued at
  exp: number;      // Expires at
}

// Token lifetimes
const ACCESS_TOKEN_LIFETIME = 15 * 60;  // 15 minutes
const REFRESH_TOKEN_LIFETIME = 30 * 24 * 60 * 60;  // 30 days
```

### Authorization Model

```typescript
// Role-based permissions
type Role = 'admin' | 'creator' | 'fan';

interface Permissions {
  // Creator permissions
  'creators.profile:update': 'creator';
  'creators.products:create': 'creator';
  'creators.payouts:create': 'creator';
  'creators.analytics:view': 'creator';

  // Fan permissions
  'fans.tips:create': 'fan';
  'fans.subscriptions:create': 'fan';
  'fans.purchases:create': 'fan';
}

// Middleware for route protection
export function requireRole(...roles: Role[]) {
  return async (request: Request) => {
    const user = await getUser(request);
    if (!user || !roles.includes(user.role)) {
      return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
    }
    return user;
  };
}
```

### Session Management

```typescript
// Redis session storage
interface Session {
  userId: string;
  token: string;
  expiresAt: Date;
  userAgent?: string;
  ipAddress?: string;
}

// Rate limiting
interface RateLimitConfig {
  window: number;  // Time window in seconds
  maxRequests: number;
}

const rateLimits: Record<string, RateLimitConfig> = {
  'api/v1/auth/login': { window: 300, maxRequests: 5 },      // 5 per 5 min
  'api/v1/auth/register': { window: 3600, maxRequests: 3 },  // 3 per hour
  'api/v1/fans/tips': { window: 60, maxRequests: 10 },       // 10 per minute
};
```

---

## Payment Processing

### Stripe Integration

```typescript
// Payment flow
1. Client creates payment intent on server
2. Server creates Stripe PaymentIntent
3. Client confirms payment with Stripe.js
4. Stripe webhook confirms payment
5. Server updates database and processes fulfillment

// Payment intent creation
interface CreatePaymentIntent {
  amount: number;      // in cents
  currency: string;
  customerId: string;
  metadata: {
    type: 'tip' | 'product' | 'subscription';
    resourceId?: string;
    creatorId: string;
  };
}
```

### Tip Processing Flow

```
Fan → Creator Tip Page
  │
  ├─> POST /api/v1/fans/tips
  │     ├─> Validate input
  │     ├─> Create Stripe PaymentIntent
  │     └─> Return client_secret
  │
  ├─> Fan confirms with Stripe.js
  │
  └─> Stripe webhook: payment_intent.succeeded
        ├─> Verify signature
        ├─> Create tip record
        ├─> Update creator balance
        ├─> Send notification to creator
        └─> Send receipt to fan
```

### Subscription Flow

```
Fan → Creator Profile → Subscribe
  │
  ├─> POST /api/v1/fans/subscriptions
  │     ├─> Create Stripe Customer (if needed)
  │     ├─> Create Stripe Subscription
  │     └─> Return subscription details
  │
  └─> Stripe webhook: checkout.session.completed
        ├─> Update subscription record
        ├─> Grant tier access
        └─> Send welcome email
```

### Webhook Handling

```typescript
// Webhook event handlers
const webhookHandlers = {
  'payment_intent.succeeded': handlePaymentSuccess,
  'payment_intent.failed': handlePaymentFailure,
  'checkout.session.completed': handleCheckoutComplete,
  'customer.subscription.created': handleSubscriptionCreated,
  'customer.subscription.updated': handleSubscriptionUpdated,
  'customer.subscription.deleted': handleSubscriptionCancelled,
  'invoice.payment_succeeded': handleInvoicePaid,
  'invoice.payment_failed': handleInvoiceFailed,
};

// Security: Verify webhook signature
function verifyWebhookSignature(payload: string, signature: string): boolean {
  const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;
  const event = stripe.webhooks.constructEvent(payload, signature, webhookSecret);
  return event !== null;
}
```

---

## Payout System

### Balance Tracking

```typescript
interface CreatorBalance {
  available: number;   // Available for withdrawal
  pending: number;     // Pending clearance (7-day hold)
  lifetime: number;    // Total earnings
}

// Balance calculation
available = tips + (subscription_fees × 0.7) + product_revenue - refunds - fees
pending = recent_transactions (< 7 days) - pending_refunds
```

### Instant Payout Flow

```
Creator → Dashboard → Withdraw
  │
  ├─> POST /api/v1/creators/balance/withdraw
  │     ├─> Validate balance > minimum
  │     ├─> Check for pending withdrawals
  │     ├─> Create FlowPay transfer
  │     ├─> Update balance (deduct amount)
  │     └─> Create payout record
  │
  └─> FlowPay webhook: transfer.completed
        ├─> Update payout status
        └─> Send notification
```

### Daily Auto-Payout

```typescript
// Cron job: runs daily at midnight UTC
export async function dailyAutoPayouts() {
  const creators = await getCreatorsWithAutoPayoutEnabled();

  for (const creator of creators) {
    const balance = await getCreatorBalance(creator.id);

    if (balance.available >= MINIMUM_PAYOUT) {
      await initiatePayout({
        creatorId: creator.id,
        amount: balance.available,
        bankAccountId: creator.defaultBankAccountId,
        type: 'daily'
      });
    }
  }
}
```

---

## Content Delivery

### File Upload & Storage

```typescript
// Upload flow
1. Client requests upload URL from server
2. Server generates presigned URL (R2)
3. Client uploads directly to R2
4. Client confirms upload completion
5. Server verifies and stores file reference

// Upload endpoint response
interface UploadResponse {
  uploadUrl: string;      // Presigned R2 URL
  fileKey: string;        // Storage key
  fileUrl: string;        // Public URL (after upload)
  expiresAt: Date;        // URL expiration
}
```

### File Type Handling

```typescript
// Allowed file types
const ALLOWED_FILE_TYPES = {
  'image/png': ['.png'],
  'image/jpeg': ['.jpg', '.jpeg'],
  'image/gif': ['.gif'],
  'image/webp': ['.webp'],
  'application/pdf': ['.pdf'],
  'video/mp4': ['.mp4'],
  'audio/mpeg': ['.mp3'],
  'audio/wav': ['.wav'],
  'application/zip': ['.zip'],
};

// Maximum file sizes
const MAX_FILE_SIZES = {
  avatar: 5 * 1024 * 1024,      // 5MB
  banner: 10 * 1024 * 1024,     // 10MB
  product_file: 2 * 1024 * 1024 * 1024,  // 2GB
};
```

### Digital Product Delivery

```typescript
// Purchase fulfillment flow
1. Payment confirmed
2. Generate access token (time-limited URL)
3. Send email with download link
4. Link expires after 7 days or 5 downloads

interface DownloadToken {
  purchaseId: string;
  fileIds: string[];
  expiresAt: Date;
  maxDownloads: number;
  downloadCount: number;
}
```

---

## Infrastructure

### Environment Strategy

| Environment | Purpose | URL |
|-------------|---------|-----|
| Local | Development | localhost:3000 |
| Preview | Pull Request previews | pr-{number}.vercel.app |
| Staging | Pre-production testing | staging.tap2.gg |
| Production | Live environment | tap2.gg |

### Database Strategy

```typescript
// Database configuration
const config = {
  development: {
    url: process.env.DATABASE_URL,
    ssl: false,
    logLevel: 'debug',
  },
  production: {
    url: process.env.DATABASE_URL,
    ssl: { rejectUnauthorized: false },
    pooler: true,  // Use connection pooler
  },
};

// Migration strategy
- Drizzle Kit for schema management
- Migrations run via CI/CD before deployment
- Rollback capability for all migrations
```

### Deployment Pipeline

```
┌────────────┐
│   Push     │
│   to main  │
└─────┬──────┘
      │
      ▼
┌────────────┐
│    CI      │  Run tests, lint, type check
└─────┬──────┘
      │
      ▼
┌────────────┐
│  Build     │  Build Next.js app
└─────┬──────┘
      │
      ▼
┌────────────┐
│  Deploy    │  Deploy to Vercel
└─────┬──────┘
      │
      ▼
┌────────────┐
│  Migrate   │  Run DB migrations
└─────┬──────┘
      │
      ▼
┌────────────┐
│  Smoke     │  Health check
│   Test     │
└────────────┘
```

---

## Security

### Security Checklist

- [ ] All API endpoints authenticated (except public)
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (input sanitization, CSP headers)
- [ ] CSRF protection (SameSite cookies, token validation)
- [ ] Rate limiting on all public endpoints
- [ ] File upload validation (type, size, content)
- [ ] Secrets never committed to git
- [ ] Dependencies scanned for vulnerabilities
- [ ] Webhook signature verification
- [ ] PII data encrypted at rest
- [ ] HTTPS only (redirect HTTP to HTTPS)
- [ ] Security headers (HSTS, X-Frame-Options, etc.)

### Security Headers

```typescript
// next.config.js
const securityHeaders = [
  {
    key: 'X-DNS-Prefetch-Control',
    value: 'on'
  },
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=63072000; includeSubDomains; preload'
  },
  {
    key: 'X-Frame-Options',
    value: 'SAMEORIGIN'
  },
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff'
  },
  {
    key: 'X-XSS-Protection',
    value: '1; mode=block'
  },
  {
    key: 'Referrer-Policy',
    value: 'strict-origin-when-cross-origin'
  },
  {
    key: 'Content-Security-Policy',
    value: CSP_DIRECTIVE
  }
];
```

### Input Validation

```typescript
// Zod schemas for validation
import { z } from 'zod';

// Tip creation schema
export const createTipSchema = z.object({
  amount: z.number().min(100).max(1000000), // $1 - $10,000
  message: z.string().max(500).optional(),
  anonymous: z.boolean().default(false),
  creatorSlug: z.string().min(1).max(100),
});

// Product creation schema
export const createProductSchema = z.object({
  title: z.string().min(1).max(255),
  description: z.string().max(5000).optional(),
  type: z.enum(['download', 'course', 'unlockable']),
  price: z.number().min(0),
  tierId: z.string().uuid().optional(),
  fileUrls: z.array(z.string().url()).min(1),
});
```

---

## Monitoring & Observability

### Error Tracking

```typescript
// Sentry integration
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,  // 10% of transactions
  beforeSend(event) {
    // Filter out sensitive data
    if (event.request) {
      delete event.request.cookies;
    }
    return event;
  },
});
```

### Logging Strategy

```typescript
// Structured logging
interface LogContext {
  userId?: string;
  requestId?: string;
  action?: string;
  [key: string]: any;
}

export function logger(level: 'info' | 'warn' | 'error', message: string, context: LogContext = {}) {
  const logEntry = {
    level,
    message,
    ...context,
    timestamp: new Date().toISOString(),
  };

  if (level === 'error') {
    Sentry.captureException(new Error(message), { extra: context });
  }

  console.log(JSON.stringify(logEntry));
}
```

### Metrics & Analytics

```typescript
// Key metrics to track
const metrics = {
  // Business metrics
  'revenue.total': 'Gauge',
  'tips.count': 'Counter',
  'subscriptions.new': 'Counter',
  'products.purchased': 'Counter',

  // Technical metrics
  'api.requests': 'Counter',
  'api.latency': 'Histogram',
  'api.errors': 'Counter',
  'db.queries': 'Counter',
  'db.latency': 'Histogram',

  // User metrics
  'users.active': 'Gauge',
  'users.churn': 'Gauge',
};
```

### Performance Targets

| Metric | Target | P95 | P99 |
|--------|--------|-----|-----|
| API Response Time | < 200ms | < 400ms | < 1s |
| Page Load | < 2s | < 3s | < 5s |
| Webhook Processing | < 1s | < 2s | < 5s |
| Database Query | < 50ms | < 100ms | < 200ms |

---

## Future Considerations

### Scalability

- **Database**: Consider read replicas for analytics queries
- **Cache**: Implement Redis caching for frequently accessed data
- **Queue**: Move to dedicated job queue (SQS, RabbitMQ) for high volume
- **CDR**: Implement CDN caching for static assets

### Internationalization

- Multi-currency support for payments
- Multi-language support for UI
- Localized payment methods

### Advanced Features

- Live streaming integration
- NFT/digital collectibles
- AI-powered content recommendations
- Creator collaboration features

---

*Last Updated: February 2026*
