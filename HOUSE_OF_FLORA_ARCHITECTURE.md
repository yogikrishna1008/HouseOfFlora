# House of Flora — E-Commerce Architecture

**Version:** 1.0
**Date:** 13 September 2026
**Status:** Draft for review
**Author:** Prepared for Jyogi Systems

---

## 0. Read This First

This is your first live e-commerce build. The difference between a storefront demo and a real store is not features — it is **money, stock, and trust**. Three things will hurt you if you get them wrong, and they are the reason this document is longer than a typical site spec:

1. **Price must never be trusted from the browser.** A customer can edit their cart in DevTools and pay ₹1 for a ₹9,000 lehenga. The server recalculates every rupee.
2. **Stock must be reserved, not just decremented.** Two customers buying the last piece simultaneously is not a rare edge case — it is what happens on every sale day.
3. **Payment webhooks fire more than once.** If your handler is not idempotent, you will ship the same order twice or mark it paid twice.

Everything below is built around those three constraints. Sections 6, 7 and 8 are the ones to read twice.

---

## 1. System Overview

```
┌──────────────────────────────────────────────────────────────┐
│  CLIENT (Browser)                                             │
│  houseofflora.com  →  Static frontend (Cloudflare Pages)      │
│  • Catalogue, filters, cart, wishlist (localStorage)          │
│  • Razorpay Checkout.js (client-side SDK)                     │
└───────────────┬──────────────────────────────────────────────┘
                │ HTTPS / JSON
┌───────────────▼──────────────────────────────────────────────┐
│  API (FastAPI on Render)                                      │
│  api.houseofflora.com                                         │
│  /products  /cart  /orders  /payments  /webhooks  /admin      │
└───────┬──────────────────┬──────────────────┬────────────────┘
        │                  │                  │
┌───────▼──────┐  ┌────────▼────────┐  ┌──────▼───────────────┐
│ PostgreSQL   │  │ Cloudflare R2   │  │ External Services    │
│ Orders,      │  │ Product images  │  │ Razorpay  (payments) │
│ stock,       │  │ + CDN delivery  │  │ Shiprocket (logistics)│
│ customers    │  │                 │  │ Resend    (email)    │
│              │  │                 │  │ WhatsApp  (support)  │
└──────────────┘  └─────────────────┘  └──────────────────────┘
```

**Why this shape:** it mirrors your jyogi.in stack (FastAPI on Render), so you reuse deployment knowledge, environment handling, and the same mental model. One genuinely new skill to absorb (PostgreSQL instead of SQLite), not five.

---

## 2. Technology Decisions

| Layer | Choice | Reasoning |
|---|---|---|
| Frontend | Vanilla HTML/CSS/JS, modular files | Your Riya build proves the pattern works. No build step, no framework churn, instant deploys, trivial to hand over. |
| Hosting (FE) | Cloudflare Pages | Free tier, global CDN with good India PoPs, auto-deploy from Git, free SSL. Measurably faster than GitHub Pages for Indian traffic. |
| Backend | FastAPI (Python 3.11+) | Same as jyogi.in. Async, Pydantic validation, auto-generated OpenAPI docs. |
| Hosting (BE) | Render (Starter, paid) | Familiar. **See cold-start warning below.** |
| Database | PostgreSQL (Render or Neon) | Concurrent checkout writes will corrupt SQLite. Non-negotiable. |
| Images | Cloudflare R2 | Zero egress fees, S3-compatible API. Dramatically cheaper than S3 for an image-heavy fashion catalogue. |
| Payments | Razorpay | UPI, cards, netbanking, wallets, COD. Already scaffolded in your Riya file. |
| Email | Resend (or Brevo) | Order confirmation, shipping updates, abandoned cart. |
| Shipping | Shiprocket | Best India coverage — aggregates Delhivery, BlueDart, Ekart, DTDC. |
| Admin panel | FastAPI + Jinja2 + HTMX | Same pattern as Jyogi Manager. Already built once. |
| Error tracking | Sentry (free tier) | You cannot debug a customer's failed checkout from logs alone. |

### ⚠️ Cold-start warning

Render's free tier sleeps after 15 minutes idle. A customer reaching checkout on a sleeping instance waits 30–50 seconds and abandons. **Budget the $7/month Starter plan from day one.** This is not an optimisation — it is the difference between a store and a broken store.

