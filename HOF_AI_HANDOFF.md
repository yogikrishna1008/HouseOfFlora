# House of Flora — AI Handoff Pack

Paste **Section 1** into any new ChatGPT / Claude Code / Cursor session before asking for work.
Then use the task prompt from **Section 3** for whatever you are building that day.

The point of this document: AI tools do not fail on this project because the code is hard. They fail because they re-decide things you have already decided. Without the constraints below, you will get a checkout that calculates prices in the browser, a stock column that gets decremented with no locking, and a webhook handler that processes duplicates. All three cost real money.

---

# SECTION 1 — Paste This First

```
You are working on House of Flora, an e-commerce store for an Indian
womenswear label. I am the founder and sole developer. Read all of this
before writing any code.

## STACK — already decided, do not propose alternatives
- Frontend: vanilla HTML/CSS/JS in modular files. NO React, Vue, Next.js,
  Tailwind, or any build step. This is deliberate.
- Backend: FastAPI (Python 3.11+), SQLAlchemy ORM, Alembic migrations
- Database: PostgreSQL
- Hosting: Cloudflare Pages (frontend), Render (backend)
- Images: Cloudflare R2
- Payments: Razorpay
- Shipping: Shiprocket
- Market: India only. Currency INR. GST applies.

## NON-NEGOTIABLE RULES
1. SERVER-SIDE PRICE AUTHORITY. The browser sends product_id, size,
   colour and quantity ONLY. It never sends a price, subtotal or total.
   The server recalculates every amount from the database. Assume the
   cart JSON has been edited by the customer.

2. STOCK RESERVATION, NOT DECREMENT. Variants have both
   stock_quantity and reserved_quantity. Available stock is
   (stock_quantity - reserved_quantity). Reserve at checkout start
   using SELECT ... FOR UPDATE inside a transaction, with a 15-minute
   expiry. Only commit the decrement after payment confirms.
   Never decrement stock at "add to cart".

3. IDEMPOTENT WEBHOOKS. Razorpay retries and sends duplicates. Every
   webhook checks a webhook_events table with a UNIQUE constraint on
   event_id before processing. The webhook is the source of truth for
   fulfilment, not the browser callback — a customer can pay and then
   close the tab.

4. ORDER ITEMS ARE IMMUTABLE SNAPSHOTS. order_items copies
   product_name, sku, size, colour and unit_price as literal values.
   It does NOT join to products for display. Old invoices must survive
   product renames and price changes.

5. NEVER log or expose full addresses, phone numbers or emails in
   error output.

## STYLE RULES
- Vanilla JS: const/let, no var. Event delegation over per-element
  listeners. No jQuery.
- Python: type hints on all function signatures. Pydantic schemas for
  every request and response. Never build SQL with f-strings.
- All secrets from environment variables via Pydantic Settings.
  The Razorpay key_secret NEVER reaches the browser — only key_id does.
- CSS: use the existing custom properties in :root. Do not introduce
  a new colour outside that palette.

## HOW TO RESPOND
- If my request conflicts with a rule above, say so and explain why
  before writing anything.
- If you need a file you have not seen, ask for it. Do not guess at
  its contents or invent function names.
- Give me complete working files, not fragments with "... rest of code
  here". If a file is too long, tell me and we will split the task.
- Do not add features I did not ask for.
- Flag anything that needs a decision from me rather than silently
  picking a default.
```

---

# SECTION 2 — Files to Attach

Attach these to the session depending on what you are working on.

| Working on | Attach |
|---|---|
| Anything at all | `HOUSE_OF_FLORA_ARCHITECTURE.md` |
| Frontend changes | `index.html` |
| Database / models | Architecture doc §4 |
| Checkout or payments | Architecture doc §6, §7, §8 |
| Planning / scheduling | `HOF_30_DAY_SPRINT.md` |

**Context-window warning:** `index.html` is ~78KB. Attaching it plus the full architecture doc will eat a large chunk of most context windows and degrade output quality. For backend work, attach only the relevant architecture sections, not the HTML.

---

# SECTION 3 — Task Prompts

Copy the one you need. Each assumes Section 1 is already in the session.

### Database models

```
Build the SQLAlchemy models for House of Flora using the schema in
Section 4 of the attached architecture doc.

Requirements:
- One file per domain: models/catalogue.py, models/orders.py,
  models/support.py
- Type hints throughout, SQLAlchemy 2.0 declarative style
- product_variants needs the UNIQUE(product_id, size, colour)
  constraint and both stock_quantity and reserved_quantity
- order_items stores product_name, sku, size, colour and unit_price
  as literal columns, not relationships
- Include the Alembic migration

Do not add tables that are not in the schema.
```

### Products endpoint

```
Build GET /api/products.

It must accept exactly these query parameters, because the existing
frontend already sends them:

  q, category, min, max, size (repeatable), colour (repeatable),
  sort, page, limit

sort accepts: featured, new, price-asc, price-desc, name

Rules:
- size and colour are repeatable and behave as OR within themselves,
  AND across the two
- q searches name, fabric, subtitle and colour names
- Return available_stock as (stock_quantity - reserved_quantity),
  never the raw stock_quantity
- Response shape: { items: [...], total: int, page: int, pages: int }

Include the Pydantic response schemas.
```

### Pricing service — the critical one

