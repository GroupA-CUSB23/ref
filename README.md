# Project Specification: Multi-Vendor E-Commerce Platform

## 1. Technology Stack

**Backend (Laravel API)**
- **Framework:** Laravel 13 (PHP 8.3+)
- **Database:** PostgreSQL 16
- **Auth/SSO:** Laravel Sanctum + Laravel Socialite
- **Security:** Spatie Permission (RBAC) & Spatie Activitylog
- **Admin Panel:** Filament PHP v3
- **Cache/Jobs:** Redis 7+

**Frontend (React SPA)**
- **Core:** React 19 (via Vite)
- **Routing & State:** TanStack Router v1 + TanStack Query v5
- **UI/Styling:** Tailwind CSS + shadcn/ui
- **Forms:** React Hook Form + Zod

**Infrastructure (Self-Hosted)**
- **OS:** Ubuntu 24.04 LTS
- **Web Server:** Nginx + Let's Encrypt SSL
- **Deployment:** Native LEMP or Coolify

## 2. Platform Features

**Customer Experience (Storefront)**
- **IAM & Auth:** Email/password and SSO (Google, Apple).
- **Marketplace Discovery:** Global search, categories, and dedicated Merchant Shop pages.
- **Shopping Cart:** Multi-vendor cart visually grouped by shop.
- **Unified Checkout:** Single payment for multi-vendor carts.
- **Customer Portal:** Saved payment methods, address book, wishlists, and order history.
- **Post-Purchase:** Real-time hybrid order tracking, RMAs (returns), and verified reviews.

**Vendor Management (Merchant Dashboard)**
- **Shop Management:** Shop onboarding, logo uploads, and profile management.
- **Product Catalog:** Products with complex JSONB variants (size/color), stock tracking, and image galleries.
- **Isolated Orders:** View and fulfill only shop-specific sub-orders.
- **Financials:** Track shop revenue, sub-totals, and platform fee deductions.

**Hybrid Logistics & Delivery**
- **Smart Routing:** Auto-route to in-house drivers or third-party couriers based on radius.
- **Third-Party Integration:** API integration (e.g., J&T, DHL) for labels, tracking, and webhooks.
- **Driver PWA:** Mobile dashboard for in-house drivers (assigned routes, status updates, photo proof-of-delivery).

**Super Admin Console**
- **Advanced RBAC:** Granular role and permission management.
- **Moderation:** Approve/suspend merchant shops and moderate reviews.
- **Platform Settings:** Manage tax rules, shipping methods, and global discount coupons.
- **Financial Oversight:** Monitor all unified checkouts and process refunds.

## 3. System Architecture Flows

```mermaid
flowchart TD
    %% Styles
    classDef actor fill:#2d3748,stroke:#4a5568,color:#fff

    %% Actors
    Customer((Customer)):::actor
    Vendor((Vendor)):::actor

    %% 1. IAM & SSO Authentication
    subgraph Phase 1: IAM & SSO Authentication
        direction TB
        SSO[Google / Apple Provider]
        AuthAPI[Laravel Socialite API]
        DB_Auth[(Users & SSO Identities)]
        SPA[TanStack React SPA]
        
        Customer -->|Clicks Login| SSO
        Vendor -->|Clicks Login| SSO
        SSO -->|OAuth Callback| AuthAPI
        AuthAPI -->|Check/Create Profile| DB_Auth
        DB_Auth -->|Issue Sanctum Token| SPA
    end

    %% Routing based on Spatie RBAC Roles
    SPA -->|RBAC: Vendor Role| V_Dash[Vendor Dashboard]
    SPA -->|RBAC: Customer Role| C_Shop[Marketplace Browse]

    %% 2. Vendor Flow (Product Creation & Audit)
    subgraph Phase 2: Vendor Management & Auditing
        direction TB
        V_Dash -->|Submit Multipart Form| API_Prod[Product API]
        API_Prod -->|Upload Images| S3[AWS S3]
        API_Prod -->|Insert Product & Variants| DB_Prod[(Products Table)]
        DB_Prod -.->|Spatie Observer Triggers| DB_Audit[(Audit Logs)]
    end

    %% 3. Customer Flow (Cart & Split Checkout)
    subgraph Phase 3: Multi-Vendor Checkout & Order Splitting
        direction TB
        C_Shop -->|Fetch Catalog| DB_Prod
        C_Shop -->|Add Items to Cart| DB_Cart[(Cart & Items)]
        
        DB_Cart -->|Initiate Payment| API_Check[Checkout API]
        API_Check -->|DB Transaction: Lock Rows| DB_Prod
        
        API_Check -->|Charge Grand Total| Stripe[Stripe Gateway]
        Stripe -->|Success Token| DB_Pay[(Checkouts & Payments)]
        
        DB_Pay -->|Backend Order Split| Split{Split by Shop_ID}
        Split -->|Sub-order 1| OrderA[(Shop A Orders)]
        Split -->|Sub-order 2| OrderB[(Shop B Orders)]
        
        OrderA -->|Decrement Stock| DB_Prod
        OrderB -->|Decrement Stock| DB_Prod
    end
```