---

## 3. Repository Structure

```
house-of-flora/
├── frontend/
│   ├── index.html                  # Homepage
│   ├── shop.html                   # Catalogue + filters
│   ├── product.html                # PDP  (?slug=...)
│   ├── cart.html
│   ├── checkout.html
│   ├── order-confirmation.html
│   ├── track-order.html            # Lookup by order no. + phone
│   ├── pages/                      # about, size-guide, shipping,
│   │                               # returns, privacy, terms
│   ├── css/
│   │   ├── tokens.css              # Design system variables
│   │   ├── base.css
│   │   ├── components.css
│   │   └── pages.css
│   ├── js/
│   │   ├── config.js               # API base, Razorpay key, WA number
│   │   ├── api.js                  # Fetch wrapper + error handling
│   │   ├── state.js                # Cart, wishlist, persistence
│   │   ├── catalogue.js            # Filters, sort, URL sync ← PORT FROM RIYA
│   │   ├── product.js              # Gallery, variant selection
│   │   ├── cart.js
│   │   ├── checkout.js             # Razorpay flow
│   │   ├── whatsapp.js             # Message builders
│   │   └── ui.js                   # Toast, drawer, modal
│   └── assets/
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py               # Pydantic Settings from env
│   │   ├── database.py
│   │   ├── dependencies.py
│   │   ├── models/                 # SQLAlchemy ORM
│   │   ├── schemas/                # Pydantic request/response
│   │   ├── routers/
│   │   │   ├── products.py
│   │   │   ├── cart.py
│   │   │   ├── orders.py
│   │   │   ├── payments.py
│   │   │   ├── webhooks.py
│   │   │   ├── coupons.py
│   │   │   ├── shipping.py
│   │   │   └── admin.py
│   │   ├── services/
│   │   │   ├── pricing_service.py       # SERVER-SIDE PRICE AUTHORITY
│   │   │   ├── inventory_service.py     # Stock reservation
│   │   │   ├── razorpay_service.py
│   │   │   ├── email_service.py
│   │   │   ├── whatsapp_service.py
│   │   │   └── shiprocket_service.py
│   │   ├── admin/                  # Jinja2 + HTMX templates
│   │   └── utils/
│   ├── alembic/                    # DB migrations
│   ├── tests/
│   ├── requirements.txt
│   └── render.yaml
│
└── docs/
    ├── ARCHITECTURE.md             # This file
    ├── API.md
    └── RUNBOOK.md                  # What to do when things break
```

---

## 4. Data Model

### 4.1 Catalogue

```sql
categories
  id              SERIAL PK
  slug            VARCHAR UNIQUE NOT NULL
  name            VARCHAR NOT NULL
  description     TEXT
  image_url       VARCHAR
  parent_id       INT REFERENCES categories(id)   -- sub-categories
  sort_order      INT DEFAULT 0
  is_active       BOOLEAN DEFAULT TRUE
  created_at      TIMESTAMPTZ

products
  id              SERIAL PK
  slug            VARCHAR UNIQUE NOT NULL          -- URL + SEO
  name            VARCHAR NOT NULL
  subtitle        VARCHAR
  description     TEXT
  category_id     INT REFERENCES categories(id)
  fabric          VARCHAR
  care_instructions TEXT
  base_price      NUMERIC(10,2) NOT NULL
  compare_at_price NUMERIC(10,2)                   -- strikethrough price
  tag             VARCHAR                          -- New / Bestseller / Sale
  is_active       BOOLEAN DEFAULT TRUE
  is_featured     BOOLEAN DEFAULT FALSE
  meta_title      VARCHAR                          -- per-product SEO
  meta_description VARCHAR
  hsn_code        VARCHAR                          -- GST classification
  created_at, updated_at TIMESTAMPTZ
```

### 4.2 Variants — the most important table

```sql
product_variants
  id                SERIAL PK
  product_id        INT REFERENCES products(id)
  sku               VARCHAR UNIQUE NOT NULL
  size              VARCHAR NOT NULL
  colour            VARCHAR NOT NULL
  colour_hex        VARCHAR(7)
  price_override    NUMERIC(10,2)      -- NULL → use products.base_price
  stock_quantity    INT NOT NULL DEFAULT 0
  reserved_quantity INT NOT NULL DEFAULT 0   -- ← see §6
  weight_grams      INT                      -- shipping calculation
  is_active         BOOLEAN DEFAULT TRUE
  UNIQUE (product_id, size, colour)

product_images
  id              SERIAL PK
  product_id      INT REFERENCES products(id)
  variant_colour  VARCHAR            -- NULL = applies to all colours
  url             VARCHAR NOT NULL
  alt_text        VARCHAR            -- accessibility + SEO
  sort_order      INT DEFAULT 0
```

