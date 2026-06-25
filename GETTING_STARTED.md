# Getting started

Ship your first **Niilox API** call in about 15 minutes.

## 1. Create your tenant

1. Open https://admin.driftin.live  
2. Sign in (Google, email, or magic link).  
3. **Create account** → choose a tenant id (e.g. `myapp`) and company name.  
4. You land on the developer dashboard with `X-App-ID: myapp`.

Your tenant is an isolated slice of the platform (users, rooms or gigs, keys, payments).

Existing first-party tenants: `drift` (live video), `geogig` (local gigs), `rodent` (P2P sessions). See [README.md](./README.md#first-party-tenants).

## 2. Create an API key

1. Go to **API keys** in the portal.  
2. **Create key** → copy the secret once (`drift_sk_…`).  
3. Store it in your server env or Bubble **Backend workflow** secrets — never in a public page.

## 3. Verify connectivity

```bash
curl -s https://api.driftin.live/health
# {"ok":true}

curl -s https://api.driftin.live/api/v1/platform/ping \
  -H "Authorization: Bearer drift_sk_YOUR_KEY" \
  -H "X-App-ID: myapp"
```

Use your portal **app id** exactly for `X-App-ID` (no extra spaces). The API key must belong to that same tenant.

Expected ping response:

```json
{"ok":true,"app_id":"myapp","auth":"api_key"}
```

## 4. Sign in a test user

For a quick test user session (no UI):

```bash
# Guest (watch-only, 24h)
curl -s https://api.driftin.live/api/v1/auth/guest \
  -H "X-App-ID: myapp" \
  -H "Content-Type: application/json" \
  -d "{}"
```

Save `token` from the response. For production sign-in use magic link, password, Google, Apple, or phone OTP — see [Native auth](./NATIVE_AUTH.md).

**GeoGig example (phone OTP):**

```bash
curl -s https://api.driftin.live/api/v1/auth/phone/send \
  -H "X-App-ID: geogig" \
  -H "Content-Type: application/json" \
  -d '{"phone":"+2348012345678"}'
```

## 5. First product call

**Live video tenant:**

```bash
curl -s "https://api.driftin.live/api/v1/rooms" \
  -H "Authorization: Bearer USER_JWT" \
  -H "X-App-ID: myapp"
```

**Gig marketplace (GeoGig):**

```bash
curl -s "https://api.driftin.live/api/v1/gigs" \
  -H "X-App-ID: geogig"
```

See [GeoGig guide](./GEOGIG.md) for the full gig lifecycle.

## 6. Configure payments (optional)

In the portal → **Billing**:

- Set **host / platform split** (not hardcoded 70/30).  
- Choose **hybrid**, **tokens only**, or **fiat only**.  
- Add **custom token packs** or use defaults.  
- Add **Stripe Connect** `acct_…` so card revenue goes to your account (Niilox takes the platform share as a fee).

Details: [Tenant business](./TENANT_BUSINESS.md).

## 7. Realtime (web / native)

Connect WebSockets after the user has a JWT:

- Room: `wss://api.driftin.live/ws/rooms/{roomID}?token=JWT`  
- Personal: `wss://api.driftin.live/ws/me?token=JWT`

Event list: [Developer integration](./DEVELOPER_INTEGRATION.md#5-realtime-events-you-should-handle-first). GeoGig gig events (`gig:checked_in`, etc.) also arrive on `/ws/me`.

## 8. No-code on Bubble?

Follow [**Bubble.io integration**](./BUBBLE_IO.md) — API Connector setup, auth workflows, and safe key storage.

## Next

- Full wiring checklist: [WIRING_CHECKLIST.md](./WIRING_CHECKLIST.md)  
- GeoGig reference app: [GEOGIG.md](./GEOGIG.md)  
- Endpoint reference: [API.md](./API.md)  
- TypeScript SDK (monorepo): [packages/niilox-sdk/README.md](../../packages/niilox-sdk/README.md)
