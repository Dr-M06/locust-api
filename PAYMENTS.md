# Payments

Tokens are the unit of in-app spend (gifts, future paid features). They are bought via a payment provider and credited atomically after a verified webhook.

## Mobile (RevenueCat)

Native apps use **App Store / Google Play** via RevenueCat — not Stripe/Flutterwave checkout. See [MOBILE_PAYMENTS.md](./MOBILE_PAYMENTS.md).

## Provider interface

```go
// internal/payments/provider.go
type Provider interface {
    Name() string
    CreateCheckout(ctx context.Context, pack TokenPack,
                   userID, purchaseID, successURL, cancelURL string) (string, error)
    VerifyWebhook(r *http.Request) (*WebhookEvent, error)
}
```

`WebhookEvent` is the normalized success event the API consumes:

```go
type WebhookEvent struct {
    PurchaseID string  // matches token_purchases.id
    UserID     string  // matches users.id
    Tokens     int     // tokens to credit
    ExternalID string  // provider-side reference (Stripe session id, Flutterwave tx_ref)
}
```

Implementations live in `internal/payments/stripe.go` and `internal/payments/flutterwave.go`. They are registered in `payments.NewRegistry(cfg)`.

## Token packs

Static catalog in `internal/payments/provider.go`:

| ID | Tokens | USD | Note |
|----|--------|------|------|
| `pack_50` | 50 | $0.99 | |
| `pack_150` | 150 | $2.49 | popular |
| `pack_500` | 500 | $6.99 | |
| `pack_1200` | 1200 | $14.99 | |

Returned by `GET /api/v1/tokens/packs`. The frontend converts USD ↔ tokens via the `TOKENS_PER_USD = 50` constant in `lib/tokenPricing.ts` (worst-pack rate — see the doc comment for why) when displaying VIP-room prices.

## Tokens-as-currency: spend paths

Tokens are the single in-app unit of spend. There are three independent flows that burn them, all sharing one balance column (`users.token_balance`) and one ledger-style transaction style:

| Flow | Where | Split | Server enforces |
|------|-------|-------|-----------------|
| **Gifts** | `POST /rooms/{id}/gifts` | host 70 % / platform 30 % | atomic `UPDATE users SET token_balance = token_balance - cost WHERE … >= cost` + `INSERT gift_transactions` |
| **VIP entry** | `POST /rooms/{id}/access` | host 70 % / platform 30 % | `SELECT … FOR UPDATE` on rooms row → seat-cap check → token burn + host credit → `INSERT room_access` (UNIQUE on `(room_id, user_id)`) |
| **Withdrawals** | `POST /users/me/withdrawals` | full debit, off-platform payout | capped at `earned_from_gifts - paid - pending` |

The 70/30 split is governed by **one** constant (`hostShareBps = 7000`, basis points) shared between the gift handler and the room-access handler so a future tuning change ripples through both without drift.

## VIP private rooms

Paid VIP rooms are **not** a separate fiat checkout per entry. Viewers spend **tokens they already hold** (bought earlier via Stripe / Flutterwave, or granted in dev). Entry is instant — no webhook wait on the door.

Migration: `migrations/012_private_rooms.sql` (not `010` — that file adds live thumbnails).

### Schema

**`rooms`** (host configures at go-live):

| Column | Type | Meaning |
|--------|------|---------|
| `is_private` | `BOOLEAN` | When `true`, `POST /rooms/{id}/join` requires a `room_access` row (host bypassed). |
| `entry_price_tokens` | `INT` | Per-viewer charge in tokens (`0` = free + tip-only tier). Capped at 10 000 server-side. |
| `seat_limit` | `INT` | Max paid viewers (`0` = unlimited). Capped at 500 server-side. |

**`room_access`** (one row per viewer per room, for the lifetime of that stream):

| Column | Type | Meaning |
|--------|------|---------|
| `id` | `UUID` | Primary key. |
| `app_id` | `TEXT` | Tenant scope (`REFERENCES apps`). |
| `room_id` | `UUID` | `REFERENCES rooms(id) ON DELETE CASCADE` — access ends when the host ends the room. |
| `user_id` | `UUID` | Viewer who paid. |
| `paid_tokens` | `INT` | Gross tokens burned from the viewer. |
| `host_earning` | `INT` | Net credited to the host (`paid_tokens × 70 %`). |
| `granted_at` | `TIMESTAMPTZ` | When access was granted. |
| | | `UNIQUE (room_id, user_id)` — idempotent re-pay; duplicate `POST /access` returns 200 without charging again. |

Indexes: `idx_room_access_room`, `idx_room_access_user`.

### Seat counting

**Source of truth: PostgreSQL**, not Redis.

- **Capacity check** — inside `BuyAccess`, after `SELECT … FOR UPDATE` on the `rooms` row: `SELECT COUNT(*) FROM room_access WHERE room_id = $1` compared to `seat_limit` (skipped when `seat_limit = 0`).
- **Lobby display** — `ListRooms` / `GetRoom` join `COUNT(*) FROM room_access` (or a lateral subquery) so cards show `seats_taken` without N+1 queries.
- **Race safety** — row lock on `rooms` + `UNIQUE (room_id, user_id)` prevents overselling and double-charges; two simultaneous payers cannot both pass a “1 seat left” check.