**Available stock = `stock_quantity - reserved_quantity`.** Never read `stock_quantity` alone when deciding whether something can be sold.

### 4.3 Orders

```sql
customers
  id, email, phone (UNIQUE), first_name, last_name,
  marketing_consent BOOLEAN, created_at

addresses
  id, customer_id FK, type (shipping|billing),
  line1, line2, city, state, pincode, country,
  is_default BOOLEAN

orders
  id                    SERIAL PK
  order_number          VARCHAR UNIQUE     -- "HOF-2627-0001"
  customer_id           INT FK
  status                VARCHAR            -- see §7 state machine
  payment_status        VARCHAR
  subtotal              NUMERIC(10,2)
  discount_amount       NUMERIC(10,2)
  shipping_amount       NUMERIC(10,2)
  tax_amount            NUMERIC(10,2)
  total_amount          NUMERIC(10,2)
  coupon_code           VARCHAR
  shipping_address_id   INT FK
  billing_address_id    INT FK
  razorpay_order_id     VARCHAR
  razorpay_payment_id   VARCHAR
  razorpay_signature    VARCHAR
  shiprocket_order_id   VARCHAR
  awb_number            VARCHAR
  courier_name          VARCHAR
  source                VARCHAR            -- web | whatsapp | manual
  notes                 TEXT
  created_at, updated_at

order_items                                -- IMMUTABLE SNAPSHOT
  id, order_id FK, variant_id FK,
  product_name  VARCHAR,   -- COPIED, not joined
  sku, size, colour VARCHAR,
  unit_price NUMERIC(10,2),
  quantity INT,
  line_total NUMERIC(10,2)

order_events                               -- AUDIT TRAIL
  id, order_id FK, event_type VARCHAR,
  payload_json JSONB, created_by VARCHAR, created_at
```

**Why `order_items` copies the product name and price instead of joining:** if you rename a product or change its price six months from now, old invoices must still show what the customer actually bought and paid. An order is a historical record, not a live view. This is the single most common beginner mistake in e-commerce schemas.

### 4.4 Supporting tables

```sql
coupons
  id, code UNIQUE, type (percent|fixed|free_shipping),
  value NUMERIC, min_order_value NUMERIC, max_discount NUMERIC,
  usage_limit INT, usage_count INT DEFAULT 0,
  per_customer_limit INT,
  valid_from, valid_until TIMESTAMPTZ,
  is_active BOOLEAN

coupon_redemptions
  id, coupon_id FK, order_id FK, customer_id FK, created_at

cart_reservations                          -- see §6
  id, session_id VARCHAR, variant_id FK,
  quantity INT, expires_at TIMESTAMPTZ

webhook_events                             -- see §8
  id, provider VARCHAR, event_id VARCHAR UNIQUE,
  payload JSONB, processed_at TIMESTAMPTZ, status VARCHAR

shipping_zones
  id, name, pincode_pattern, base_rate,
  per_kg_rate, free_above NUMERIC, estimated_days INT
```

---

## 5. API Surface

### Public — no auth

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/products` | List + filter + sort + paginate |
| GET | `/api/products/{slug}` | Single product with variants and images |
| GET | `/api/categories` | Category tree |
| POST | `/api/cart/validate` | **Recalculate cart server-side** |
| POST | `/api/coupons/validate` | Check coupon against cart |
| POST | `/api/shipping/estimate` | Rate by pincode + weight |
| POST | `/api/orders` | Create order (pending) + Razorpay order |
| POST | `/api/payments/verify` | Verify signature after checkout |
| GET | `/api/orders/track` | Lookup by order number + phone |
| POST | `/api/webhooks/razorpay` | Payment events (HMAC-verified) |
| POST | `/api/webhooks/shiprocket` | Shipment status |

### Admin — authenticated

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/admin/login` | JWT issue |
| CRUD | `/api/admin/products` | Catalogue management |
| CRUD | `/api/admin/variants` | Stock + SKU management |
| GET/PATCH | `/api/admin/orders` | Order list, status transitions |
| POST | `/api/admin/orders/{id}/ship` | Push to Shiprocket, store AWB |
| POST | `/api/admin/orders/{id}/refund` | Razorpay refund |
| CRUD | `/api/admin/coupons` | |
| GET | `/api/admin/reports/sales` | Revenue, top products, low stock |

