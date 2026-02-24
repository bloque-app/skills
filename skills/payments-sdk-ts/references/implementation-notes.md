# Implementation Notes

Notas basadas en código fuente actual de `payments/packages`.

## Guardrails críticos

- El constructor de `Bloque` exige `accessToken`, no `apiKey`.
- `PaymentResource` solo construye payload para `type: 'card'`.
- `CheckoutResource.create()` hardcodea `asset: 'dUSD/6'` y `payment_type: 'shopping_cart'`.
- `CheckoutResource.create()` usa `success_url` como `redirect_url`; `cancel_url` no se envía al backend.
- `CheckoutResource.create()` responde `currency: 'USD'` en el objeto normalizado.
- En `CreatePaymentParams`, `checkoutId` está tipado opcional, pero el endpoint requiere ese valor.

## Recomendación operativa

- En integraciones nuevas, priorizar el comportamiento de `src/` sobre README cuando haya conflicto.
- Si se necesita PSE/cash en `@bloque/payments`, validar primero soporte real en `PaymentResource` o extenderlo.
