# Farm Direct: Farmer Marketplace (HACK-P309)

A working prototype where farmers list produce and buyers (households, retailers, restaurants, wholesalers) browse, compare and order directly, with a clear order lifecycle, a farmer dashboard and admin moderation.

## Problem

Farmers struggle to reach buyers and to communicate what they have, how much, and at what price. Buyers can't easily compare availability across farms. Farm Direct makes listings transparent (price, unit, quantity, harvest date, pickup/delivery) and supports a simple request → accept → ready → complete flow, with no logistics network required.

## What's in the prototype

| Requirement | Where |
|---|---|
| Farmer / buyer / admin accounts | `/register`, `/login` (admin accounts are seeded only, never self-registered) |
| Product listing: name, category, photo, unit, quantity, price, harvest date, pickup/delivery | `/farmer/products/new` |
| Search + filters: product, price range, location, availability, sort | home page `/` |
| Cart and order request with quantity validation | `/cart`, `/checkout` |
| Order status: requested, accepted, ready, completed, cancelled | `marketplace/services.py::update_status` |
| Farmer dashboard (inventory, incoming orders, sales) | `/farmer/dashboard` |
| Buyer dashboard (order history, progress tracker, cancel) | `/orders` |
| Admin moderation (duplicates, flagged wording, remove/restore) | `/admin` |
| 5 farmer profiles, 16 sample products | `python seed.py` (plus one deliberately flagged duplicate) |
| Tests for stock limits and cancellation | `tests/` (40 tests) |

## Setup

Requires Python 3.10+.

```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python seed.py --reset      # creates marketplace.db with demo data
python app.py               # http://127.0.0.1:5000
python -m pytest            # run the tests
```

### Environment variables (all optional)

| Variable | Default | Purpose |
|---|---|---|
| `SECRET_KEY` | `dev-only-change-me` | Signs session cookies. **Set a real value anywhere beyond your laptop.** |
| `DATABASE` | `./marketplace.db` | SQLite file path (used by both the app and `seed.py`). |
| `FLASK_DEBUG` | off | Set to `1` for auto-reload while developing. |

### Demo credentials (demo only)

Password for **every** seeded account: `demo123`

| Role | Email | Notes |
|---|---|---|
| Farmer | `ravi@farm.demo` | Has new + in-progress orders and a listing under review |
| Farmer | `lakshmi@farm.demo`, `mohan@farm.demo`, `shabana@farm.demo`, `venkat@farm.demo` | |
| Buyer | `asha@buyer.demo` (household) | Has a requested and an accepted order |
| Buyer | `kitchen@buyer.demo` (restaurant), `store@buyer.demo` (retailer) | |
| Admin | `admin@farm.demo` | Moderation queue |

### Suggested demo walkthrough (about 3 minutes)

1. Browse `/` as a visitor: filter by price, location, "Coming soon".
2. Log in as `asha@buyer.demo`, add Okra, check out (cash on pickup).
3. Try ordering more than the stock: the app refuses and tells you what's left.
4. Log in as `ravi@farm.demo`: accept the order, mark ready, complete it, and watch Sales update.
5. Cancel another order and watch the stock come back on the inventory table.
6. Log in as `admin@farm.demo`: approve or remove the flagged duplicate tomato listing.

## Architecture

```
┌────────────────────────────┐
│ Browser (desktop / mobile) │   Server-rendered HTML + one CSS file, no JS framework
└─────────────┬──────────────┘
              │ HTTP (forms, session cookie)
┌─────────────▼──────────────┐
│ Flask app  (app.py)        │   Routing, sessions, role checks, file upload, templates (Jinja2)
│  • auth + role_required    │
│  • cart in session         │
└─────────────┬──────────────┘
              │ plain function calls
┌─────────────▼──────────────┐
│ Service layer              │   ALL business rules live here: stock, lifecycle, permissions,
│ marketplace/services.py    │   moderation, search, dashboard stats. Fully unit-tested.
└─────────────┬──────────────┘
              │ SQL (explicit BEGIN IMMEDIATE transactions)
┌─────────────▼──────────────┐
│ SQLite  (marketplace.db)   │   users · products · orders · order_items
└────────────────────────────┘

External services: none. Payments are simulated/cash, delivery is represented only, photos are stored on local disk.
```

