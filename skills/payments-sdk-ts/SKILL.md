---
name: bloque-payments-sdk-ts
description: Integración del SDK de pagos de Bloque en TypeScript/JavaScript. Úsala cuando debas crear checkouts, procesar pagos con @bloque/payments, verificar webhooks o embeber el checkout con @bloque/payments-core o @bloque/payments-react.
---

# Bloque Payments SDK TS

Implementa integraciones de pago con los paquetes `@bloque/payments`, `@bloque/payments-core` y `@bloque/payments-react`.

## Flujo recomendado

1. Crear checkout en backend con `@bloque/payments`.
2. Entregar `checkoutId` al frontend.
3. Montar checkout hosted con `@bloque/payments-core` o `@bloque/payments-react`.
4. Verificar webhooks en backend antes de procesar eventos.

## Reglas de implementación

- Usar `accessToken` al inicializar `new Bloque(...)`.
- Tratar el SDK de `@bloque/payments` como backend-only.
- Verificar firma HMAC-SHA256 de webhooks con `bloque.webhooks.verify(...)`.
- En frontend, usar `publicApiKey` (no `accessToken`).
- Mantener `mode` consistente entre backend y frontend (`sandbox` o `production`).

## Selección rápida de paquete

- Backend API de pagos: `@bloque/payments`
- Iframe hosted checkout (vanilla JS/TS): `@bloque/payments-core`
- Componente React para hosted checkout: `@bloque/payments-react`

## Referencias

- `references/quick-start.md`: Arranque rápido backend + frontend.
- `references/api-reference.md`: Métodos, tipos y payloads principales.
- `references/implementation-notes.md`: Diferencias detectadas entre README y código fuente, con guardrails de uso.