### `GET /api/products` — query parameters

Mirror the Riya URL-sync scheme so your existing `catalogue.js` ports over almost unchanged:

```
?q=emerald&category=lehengas&min=5000&max=15000
&size=M&size=L&colour=Ivory&sort=price-asc&page=1&limit=24
```

---

## 6. Inventory & Stock Reservation ⚠️

**The problem:** one piece left. Two customers add it to cart at the same moment. Both reach Razorpay. Both pay. You now owe someone a refund and an apology.

**The solution — a three-phase lifecycle:**

```
Phase 1 — ADD TO CART
  No reservation. Cart is browsing, not commitment.
  Show live availability only.

Phase 2 — CHECKOUT INITIATED
  Reserve stock atomically, with a 15-minute expiry:

      BEGIN;
        SELECT stock_quantity, reserved_quantity
          FROM product_variants
         WHERE id = :vid
           FOR UPDATE;                          -- row lock

        -- verify (stock_quantity - reserved_quantity) >= qty
        UPDATE product_variants
           SET reserved_quantity = reserved_quantity + :qty
         WHERE id = :vid;

        INSERT INTO cart_reservations
          (session_id, variant_id, quantity, expires_at)
        VALUES (:sid, :vid, :qty, NOW() + INTERVAL '15 minutes');
      COMMIT;

  If the check fails → return 409 with what IS available.

Phase 3 — PAYMENT CONFIRMED
  stock_quantity    -= qty
  reserved_quantity -= qty
  DELETE the reservation row.

EXPIRY SWEEPER (background job, every 5 min)
  For each reservation past expires_at:
    reserved_quantity -= quantity
    DELETE reservation
```

`SELECT ... FOR UPDATE` is what makes this safe. It locks the row so the second concurrent request waits for the first to commit, then reads the updated number. Without it, both requests read "1 available" and both succeed.

**Oversell policy:** decide now, not during your first sale.
- *Strict* (recommended for launch): never allow oversell. Simpler, protects reputation.
- *Backorder*: allow, flag the item, communicate a longer dispatch window.

---

## 7. Order State Machine

```
                  ┌─────────┐
                  │ PENDING │  ← order created, awaiting payment
                  └────┬────┘
           ┌───────────┼───────────┐
           ▼           ▼           ▼
     ┌──────────┐ ┌────────┐ ┌───────────┐
     │PAYMENT_  │ │ PAID   │ │ EXPIRED   │ (no payment in 30 min)
     │FAILED    │ └───┬────┘ └───────────┘
     └────┬─────┘     │
          │           ▼
          │     ┌───────────┐
          │     │ CONFIRMED │  ← stock committed, email sent
          │     └─────┬─────┘
          │           ▼
          │     ┌───────────┐
          │     │ PACKED    │
          │     └─────┬─────┘
          │           ▼
          │     ┌───────────┐
          │     │ SHIPPED   │  ← AWB assigned
          │     └─────┬─────┘
          │           ▼
          │     ┌───────────┐      ┌──────────┐
          │     │ DELIVERED ├─────►│ RETURNED │
          │     └───────────┘      └────┬─────┘
          ▼                             ▼
     ┌───────────┐                ┌──────────┐
     │ CANCELLED │───────────────►│ REFUNDED │
     └───────────┘                └──────────┘
```

**Rules:**
- Every transition writes a row to `order_events`. No exceptions.
- Only `PENDING`, `PAID` and `CONFIRMED` are cancellable without a return flow.
- Cancelling before `SHIPPED` releases stock back.
- `PAID → CONFIRMED` is the only place stock is permanently decremented.

---

## 8. Payment Flow (Razorpay)