Redis (`REDIS_URL`) is only used to **fan out WebSocket events** across API replicas. It does **not** store or increment seat counters. If Redis is unset, single-node mode still enforces seats correctly via Postgres.

**Live UI** — after a successful `BuyAccess`, the hub broadcasts `lobby:seats` on every `/ws/me` connection:

```json
{ "room_id": "<uuid>", "seats_taken": 3, "seat_limit": 10 }
```

Lobby cards and open paywall sheets update without polling.

### 402 paywall response

When a viewer lacks access, these endpoints return **HTTP 402 Payment Required** with the same JSON body (`AccessInfo`):

- `POST /rooms/{id}/join` (no LiveKit token until paid)
- `POST /rooms/{id}/access` (insufficient balance mid-purchase)

```json
{
  "has_access": false,
  "is_private": true,
  "entry_price_tokens": 250,
  "seat_limit": 10,
  "seats_taken": 3,
  "balance": 120,
  "is_host": false,
  "error": "payment_required"
}
```

Hosts always receive `has_access: true` / `is_host: true` without paying. `GET /rooms/{id}/access` returns the same shape for the paywall UI (200 even when unpaid).

Other statuses: **409** `this vip room is full` (seat cap hit), **403** for guests / adult gates (separate from paywall).

### Token purchase → room entry flow

End-to-end path when a viewer taps a VIP card:

```
1. Lobby card shows 🔒, USD price, "N of M seats left"
        │
        ▼
2. Viewer taps → POST /rooms/{id}/join
        │
        ├─ has room_access row? ──yes──► 200 + livekit_token → enter stream
        │
        no
        ▼
3. 402 AccessInfo → frontend opens PaywallModal
        │
        ├─ balance >= entry_price_tokens?
        │       │
        │      yes → POST /rooms/{id}/access
        │              BEGIN
        │                SELECT rooms … FOR UPDATE
        │                COUNT room_access (seat cap)
        │                UPDATE users SET token_balance -= price (viewer)
        │                UPDATE users SET token_balance += host_earning (host, 70 %)
        │                INSERT room_access
        │              COMMIT
        │              FanoutLobbySeats → lobby:seats WS
        │              200 AccessInfo (has_access: true)
        │              → POST /rooms/{id}/join → LiveKit token
        │
        no (insufficient balance)
        ▼
4. PaywallModal → /tokens?return=/lobby/{category}
        │
        ▼
5. POST /tokens/checkout { pack_id, return: "/lobby/…" }
        → Stripe / Flutterwave hosted checkout
        │
        ▼
6. Webhook checkout.session.completed
        → token_purchases.status = completed
        → users.token_balance += pack tokens
        │
        ▼
7. Redirect /tokens?purchase=success&return=/lobby/…
        → frontend refreshes /users/me, reopens paywall (VIP_RESUME_KEY)
        │
        ▼
8. Back to step 3 — POST /rooms/{id}/access → join
```

**Re-entry without re-pay** — leaving the room (`POST /leave` or navigating away) does **not** delete `room_access`. The same `room_id` keeps the row until the host ends the stream (`rooms.ended_at` set / row cascades). `POST /join` mints a fresh LiveKit token on every return.

**Wallet history** — `GET /users/me/wallet` activity includes `vip_paid` (viewer) and `vip_earned` (host) rows sourced from `room_access`. Summary `spent_on_gifts` / `earned_from_gifts` include VIP totals (field names kept for wire-compat).

## Checkout flow

```
1. Client                ──POST /api/v1/tokens/checkout {"pack_id":"pack_150","return":"/lobby/all"}──►
2. handler.CreateCheckout
     a) Pick provider from app.PaymentProvider ('stripe' | 'flutterwave')
     b) INSERT INTO token_purchases (app_id, user_id, tokens, amount_usd, 'pending')
     c) provider.CreateCheckout(...) with success/cancel URLs appended with ?return=…
        (only when "return" starts with "/" — open-redirect guard)
3. Client                ◄──{"checkout_url":"https://...","purchase_id":"<uuid>"}
4. Client                ──redirect to checkout_url──►  provider hosted page
5. User pays
6. Provider              ──POST /api/v1/webhooks/stripe (or /payments/flutterwave)──►
7. handler.handleWebhook
     a) provider.VerifyWebhook(r) → *WebhookEvent or error
     b) BEGIN
        UPDATE token_purchases SET status='completed', stripe_session=$ext
          WHERE id=$pid AND status='pending'   -- idempotent
        UPDATE users SET token_balance = token_balance + $tokens WHERE id=$uid
        COMMIT
```

Replays are safe: the `WHERE status='pending'` guard makes the UPDATE a no-op the second time, so a webhook delivered twice (or out of order) credits exactly once.

