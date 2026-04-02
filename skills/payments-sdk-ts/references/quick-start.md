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
      paymentMethods={['card', 'pse']}
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
  checkoutId: 'checkout_abc123',
  payment: {
    type: 'card',
    data: {
      cardNumber: '4111111111111111',
      cardholderName: 'John Doe',
      expiryMonth: '12',
      expiryYear: '2028',
      cvv: '123',
      email: 'john@example.com',
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

  // Poll for terminal status
  const MAX_ATTEMPTS = 60;
  const INTERVAL_MS = 3_000;

  for (let i = 0; i < MAX_ATTEMPTS; i++) {
    await new Promise((r) => setTimeout(r, INTERVAL_MS));
    const status = await bloque.payments.getStatus(payment.id);
    console.log(`[${i + 1}/${MAX_ATTEMPTS}] status: ${status.status}`);

    if (status.status === 'completed' || status.status === 'failed') {
      console.log('Terminal:', JSON.stringify(status, null, 2));
      break;
    }
  }
}
```
