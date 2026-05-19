# billing-service

> **Status:** 📋 Planned (Phase 5)  
> **Role:** Subscriptions, payments, invoices — Stripe integration

## Description

Maps teams (and individual users on solo plans) to Stripe customers. Manages subscription plans, payment methods, and billing lifecycle events. Listens to Stripe webhooks for asynchronous payment state changes.

## Tech Stack

| Concern | Technology |
|---|---|
| Framework | FastAPI |
| Database | PostgreSQL (own instance) |
| Cache | Redis (idempotency keys for Stripe calls) |
| Payments | `stripe-python` SDK |

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/billing/plans` | List available subscription plans |
| `GET` | `/billing/subscriptions/me` | Current subscription (team or user) |
| `POST` | `/billing/subscriptions` | Create or upgrade subscription |
| `DELETE` | `/billing/subscriptions/me` | Cancel subscription (at period end) |
| `GET` | `/billing/invoices` | Paginated billing history |
| `GET` | `/billing/invoices/{id}` | Single invoice details + PDF link |
| `POST` | `/billing/portal` | Create Stripe Customer Portal session URL |
| `POST` | `/billing/webhooks/stripe` | Stripe webhook receiver (public, HMAC gated) |

## Events Consumed (Redis Streams)

| Stream | Action |
|---|---|
| `team.deleted` | Cancel active subscriptions, archive Stripe customer |

## Events Published (Redis Streams)

| Stream | Trigger |
|---|---|
| `billing.subscription.activated` | Subscription created or upgraded |
| `billing.subscription.cancelled` | Subscription cancelled |
| `billing.payment.failed` | Stripe `invoice.payment_failed` webhook received |

## Stripe Webhook Events Handled

| Stripe event | Action |
|---|---|
| `customer.subscription.created` | Activate subscription record |
| `customer.subscription.updated` | Update plan/status |
| `customer.subscription.deleted` | Mark cancelled |
| `invoice.payment_succeeded` | Record invoice, publish activated event |
| `invoice.payment_failed` | Publish `billing.payment.failed` |

## Data Model (key fields)

```
billing_customers
├── id              UUID PK
├── entity_id       UUID (team_id or user_id)
├── entity_type     ENUM (team, user)
├── stripe_cust_id  TEXT UNIQUE
└── created_at      TIMESTAMPTZ

subscriptions
├── id              UUID PK
├── customer_id     UUID FK → billing_customers.id
├── stripe_sub_id   TEXT UNIQUE
├── plan_id         TEXT (Stripe price ID)
├── status          ENUM (active, cancelled, past_due, trialing)
├── current_period_start  TIMESTAMPTZ
└── current_period_end    TIMESTAMPTZ
```

## Security Notes

- Webhook endpoint validates `Stripe-Signature` HMAC before any processing
- Idempotency keys stored in Redis to prevent duplicate Stripe API calls
- Billing routes require active authenticated session
- Sensitive card data never touches this service — handled entirely by Stripe
