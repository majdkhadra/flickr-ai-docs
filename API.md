# Flickr API — Client Reference

How the iOS app talks to the Supabase backend.

## Connection

- **Base URL:** `https://mrnuzqvmriblffeumksa.supabase.co`
- **Anon key:** `sb_publishable_q23dj0AJv5LQySOLYT_xIw_yHKxUuXJ` (safe to ship in the app)
- **SDK:** `supabase-swift`. The SDK auto-attaches the `apikey` and the user's `Authorization: Bearer <jwt>` on every call — don't set them manually.

## Auth

- Users can browse after skipping onboarding sign-in, but protected actions require a real provider session.
- Google sign-in is the current production sign-in path.
- Store the session in the **Keychain** so it survives reinstall. The SDK auto-refreshes tokens; never handle expiry manually.
- Every call below requires a signed-in session. Calls without a valid JWT return **401**.

---

## Edge Functions (call via `functions.invoke`)

### `POST /functions/v1/generate-image`
Generate an image from a prompt (optionally editing a supplied photo). Charges 1 credit on success only.

**Request body**
| Field | Type | Required | Notes |
|---|---|---|---|
| `prompt` | string | yes | the edit/generation instruction |
| `inputImageBase64` | string | no | base64 of the source photo for edits. Compress/downscale before sending (request size limit). |
| `templateId` | uuid | no | set if started from a template |

**Success — 200**
```json
{ "creation": {
  "id": "uuid",
  "prompt": "string",
  "output_image_url": "https://.../creations/<uid>/<file>.png",
  "credits_cost": 1,
  "created_at": "ISO8601"
}}
```
`output_image_url` is a public URL — load it directly.

**Errors**
| Status | Body | Client action |
|---|---|---|
| 401 | `not authenticated` / `Invalid JWT` | ensure signed in, retry |
| 402 | `insufficient_credits` | route to paywall |
| 500 | `model error: ...` / `no image returned` | show error toast; no credit charged |

---

### `POST /functions/v1/verify-purchase`
Verify an Apple StoreKit purchase/subscription transaction and grant matching credits.

**Request body**
| Field | Type | Required | Notes |
|---|---|---|---|
| `jws` | string | yes | the signed transaction from StoreKit 2 |

**Success — 200**
```json
{ "ok": true, "granted": 200 }
```

**Errors**
| Status | Body | Meaning |
|---|---|---|
| 400 | `jws required` / `wrong bundle` / `unknown product` | bad/unsupported transaction |
| 401 | not authenticated | no valid session |
| 500 | error string | verification or grant failed |

**Flow:** StoreKit purchase → Apple returns `jws` → call this → on `ok`, refresh the credits balance (below).

### Product IDs expected by backend
These are the SKUs currently used by the app and local StoreKit config, and
must match `CREDITS_BY_PRODUCT` keys in `verify-purchase` **exactly**:

- `abdalla_khadra.flickr_ai.credits.300` → 300 credits
- `abdalla_khadra.flickr_ai.credits.600` → 600 credits
- `abdalla_khadra.flickr_ai.credits.900` → 900 credits
- `abdalla_khadra.flickr_ai.weekly.trial` → 200 credits (initial purchase)
- `abdalla_khadra.flickr_ai.weekly` → 200 credits (initial purchase)
- `abdalla_khadra.flickr_ai.annual` → 200 credits (initial purchase)

Do not use placeholder IDs like `credits_50` / `credits_200` / `credits_500` in
the backend map unless App Store Connect and the iOS app were also changed to
those exact IDs.

If the purchased `productId` is missing from the function allowlist/mapping, the
function returns `400 unknown product`.

### Bundle check
`verify-purchase` must use the real bundle id (`abdalla-khadra.flickr-ai`) for Apple transaction bundle validation.

If `tx.bundleId !== BUNDLE_ID`, the function returns `400 wrong bundle`.

### Quick troubleshooting
- `400 wrong bundle`: transaction bundle id does not equal
  `abdalla-khadra.flickr-ai`.
- `400 unknown product`: transaction `productId` is not in the backend
  `CREDITS_BY_PRODUCT` map.

---

## Direct reads (call via `from(...).select()`)

Each query returns only the signed-in user's own data.

### Credits balance
```
from("users").select()
```
Returns the current user's profile row. The app currently decodes:

- `credits` → current credit balance.
- `resets` → optional account reset count/allowance, if the backend exposes it.
- `permanent_credits` → optional permanent-credit balance, if the backend exposes it.

Use the first row. Refresh after each generation and after a purchase.

### Past creations (Creations screen)
```
from("creations").select().order("created_at", desc)
```
Returns the user's creations, newest first. Each row matches the `creation` shape above.

---

## Notes
- The onboarding plan screen now uses **real StoreKit subscription purchases**:
  - weekly trial: 3 days free, then `$7.99/week`
  - annual: `$59.99/year`
- In-app credit top-up paywall continues to use consumable packs.
- New users start at **0 credits**. Credits come from verified purchases/subscription grants.
- `speed`/`mode` are not yet honored by generation (default `fast`/`auto`).
- Video generation: not available.
