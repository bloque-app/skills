# Quick Start

## 1) Backend: create checkout

```ts
import { Bloque } from '@bloque/payments';

const bloque = new Bloque({
  mode: 'sandbox',
  secretKey: process.env.BLOQUE_SECRET_KEY!,
});

const checkout = await bloque.checkout.create({
  name: 'Pack Profesional de Productividad',
  description: 'Premium peripherals for high-performance work',
  asset: 'COPM/2',
  image_url: 'https://cdn.bloque.app/checkouts/productivity-pack.png',
  items: [
    {
      name: 'Teclado mecanico Keychron K2',
      amount: 450_000_00,
      quantity: 2,
      image_url: 'https://cdn.bloque.app/items/keychron-k2.png',
    },
    {
      name: 'Mouse Logitech MX Master 3S',
      amount: 380_000_00,
      quantity: 1,
      image_url: 'https://cdn.bloque.app/items/mx-master-3s.png',
    },
  ],
  success_url: 'https://yourapp.com/success',
  cancel_url: 'https://yourapp.com/cancel',
  webhook_url: 'https://yourapp.com/webhooks/payment',
});

console.log('Checkout ID:', checkout.id);
console.log('Checkout URL:', checkout.url);
console.log('Client Secret:', checkout.client_secret);

// Pass checkout.id and checkout.client_secret to the frontend
```

## 2) Frontend JS/TS: mount checkout iframe

```ts
import { createCheckout, init } from '@bloque/payments-core';

init({
  publishableKey: 'pk_test_...',
  mode: 'sandbox',
});

const checkout = createCheckout({
  checkoutId: 'checkout_123abc',
  clientSecret: 'eyJ...',          // from your backend (checkout.client_secret)
  container: '#checkout-container',
  paymentMethods: ['card', 'pse', 'cash'],
  onSuccess: (data) => console.log('approved', data),
  onPending: (data) => console.log('pending', data),
  onError: (error) => console.error('error', error),
  onThreeDSChallenge: () => console.log('3DS challenge started'),
});

// later
checkout.destroy();
```

## 3) Frontend React: component

```tsx
import { BloqueCheckout } from '@bloque/payments-react';

export function CheckoutPage({
  checkoutId,
  clientSecret,
}: {
  checkoutId: string;
  clientSecret: string;
}) {
  return (
    <BloqueCheckout
      checkoutId={checkoutId}
      publishableKey="pk_test_..."
      clientSecret={clientSecret}
      mode="sandbox"
      paymentMethods={['card', 'pse', 'cash']}
      threeDsAuthType="challenge_v2"    // sandbox only — omit in production
      onThreeDSChallenge={() => {
        console.log('3DS challenge overlay is visible');
      }}
      onSuccess={(data) => console.log('Payment approved:', data)}
      onPending={(data) => console.log('Payment pending:', data)}
      onError={(error) => console.error('Payment failed:', error)}
    />
  );
}
```

## 4) Webhooks: verify signature

```ts
import { Bloque } from '@bloque/payments';

const bloque = new Bloque({
  mode: 'production',
  secretKey: process.env.BLOQUE_SECRET_KEY!,
  webhookSecret: process.env.BLOQUE_WEBHOOK_SECRET,
});

const valid = bloque.webhooks.verify(
  rawBody,
  req.headers['x-bloque-signature'] as string,
);

if (!valid) {
  throw new Error('Invalid webhook signature');
}
```

## 5) 3D Secure card payment (server-side direct API)

Use this when processing card payments server-side with 3DS (not via embedded checkout).

