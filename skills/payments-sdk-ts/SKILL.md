---
name: bloque-payments-sdk-ts
description: >
  Integration guide for the Bloque Payments SDK in TypeScript/JavaScript.
  Use when the user asks to create checkouts, process payments with
  @bloque/payments, handle 3D Secure, verify webhooks, or embed the
  checkout with @bloque/payments-core or @bloque/payments-react.
---

# Bloque Payments SDK TS

Implement payment integrations with `@bloque/payments`, `@bloque/payments-core`, and `@bloque/payments-react`.

## Security Boundaries (Mandatory)

| Credential | Prefix | Where | Rules |
|---|---|---|---|
| **Secret key** | `sk_test_` / `sk_live_` | Server only | Never expose in browser, logs, or client bundles. Auto-exchanged for a short-lived JWT by the SDK. |
| **Publishable key** | `pk_test_` / `pk_live_` | Browser / mobile | Read-only. Identifies the merchant but cannot authorize payments. Safe to embed in client code. |
| **Client secret** | JWT (`eyJ...`) | Browser | Short-lived (~1 hr), scoped to a single checkout. Do NOT persist in localStorage/cookies. Do NOT log or send to analytics. Treat as ephemeral per-session. |

Additional rules:
- Verify HMAC-SHA256 webhook signatures before processing any event.
- Keep `mode` consistent between backend (`sandbox`/`production`) and frontend.
- Require explicit human confirmation before money-moving operations in agent workflows.

## When to Apply

Use this skill when:
- Creating checkout sessions (one-time or subscription) with `@bloque/payments`
- Embedding hosted checkout in a web app (`@bloque/payments-core` or `@bloque/payments-react`)
- Processing card payments with 3D Secure (3DS)
- Polling payment status after a 3DS challenge
- Verifying payment webhooks
- Setting up the three-credential auth model (`secretKey` / `publishableKey` / `clientSecret`)
- Handling `KeyRevokedError` for rotated API keys

## Package Selection

```
@bloque/payments        Server SDK (Node/Bun/Deno). secretKey auth. Checkout (one-time + subscription), payments, webhooks.
@bloque/payments-core   Browser SDK (vanilla JS/TS). Iframe checkout + 3DS overlay.
@bloque/payments-react  React wrapper for payments-core. <BloqueCheckout> component.
```

## Recommended Flow

```
Backend (server)                          Frontend (browser)
────────────────                          ──────────────────
1. new Bloque({ secretKey, mode })
2. bloque.checkout.create({ ... })
   ├── returns checkout.id
   ├── returns checkout.url
   └── returns checkout.client_secret ──► 3. Mount <BloqueCheckout>
                                             ├── publishableKey = pk_test_...
                                             ├── clientSecret = checkout.client_secret
                                             └── checkoutId = checkout.id
                                          4. User fills card form
                                          5. Hosted checkout submits payment
                                             │
                                             ├── No 3DS ──► onSuccess / onError
                                             │
                                             └── 3DS required
                                                  ├── onThreeDSChallenge fires
                                                  ├── 3DS overlay rendered in parent page
                                                  ├── User completes challenge
                                                  └── Polls status until terminal
                                                       ├── approved ──► onSuccess
                                                       ├── rejected ──► onError
                                                       └── timeout  ──► onError

6. Receive webhook (server)
   └── bloque.webhooks.verify(body, signature)
```

## Quick Start (minimal)

```typescript
// Backend: create checkout
import { Bloque } from '@bloque/payments';

const bloque = new Bloque({ mode: 'sandbox', secretKey: process.env.BLOQUE_SECRET_KEY! });
const checkout = await bloque.checkout.create({
  name: 'Order #123', items: [{ name: 'Widget', amount: 500000, quantity: 1 }],
  asset: 'DUSD/6', success_url: 'https://app.com/success', cancel_url: 'https://app.com/cancel',
});
// Pass checkout.id and checkout.client_secret to the frontend
```

```tsx
// Frontend (React): mount checkout
import { BloqueCheckout } from '@bloque/payments-react';

<BloqueCheckout
  checkoutId={checkoutId}
  publishableKey="pk_test_..."
  clientSecret={clientSecret}
  mode="sandbox"
  onSuccess={(data) => console.log('Paid!', data)}
  onError={(err) => console.error(err)}
/>
```

## References

| File | When to read |
|------|-------------|
| `references/api-reference.md` | All methods, types, params, return shapes, error classes |
| `references/quick-start.md` | Full backend + frontend + 3DS + webhook examples |
| `references/implementation-notes.md` | Gotchas, SDK internals, 3DS sandbox values |
