# API Reference

## `@bloque/payments`

### `new Bloque(config)`

```ts
type BloqueConfig = {
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

Tipos principales:

```ts
type CheckoutParams = {
  name: string;
  description?: string;
  image_url?: string;
  items: {
    name: string;
    description?: string;
    amount: number;
    quantity: number;
    image_url?: string;
  }[];
  currency?: 'COP' | 'USD';
  success_url: string;
  cancel_url: string;
  metadata?: Record<string, string | number | boolean>;
  expires_at?: string;
  payment_methods?: ('card' | 'pse' | 'cash')[];
};

type Checkout = {
  id: string;
  object: 'checkout';
  url: string;
  status: 'pending' | 'completed' | 'expired' | 'canceled';
  amount_total: number;
  amount_subtotal: number;
  currency: 'COP' | 'USD';
  items: CheckoutItem[];
  metadata?: Record<string, string | number | boolean>;
  created_at: string;
  updated_at: string;
  expires_at: string | null;
};
```

### `bloque.payments`

- `create(params: CreatePaymentParams): Promise<PaymentResponse>`

```ts
type PaymentSubmitPayload = {
  type: 'card';
  data: {
    cardNumber: string;
    cardholderName: string;
    expiryMonth: string;
    expiryYear: string;
    cvv: string;
    email: string;
  };
};

type CreatePaymentParams = {
  checkoutId?: string;
  payment: PaymentSubmitPayload;
};

type PaymentResponse = {
  id: string;
  object: 'payment';
  status: 'pending' | 'processing' | 'completed' | 'failed';
  amount: number;
  currency: string;
  created_at: string;
  updated_at: string;
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

## `@bloque/payments-core`

- `init(config)`
- `new BloqueCheckout(options)`
- `createCheckout({ ...options, container })`

Eventos:

- `onReady()`
- `onSuccess(PaymentResult)`
- `onPending(PaymentResult)`
- `onError(string)`

## `@bloque/payments-react`

- `BloqueCheckout` (wrapper React del core)
- `init(...)` y `BloqueCheckoutCore` re-exportados desde `@bloque/payments-core`