```ts
import { Bloque } from '@bloque/payments';
import type { BrowserInfo } from '@bloque/payments';

const bloque = new Bloque({
  mode: 'sandbox',
  secretKey: process.env.BLOQUE_SECRET_KEY!,
});

// browser_info MUST be collected from the user's browser and sent to the server
const browserInfo: BrowserInfo = {
  browser_color_depth: '24',
  browser_screen_height: '1080',
  browser_screen_width: '1920',
  browser_language: 'es',
  browser_user_agent: 'Mozilla/5.0 ...',
  browser_tz: '-300',
};

const payment = await bloque.payments.create({
  paymentUrn: 'did:bloque:payments:abc123',
  payment: {
    type: 'card',
    data: {
      cardNumber: '4111111111111111',
      cardholderName: 'John Doe',
      expiryMonth: '12',
      expiryYear: '2028',
      cvv: '123',
      email: 'john@example.com',
      installments: 1,
      currency: 'COP',
      is_three_ds: true,
      browser_info: browserInfo,
      three_ds_auth_type: 'challenge_v2', // sandbox only — omit in production
    },
  },
});

console.log('Payment:', payment.id, 'Status:', payment.status);

// If 3DS is required, payment.three_ds.iframe contains the challenge HTML/URL
if (payment.three_ds?.iframe) {
  console.log('3DS challenge — render iframe in browser');

  // Poll for terminal status — getStatus() returns a Checkout object
  const MAX_ATTEMPTS = 60;
  const INTERVAL_MS = 3_000;

  for (let i = 0; i < MAX_ATTEMPTS; i++) {
    await new Promise((r) => setTimeout(r, INTERVAL_MS));
    const checkout = await bloque.payments.getStatus(payment.id);
    console.log(`[${i + 1}/${MAX_ATTEMPTS}] status: ${checkout.status}`);

    if (checkout.status === 'paid' || checkout.status === 'cancelled') {
      console.log('Terminal:', JSON.stringify(checkout, null, 2));
      break;
    }
  }
}
```

## 6) PSE bank transfer (server-side direct API)

```ts
const payment = await bloque.payments.create({
  paymentUrn: 'did:bloque:payments:abc123',
  payment: {
    type: 'pse',
    data: {
      email: 'user@example.com',
      personType: 'natural',
      documentType: 'CC',
      documentNumber: '1234567890',
      bankCode: '1007',         // Bancolombia
      name: 'Maria Lopez',
      phone: '+573001234567',
    },
  },
});

if (payment.checkout_url) {
  // Redirect user to complete PSE payment
  console.log('PSE redirect:', payment.checkout_url);
}
```

## 7) Cash payment (server-side direct API)

```ts
const payment = await bloque.payments.create({
  paymentUrn: 'did:bloque:payments:abc123',
  payment: {
    type: 'cash',
    data: {
      fullName: 'Carlos Martínez',
      email: 'carlos@example.com',
      documentType: 'CC',
      documentNumber: '1122334455',
    },
  },
});

if (payment.payment_code) {
  // Share the code with the user for in-store payment
  console.log('Cash code:', payment.payment_code);
}
```

## 8) Subscription checkout

```ts
const subscription = await bloque.checkout.create({
  name: 'Plan Profesional Mensual',
  description: 'Acceso completo a todas las herramientas premium',
  asset: 'USD/6',
  payment_type: 'subscription',
  items: [{ name: 'Suscripción mensual', amount: 29_000000, quantity: 1 }],
  subscription: {
    type: 'cron',
    cron: '0 0 1 * *',
    status: 'active',
  },
  success_url: 'https://yourapp.com/success',
  cancel_url: 'https://yourapp.com/cancel',
  redirect_url: 'https://yourapp.com/dashboard',
});

console.log('Subscription checkout:', subscription.url);
console.log('Payment type:', subscription.payment_type); // 'subscription'
```

## 9) List checkouts

```ts
const checkouts = await bloque.checkout.list({
  status: 'pending',
  limit: 10,
  order: 'desc',
});

for (const c of checkouts) {
  console.log(c.urn, c.status, c.amount_total);
}
```

## 10) Cancel a checkout

```ts
const cancelled = await bloque.checkout.cancel('did:bloque:payments:abc123');
console.log('Status:', cancelled.status); // 'cancelled'
```

## 11) Retrieve a public checkout (no auth)

```ts
const checkout = await bloque.checkout.retrievePublic('18f9c4e3-...');
console.log(checkout.urn, checkout.status);
```
