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

- `create(params: CheckoutParams): Promise<Checkout>` — maps to `POST /`
- `retrievePublic(urlId: string): Promise<Checkout>` — maps to `GET /by-link/:url_id` (no auth)
- `retrieve(checkoutId: string): Promise<Checkout>` — maps to `GET /link/:url_id`
- `cancel(paymentUrn: string): Promise<Checkout>` — maps to `PATCH /:payment_urn/status`
- `list(params?: ListCheckoutParams): Promise<Checkout[]>` — maps to `GET /`

```ts
type ASSETS = 'DUSD/6' | 'COPM/2' | 'COP/2' | 'USD/6';
type PaymentType = 'shopping_cart' | 'subscription';
type SubscriptionStatus = 'active' | 'expired' | 'eliminated' | 'paid';

type SubscriptionConfig = {
  type: 'cron';
  cron: string;                            // e.g. "0 0 1 * *" for monthly
  startDate?: string;
  endDate?: string;
  status?: SubscriptionStatus;
};

type TaxInfo = {
  name: string;
  rate: number;                            // e.g. 0.19 for 19%
};

type IdType = 'CC' | 'NIT' | 'RUT' | 'PASSPORT' | 'DRIVER_LICENSE';

type PayeerInfo = {
  name: string;
  email?: string;
  phone?: string;
  address_line1?: string;
  address_line2?: string;
  neighborhood?: string;
  city?: string;
  state?: string;
  country?: string;
  postal_code?: string;
  id_type?: IdType;
  id_number?: string;
};

type CheckoutParams = {
  name: string;
  description?: string;
  image_url?: string;
  asset?: ASSETS;                          // default 'DUSD/6'
  payment_type?: PaymentType;              // default 'shopping_cart'
  items: {
    name: string;
    description?: string;
    amount: number;
    quantity: number;
    image_url?: string;
  }[];
  subscription?: SubscriptionConfig;       // required when payment_type is 'subscription'
  success_url?: string;
  cancel_url?: string;
  redirect_url?: string;                   // subscription redirect after checkout
  webhook_url?: string;
  metadata?: Record<string, unknown>;
  expires_at?: string;
  payment_methods?: ('card' | 'pse' | 'cash')[];  // hosted checkout UI only, not sent to server
  payout_route?: PayoutRoute[];
  tax?: TaxInfo[];
  discount_code?: string;
  payeer?: PayeerInfo;
};

type CheckoutStatus = 'pending' | 'paid' | 'expired' | 'deposited' | 'cancelled';

type Checkout = {
  id: string;
  urn: string;
  object: 'checkout';
  url: string;
  client_secret?: string;                  // checkout-scoped JWT for browser-side auth
  status: CheckoutStatus;
  payment_type: PaymentType;
  amount_total: number;
  amount_subtotal: number;
  asset: ASSETS;
  items: CheckoutItem[];
  subscription?: SubscriptionConfig;       // present when payment_type is 'subscription'
  metadata?: Record<string, unknown>;
  created_at: string;
  updated_at: string;
  expires_at: string | null;
};

type ListCheckoutParams = {
  status?: CheckoutStatus;
  payment_type?: PaymentType;
  payeer_search?: string;
  from_date?: string;
  to_date?: string;
  limit?: number;
  offset?: number;
  order?: 'asc' | 'desc';
};
```

### `bloque.payments`

- `create(params: CreatePaymentParams): Promise<PaymentResponse>` — maps to `POST /:type`
- `getStatus(paymentId: string): Promise<PaymentResponse>` — maps to `GET /:payment_urn`

```ts
type PaymentMethodType = 'card' | 'pse' | 'cash';

type PayeeIdType = 'CC' | 'NIT' | 'RUT' | 'PASSPORT' | 'DRIVER_LICENSE';

type Payee = {
  name: string;
  email: string;
  phone?: string;
  id_type?: PayeeIdType;
  id_number?: string;
  address_line1?: string;
  address_line2?: string;
  neighborhood?: string;
  city?: string;
  state?: string;
  country?: string;
  postal_code?: string;
};

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
  installments: number;                 // 1 for single payment
  currency: string;                     // e.g. 'COP', 'USD'
  phone?: string;
  webhookUrl?: string;
  is_three_ds?: boolean;
  browser_info?: BrowserInfo;
  three_ds_auth_type?: string;          // sandbox only
};

type PsePaymentFormData = {
  email: string;
  personType: 'natural' | 'juridica';
  documentType: string;
  documentNumber: string;
  bankCode: string;
  name: string;
  phone: string;
  webhookUrl?: string;
};

type CashPaymentFormData = {
  email: string;
  documentType: string;
  documentNumber: string;
  fullName: string;
  phone?: string;
  webhookUrl?: string;
};

type PaymentSubmitPayload =
  | { type: 'card'; data: CardPaymentFormData }
  | { type: 'pse';  data: PsePaymentFormData }
  | { type: 'cash'; data: CashPaymentFormData };

type CreatePaymentParams = {
  paymentUrn: string;          // did:bloque:payments:... (maps to payment_urn in body)
  payment: PaymentSubmitPayload;
  /** @deprecated Use paymentUrn instead */
  checkoutId?: string;
};

type DirectPaymentStatus = 'approved' | 'rejected' | 'pending';

type ThreeDSData = {
  current_step: string;
  iframe: string;                       // HTML string or URL for 3DS challenge
};

type PaymentResponse = {
  id: string;                           // payment URN (payment_id on the wire)
  object: 'payment';
  status: DirectPaymentStatus;
  message: string;
  amount: number;
  currency: string;
  reference?: string;
  checkout_url?: string;                // PSE redirect URL
  payment_code?: string;               // cash payment code
  order_id?: string;
  order_status?: string;
  three_ds?: ThreeDSData;
  created_at: string;
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
type PaymentMethod = 'card' | 'pse' | 'cash';

type BloqueCheckoutOptions = {
  checkoutId: string;
  clientSecret?: string;       // checkout JWT from bloque.checkout.create()
  publishableKey?: string;
  /** @deprecated */ publicApiKey?: string;
  mode?: 'production' | 'sandbox';
  checkoutUrl?: string;
  appearance?: AppearanceConfig;
  showInstallments?: boolean;
  paymentMethods?: PaymentMethod[];
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
  paymentMethods?: PaymentMethod[];
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

Re-exports from `@bloque/payments-core`: `init`, `BloqueCheckoutCore`, `AppearanceConfig`, `PaymentResult`, `PaymentMethod`, `ThreeDSChallengeData`.
