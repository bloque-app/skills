# Quick Start

## 1) Backend: crear checkout

```ts
import { Bloque } from '@bloque/payments';

const bloque = new Bloque({
  mode: 'sandbox',
  accessToken: process.env.BLOQUE_ACCESS_TOKEN!,
});

const checkout = await bloque.checkout.create({
  name: 'Pack Profesional de Productividad',
  description: 'Periféricos premium',
  currency: 'COP',
  image_url: 'https://cdn.bloque.app/checkouts/productivity-pack.png',
  items: [
    {
      name: 'Teclado mecánico Keychron K2',
      amount: 450_000,
      quantity: 2,
      image_url: 'https://cdn.bloque.app/items/keychron-k2.png',
    },
  ],
  success_url: 'https://tuapp.com/success',
  cancel_url: 'https://tuapp.com/cancel',
});

console.log(checkout.id, checkout.url);
```

## 2) Frontend JS/TS: montar checkout iframe

```ts
import { createCheckout, init } from '@bloque/payments-core';

init({
  publicApiKey: 'pk_test_xxx',
  mode: 'sandbox',
});

const checkout = createCheckout({
  checkoutId: 'checkout_123abc',
  container: '#checkout-container',
  onSuccess: (data) => console.log('approved', data),
  onPending: (data) => console.log('pending', data),
  onError: (error) => console.error('error', error),
});

// later
checkout.destroy();
```

## 3) Frontend React: componente

```tsx
import { BloqueCheckout } from '@bloque/payments-react';

export function CheckoutPage({ checkoutId }: { checkoutId: string }) {
  return (
    <BloqueCheckout
      checkoutId={checkoutId}
      publicApiKey="pk_test_xxx"
      mode="sandbox"
      paymentMethods={['card', 'pse']}
      onSuccess={(data) => console.log(data)}
      onError={(error) => console.error(error)}
    />
  );
}
```

## 4) Webhooks: validar firma

```ts
import { Bloque } from '@bloque/payments';

const bloque = new Bloque({
  mode: 'production',
  accessToken: process.env.BLOQUE_ACCESS_TOKEN!,
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
