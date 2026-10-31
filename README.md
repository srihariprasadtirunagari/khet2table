# Khet2Table: Farmer Marketplace (HACK-P309)

A student-hackathon prototype where farmers list produce and buyers (households, retailers, restaurants, wholesalers) browse, order and track requests directly. No logistics network is needed.

## Contents
| Path | Purpose |
|---|---|
| `../farmer-marketplace.html` | Clickable front-end demo (all rules run in the browser, data in localStorage) |
| `rules.js` | Business rules: stock validation, order lifecycle, cancellation, duplicate detection |
| `server.js` | Zero-dependency REST API (Node 18+), data saved to `data.json` |
| `test/rules.test.js` | Automated tests (stock limits, cancellation, overselling, hidden listings) |

## Setup
```bash
npm test          # runs 9 tests, no install needed
npm start         # API on http://localhost:3000  (PORT env var optional)
```
Environment variables: `PORT` (default 3000). No secrets, no database server, no external services.

## Demo credentials (demo only, no passwords)
Send the `x-user` header: buyers `b1` (Meera, household), `b2` (Spice Route, restaurant); farmers `f1` to `f5`; admin `a1`. The HTML demo has a "Demo login" dropdown instead.

## API
| Method and path | Who | Action |
|---|---|---|
| `GET /api/products` | anyone | Visible listings |
| `POST /api/products` | farmer | Create listing (`name, cat, unit, price, qty, date, mode`) |
| `POST /api/products/:id/hide` | admin | Hide or restore a listing |
| `POST /api/orders` | buyer | `{items:{productId:qty}, fulfil:"pickup"\|"delivery"}`; splits per farmer |
| `GET /api/orders` | any | Own orders (all for admin) |
| `POST /api/orders/:id/advance` | farmer | requested → accepted → ready → completed |
| `POST /api/orders/:id/cancel` | buyer or farmer | Only while requested or accepted; stock returns |

## Business rules
- Stock is reserved when an order is requested and returned on cancellation, so inventory stays consistent.
- Orders above available stock are rejected; quantities must be whole numbers.
- Payment is simulated cash on pickup. The app never collects card details.
- Delivery is **not operational**. It is a demo option only, and farmers arrange it by phone.

## Architecture
Browser UI → REST API (`server.js`) → business rules (`rules.js`) → JSON file store. In production, swap the file for PostgreSQL, add real authentication and image upload, and keep `rules.js` unchanged behind the API.

## Known limitations
- No real authentication (header-based demo users) and no passwords.
- Photos are emoji placeholders; no image upload.
- JSON file storage is single-process and not safe for concurrent servers.
- The API server was not smoke-tested end to end here; only `rules.js` is covered by tests.
- No multilingual UI, bulk orders, price trends, or traceability yet.

## Future improvements
Real accounts and phone OTP, image upload, ratings, seasonal availability, bulk pricing, logistics partner integration, price-trend dashboard, multilingual (Telugu, Hindi) UI.
