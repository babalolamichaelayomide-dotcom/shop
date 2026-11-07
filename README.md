# ROYSTORE (Next.js, built for Vercel)

Online store with separate Home, Shop, About Us, and Testimonials pages at `/`, `/shop`, `/about`, and `/testimonials`. It includes 5 products with 30 units of stock each, a cart with quantity updates, free delivery, Paystack payment, and a payment confirmation pop-up. Blue, clean design.

## Stack
- Next.js (App Router) with React, plain JavaScript, no TypeScript
- API routes in `app/api/*` (these become Vercel serverless functions)
- Orders and stock stored as JSON: a file locally, Upstash Redis on Vercel
- Paystack for payments

## Run locally
```
npm install
cp .env.example .env.local     # add your Paystack TEST keys
npm run dev                    # http://localhost:3000
npm test                       # 19 unit tests
```
Locally, orders and stock are saved to `data/store.json`. Delete that file to reset stock to 30 each.

## Deploy to Vercel
1. Push this folder to a GitHub repository.
2. In Vercel: **Add New > Project**, import the repository. Framework is detected as Next.js.
3. In the project, open **Storage** (or the **Marketplace**) and add **Upstash Redis**. It adds `UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN` automatically. Without it, the store refuses to start on Vercel (a local file would lose orders there).
4. In **Settings > Environment Variables**, add `PAYSTACK_PUBLIC_KEY` and `PAYSTACK_SECRET_KEY` (test keys first), plus optionally `DELIVERY_FEE=0` and `RESERVATION_MINUTES=15`.
5. Deploy. Make a test payment with a Paystack test card.
6. In the Paystack dashboard (Settings > API Keys & Webhooks), set the webhook URL to `https://YOUR-DOMAIN/api/webhook`.
7. When the owner approves, switch to the live keys and redeploy.

## How payment and stock work
1. Checkout calls `POST /api/orders`. The server recalculates the total from its own prices, **reserves** the items (atomically in Redis), and returns the amount and reference.
2. The Paystack popup opens with those server values.
3. On success the browser calls `POST /api/verify`. The server asks Paystack, checks status, currency and amount, then marks the order paid. The pop-up shows the result.
4. `POST /api/webhook` does the same on Paystack's `charge.success` event (signature checked), so orders are confirmed even if the customer closes the page.
5. If the customer closes the payment window, `POST /api/cancel` returns the reserved items. Unpaid reservations also expire after 15 minutes.
6. If payment arrives after a hold expired and the item has sold out, the order is marked `needs_attention` so you can refund or contact the customer.

## Edit content
- Products, prices, stock per product: `lib/catalog.js`
- About Us text and testimonials: `lib/content.js` (testimonials are samples until you replace them and set `TESTIMONIALS_ARE_SAMPLES = false`)
- Colours: CSS variables at the top of `app/globals.css`
- Contact details in the footer: `components/Store.js`

Checkout loads Paystack before reserving stock, so a failed payment-script load does not hold items from other customers.

## Reading orders
Orders are saved as JSON. On Vercel they are in Upstash Redis under keys `order:<reference>`; use the Upstash console data browser. Statuses: `pending`, `paid`, `expired`, `cancelled`, `needs_attention`. Paystack's dashboard also lists every payment.

## Stock note
Stock is changed only when an order is reserved, cancelled, expired or paid. To restock, set the `stock:<id>` keys in Upstash (or edit `data/store.json` locally).

## Decisions
- Next.js instead of plain HTML so it deploys on Vercel with no extra setup.
- Storage is JSON, but kept in Redis on Vercel because Vercel functions cannot keep files between requests. Redis also makes stock changes atomic, so the last unit cannot be sold twice.
- Delivery is free (`DELIVERY_FEE=0`); the setting remains in case it changes.
- Prices are placeholders and descriptions are lorem ipsum.
