# Native auth and own Postgres

**Niilox API** runs on **your own Postgres** with **native auth** (Google, magic link, phone OTP, Apple, password) while keeping the existing `/api/v1` contract. **Production (driftin.live) uses native auth only** — no `SUPABASE_*` env vars and no `/auth/verify` bridge.

For historical cutover steps, see [POSTGRES_MIGRATION.md](./POSTGRES_MIGRATION.md) and [MIGRATION_STATUS.md](./MIGRATION_STATUS.md).

## New environment setup

1. Provision Postgres (`scripts/install-postgres-hetzner.sh` or `docker compose up -d postgres`).
2. Apply migrations: `go run ./cmd/migrate-up`.
3. Configure native auth env vars (below).
4. Frontend: `NEXT_PUBLIC_AUTH_MODE=native` in `.env.local`.

## API env (backend `.env`)

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | Your Postgres (required) |
| `SUPABASE_URL` | **Removed in production** — legacy bridge only if you maintain a fork |
| `API_PUBLIC_URL` | Public API origin, e.g. `https://api.driftin.live` |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google OAuth for Drift |
| `GOOGLE_REDIRECT_URL` | Drift callback, e.g. `https://api.driftin.live/auth/google/callback` |
| `GOOGLE_REDIRECT_URL_RODENT` | Rodent callback (optional separate OAuth client) |
| `GEOGIG_APP_URL` | GeoGig web origin for OAuth/magic finish (e.g. `https://geogig.example.com`) |
| `RABBALY_APP_URL` | Rabbaly app origin for OAuth finish |

**“Access blocked” on Google** usually means:

1. **Redirect URI not registered** — add the exact URL from the OAuth link (`redirect_uri=…`) in Google Cloud Console.
2. **OAuth consent screen in Testing** — add your Gmail under **Test users**, or publish the app.
| `RESEND_API_KEY` / `RESEND_FROM_EMAIL` | Magic links — **verify `driftin.live` at [resend.com/domains](https://resend.com/domains)** then set `from` to e.g. `noreply@driftin.live`. Until verified, Resend only delivers to the account owner email. |
| `RESEND_MAGIC_SUBJECT` | Email subject line (default `sign in to drift`) |
| `RESEND_MAGIC_APP_NAME` | Brand name in template (default `drift`) |
| `RESEND_MAGIC_TEMPLATE` | Path to HTML template (default `/opt/drift/templates/email/magic-link.html`) — see `templates/email/README.md` |
| `AFRICASTALKING_USERNAME` / `AFRICASTALKING_API_KEY` | SMS OTP |
| `APPLE_CLIENT_IDS` | Comma-separated iOS bundle IDs (must match the `aud` claim in Apple identity tokens) |
| `APPLE_CLIENT_ID_GEOGIG` | Optional — GeoGig bundle ID (`cc.nomli.geogig`), merged into `APPLE_CLIENT_IDS` |
| `APPLE_CLIENT_ID_RODENT` | Optional — Rodent iOS bundle ID, merged into `APPLE_CLIENT_IDS` |

**Sign in with Apple** — enable the capability on each app in Apple Developer, then add its bundle ID here. GeoGig uses `cc.nomli.geogig`. The mobile app calls `POST /api/v1/auth/apple` with `{ "identity_token": "..." }` (optional `name` on first sign-in).

## Routes

| Method | Path | Notes |
|--------|------|-------|
| POST | `/api/v1/auth/google/init` | Returns `{ "url": "..." }` |
| GET | `/auth/google/callback` | Redirects to `{APP_URL}/auth/finish?token=` |
| POST | `/api/v1/auth/magic/send` | Body `{ "email" }` |
| POST | `/api/v1/auth/magic/verify` | Body `{ "token" }` or `?magic_token=` on finish page |
| POST | `/api/v1/auth/phone/send` | Body `{ "phone" }` E.164 |
| POST | `/api/v1/auth/phone/verify` | Body `{ "phone", "code" }` |
| GET | `/api/v1/auth/providers` | `{ "google", "magic", "phone", "apple", "password" }` booleans |
| POST | `/api/v1/auth/apple` | Body `{ "identity_token", "name"? }` — iOS Sign in with Apple |
| POST | `/api/v1/auth/refresh` | Body `{ "refresh_token" }` |
| POST | `/api/v1/auth/logout` | Revokes refresh session |

## Frontend

`NEXT_PUBLIC_AUTH_MODE`:

- `dual` — native Google/magic/phone when API providers are set; Supabase password/magic still available
- `native` — only Drift API auth
- `supabase` — legacy bridge only

## Tools

```bash
# Linux / macOS (make installed)
make migrate019

# Windows PowerShell (no make)
cd backend/drift
powershell -File scripts\migrate019.ps1
# or: go run ./scripts/migrate019

go run ./cmd/migrate-from-supabase          # row counts
go run ./cmd/backup                         # pg_dump + optional encrypt
DATABASE_URL=... go run ./cmd/restore backup.dump
```
