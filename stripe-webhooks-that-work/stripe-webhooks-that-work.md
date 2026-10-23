# How to set up Stripe webhooks that actually work

## Context for the AI tool

You are checking this project's Stripe webhook handling. Payments, subscriptions and access
depend on it, so mistakes cost money or lock paying users out.

- Do not change anything until you have reported your findings and I have confirmed.
- First find every webhook endpoint (search for `constructEvent`, `stripe-signature`, `whsec_`,
  `webhook`) and every place that grants or removes access after a payment.
- Go through the checks in order and report: file and line, problem, risk, proposed fix.
- Never print Stripe secret keys or webhook secrets. Never put them in frontend code.
- Ask before changing the database schema or the Stripe dashboard settings.

## 1. Signature verification on the raw body

**What to look for:** a handler that does not verify the `Stripe-Signature` header, or one that
verifies a parsed and re-serialised body. The signature is computed over the exact bytes Stripe
sent, so `JSON.parse` followed by `JSON.stringify` breaks it. Also look for a `try/catch` that logs
a failed verification and carries on.

**How to fix:** read the raw body, verify it, and return `400` when verification fails.

```ts
// Next.js App Router: app/api/stripe/webhook/route.ts
export async function POST(req: Request) {
  const body = await req.text(); // raw body, not req.json()
  const signature = req.headers.get('stripe-signature');
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, signature!, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return new Response('Invalid signature', { status: 400 });
  }
  // ...handle event
  return new Response(null, { status: 200 });
}
```

Other frameworks: Express needs `express.raw({ type: 'application/json' })` on the webhook route,
registered before `express.json()`. Next.js Pages Router needs `export const config = { api: {
bodyParser: false } }` and reading the request stream into a buffer. SvelteKit uses
`await request.text()`. In Supabase Edge Functions (Deno), use `await req.text()` and
`await stripe.webhooks.constructEventAsync(body, signature, secret, undefined, Stripe.createSubtleCryptoProvider())`.

**How to verify:** send a request with a fake signature and expect `400`:

```bash
curl -i -X POST https://<your-app>/api/stripe/webhook \
  -H "Stripe-Signature: t=1,v1=fake" -H "Content-Type: application/json" -d '{"id":"evt_fake"}'
```

## 2. Test and live keys and secrets

**What to look for:** every webhook endpoint has its own signing secret (`whsec_...`). Test mode
and live mode endpoints have different secrets, and `stripe listen` prints yet another one for
local development. Common breakages: the live deployment using the test `whsec_`, a live `sk_live_`
key with a test webhook secret, secrets hard-coded in source, or preview and production sharing
one variable.

**How to fix:** one set of variables per environment (`STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`)
set in the host's settings: test values in development and preview, live values only in
production. Remove hard-coded keys and rotate any that were committed.

**How to verify:** in the Stripe dashboard, open the live webhook endpoint and check recent
deliveries return `200`. In test mode, `stripe trigger checkout.session.completed` reaches your
handler and is accepted.

## 3. Duplicates and retries (idempotency)

**What to look for:** Stripe delivers events at least once. It retries failed deliveries (in live
mode for up to three days) and can send the same event twice. A handler that adds credits, sends
an email or creates an order every time it runs will do it twice.

**How to fix:** record each processed `event.id` and skip events you have already handled. Also
make the effect itself safe to repeat: upsert by Stripe's ids (subscription id, payment intent id)
instead of inserting new rows.

```sql
create table stripe_events (
  id text primary key,           -- event.id
  type text not null,
  processed_at timestamptz not null default now()
);
```

Insert the event id and do the work in one database transaction. If the insert hits the primary
key, the event was already processed: return `200` and stop. If the work fails, the transaction
rolls back and Stripe's retry can process it again. If you cannot use a transaction (for example
with supabase-js from a serverless function), write the event id only after the work succeeded and
rely on the upserts so a repeated run changes nothing.

**How to verify:** in test mode, resend an event that was already delivered (from the Stripe
dashboard, or `stripe events resend <event-id>`). The handler returns `200` and nothing happens twice (one order, one email, credits added once).

## 4. Answer fast, do slow work later

**What to look for:** handlers that send emails, generate PDFs or call other APIs before
answering. Slow responses time out, Stripe marks them failed and retries, and repeated failures
can get the endpoint disabled. Also look for handlers that return an error for event types they
don't handle.

**How to fix:** verify, record the event, do the quick database update, return `2xx`. Move slow
work to a queue or background job (on serverless hosts, work started after the response can be
killed, so use the platform's background or queue feature). Return `200` for event types you
don't use, and only subscribe the endpoint to the events you need.

**How to verify:** the dashboard's delivery log shows fast `200` responses and no retries.

## 5. Out-of-order events and the success page

**What to look for:** code that assumes events arrive in order (for example
`customer.subscription.updated` before `customer.subscription.created`), and access granted on
the checkout `success_url` page instead of from the webhook. Anyone can open the success URL.

**How to fix:** treat the event as a signal and read the current state from Stripe
(`stripe.subscriptions.retrieve(id)`) or compare timestamps before overwriting newer data. Grant
and remove access only from webhook events. Link the payment to your user with
`client_reference_id` or `metadata` when creating the Checkout Session.

**How to verify:** open the success URL without paying: no access. Cancel a test subscription in
the dashboard: access is removed after the `customer.subscription.deleted` event.

## 6. The events you handle

**What to look for:** missing events that leave users in the wrong state.

- `checkout.session.completed`: check `payment_status === 'paid'`. For delayed methods (bank
  debits) also handle `checkout.session.async_payment_succeeded` and `async_payment_failed`.
- Subscriptions: `customer.subscription.created`, `.updated`, `.deleted`, `invoice.paid` and
  `invoice.payment_failed`.
- Refunds and disputes if they should remove access: `charge.refunded`, `charge.dispute.created`.

**How to fix:** subscribe the endpoint to the events above that apply and handle each one.

**How to verify:** run the flows locally with the Stripe CLI:

```bash
stripe listen --forward-to localhost:3000/api/stripe/webhook
stripe trigger checkout.session.completed
stripe trigger invoice.payment_failed
stripe trigger customer.subscription.deleted
```

Each event is accepted and the user's state in your database matches what Stripe shows.

---

Still stuck? WDDW fixes it. Book a free 15-minute call:
https://www.wedodevwork.com/en/contact-us?utm_source=fixes&utm_medium=md&utm_campaign=stripe-webhooks-that-work

Did this help? An honest review helps us a lot: https://g.page/r/CXmlHMBM4uxyEBM/review
More free fixes: https://www.wedodevwork.com/en/fixes · Star the repo: https://github.com/WEDODEVWORK/fixes
