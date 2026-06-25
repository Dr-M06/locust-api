# Integration wiring checklist

> **Public docs note:** Step-by-step flows for VIP access, worker safety, geofence check-in, and payment webhooks are withheld from this repository. Contact **dev@niilox.com** for the full integration guide.

Use this before going to production. Check each box in order.

## Account & tenant

- [ ] Developer account at https://admin.driftin.live  
- [ ] Tenant `app_id` chosen (lowercase, e.g. `mybrand`)  
- [ ] `X-App-ID: <app_id>` documented for your team  

## Credentials

- [ ] At least one **API key** (`drift_sk_…`) created and stored server-side only  
- [ ] `/platform/ping` succeeds with key + `X-App-ID`  
- [ ] API key rotation process documented (create new → deploy → revoke old)  

## End-user auth

Pick one or more:

- [ ] Magic link  
- [ ] Google  
- [ ] Email + password  
- [ ] Guest mode for browse-only  

Storage:

- [ ] Access JWT stored securely (httpOnly cookie or secure storage on mobile)  
- [ ] Refresh token stored and refresh on 401  
- [ ] Logout revokes refresh tokens when applicable  

## Core product flows

- [ ] Profile after login  
- [ ] Rooms lobby and go-live  
- [ ] LiveKit join in your UI  
- [ ] Room chat  
- [ ] Gifts or fiat checkout (per your payment mode)  
- [ ] VIP / capped events (if applicable) — see integration guide  

## Realtime

- [ ] WebSocket reconnect with backoff  
- [ ] Handle chat, gift, presence, and end on room socket  
- [ ] Handle notifications and lobby updates on `/ws/me` if using VIP  

## Payments

- [ ] [Tenant business](./TENANT_BUSINESS.md) configured (mode, packs)  
- [ ] Stripe Connect `acct_…` if you collect card payments  
- [ ] Web and mobile checkout wired per integration guide  
- [ ] Mobile: RevenueCat — [MOBILE_PAYMENTS.md](./MOBILE_PAYMENTS.md)  

## Security

- [ ] Never embed `drift_sk_…` in frontend or Bubble **page** workflows visible to users  
- [ ] CORS: your web origin allowed (contact support if custom domain)  
- [ ] Webhook signature verification if you proxy Stripe events  

## Bubble.io (if applicable)

- [ ] [Bubble guide](./BUBBLE_IO.md) completed  
- [ ] API Connector uses **backend workflows** for secrets  
- [ ] Dynamic `Authorization` header uses **user JWT**, not API key  

## Go-live

- [ ] Staging tenant tested end-to-end  
- [ ] Error handling for paywall, banned/suspended, and capacity errors  
- [ ] Monitoring on `GET /health` and your own API error rates  

## SDK (optional — TypeScript / Expo)

- [ ] `@niilox/sdk` installed and built  
- [ ] `createNiiloxClient({ appId, token })` with `X-App-ID` matching your tenant  
- [ ] VIP / ticketing via `client.seats` — integration guide on request  
- [ ] See [Platform status](./PLATFORM_STATUS.md#sdk-niiloxsdk-v01)
