# Implementation Notes

Guardrails and SDK internals. Read before making non-trivial changes.

## Constructor: `secretKey` preferred

The `Bloque` constructor accepts either `secretKey` (preferred) or `accessToken` (deprecated).

```ts
// preferred
new Bloque({ mode: 'sandbox', secretKey: 'sk_test_...' });

// deprecated — still works, skips key exchange
new Bloque({ mode: 'sandbox', accessToken: 'eyJ...' });
```

When using `secretKey`, the SDK's `HttpClient` automatically exchanges it for a short-lived JWT via `POST /origins/api-keys/exchange`. The JWT is cached and refreshed ~1 minute before expiry. No action needed on the caller's side.

## Base URLs: `baseURL` vs `exchangeBaseURL`

The SDK uses two base URLs internally:

| Config | Default (sandbox) | Used for |
|---|---|---|
| `baseURL` | `https://dev.bloque.app/api/payments` | All resource endpoints (`/checkout`, `/payments`, `/webhooks`) |
| `exchangeBaseURL` | `https://dev.bloque.app/api` | Key exchange only: `POST /origins/api-keys/exchange` |

The separation exists because the origins service lives under `/api`, not `/api/payments`. If you're pointing at a custom gateway, set both values explicitly.

## Checkout returns `client_secret`

`bloque.checkout.create()` returns a `client_secret` field — a checkout-scoped JWT the frontend uses for authentication. Always pass it to the browser SDK:

```ts
// Backend
const checkout = await bloque.checkout.create({ ... });
// Send checkout.id AND checkout.client_secret to the frontend

// Frontend
createCheckout({
  checkoutId: checkout.id,
  clientSecret: checkout.client_secret,  // <-- required for auth
  ...
});
```

If `clientSecret` is missing, the hosted checkout falls back to `publishableKey` via `X-Api-Key`, which has narrower scopes.

## 3D Secure

### `PaymentResource` 3DS fields

When sending a card payment with `is_three_ds: true`, include `browser_info` (collected from the user's browser). The response may include:

```ts
response.three_ds?.iframe   // HTML string or URL for the 3DS challenge
response.three_ds?.current_step
```

### HTML entity decoding

The `iframe` field in `three_ds` may be HTML-entity-encoded (`&lt;`, `&#39;`, `&amp;`, etc.). Decode it before setting as `srcDoc`:

```ts
function decodeHtmlEntities(html: string): string {
  const textarea = document.createElement('textarea');
  textarea.innerHTML = html;
  return textarea.value;
}

iframeEl.srcdoc = decodeHtmlEntities(payment.three_ds.iframe);
```

### `getStatus()` polling

After rendering the 3DS challenge iframe, poll `bloque.payments.getStatus(paymentId)` at 3-second intervals until the status is terminal (`completed` or `failed`). Cap at 60 attempts (~3 minutes).

### Sandbox `three_ds_auth_type` values

Use these in sandbox mode (`is_three_ds: true` must also be set):

| Value | Result |
|---|---|
| `challenge_v2` | Full 3DS challenge flow (iframe with approve/decline buttons) |
| `no_challenge_success` | 3DS succeeds silently (frictionless) |
| `challenge_denied` | 3DS challenge is denied by the user |
| `supported_version_error` | Simulates a version mismatch error |
| `authentication_error` | Simulates an authentication failure |

Omit `three_ds_auth_type` in production — it is ignored.

### 3DS iframe security

Use `sandbox="allow-scripts allow-forms allow-popups"` for `srcDoc` content. Add `allow-same-origin` only when using a `src` URL.

## Asset field replaces currency

Checkouts use `asset` (e.g. `DUSD/6`, `COPM/2`) instead of the old `currency` string. The format is `SYMBOL/DECIMALS`. Do not pass `currency` — it is no longer accepted.

## Hosted checkout proxy auth

The hosted checkout at `payments.bloque.app/checkout` uses server-side proxy routes that forward the client's auth token as `Authorization: Bearer`. The proxy reads from `X-Client-Secret` (preferred) or `X-Api-Key` headers. If neither is present, it returns 401.

The checkout page detects `clientSecret` JWT expiry by decoding the `exp` claim. If expired, it shows an error and blocks further requests rather than sending stale tokens.

## Webhook verification

Always verify HMAC-SHA256 signatures before trusting webhook payloads. The `webhookSecret` can be passed in the constructor or set later with `bloque.webhooks.setSecret()`.

## Mode consistency

Ensure `mode` is consistent between backend and frontend:
- Backend `new Bloque({ mode: 'sandbox' })` → Frontend `init({ mode: 'sandbox' })`
- Using mismatched modes will cause auth failures since keys are environment-scoped.