```
Build pricing_service.py.

This is the single most important function in the codebase. It is the
only place prices are calculated.

Signature:
  calculate_order_total(db, items, coupon_code, pincode) -> PriceBreakdown

where items is a list of (product_id, size, colour, quantity).

It must:
- Look up every price from product_variants / products in the DB.
  Never accept a price from the caller.
- Apply price_override where it is not NULL, otherwise base_price
- Validate the coupon: exists, active, in date, min_order_value met,
  usage_limit not exceeded
- Cap percentage discounts at max_discount
- Calculate shipping from shipping_zones by pincode
- Calculate GST by HSN code and price slab, splitting CGST/SGST for
  intra-state and IGST for inter-state
- Return a breakdown: subtotal, discount, shipping, tax, total

Raise a specific exception if any variant is inactive or does not exist.
Write pytest tests including a case where a caller tries to pass a price.
```

### Inventory service

```
Build inventory_service.py implementing the three-phase reservation
flow in Section 6 of the architecture doc.

Functions:
  reserve_stock(db, session_id, items) -> reservation_ids
  commit_reservation(db, session_id)
  release_reservation(db, session_id)
  sweep_expired(db)

reserve_stock must use SELECT ... FOR UPDATE inside a transaction so
two concurrent requests for the last unit cannot both succeed. On
insufficient stock, raise an exception carrying the actual available
quantity.

Write a pytest test that fires two concurrent reservations for a
variant with stock_quantity=1 and asserts exactly one succeeds.
```

### Order creation + Razorpay

```
Build POST /api/orders per Section 8 of the architecture doc.

Flow:
1. Validate the request body
2. Reserve stock via inventory_service
3. Calculate the total via pricing_service — never trust the client
4. Create or find the customer, store the address
5. Create the order with status=PENDING
6. Create the Razorpay order with the server-calculated amount
7. Return { order_number, razorpay_order_id, amount, key_id }

If any step fails, release the reservation before raising.
Order numbers follow HOF-{FY}-{sequence}, e.g. HOF-2627-0001.

Include the Pydantic request/response schemas.
```

### Webhook handler

```
Build POST /api/webhooks/razorpay.

Requirements:
- Verify the HMAC signature against the webhook secret BEFORE parsing
  the body. Reject unsigned requests with 400.
- Check webhook_events for the event_id. If already processed, return
  200 immediately without re-processing.
- Handle payment.captured and payment.failed
- On captured: order to PAID then CONFIRMED, commit the stock
  reservation, write an order_events row, send the confirmation email
- On failed: order to PAYMENT_FAILED, release the reservation
- Record the event_id after processing

This must be safe to call five times with the same payload and produce
the same result as calling it once. Write a test proving that.
```

### Connect the frontend

```
The attached index.html currently reads from a hardcoded PRODUCTS
constant. Change it to fetch from GET /api/products.

Constraints:
- Do NOT change the filter UI, the URL sync scheme, or the CSS.
  The query parameters already match the API contract.
- Move filtering and sorting server-side; the client sends params
  and renders what comes back
- Add a loading state and an error state to the grid
- Keep cart and wishlist in localStorage under the hof_ prefix
- Debounce search at 220ms as it currently is

Give me the complete modified <script> block.
```

---

# SECTION 4 — Things These Tools Get Wrong on This Project

Watch for these. They are the failures I expect, and each one is a rule from Section 1 being ignored.

| What it will do | Why it is wrong |
|---|---|
| Send `total` or `price` from the browser to the API | Customer edits it in DevTools and pays ₹1 |
| `UPDATE variants SET stock = stock - 1` with no lock | Two buyers, one unit, both succeed |
| Process webhooks without an idempotency check | Duplicate delivery ships the order twice |
| Join `order_items` to `products` for the invoice | Renaming a product rewrites last year's invoices |
| Suggest React "to make this more maintainable" | Adds a build step to a project deliberately without one |
| Put `key_secret` in frontend config | Anyone can create orders against your account |
| Store money as `float` | Use `NUMERIC(10,2)` / `Decimal`. Floats lose paise |
| Assume US addresses and sales tax | Six-digit pincode, GST by HSN slab, state-based CGST/SGST split |
| Return the raw `stock_quantity` in the API | Ignores reservations, oversells |
| Write `# ... rest of implementation` | You cannot ship an ellipsis |

---

# SECTION 5 — Which Tool for Which Job

Honest assessment, since you will be paying for at least one of these:

**ChatGPT / Claude in a chat window** — good for single files, schema design, debugging a pasted traceback, and talking through a decision. Bad at multi-file changes, because it cannot see your repo and will invent function names that do not exist.

**Claude Code / Cursor** — better for this project specifically, because they read the actual files. Multi-file refactors, running tests, and wiring the frontend to the API are where they earn their cost. You already used a project brief this way for jyogi.in, so the pattern is familiar.

**Where to keep this pack:** commit it to the repo as `docs/AI_HANDOFF.md`. In Claude Code, the Section 1 block belongs in `CLAUDE.md` at the repo root — it gets loaded automatically every session, so you stop pasting it.

---

# SECTION 6 — Review Before You Ship

Whatever generated it, check these by hand. Generated code is confident, and confidence is not correctness.

- [ ] Open DevTools, edit the cart to a ₹1 total, submit. **Order must be rejected.**
- [ ] Set a variant to stock 1. Fire two checkouts at once. **Exactly one succeeds.**
- [ ] Send the same webhook payload three times. **Order processed once.**
- [ ] Search the frontend bundle for `key_secret`. **Zero results.**
- [ ] Apply a coupon twice. **Discount applies once.**
- [ ] Abandon a checkout. Wait 15 minutes. **Reserved stock is released.**
- [ ] Check money columns are `NUMERIC`, not `FLOAT` or `REAL`.
- [ ] Confirm no invented imports or functions that do not exist.

The first three are the ones that cost money. Run them deliberately, not casually.