## Stripe configuration

Per-tenant credentials are not (yet) stored in the `apps` table — the Stripe key is a single value at process scope (`STRIPE_KEY`, `STRIPE_WEBHOOK_SECRET`). If you need per-tenant Stripe accounts, move those two fields onto the `apps` row and update `payments.Registry` to look them up per request — small change.

Webhook URL to register in the Stripe dashboard:

```
https://api.yourdomain.com/api/v1/webhooks/stripe
```

Event filter: `checkout.session.completed`. Copy the signing secret to `STRIPE_WEBHOOK_SECRET`.

## Flutterwave configuration

Flutterwave Standard hosted checkout. Per-process credentials:

```
FLW_PUBLIC_KEY=...
FLW_SECRET_KEY=...
FLW_WEBHOOK_HASH=...
# Optional — omit to use methods enabled in the Flutterwave dashboard:
# FLW_PAYMENT_OPTIONS=card,ussd,banktransfer,account
```

**Payment methods on checkout**

- Flutterwave checkout uses **`FLW_CHECKOUT_CURRENCY=NGN`** by default (USD only shows card on the hosted page).
- Pack prices are converted with **`FLW_USD_TO_NGN_RATE`** (default `1600`). Tokens credited are still from the pack catalog; `token_purchases.amount_usd` stays in USD cents.
- **`FLW_PAYMENT_OPTIONS`** defaults to `card,ussd,banktransfer,account,opay`. Set `FLW_PAYMENT_OPTIONS=dashboard` to omit the field and use only Flutterwave dashboard settings.
- Ensure NGN and those methods are enabled on your Flutterwave account.

Webhook URL to register in the Flutterwave dashboard:

```
https://api.yourdomain.com/api/v1/webhooks/payments/flutterwave
```

`FLW_WEBHOOK_HASH` is the shared secret you set in Flutterwave; the handler accepts either the raw value or its SHA-512 hex digest in the `verif-hash` header.

## Choosing the provider per tenant

`apps.payment_provider` is a single text column:

```sql
UPDATE apps SET payment_provider = 'flutterwave' WHERE id = 'rabbaly';
```

`POST /api/v1/tokens/checkout` then routes to Flutterwave for callers sending `X-App-ID: rabbaly` and Stripe for `X-App-ID: drift`.

## Adding a new provider

1. Implement the interface:

   ```go
   // internal/payments/paystack.go
   package payments

   type PaystackProvider struct { secretKey, webhookSecret string }
   func (p *PaystackProvider) Name() string { return "paystack" }
   func (p *PaystackProvider) CreateCheckout(...) (string, error) { ... }
   func (p *PaystackProvider) VerifyWebhook(r *http.Request) (*WebhookEvent, error) { ... }
   ```

2. Register it in `payments.NewRegistry`:

   ```go
   r.paystack = NewPaystackProvider(cfg.PaystackKey, cfg.PaystackWebhookSecret)
   ```

3. Extend `Registry.For`:

   ```go
   case "paystack":
       if r.paystack == nil || r.paystack.secretKey == "" {
           return nil, fmt.Errorf("paystack not configured")
       }
       return r.paystack, nil
   ```

4. Add env vars to `config.Config` and `.env.example`.

5. Run `UPDATE apps SET payment_provider = 'paystack' WHERE id = '...'` for the tenants that should use it.

No other code change needed. The webhook URL `POST /api/v1/webhooks/payments/paystack` is automatically routed.

## Withdrawals

Withdrawals are NOT a payment-provider concern — they're a request queue. Users withdraw earned tokens; an admin or back-office process pays them out off-platform.

`POST /api/v1/users/me/withdrawals` with `{"tokens": N}` (N ≥ 100):

1. `BEGIN`
2. `SELECT token_balance FOR UPDATE`
3. Compute `withdrawable = earned_from_gifts - paid - pending`
4. If `tokens > withdrawable`: reject (`only gift earnings can be withdrawn`).
5. `UPDATE users SET token_balance = token_balance - $tokens WHERE token_balance >= $tokens`
6. `INSERT INTO withdrawal_requests (status='pending')`
7. `COMMIT`

When the off-platform payout completes, an admin tool updates `withdrawal_requests.status = 'paid'` (out of scope for this codebase — wire whatever ops dashboard you prefer).

## Common questions

**Q: What happens if a checkout completes but the webhook never arrives?**  Stripe and Flutterwave both retry webhooks for several days. The purchase row stays `pending` until the webhook lands. Add an admin reconcile job if you need belt-and-braces.

**Q: Can I let users buy from multiple providers in the same tenant?**  Today the column is single-valued. Move `payment_provider` to a JSON array (`['stripe','flutterwave']`) and accept a `provider` field in `POST /tokens/checkout`. The provider interface is already ready for it.

**Q: How do refunds work?**  The backend does not initiate refunds. Manual refund + `UPDATE token_purchases SET status = 'refunded'` + a negative balance adjust is the recommended flow. Build it into your admin tool.
