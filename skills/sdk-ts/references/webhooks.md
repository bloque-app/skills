# Webhooks

Webhooks deliver real-time notifications about card transactions to your server.

## Setup

Set `webhookUrl` when creating a card:

```typescript
const card = await user.accounts.card.create(
  {
    ledgerId: pocket.ledgerId,
    webhookUrl: 'https://api.example.com/webhooks/card',
    metadata: { spending_control: 'default', preferred_asset: 'DUSD/6' },
  },
  { waitLedger: true },
);
```

Or update it on a batch transfer:

```typescript
await user.accounts.batchTransfer({
  operations: [...],
  webhookUrl: 'https://api.example.com/webhooks/batch',
});
```

## Event Types

| Event | Type | Direction | Description |
|-------|------|-----------|-------------|
| `purchase` | `authorization` | `debit` | Card purchase authorized and debited. |
| `rejected_insufficient_funds` | `authorization` | `debit` | Purchase rejected — insufficient balance. |
| `rejected_credit` | `authorization` | `credit` | Credit authorization rejected (not supported). |
| `rejected_currency` | `authorization` | `debit` | Purchase rejected — unsupported currency. |
| `credit_adjustment` | `adjustment` | `credit` | Refund or credit adjustment processed. |
| `debit_adjustment` | `adjustment` | `debit` | Debit adjustment processed. |

## Webhook Payload Shape

```typescript
type CardEventType = 
  | 'purchase'
  | 'rejected_insufficient_funds'
  | 'rejected_credit'
  | 'rejected_currency'
  | 'credit_adjustment'
  | 'debit_adjustment';

interface WebhookPayload {
  /** URN of the card account */
  account_urn: string;

  /** Unique transaction identifier */
  transaction_id: string;

  /** Event classification */
  type: 'authorization' | 'adjustment';

  /** Fund flow direction */
  direction: 'debit' | 'credit';

  /** Specific event type */
  event: CardEventType;

  /** Amount debited/credited in the ledger asset (scaled BigInt string) */
  amount?: string;

  /** Asset used for the debit/credit (e.g., "DUSD/6") */
  asset?: string;

  /** Purchase amount in the merchant's local currency */
  local_amount?: number;

  /** ISO 4217 currency code (e.g., "USD", "COP") */
  local_currency?: string;

  /** Conversion rate applied (1 if direct match) */
  exchange_rate?: number;

  /** Merchant information */
  merchant?: {
    name: string;
    mcc: string;          // Merchant Category Code
    city?: string;
    country?: string;
    address?: string;
    terminal_id?: string;
  };

  /** Transaction medium details */
  medium?: {
    entry_mode: string;           // e.g., "CONTACTLESS", "CHIP", "CREDENTIAL_ON_FILE"
    point_type: string;           // e.g., "POS", "ECOMMERCE"
    origin: string;               // e.g., "DOMESTIC", "INTERNATIONAL"
    network: string;              // e.g., "VISA", "MASTERCARD"
    cardholder_presence?: string;
    pin_presence?: string;
    tokenization_wallet_name?: string; // e.g., "apple_pay", "google_pay"
  };

  /** Fee breakdown (present on successful transactions) */
  fee_breakdown?: {
    total_fees: string;          // Total fees as scaled BigInt string
    fees: Array<{
      fee_name: string;          // Fee identifier
      amount: string;            // Fee amount as scaled BigInt string
      rate: number;              // The rate or flat value used
    }>;
  };

  /** Reason for rejection (present on rejection events) */
  reason?: string;

  /** Required USD amount (present on some rejection events) */
  required_usd?: number;
}
```

## Example Payloads

### Purchase Approved (Authorization Debit)

```json
{
  "account_urn": "did:bloque:account:card:usr-abc:crd-123",
  "transaction_id": "ctx-200kXoaEJLNzcsvNxY1pmBO7fEx",
  "type": "authorization",
  "direction": "debit",
  "event": "purchase",
  "amount": "50000000",
  "asset": "DUSD/6",
  "local_amount": 50.0,
  "local_currency": "USD",
  "exchange_rate": 1,
  "merchant": {
    "name": "AMAZON.COM",
    "mcc": "5411",
    "city": "Seattle",
    "country": "USA"
  },
  "medium": {
    "entry_mode": "CREDENTIAL_ON_FILE",
    "point_type": "ECOMMERCE",
    "origin": "INTERNATIONAL",
    "network": "VISA"
  },
  "fee_breakdown": {
    "total_fees": "720000",
    "fees": [
      { "fee_name": "interchange", "amount": "720000", "rate": 0.0144 }
    ]
  }
}
```

### Refund (Adjustment Credit)

```json
{
  "account_urn": "did:bloque:account:card:usr-abc:crd-123",
  "transaction_id": "ctx-adj-456",
  "type": "adjustment",
  "direction": "credit",
  "event": "credit_adjustment",
  "amount": "25000000",
  "asset": "DUSD/6",
  "local_amount": 25.0,
  "local_currency": "USD",
  "exchange_rate": 1,
  "merchant": {
    "name": "AMAZON.COM",
    "mcc": "5411",
    "city": "Seattle",
    "country": "USA"
  },
  "fee_breakdown": {
    "total_fees": "360000",
    "fees": [
      { "fee_name": "interchange", "amount": "360000", "rate": 0.0144 }
    ]
  }
}
```


### Purchase Rejected (Insufficient Funds)

When a purchase is rejected, the webhook is still sent but with no `fee_breakdown` (since no fees were charged) and the amount reflects what was attempted.

