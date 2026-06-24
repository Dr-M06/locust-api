# Mobile token purchases (RevenueCat)

Consumer **web** buys tokens via Stripe / Flutterwave (`POST /api/v1/tokens/checkout`).

Consumer **mobile** (React Native) must use **App Store / Google Play** through [RevenueCat](https://www.revenuecat.com/) — not Stripe checkout URLs (Apple/Google policy).

## Architecture

```text
Mobile app                    RevenueCat              Drift API
──────────                    ──────────              ─────────
Purchases.configure({ appUserID: drift_user_uuid })
Purchases.purchasePackage() ──► Store billing ──► Webhook POST /webhooks/revenuecat
                                                      └─► credit token_balance
```

1. User signs in → JWT `sub` = Drift `users.id`.
2. On app launch: `Purchases.logIn(driftUserId)` (same UUID).
3. Fetch catalog: `GET /api/v1/tokens/mobile-config` (public RC SDK keys + packs with `store_product_id`).
4. Map RevenueCat **Offerings** to those product ids.
5. After purchase, RC sends webhook → API credits tokens (idempotent per transaction).

**Do not** call `POST /tokens/checkout` from mobile.

## Store product ids

Configured in `internal/payments/provider.go` (`StoreProductID` on each pack):

| Pack | Tokens | Store product id |
|------|--------|------------------|
| pack_50 | 50 | `com.driftin.live.tokens.pack_50` |
| pack_150 | 150 | `com.driftin.live.tokens.pack_150` |
| pack_500 | 500 | `com.driftin.live.tokens.pack_500` |
| pack_1200 | 1200 | `com.driftin.live.tokens.pack_1200` |

Create matching **consumable** IAP products in App Store Connect and Google Play Console, then attach them to RevenueCat products / offerings.

## API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/tokens/packs` | — | Pack list (includes `store_product_id`; web ignores extra field) |
| GET | `/api/v1/tokens/mobile-config` | — | `{ provider, revenuecat_api_key_apple, revenuecat_api_key_google, packs }` |
| POST | `/webhooks/revenuecat` | Bearer `REVENUECAT_WEBHOOK_AUTH` | Credits tokens on `INITIAL_PURCHASE` / `NON_RENEWING_PURCHASE`; clawback on `REFUND` |

## Environment

```env
REVENUECAT_WEBHOOK_AUTH=...        # RevenueCat dashboard → Integrations → Webhooks → Authorization header
REVENUECAT_APPLE_API_KEY=appl_...  # Public SDK key (iOS)
REVENUECAT_GOOGLE_API_KEY=goog_... # Public SDK key (Android)
```

Webhook URL: `https://api.driftin.live/webhooks/revenuecat`

## React Native (Expo) sketch

```bash
npx expo install react-native-purchases
```

```typescript
import Purchases from 'react-native-purchases'
import { Platform } from 'react-native'

// After auth:
const cfg = await fetch(`${API}/tokens/mobile-config`).then(r => r.json())
await Purchases.configure({
  apiKey: Platform.OS === 'ios' ? cfg.revenuecat_api_key_apple : cfg.revenuecat_api_key_google,
  appUserID: user.id,
})

const offerings = await Purchases.getOfferings()
// Match offering packages to cfg.packs by store_product_id
const { customerInfo } = await Purchases.purchasePackage(selectedPackage)
// Tokens arrive when webhook fires — refresh GET /users/me or poll balance
```

VIP private rooms still spend **in-app tokens** (same as web) after balance updates.

## Refunds

RevenueCat `REFUND` events run through the same clawback path as Stripe disputes (`payment_disputes` audit + balance debit).