## 4. Database Schema (ER Diagram)

```mermaid
erDiagram
    %% 1. USERS, IAM & ACCESS CONTROL
    users {
        bigint id PK
        varchar name
        varchar email UK
        varchar password "nullable for SSO"
        timestamp created_at
        timestamp updated_at
    }
    sso_identities {
        bigint id PK
        bigint user_id FK
        varchar provider_name "google, apple"
        varchar provider_id UK
        timestamp created_at
    }
    user_addresses {
        bigint id PK
        bigint user_id FK
        enum type "billing, shipping"
        varchar street
        varchar city
        varchar zip
        boolean is_default
    }

    %% 2. ADVANCED RBAC (Spatie Permission)
    roles {
        bigint id PK
        varchar name UK "Super Admin, Vendor, Driver"
    }
    permissions {
        bigint id PK
        varchar name UK
    }
    role_has_permissions {
        bigint role_id FK
        bigint permission_id FK
    }
    model_has_roles {
        bigint role_id FK
        bigint model_id FK "user_id"
    }
    model_has_permissions {
        bigint permission_id FK
        bigint model_id FK "user_id"
    }

    %% 3. SECURITY & AUDIT LOGS
    audit_logs {
        bigint id PK
        bigint user_id FK "nullable"
        varchar event "created, updated, deleted"
        varchar auditable_type
        bigint auditable_id
        jsonb old_values
        jsonb new_values
        timestamp created_at
    }

    %% 4. MULTI-VENDOR MARKETPLACE
    shops {
        bigint id PK
        bigint vendor_id FK "users.id"
        varchar name
        varchar slug UK
        text description
        varchar logo_url
        enum status "pending, active, suspended"
        timestamp created_at
    }

    %% 5. PRODUCT CATALOG
    categories {
        bigint id PK
        bigint parent_id FK "nullable"
        varchar name
        varchar slug UK
    }
    products {
        bigint id PK
        bigint shop_id FK
        bigint category_id FK
        varchar name
        varchar slug UK
        text description
        decimal base_price
        enum status
        timestamp deleted_at
    }
    product_variants {
        bigint id PK
        bigint product_id FK
        varchar sku UK
        integer stock_quantity
        decimal price_modifier
        jsonb attributes "e.g., size, color"
    }
    product_images {
        bigint id PK
        bigint product_id FK
        varchar image_url
        boolean is_primary
        integer sort_order
    }

    %% 6. CUSTOMER EXPERIENCE
    wishlists {
        bigint id PK
        bigint user_id FK
        bigint product_id FK
        timestamp created_at
    }
    reviews {
        bigint id PK
        bigint product_id FK
        bigint user_id FK
        smallint rating "1-5"
        text comment
        boolean is_verified_purchase
        timestamp created_at
    }

    %% 7. SHOPPING CART
    carts {
        bigint id PK
        bigint user_id FK "nullable"
        varchar session_id "for guests"
    }
    cart_items {
        bigint id PK
        bigint cart_id FK
        bigint product_variant_id FK
        integer quantity
    }

    %% 8. MARKETING & TAXES
    coupons {
        bigint id PK
        varchar code UK
        enum type "percentage, fixed"
        decimal value
        timestamp valid_until
        integer usage_limit
    }
    shipping_methods {
        bigint id PK
        varchar name
        decimal base_cost
    }
    tax_rates {
        bigint id PK
        varchar country
        varchar state
        decimal rate_percentage
    }

    %% 9. CHECKOUT & PAYMENTS (Unified)
    checkouts {
        bigint id PK
        bigint user_id FK
        varchar checkout_reference UK
        decimal grand_total
        enum status
        timestamp created_at
    }
    payment_methods {
        bigint id PK
        bigint user_id FK
        varchar provider "stripe, paypal"
        varchar provider_token
        varchar card_brand
        varchar last_four
        boolean is_default
    }
    payments {
        bigint id PK
        bigint checkout_id FK
        bigint payment_method_id FK "nullable"
        varchar provider
        varchar transaction_id
        enum status "pending, completed, failed"
        decimal amount
    }

    %% 10. SPLIT ORDERS & POST-PURCHASE
    orders {
        bigint id PK
        bigint checkout_id FK
        bigint shop_id FK
        bigint shipping_method_id FK
        varchar order_number UK
        enum status "pending, ready, shipped, delivered"
        enum fulfillment_type "in_house, third_party"
        decimal shop_total
        jsonb shipping_address "snapshot"
        timestamp created_at
    }
    order_items {
        bigint id PK
        bigint order_id FK
        bigint product_variant_id FK
        integer quantity
        decimal unit_price "snapshot"
        decimal subtotal
    }
    order_coupons {
        bigint id PK
        bigint order_id FK
        bigint coupon_id FK
        decimal discount_applied
    }
    returns {
        bigint id PK
        bigint order_id FK
        enum status "pending, approved, refunded"
        text reason
        decimal refund_amount
        timestamp created_at
    }

    %% 11. HYBRID LOGISTICS (In-House vs Express)
    deliveries {
        bigint id PK
        bigint order_id FK "In-House Route"
        bigint driver_id FK "users.id"
        enum status "assigned, picked_up, out_for_delivery, delivered"
        varchar proof_of_delivery_url "S3 image"
        timestamp delivered_at
    }
    shipments {
        bigint id PK
        bigint order_id FK "Third-Party Route"
        varchar courier_name "e.g., J&T Express"
        varchar tracking_number
        varchar label_url "PDF URL"
        enum status "label_created, in_transit, delivered"
        timestamp delivered_at
    }

    %% RELATIONSHIPS DEFINITION
    users ||--o{ sso_identities : "logs_in_via"
    users ||--o{ user_addresses : "has"
    users ||--o| shops : "manages"
    users ||--o{ wishlists : "saves"
    users ||--o{ payment_methods : "saves"
    users ||--o{ audit_logs : "performs_action"
    users ||--o{ reviews : "writes"
    users ||--o| carts : "owns"
    users ||--o{ checkouts : "initiates"
    users ||--o{ deliveries : "drives_for"
    
    roles ||--o{ role_has_permissions : "grants"
    permissions ||--o{ role_has_permissions : "assigned_to"
    roles ||--o{ model_has_roles : "assigned_to"
    users ||--o{ model_has_roles : "has_role"
    permissions ||--o{ model_has_permissions : "assigned_to"
    users ||--o{ model_has_permissions : "has_direct_permission"

    shops ||--o{ products : "sells"
    shops ||--o{ orders : "fulfills"

    categories ||--o{ categories : "parent_of"
    categories ||--o{ products : "contains"
    
    products ||--o{ product_variants : "has"
    products ||--o{ product_images : "has"
    products ||--o{ wishlists : "saved_in"
    products ||--o{ reviews : "receives"
    
    carts ||--o{ cart_items : "contains"
    product_variants ||--o{ cart_items : "added_to"
    
    checkouts ||--o{ orders : "splits_into"
    checkouts ||--o| payments : "paid_via"
    payment_methods ||--o{ payments : "used_for"
    
    orders ||--o{ order_items : "contains"
    orders ||--o| returns : "has_returns"
    orders ||--o| order_coupons : "uses"
    orders ||--o| deliveries : "dispatched_internally"
    orders ||--o| shipments : "dispatched_externally"
    
    product_variants ||--o{ order_items : "fulfilled_as"
    shipping_methods ||--o{ orders : "used_by"
    coupons ||--o{ order_coupons : "applied_to"
```