```json
{
  "account_urn": "did:bloque:account:card:usr-abc:crd-123",
  "transaction_id": "ctx-789",
  "type": "authorization",
  "direction": "debit",
  "event": "rejected_insufficient_funds",
  "required_usd": 150.0,
  "reason": "Insufficient funds"
}
```

## Webhook Handler Examples

### Basic Handler (Express / Hono / Fastify)

```typescript
app.post('/webhooks/card', async (req, res) => {
  const payload = req.body as WebhookPayload;

  switch (payload.event) {
    case 'purchase':
      console.log(`Purchase: ${payload.local_amount} ${payload.local_currency}`);
      console.log(`Merchant: ${payload.merchant?.name} (MCC ${payload.merchant?.mcc})`);
      console.log(`Debited: ${payload.amount} ${payload.asset}`);
      console.log(`Fees: ${payload.fee_breakdown?.total_fees}`);

      await notifyUser(payload.account_urn, {
        type: 'purchase',
        merchant: payload.merchant?.name,
        amount: payload.local_amount,
        currency: payload.local_currency,
      });
      break;

    case 'credit_adjustment':
      console.log(`Refund: ${payload.amount} ${payload.asset}`);
      await notifyUser(payload.account_urn, { type: 'refund', amount: payload.amount });
      break;

    case 'rejected_insufficient_funds':
      console.log(`Rejected: ${payload.reason}`);
      console.log(`Required: ${payload.required_usd} USD`);
      break;
  }

  res.status(200).json({ received: true });
});
```

### Full Handler with Idempotency and Logging

```typescript
const processedEvents = new Set<string>();

app.post('/webhooks/card', async (req, res) => {
  const event = req.body as WebhookPayload;

  // Idempotency check — webhook may be delivered more than once
  const eventKey = `${event.type}-${event.transaction_id}`;
  if (processedEvents.has(eventKey)) {
    return res.status(200).json({ received: true, deduplicated: true });
  }
  processedEvents.add(eventKey);

  // Log for audit
  console.log(JSON.stringify({
    webhook: payload.event,
    type: payload.type,
    direction: payload.direction,
    account: payload.account_urn,
    tx: payload.transaction_id,
    merchant: payload.merchant?.name,
    mcc: payload.merchant?.mcc,
    local: `${payload.local_amount} ${payload.local_currency}`,
    debited: `${payload.amount} ${payload.asset}`,
    entry: payload.medium?.entry_mode,
    wallet: payload.medium?.tokenization_wallet_name || 'physical',
    fees: payload.fee_breakdown?.total_fees,
  }));

  // Route by event type
  if (payload.event === 'purchase') {
    // Purchase — update balance UI, send push notification
    await db.transactions.create({
      accountUrn: payload.account_urn,
      transactionId: payload.transaction_id,
      merchantName: payload.merchant?.name,
      merchantMcc: payload.merchant?.mcc,
      localAmount: payload.local_amount,
      localCurrency: payload.local_currency,
      amount: payload.amount,
      asset: payload.asset,
      fees: payload.fee_breakdown?.total_fees,
      entryMode: payload.medium?.entry_mode,
      wallet: payload.medium?.tokenization_wallet_name,
    });

    await pushNotification(payload.account_urn, {
      title: `Purchase at ${payload.merchant?.name}`,
      body: `${payload.local_amount} ${payload.local_currency}`,
    });
  }

  if (payload.event === 'credit_adjustment') {
    // Refund — update original transaction, notify user
    await db.transactions.update({
      transactionId: payload.transaction_id,
      refundAmount: payload.amount,
    });

    await pushNotification(payload.account_urn, {
      title: 'Refund received',
      body: `${payload.local_amount} ${payload.local_currency} from ${payload.merchant?.name}`,
    });
  }

  res.status(200).json({ received: true });
});
```

## Transaction Lifecycle

Understanding the full lifecycle of a card transaction:

```
                         ┌──────────────┐
   Card swipe/tap ──────▶│  Authorization│
                         │   Request     │
                         └──────┬───────┘
                                │
                    ┌───────────▼────────────┐
                    │  Resolve Spending       │
                    │  Control (default/smart)│
                    └───────────┬────────────┘
                                │
              ┌─────────────────▼──────────────────┐
              │                                     │
     ┌────────▼────────┐              ┌─────────────▼─────────┐
     │ Default Control │              │  Smart Control         │
     │ Debit pocket    │              │  1. Match MCC          │
     │                 │              │  2. Walk priority list  │
     │                 │              │  3. Debit best pocket   │
     └────────┬────────┘              └─────────────┬─────────┘
              │                                     │
              └──────────────┬──────────────────────┘
                             │
                    ┌────────▼────────┐
                    │ Execute Batch   │
                    │ Movements       │
                    │ (atomic)        │
                    └────────┬────────┘
                             │
               ┌─────────────▼──────────────┐
               │  Fire Events               │
               │  • Webhook (async)         │
               │  • WhatsApp notification   │
               └─────────────┬──────────────┘
                             │
                    ┌────────▼────────┐
                    │ Return APPROVED │
                    │ or REJECTED     │
                    └─────────────────┘
```

### Key Points

- **Idempotent delivery**: Webhooks use idempotency keys (`{type}-{transactionId}`). Your handler should be idempotent.
- **Async delivery**: Webhooks are delivered asynchronously via a task queue. Expect slight delay (seconds).
- **Retry**: Failed webhook deliveries are retried automatically.
- **Refund routing (smart spending)**: Refunds go to the main wallet (last pocket in `priority_mcc`), not the original category pocket.
- **Fee handling on refunds**: Credit adjustments return fees to the platform first, then credit the user account.
