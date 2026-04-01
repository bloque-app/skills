# API Reference

## `@bloque/payments` (Server SDK)

### `new Bloque(config)`

```ts
type BloqueConfig = {
  mode: 'sandbox' | 'production';
  secretKey: string;       // sk_test_... or sk_live_... (auto-exchanged for JWT)
  timeout?: number;        // request timeout in ms (default 10_000)
  maxRetries?: number;     // retry count (default 2)
  webhookSecret?: string;
};

/** @deprecated Use BloqueConfig with secretKey instead */
type BloqueConfigLegacy = {
  mode: 'sandbox' | 'production';
  accessToken: string;
  timeout?: number;
  maxRetries?: number;
  webhookSecret?: string;
};
```

### `bloque.checkout`

- `create(params: CheckoutParams): Promise<Checkout>`
- `retrieve(checkoutId: string): Promise<Checkout>`
- `cancel(checkoutId: string): Promise<Checkout>`

```ts
type ASSETS = 'DUSD/6' | 'COPM/2';

type CheckoutParams = {
  name: string;
  description?: string;
  image_url?: string;
  asset?: ASSETS;                          // default 'DUSD/6'
  items: {
    name: string;
    description?: string;
    amount: number;
    quantity: number;
    image_url?: string;
  }[];
  success_url: string;
  cancel_url: string;
  metadata?: Record<string, string | number | boolean>;
  expires_at?: string;
  payment_methods?: ('card' | 'pse' | 'cash')[];
  payout_route?: PayoutRoute[];
};

type Checkout = {
  id: string;
  urn: string;
  object: 'checkout';
  url: string;
  client_secret?: string;    // checkout-scoped JWT for browser-side auth
  status: 'pending' | 'completed' | 'expired' | 'canceled';
  amount_total: number;
  amount_subtotal: number;
  asset: ASSETS;
  items: CheckoutItem[];
  metadata?: Record<string, string | number | boolean>;
  created_at: string;
  updated_at: string;
  expires_at: string | null;
};
```

### `bloque.payments`

- `create(params: CreatePaymentParams): Promise<PaymentResponse>`
- `getStatus(paymentId: string): Promise<PaymentResponse>`

```ts
type BrowserInfo = {
  browser_color_depth: string;
  browser_screen_height: string;
  browser_screen_width: string;
  browser_language: string;
  browser_user_agent: string;
  browser_tz: string;
};

type CardPaymentFormData = {
  cardNumber: string;
  cardholderName: string;
  expiryMonth: string;
  expiryYear: string;
  cvv: string;
  email: string;
  is_three_ds?: boolean;
  browser_info?: BrowserInfo;
  three_ds_auth_type?: string;          // sandbox only
};

type PaymentSubmitPayload = {
  type: 'card';
  data: CardPaymentFormData;
};

type CreatePaymentParams = {
  checkoutId?: string;
  payment: PaymentSubmitPayload;
};

type ThreeDSData = {
  current_step: string;
  iframe: string;                       // HTML string or URL for 3DS challenge
};

type PaymentResponse = {
  id: string;
  object: 'payment';
  status: 'pending' | 'processing' | 'completed' | 'failed';
  amount: number;
  currency: string;
  created_at: string;
  updated_at: string;
  three_ds?: ThreeDSData;
};
```

### `bloque.webhooks`

- `verify(body, signature, options?): boolean`
- `setSecret(secret): void`

```ts
type WebhookVerifyOptions = {
  secret: string;
};
```

### Error classes

| Class | When |
|---|---|
| `BloqueError` | Base class for all SDK errors |
| `APIError` | Non-2xx API response (has `status`, `code`) |
| `AuthenticationError` | 401 — bad key or expired JWT |
| `RateLimitError` | 429 — too many requests |
| `ValidationError` | 400 — invalid params (has `field`) |
| `KeyRevokedError` | 401 — API key has been revoked (has `keyId`) |

All errors are importable from `@bloque/payments`.

---

## `@bloque/payments-core` (Browser SDK)

### `init(config: BloqueInitOptions)`

```ts
type BloqueInitOptions = {
  publishableKey?: string;     // pk_test_... or pk_live_...
  /** @deprecated */ publicApiKey?: string;
  mode?: 'production' | 'sandbox';
  checkoutUrl?: string;        // default 'https://payments.bloque.app/checkout'
};
```

### `new BloqueCheckout(options)` / `createCheckout({ ...options, container })`

```ts
type BloqueCheckoutOptions = {
  checkoutId: string;
  clientSecret?: string;       // checkout JWT from bloque.checkout.create()
  publishableKey?: string;
  /** @deprecated */ publicApiKey?: string;
  mode?: 'production' | 'sandbox';
  checkoutUrl?: string;
  appearance?: AppearanceConfig;
  showInstallments?: boolean;
  paymentMethods?: ('card' | 'pse')[];
  iframeStyles?: Record<string, string>;
  three_ds_auth_type?: string;          // sandbox only
  onReady?: () => void;
  onSuccess?: (data: PaymentResult) => void;
  onPending?: (data: PaymentResult) => void;
  onError?: (error: string) => void;
  onThreeDSChallenge?: () => void;      // fires when 3DS overlay appears
};

type AppearanceConfig = {
  primaryColor?: string;
  borderRadius?: string;
  fontFamily?: string;
};

type PaymentResult = {
  payment_id: string;
  status: 'approved' | 'pending' | 'rejected';
  message: string;
  amount: number;
  currency: string;
  reference: string;
  created_at: string;
};
```

---

## `@bloque/payments-react`

### `<BloqueCheckout>` component

Props extend `BloqueCheckoutOptions` with React-friendly naming:

```tsx
type BloqueCheckoutProps = {
  checkoutId: string;
  clientSecret?: string;
  publishableKey?: string;
  /** @deprecated */ publicApiKey?: string;
  mode?: 'production' | 'sandbox';
  checkoutUrl?: string;
  appearance?: AppearanceConfig;
  showInstallments?: boolean;
  paymentMethods?: ('card' | 'pse')[];
  iframeStyles?: Record<string, string>;
  threeDsAuthType?: string;              // camelCase (React convention)
  onReady?: () => void;
  onSuccess?: (data: PaymentResult) => void;
  onPending?: (data: PaymentResult) => void;
  onError?: (error: string) => void;
  onThreeDSChallenge?: () => void;
  className?: string;
  style?: React.CSSProperties;
};
```

Re-exports from `@bloque/payments-core`: `init`, `BloqueCheckoutCore`, `AppearanceConfig`, `PaymentResult`.
