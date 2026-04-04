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
| `baseURL` | `https://dev.bloque.app/api/payments` | All resource endpoints (`/`, `/link/:id`, `/:type`, `/:urn/status`) |
| `exchangeBaseURL` | `https://dev.bloque.app/api` | Key exchange only: `POST /origins/api-keys/exchange` |

The separation exists because the origins service lives under `/api`, not `/api/payments`. If you're pointing at a custom gateway, set both values explicitly.

## Route mapping

The SDK builds URLs relative to `baseURL` (`{apiRoot}/payments`). The server routes (`@Controller("payments")` with global prefix `api`) are:

| SDK method | SDK path (relative) | Server route |
|---|---|---|
| `checkout.create()` | `POST /` | `POST /api/payments` |
| `checkout.retrievePublic(id)` | `GET /by-link/${id}` | `GET /api/payments/by-link/:url_id` (public) |
| `checkout.retrieve(id)` | `GET /link/${id}` | `GET /api/payments/link/:url_id` |
| `checkout.cancel(urn)` | `PATCH /${urn}/status` | `PATCH /api/payments/:payment_urn/status` |
| `checkout.list(params)` | `GET /` + query | `GET /api/payments` |
| `payments.create(params)` | `POST /${type}` | `POST /api/payments/:type` |
| `payments.getStatus(id)` | `GET /${id}` | `GET /api/payments/:payment_urn` |

No Envoy gateway rewrites exist — paths are forwarded unchanged.

## `cancel()` requires a payment URN

`checkout.cancel()` accepts a **payment URN** (`did:bloque:payments:...`), not a checkout ID. It sends `PATCH /${urn}/status` with `{ status: 'cancelled' }` in the body.

```ts
const cancelled = await bloque.checkout.cancel(checkout.urn);
```

## `payments.create()` uses `paymentUrn`

Direct payments (card / PSE / cash) require `paymentUrn` in the params object. The URN is sent as `payment_urn` in the request body to `POST /:type`:

```ts
const payment = await bloque.payments.create({
  paymentUrn: 'did:bloque:payments:abc123',
  payment: { type: 'card', data: { ... } },
});
```

The deprecated `checkoutId` field is still accepted as a fallback.

## Payment types: card, PSE, cash

All three payment types are supported with distinct form data shapes:

| Type | Form data interface | Required fields | Type-specific response fields |
|---|---|---|---|
| `card` | `CardPaymentFormData` | `cardNumber`, `cardholderName`, `expiryMonth`, `expiryYear`, `cvv`, `email`, `installments`, `currency` + 3DS fields | `three_ds?` |
| `pse` | `PsePaymentFormData` | `email`, `personType`, `documentType`, `documentNumber`, `bankCode`, `name`, `phone` | `checkout_url` (redirect URL) |
| `cash` | `CashPaymentFormData` | `email`, `documentType`, `documentNumber`, `fullName` | `payment_code` (10-digit code) |

The `PaymentResource` builds the correct server payload for each type, mapping camelCase SDK fields to snake_case server fields.

### Card: `installments` and `currency`

Card payments require `installments` (number, use `1` for single payment) and `currency` (string, e.g. `'COP'`, `'USD'`). The server uses `currency` to determine the source asset for the swap rate. Omitting these fields causes server-side validation failure.

### Payee forwarding

All payment types build a `payee` object in the wire payload. The SDK exposes optional fields (`phone`, `webhookUrl`) on each form data interface. When present, they are forwarded to the server:

- **Card**: `payee.phone` maps to `customer_data.phone_number` in the swap args.
- **PSE**: `name` and `phone` are required fields. They map to `customer_data.full_name` / `customer_data.phone_number` in the swap args.
- **Cash**: `payee.phone` maps to `phone_number`.

### `webhookUrl`

All three form data interfaces accept an optional `webhookUrl` (camelCase). The builder maps it to `webhook_url` (snake_case) on the wire payload. The server `Payee` class and each payment input class include `webhook_url` support.

### `Payee` type

The SDK exports a `Payee` interface and `PayeeIdType` union from `@bloque/payments`. This mirrors the server-side `Payee` class with all fields (name, email, phone, id_type, id_number, address fields). For card payments, the builder uses the full `Payee` shape on the wire payload.

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

After rendering the 3DS challenge iframe, poll `bloque.payments.getStatus(paymentId)` at 3-second intervals until the status is terminal (`approved` or `rejected`). Cap at 60 attempts (~3 minutes).

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

## Checkout `payment_type` and subscriptions

`checkout.create()` accepts `payment_type` (default `'shopping_cart'`). For recurring payments, pass `payment_type: 'subscription'` with a `subscription` config:

```ts
const sub = await bloque.checkout.create({
  name: 'Monthly Plan',
  payment_type: 'subscription',
  items: [{ name: 'Plan', amount: 29_000000, quantity: 1 }],
  subscription: { type: 'cron', cron: '0 0 1 * *', status: 'active' },
  success_url: 'https://app.com/success',
  redirect_url: 'https://app.com/dashboard',
});
```

The returned `Checkout` includes `payment_type` and `subscription` fields.

## Checkout `success_url` field

`checkout.create()` sends `success_url` to the server (matching the shopping cart DTO). For subscriptions, also pass `redirect_url` (the subscription DTO accepts both). The `payment_methods` field in `CheckoutParams` is **not** sent to the server — it only controls which tabs the hosted checkout UI displays.

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