```
1. CLIENT   POST /api/orders  { items:[{variant_id, qty}], address, coupon }
                 ⚠️ Client sends IDs and quantities ONLY — never prices.

2. SERVER   • Reserve stock (§6)
            • pricing_service recalculates subtotal from DB
            • Validate coupon, compute discount
            • Compute shipping + GST
            • Create local order (status=PENDING)
            • razorpay.order.create(amount = server total)
            → return { order_number, razorpay_order_id, amount, key_id }

3. CLIENT   Open Razorpay Checkout with returned order_id
            Customer pays

4. CLIENT   POST /api/payments/verify
            { razorpay_order_id, razorpay_payment_id, razorpay_signature }

5. SERVER   • Verify HMAC-SHA256 signature with key_secret
            • If valid → PAID → CONFIRMED, commit stock, send email
            • If invalid → log, alert, do NOT fulfil

6. WEBHOOK  POST /api/webhooks/razorpay   (source of truth)
            • Verify webhook HMAC signature
            • Check webhook_events for event_id → skip if seen
            • Process, record event_id
```

### Three non-negotiable rules

1. **Never trust a client-supplied price.** The browser sends `variant_id` and `quantity`. The server derives every rupee. Assume the cart JSON has been edited.

2. **Webhooks must be idempotent.** Razorpay retries on non-2xx and can deliver duplicates. The `webhook_events` table with a `UNIQUE` constraint on `event_id` is your protection. Check before processing, record after.

3. **The webhook is the source of truth, not step 5.** A customer can close the browser after paying but before the verify call. If you only fulfil on step 5, that order silently never ships. Both paths must converge on the same idempotent handler.

---

## 9. Security

| Area | Measure |
|---|---|
| Secrets | Environment variables only. Razorpay `key_secret` never reaches the browser — only `key_id` does. |
| CORS | Whitelist `houseofflora.com` and `www.` explicitly. Never `*`. |
| Rate limiting | slowapi: 5/min on order creation, 10/min on coupon validation, 20/min on tracking lookup. |
| Input validation | Pydantic on every endpoint. Pincode regex, phone regex, quantity bounds. |
| SQL injection | SQLAlchemy ORM throughout. No f-string SQL, ever. |
| Admin auth | JWT, short expiry, bcrypt-hashed passwords, separate admin subdomain. |
| Webhooks | Verify HMAC signature before parsing the body. Reject unsigned requests. |
| PII | Never log full addresses or phone numbers. Mask in error reports. |
| HTTPS | Enforced everywhere. HSTS header. |
| Card data | Never touches your server. Razorpay's iframe handles it — this keeps you out of PCI-DSS scope entirely. |

---

## 10. India-Specific Requirements

These are legal and operational necessities, not nice-to-haves.

**GST.** Apparel is taxed by price slab — currently 5% below ₹1,000 and 12% above per piece. Store an `hsn_code` per product and compute tax server-side at order time. Invoices must show your GSTIN, the HSN code, and the CGST/SGST split for intra-state versus IGST for inter-state.

> **Decision needed:** Is House of Flora billing under Jyogi Studios' existing GSTIN, or is this a client project where the client holds the registration? This determines whose GSTIN goes on the invoice and who files the returns. Settle this before writing the invoicing module.

**COD.** A large share of Indian fashion orders are cash on delivery, with meaningfully higher return rates. If offering it: restrict by pincode serviceability, consider a cap on order value, and track COD-specific return rates separately.

**Address format.** Pincode is 6 digits and is the primary routing key. Validate against Shiprocket's serviceability API before accepting the order — not after.

**Returns policy.** Publish it before launch. Fashion return rates run high on fit. Your size guide is a returns-reduction tool, so treat it as a real page, not a placeholder link.

---

## 11. Build Phases

### Phase 1 — Catalogue Foundation (Week 1–2)
Database schema and migrations · Product/category CRUD in admin · Image upload to R2 · Public product endpoints · Port `catalogue.js` from the Riya build · Product detail page
**Milestone:** browsable, filterable catalogue with real products.

### Phase 2 — Cart & Checkout (Week 3–4)
Cart state and persistence · Server-side cart validation · Inventory reservation · Address capture and pincode validation · Shipping rate estimation · Coupon engine
**Milestone:** a customer can build a cart and reach a correctly-priced checkout.

### Phase 3 — Payments (Week 5)
Razorpay order creation · Signature verification · Idempotent webhook handler · Order confirmation email · Order tracking page
**Milestone:** money moves correctly. **Test failure paths as hard as success paths.**

