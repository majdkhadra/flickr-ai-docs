# Backend Handoff — for the Frontend Agent

Backend behaviors the iOS app must integrate with. Endpoint contracts are in `API.md`; this is lifecycle/behavior context.

## Stack
- Supabase: Postgres + Auth + Edge Functions + Storage.
- Base URL: `https://mrnuzqvmriblffeumksa.supabase.co`
- Anon key (ship it): `sb_publishable_q23dj0AJv5LQySOLYT_xIw_yHKxUuXJ`
- Use `supabase-swift`. One shared client. SDK auto-attaches auth headers.

## Users & sessions
- Users can skip onboarding sign-in and browse the app, but protected backend work requires a real provider session.
- Google sign-in is the current production sign-in path.
- **Profile is auto-created.** When a user signs in, a DB trigger creates their `public.users` row.
- **Persist session in Keychain.** SDK restores on relaunch and auto-refreshes tokens.

### Critical trigger fix (applied)
`public.handle_new_user()` must explicitly insert into `public.users` (not bare `users`) and be idempotent.

```sql
create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = public
as $$
begin
  insert into public.users (id)
  values (new.id)
  on conflict (id) do nothing;
  return new;
end;
$$;
```

Without this, anonymous signup can fail with duplicate key errors on `users_pkey`.

## Credits
- **Server is source of truth.** Do not treat local balance as authoritative.
- New users have **0 credits**.
- Credits are spent by `generate-image` and granted by verified purchases/renewals.
- After generation or purchase, refresh from DB (`from("users").select()`).

## Generate flow
1. Ensure signed in.
2. Call `generate-image` with `{ prompt, inputImageBase64?, templateId? }`.
3. On 200: append creation + refresh credits.
4. On 402: route to paywall.
5. On 500: show error; no charge.

## Purchase flow
1. Ensure signed in.
2. StoreKit 2 purchase → get signed transaction JWS.
3. Call `verify-purchase` with `{ jws }`.
4. On `{ ok: true, granted }`: refresh credits.

### Current app SKUs
- Consumables:
  - `abdalla_khadra.flickr_ai.credits.300`
  - `abdalla_khadra.flickr_ai.credits.600`
  - `abdalla_khadra.flickr_ai.credits.900`
- Subscriptions (onboarding paywall):
  - `abdalla_khadra.flickr_ai.weekly.trial` (3-day trial, then $7.99/week)
  - `abdalla_khadra.flickr_ai.weekly` ($7.99/week)
  - `abdalla_khadra.flickr_ai.annual` ($59.99/year)

`verify-purchase` must allow/map all six IDs and use the real bundle id `abdalla-khadra.flickr-ai`.

## Onboarding/paywall state (current)
- Onboarding plan screen now runs real StoreKit purchase for:
  - `Start FREE trial` (weekly trial SKU)
  - `Subscribe annually` (annual SKU)
- Main in-app paywall still sells consumable credit packs.

## Transaction listening (client)
A launch-time `Transaction.updates` listener is now wired in app startup to avoid missing completed transactions while app is active/resumed.

## Auto-renewals (required backend work)
Initial subscription purchase grants once. Renewals require App Store Server Notifications V2.

Implement:
- Edge Function webhook endpoint (example slug):
  - `/functions/v1/apple-server-notifications`
- Configure URL in App Store Connect (Server Notifications V2).
- Verify Apple signed payloads.
- Handle at least: `DID_RENEW`, `EXPIRED`, `GRACE_PERIOD_EXPIRED`, `REFUND`.
- Update entitlement state and grant renewal credits idempotently per renewal transaction.

## Reading data
- Account: `from("users").select()`
  - Required today: `credits`
  - Optional fields decoded by Settings if present: `resets`, `permanent_credits`
- Creations: `from("creations").select().order("created_at", desc)`

## Known gaps
- Renewal webhook flow is not fully implemented yet.
- `speed`/`mode` are ignored by generation (default `fast`/`auto`).
- Video generation not implemented.

## Reference
- `API.md` — exact request/response/error contracts.
