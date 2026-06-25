# Payments

Tokens are the unit of in-app spend (gifts, VIP access, and other paid features). They are bought via a payment provider and credited after a verified webhook.

> **Public docs note:** Step-by-step transaction flows, paywall semantics, and revenue-split implementation are withheld from this repository. Registered developers receive the full integration guide — contact **dev@niilox.com**.

## Mobile (RevenueCat)

Native apps use **App Store / Google Play** via RevenueCat — not Stripe/Flutterwave checkout. See [MOBILE_PAYMENTS.md](./MOBILE_PAYMENTS.md).

## Provider interface

Niilox supports pluggable payment providers (Stripe, Flutterwave, and extensible others). Each provider implements checkout creation and webhook verification against a normalized success event.

Implementations live in `internal/payments/`. They are registered in `payments.NewRegistry(cfg)`.

## Token packs

Per-tenant token pack catalog is configurable in the developer portal. Packs are listed via the tokens API and purchased through hosted checkout.

## In-app commerce

Niilox supports token spend for gifts, capped VIP events, and host payouts. Revenue split and seat capacity are enforced server-side per tenant.

Full endpoint reference, paywall responses, and webhook handling are provided to registered developers.

## Checkout flow (overview)

1. Client requests checkout for a pack id.
2. API creates a pending purchase row and returns a hosted checkout URL.
3. User completes payment on the provider page.
4. Provider webhook confirms the purchase; tokens are credited idempotently.

## Stripe configuration

Register a webhook for completed checkout sessions on your API host. Copy the signing secret to your server environment.

Per-tenant Stripe Connect accounts are configured in the developer portal (`acct_…`).

## Flutterwave configuration

Flutterwave Standard hosted checkout. Set public/secret keys and webhook hash in server environment.

Webhook URL: `POST /api/v1/webhooks/payments/flutterwave` on your API host.

## Choosing the provider per tenant

`apps.payment_provider` selects Stripe or Flutterwave per tenant. Update via SQL or the ops console.

## Adding a new provider

Implement the `Provider` interface, register in `payments.NewRegistry`, add env vars, and set `payment_provider` on the tenant row.

## Withdrawals

Withdrawals are a request queue — users withdraw earned tokens; an admin or back-office process pays them out off-platform.

Contact **dev@niilox.com** for withdrawal rules and admin tooling.

## Common questions

**Q: What happens if a checkout completes but the webhook never arrives?**  Providers retry webhooks for several days. The purchase row stays pending until the webhook lands.

**Q: Can I let users buy from multiple providers in the same tenant?**  Today one provider per tenant; multi-provider support is a small schema extension.

**Q: How do refunds work?**  Manual refund via your admin tool; mark the purchase refunded and adjust balance accordingly.