### Phase 4 — Fulfilment (Week 6)
Admin order management · Shiprocket integration · AWB and tracking sync · Shipping notification emails · Refund and cancellation flow
**Milestone:** an order can go from paid to delivered without touching the database manually.

### Phase 5 — Polish & Launch (Week 7–8)
SEO: meta tags, `Product` structured data, sitemap, robots.txt · Analytics · Sentry · Performance: image optimisation, lazy loading · Legal pages · Load test on checkout · Soft launch to a small group
**Milestone:** public launch.

---

## 12. Pre-Launch Checklist

**Functional**
- [ ] Checkout completes on Chrome, Safari, and Android Chrome
- [ ] Payment failure leaves order `PAYMENT_FAILED` and releases stock
- [ ] Duplicate webhook delivery does not double-process
- [ ] Concurrent purchase of the last unit — exactly one succeeds
- [ ] Coupon cannot exceed its usage limit under concurrent use
- [ ] Reservation expiry sweeper returns stock correctly
- [ ] Cart price tampering is rejected server-side (test this deliberately)

**Operational**
- [ ] Razorpay account activated, settlement bank account verified
- [ ] Webhook URL registered and signature secret set
- [ ] Automated database backups enabled and a restore tested
- [ ] Sentry capturing errors
- [ ] Admin credentials stored in a password manager, not a file
- [ ] Render on a paid plan (no cold starts)

**Legal & Content**
- [ ] Terms, Privacy, Shipping, Returns, Refund pages live
- [ ] GST configuration verified against a real invoice
- [ ] Contact details and business address on site
- [ ] Size guide complete with actual measurements

**SEO**
- [ ] Unique title and meta description per product
- [ ] `Product` + `Organization` JSON-LD structured data
- [ ] `sitemap.xml` submitted to Search Console
- [ ] Open Graph tags (fashion gets shared on WhatsApp and Instagram constantly)
- [ ] All images have real `alt` text

---

## 13. Open Decisions

Settle these before Phase 2 begins — each one changes the schema or the flow:

1. **Entity and GST** — Jyogi Studios' GSTIN, or the client's?
2. **COD** — offer at launch, or card/UPI only initially?
3. **Guest checkout** — allowed, or require account creation? (Guest converts better; accounts retain better. Recommendation: guest checkout with optional post-purchase account creation.)
4. **Oversell policy** — strict, or backorder with communication?
5. **Free shipping threshold** — what order value, if any?
6. **Return window** — 7 days, 10 days, exchange-only?
7. **Catalogue size at launch** — affects whether pagination and search need to be robust from day one.

---

## 14. Cost Estimate (Monthly, INR)

| Service | Tier | Cost |
|---|---|---|
| Render (backend) | Starter | ~₹600 |
| PostgreSQL | Render/Neon starter | ~₹600 |
| Cloudflare Pages | Free | ₹0 |
| Cloudflare R2 | Under 10 GB | ~₹0–100 |
| Domain | Already purchased | — |
| Razorpay | 2% per transaction | Variable |
| Resend | Free to 3,000 emails | ₹0 |
| Shiprocket | Pay per shipment | Variable |
| Sentry | Free tier | ₹0 |
| **Fixed monthly** | | **≈ ₹1,200–1,500** |

Transaction and shipping costs scale with revenue, which is the correct shape for a new store.

---

## Appendix A — What Ports Directly from the Riya Build

| Riya component | Reuse | Change needed |
|---|---|---|
| Filter + sort engine | ✅ High | Read from API instead of a `PRODUCTS` constant |
| URL sync (`pushState`/`replaceState`) | ✅ Direct | None — the scheme already matches the API |
| Active filter chips | ✅ Direct | None |
| Dual-handle price slider | ✅ Direct | Bounds from API min/max |
| Mobile filter drawer | ✅ Direct | None |
| Product modal + gallery | ✅ High | Becomes a full PDP page for SEO |
| Cart drawer | ✅ Medium | Add server validation call |
| localStorage keys | ⚠️ | Rename `riya_*` → `hof_*` to avoid collision |
| WhatsApp message builders | ✅ High | New number, new brand copy |
| Razorpay block | ⚠️ Scaffold only | Needs the real backend flow in §8 |
| CSS design tokens | ❌ | New brand direction |
| Product data | ❌ | Real catalogue from DB |

The filter system — which is the genuinely hard part and already working — carries across almost untouched. That is a real head start.
