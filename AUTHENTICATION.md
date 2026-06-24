# Authentication

> **Production (May 2026):** Native auth only — see **[NATIVE_AUTH.md](./NATIVE_AUTH.md)**.
> `/auth/verify` is not part of the production API.
> This file is kept as historical context for older migrations.

Drift now uses native auth providers (Google, magic link, phone OTP, Apple, password) and issues server-managed access + refresh tokens. Any historical references to Supabase flows here are archival only.

## End-to-end flow

```
1. Browser            ──POST /api/v1/auth/{provider}──►  drift API
2. drift API          verifies provider payload / OTP / password
3. drift API          upserts tenant-scoped user row
4. drift API          creates hashed refresh session (`auth_sessions`)
5. drift API          ───────────────► returns access + refresh tokens
6. Browser            includes `Authorization: Bearer <access>` on API requests
7. Browser            renews via `/api/v1/auth/refresh` when needed
```

The Drift access JWT is the credential used on protected routes.

## Drift session JWT

```json
{
  "sub": "<users.id>",   // app-scoped user UUID
  "iat": 1734567890,
  "exp": 1735172690      // iat + 7 days
}
```

Signing: `HS256` with `JWT_SECRET`. Verified by `internal/middleware.Auth`. Claims placed on `r.Context()`:

| Key | Value |
|-----|-------|
| `middleware.UserIDKey` | `<users.id>` |
| `middleware.RoleKey` | `""` for registered, `"guest"` for `/auth/guest` |
| `middleware.DisplayNameKey` | from the JWT `name` claim (set on guest tokens, blank otherwise) |

Helpers exposed by `internal/middleware`:

| Function | Returns |
|----------|---------|
| `GetUserID(ctx)` | `string` |
| `IsGuest(ctx)` | `bool` |
| `GetApp(ctx)` | `*apps.App` (set by the tenant middleware) |
| `GetAppID(ctx)` | tenant id (defaults to `drift`) |

## Guest tokens

Guests browse rooms without signing in. `POST /api/v1/auth/guest` issues:

```json
{
  "sub": "guest-<uuid>",
  "role": "guest",
  "name": "guest-XXXX",
  "iat": ...,
  "exp": ... // 24h
}
```

Routes wrapped in `middleware.RequireRegistered` (room creation, chat, gifts, DMs, payments, withdrawals, stage actions, moderation, ID verification) return `403 sign in required` for guests. Read routes — `GET /rooms`, `GET /rooms/{id}`, `GET /rooms/{id}/chat`, `GET /gifts`, `GET /tokens/packs` — accept any caller.

Adult rooms (see [`MULTI-TENANT.md`](MULTI-TENANT.md)) reject guests outright even on `JoinRoom`.

## Tenant resolution

The `X-App-ID` header is read by `middleware.Tenant`, looked up in `apps.Registry`, and stored on the context **before** the auth middleware runs. Two consequences:

- A single identity maps to **different `users.id` rows per tenant** — the same person can hold separate balances on Drift and Rabbaly.
- User rows are always resolved and written under the active tenant (`app_id`), so issuing tokens for tenant A never affects tenant B.

## LiveKit tokens

LiveKit access is gated by short-lived JWTs minted by `livekit.Service`. Three kinds:

| Method | Used for | Permissions | TTL |
|--------|----------|-------------|-----|
| `CreateHostToken(room, userID)` | `POST /api/v1/rooms` | `RoomCreate`, `RoomJoin`, `RoomAdmin`, `CanPublish`, `CanSubscribe`, `CanPublishData` | 6h |
| `CreateViewerToken(room, userID)` | `POST /api/v1/rooms/{id}/join` and media-svc watcher | `RoomJoin`, `CanSubscribe` (no publish) | 6h |
| Internal `adminToken(room)` | RoomService twirp calls (mute, kick, list participants) | `RoomAdmin` | 2 min |

Tokens are HS256-signed with the **tenant's** LiveKit secret (`apps.livekit_secret`), not the global `JWT_SECRET`.

## Admin role

`users.is_admin = true` (per tenant) gates the ID-verification review endpoints under `/api/v1/admin/verifications`. There's no separate JWT role — admins use their normal Drift JWT and the `verification.adminOnly` middleware checks the DB column on each call.

### Granting admin

Two interchangeable paths — pick whichever fits your ops workflow.

**1. Env-driven (recommended).** Set `ADMIN_EMAILS` in `.env` (or your Docker / k8s / Supabase Edge Function environment) to a comma-separated list. The server then:

- Runs a one-shot `UPDATE users SET is_admin = true WHERE lower(email) = ANY(...)` on boot to retroactively promote already-existing users.
- On every successful native sign-in / token issuance, promotes the user if their email is in the list. Idempotent.

```env
ADMIN_EMAILS=[email protected],[email protected]
```

This is **promote-only**. Removing an email from the env will *not* revoke admin — that prevents accidental self-lockout. Revoke explicitly with SQL when needed.

**2. Manual SQL.** Still works the same as before:

```sql
UPDATE users SET is_admin = true WHERE app_id = 'drift' AND email = '[email protected]';
```

Use this when you want per-tenant control (env promotion applies across tenants by email) or for one-off revocation:

```sql
UPDATE users SET is_admin = false WHERE email = '[email protected]';
```

## Security boundaries

- **`JWT_SECRET` must be at least 32 random bytes.** If rotated, every issued session JWT is invalidated and users will re-sign-in.
- **The Drift JWT is unencrypted but signed.** Do not put PII in custom claims.
- **Webhook auth is per-tenant.** LiveKit webhooks are HMAC-verified against `apps.livekit_secret`, so a rogue tenant cannot spoof events for another.
- **CORS** allows `APP_URL`, `ADMIN_APP_URL`, and `http://localhost:3000` / `:3001` for dev. The `X-App-ID` header is on the allow list.