**Why this shape.** Keeping every rule in `services.py` (no Flask imports) means the stock and cancellation logic is tested directly and can later sit behind a mobile app or JSON API without changes. SQLite keeps setup to zero for a hackathon; the schema is plain SQL and ports to PostgreSQL easily.

### Data model

- `users`: role (`farmer` / `buyer` / `admin`), location, phone, buyer type
- `products`: owned by a farmer; `quantity`, `available_from`, `fulfilment`, `status` (`active` / `pending_review` / `removed`)
- `orders`: one per farmer per checkout, status, fulfilment, payment method, total
- `order_items`: snapshot of name, unit and price at order time, so history is stable if a listing changes later

### Order lifecycle

```
requested ──► accepted ──► ready ──► completed
    │            │           │
    └────────────┴───────────┴──► cancelled   (stock is returned exactly once)
```

- Only the **farmer** moves an order forward, one step at a time (no skipping, no going back).
- A **buyer** may cancel while `requested` or `accepted`. After `ready`, only the farmer (or an admin) can cancel.
- `completed` and `cancelled` are final.

### How stock stays correct

- Stock is **reserved** when the order is requested and **returned** on cancellation.
- Reservation uses a conditional update (`... WHERE quantity >= ?`), inside a single `BEGIN IMMEDIATE` transaction for the whole checkout. If any line fails, everything rolls back, so there are no half-reserved carts.
- The database itself also enforces `quantity >= 0` as a last line of defence.
- Cart contents are re-validated at checkout, so a stale cart can't oversell.
- Duplicate lines for the same product are merged before the stock check.

### Safety and business rules implemented

- Prices, units, quantities and harvest dates shown on every card and detail page.
- Payments are cash-on-pickup or a clearly labelled simulation; **no card fields exist anywhere**.
- Delivery is labelled "demo only" on listings, cart and orders, with an explicit notice that no courier is connected.
- Passwords are hashed (Werkzeug); user content is auto-escaped; uploads are limited to PNG/JPG/WebP, 2 MB, with random filenames; login redirects only to local paths.
- Role checks on every protected route; farmers can edit only their own listings; buyers can touch only their own orders.

## Tests

`python -m pytest` runs 40 tests:

- **Stock limits:** exact-stock order, over-order rejected with no side effects, last-units race, merged duplicate lines, all-or-nothing multi-item orders, invalid quantities, future-dated produce, unsupported delivery.
- **Cancellation:** stock restored, no double restore on repeat cancel, farmer cancel restores every line, completed orders can't be cancelled, buyer can't cancel once ready, strangers can't cancel or advance.
- **Lifecycle, moderation, search, dashboard maths**, plus end-to-end web flows (role protection, escaping, upload validation, open-redirect guard, two-session stock race).

## Known limitations

- **No real delivery or logistics**; delivery is a represented option only.
- **No real payments**; simulated or cash only, by design.
- **Cart is session-based**: it's cleared on logout and isn't shared across devices.
- **No CSRF tokens** on forms (session cookies are `SameSite=Lax`, which mitigates but doesn't replace them). Add Flask-WTF before any public deployment.
- **No email/SMS notifications**; status changes are visible only inside the app.
- **No password reset or email verification.**
- **Moderation is basic**: exact name + unit duplicate match and a tiny keyword list. Real use needs fuzzy matching and a report button for buyers.
- **Reservations never expire**: a requested order holds stock until the farmer acts on it.
- **Single-node SQLite** and the Flask dev server; use PostgreSQL and a WSGI server (gunicorn) to scale.
- **English only**, INR only, quantities in a single unit per listing.
- Uploaded photos live on local disk, not object storage; image contents are not scanned.
- Order notes and phone numbers are visible to the other party in an order, which is intentional for coordination but should get consent text in production.

## Future improvements

Multilingual UI (Telugu/Hindi/English), seasonal availability calendar, bulk-order pricing, price-trend dashboard, SMS/WhatsApp order alerts, reservation expiry, ratings, traceability (farm plot, harvest batch), logistics-partner integration, map view of nearby farms, and a JSON API for a mobile app.

## Project structure

```
app.py                     Flask app factory and routes
seed.py                    demo data loader
marketplace/db.py          schema + connection helper
marketplace/services.py    business rules (no Flask dependency)
templates/  static/        Jinja2 pages and CSS (uploads/ holds photos)
tests/                     service-layer and end-to-end tests
```
