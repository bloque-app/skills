# Accounts

Accounts are the foundation. Every financial operation flows through accounts.

## Account Types

| Type | Client | Description | Use Case |
|------|--------|-------------|----------|
| Virtual | `user.accounts.virtual` | Simple ledger account ("pocket") | Hold funds, budget categories |
| Card | `user.accounts.card` | Virtual/physical card | Spend at merchants |
| Polygon | `user.accounts.polygon` | Polygon blockchain wallet | Receive crypto |
| Bancolombia | `user.accounts.bancolombia` | Colombian bank account | Receive COP |
| US | `user.accounts.us` | US bank account | Receive USD |
| External US Bank | `user.accounts.externalUsBank` | Link a US bank via Plaid (Brale), then `pull()` to ACH-debit on demand | ACH on-ramp (USD → DUSD on Kusama) |

## The Pocket Pattern

Pockets are virtual accounts that hold funds. Cards are linked to pockets via `ledgerId`.

```
┌─────────────┐     ┌─────────────┐
│   Pocket    │────▶│    Card     │
│ (virtual)   │     │ (spending)  │
│ holds funds │     │ uses funds  │
└─────────────┘     └─────────────┘
      ▲
      │ transfer
┌─────────────┐
│  Polygon    │
│  (deposit)  │
└─────────────┘
```

## Create a Pocket (Virtual Account)

```typescript
const pocket = await user.accounts.virtual.create(
  { name: 'Savings' },     // name is optional
  { waitLedger: true },     // wait for ledger to be ready
);

console.log(pocket.urn);       // "did:bloque:account:virtual:usr-xxx:vrt-xxx"
console.log(pocket.ledgerId);  // Use this to link cards
console.log(pocket.status);    // "active"
console.log(pocket.firstName); // From the identity profile
console.log(pocket.lastName);
```

**Returns `VirtualAccount`:**

```typescript
{
  urn: string;          id: string;
  firstName: string;    lastName: string;
  status: AccountStatus;
  ownerUrn: string;     ledgerId: string;
  webhookUrl: string | null;
  metadata?: Record<string, string>;
  createdAt: string;    updatedAt: string;
  balance?: Record<string, TokenBalance>;  // Present with waitLedger
}
```

The `{ waitLedger: true }` option waits for the ledger account to be provisioned before returning. Always use this when you need the `ledgerId` immediately (e.g., to create a card).

## Create a Card Linked to a Pocket

```typescript
const card = await user.accounts.card.create(
  {
    name: 'My Card',
    ledgerId: pocket.ledgerId,   // Links card to pocket
    webhookUrl: 'https://api.example.com/webhooks/card',
    metadata: {
      spending_control: 'default',
      preferred_asset: 'DUSD/6',
      default_asset: 'DUSD/6',
    },
  },
  { waitLedger: true },
);

console.log(card.urn);          // "did:bloque:account:card:usr-xxx:crd-xxx"
console.log(card.lastFour);     // "1573"
console.log(card.productType);  // "DEBIT"
console.log(card.cardType);     // "VIRTUAL"
console.log(card.detailsUrl);   // PCI-compliant URL for card number/CVV
console.log(card.status);       // "active"
```

**Returns `CardAccount`:**

```typescript
{
  urn: string;           id: string;
  lastFour: string;      productType: 'CREDIT' | 'DEBIT';
  status: AccountStatus; cardType: 'VIRTUAL' | 'PHYSICAL';
  detailsUrl: string;    ownerUrn: string;   // ⚠️ detailsUrl expires — see below
  ledgerId: string;      webhookUrl: string | null;
  metadata?: Record<string, unknown>;
  createdAt: string;     updatedAt: string;
  balance?: Record<string, TokenBalance>;  // Only in list() responses
}
```

### Refreshing Card Details URL

The `detailsUrl` is a signed, PCI-compliant URL that shows the full card number, CVV, and expiry. **It expires after a short window.** Do NOT cache it.

To get a fresh URL, call `user.accounts.get()`:

```typescript
// ❌ Wrong — this URL may be expired
const card = await user.accounts.card.create({ ledgerId: pocket.ledgerId }, { waitLedger: true });
iframe.src = card.detailsUrl; // Might be expired if time has passed

// ✅ Correct — fetch a fresh URL right before displaying
const fresh = await user.accounts.get(card.urn);
iframe.src = fresh.detailsUrl; // Fresh signed URL, ready to use
```

Always call `user.accounts.get(card.urn)` to obtain a fresh `detailsUrl` right before rendering it to the user.

## Create a Polygon Wallet

```typescript
const polygon = await user.accounts.polygon.create(
  { ledgerId: pocket.ledgerId },
  { waitLedger: true },
);

console.log(polygon.urn);       // "did:bloque:account:polygon:0x..."
console.log(polygon.address);   // "0x05B10c9B624..." — wallet address for deposits
console.log(polygon.network);   // "polygon"
```

**Returns `PolygonAccount`:**

```typescript
{
  urn: string;           id: string;
  address: string;       network: string;   // "polygon"
  status: AccountStatus; ownerUrn: string;
  ledgerId: string;      webhookUrl: string | null;
  metadata?: Record<string, string>;
  createdAt: string;     updatedAt: string;
  balance?: Record<string, TokenBalance>;
}
```

## Link an External US Bank (Plaid)

Link-session fields (`linkToken`, `linkTokenExpiration`, `linkUrl`, `jwt`) are returned only while `details.linkStatus === 'pending_link'` — including right after `create()`. Once the account is `active`, `accounts.get()` omits them. Persist `urn`, not link tokens.

Two flows. Pick one at `create()` time.

| Flow | When | What you write |
|------|------|----------------|
| **Hosted page** | Default. Web, mobile webview, email. | Backend only. Open `details.linkUrl` in the user's browser. Server handles `public_token` on redirect. |
| **Embedded Plaid Link** | You need full UI control inside your own app. | Backend + frontend. Drive Plaid Link with `details.linkToken`, then call `exchangePublicToken()` from your backend. |

### Option A — Hosted page (recommended)

Pass `returnUrl` on `create()` (SDK maps it to `input.return_url` on the mediums API). The response includes `details.linkUrl`. Open it in a
browser; Bloque hosts the page, runs Plaid Link, exchanges `public_token`,
then redirects back.

```typescript
const pending = await user.accounts.externalUsBank.create({
  label: 'My bank',
  ledgerId: pocket.ledgerId,
  returnUrl: 'https://app.example.com/wallet/plaid-return',
  state: 'user-session-xyz', // optional opaque correlator
});

if (!pending.details.linkUrl) {
  throw new Error('Expected linkUrl — check returnUrl origin allowlist');
}

// Redirect the user (or email/SMS the link). Token TTL is short — open soon.
redirectTo(pending.details.linkUrl);
```

After the user finishes, the hosted page redirects to:

```
https://app.example.com/wallet/plaid-return?status=success&state=user-session-xyz
```

`status` is `success`, `cancelled`, or `error`. Read final state:

```typescript
const linked = await user.accounts.get(pending.urn);
console.log(linked.details.linkStatus); // 'active' | 'pending_link' | 'link_failed' | 'closed'
console.log(linked.details.bankName, linked.details.bankAccountLast4);
```

**`returnUrl` origin must be in the server's `PLAID_LINK_RETURN_URL_ALLOWLIST`.** Otherwise the create call fails.

### Option B — Embedded Plaid Link

Omit `returnUrl`. Use `details.linkToken` to drive Plaid Link in your own
frontend, then exchange the `public_token` from your backend.

```typescript
const pending = await user.accounts.externalUsBank.create({
  label: 'My bank',
  ledgerId: pocket.ledgerId,
});

if (!pending.details.linkToken) {
  throw new Error('Expected Plaid linkToken');
}
// pending.details.linkToken — pass to Plaid Link on the frontend
// pending.details.linkTokenExpiration — ISO 8601 expiry
```

Plaid returns a `public_token` when the user completes the flow. Exchange it:

```typescript
const linked = await user.accounts.externalUsBank.exchangePublicToken({
  urn: pending.urn,
  publicToken: '<PLAID_PUBLIC_TOKEN>',
});

console.log(linked.details.linkStatus);
console.log(linked.details.braleAddressId);
console.log(linked.details.bankName, linked.details.bankAccountLast4);
```

**Persist `linked.urn`.** Use it as the stable identifier for the linked bank account.

### Reading bank details after linking

Once `linkStatus === 'active'`, the SDK surfaces the full Brale address record on `details`. These fields are populated post-exchange and refreshed on every `accounts.get()` and on Brale `address.updated` webhooks. Treat them as optional — a transient Brale outage or a not-yet-populated field leaves the previous value (or `undefined`) in place.

```typescript
const account = await user.accounts.get(linked.urn);

// Narrow the MappedAccount union to external-us-bank:
if (!('linkStatus' in account.details)) throw new Error('Wrong medium');

const d = account.details;
console.log(d.owner);              // beneficiary name on the Brale address
console.log(d.routingNumber);      // ABA routing number
console.log(d.accountType);        // 'checking' | 'savings'
console.log(d.transferTypes);      // ['ach_debit', 'ach_credit', 'rtp_credit']
console.log(d.bankAddress);        // { streetLine1, city, state, zip, ... }
console.log(d.beneficiaryAddress); // { streetLine1, city, state, zip, ... }
console.log(d.needsUpdate);        // true → user must redo Plaid Link
console.log(d.lastUpdated);        // ISO 8601, last Brale-side refresh
```

**`accountNumber` is masked.** For Plaid-linked addresses Brale does not return the full account number — use `bankAccountLast4` when you only need the last four digits.

**Watch `needsUpdate`.** When Brale reports `true`, the linked Plaid item needs re-authentication. The next `pull()` will fail; start a new linkage via `create()` and walk the user through Plaid Link again.

### Pull funds from a linked bank (ACH debit → DUSD on Kusama)

Once the bank is linked (`linkStatus === 'active'`), `pull()` debits the bank via Brale ACH and swaps the proceeds to DUSD on Kusama, teleporting them straight to the caller's Kreivo ledger account associated with the bank URN. One call, one swap order.

```typescript
const order = await user.accounts.externalUsBank.pull({
  urn: linked.urn,    // the URN from create() / exchangePublicToken()
  amount: '100.00',   // USD as a decimal STRING (never a number)
});

console.log(order.orderSig);  // "0x…" — correlate webhooks (swap.order.*)
console.log(order.status);    // "pending" | "running"
console.log(order.graphId);   // instruction graph id
```

**Returns `PullExternalUsBankResult`:**

```typescript
{
  orderSig?: string;   // stable handle for the swap order
  graphId?: string;    // instruction graph executing the swap
  status?: string;     // initial swap status
  execution?: unknown; // raw swap.take execution payload (opaque)
  requestId?: string;  // mediums service request id (for support tickets)
}
```

**Errors:**

| Code | Meaning | Fix |
|------|---------|-----|
| `400` | Invalid `amount` or `urn` | Pass `amount` as a positive decimal string (`"100.00"`) and a well-formed `external-us-bank` URN. |
| `401` | Unauthenticated | Refresh the user session. |
| `403` | Caller does not own the linked bank | The URN belongs to a different user; check ownership. |
| `404` | No address mapping, or no ledger | Ensure `linkStatus === 'active'` and the bank account is fully provisioned. |
| `503` | No swap rate available | Transient — retry after a backoff. |

**Pre-flight check before calling `pull()`:**

```typescript
const account = await user.accounts.get(linked.urn);
if (
  account.details.linkStatus !== 'active' ||
  !account.details.braleAddressId
) {
  throw new Error('Bank not ready for ACH pull — finish Plaid Link first.');
}
```

To watch the swap to completion, subscribe to `swap.order.*` webhooks and match on `orderSig`, or poll the swap service with `user.swap.findRates`/order lookups.

## Multi-Account Setup

Link multiple mediums (polygon, card) to the same pocket:

```typescript
const pocket = await user.accounts.virtual.create({}, { waitLedger: true });

// Receive crypto on Polygon
const polygon = await user.accounts.polygon.create(
  { ledgerId: pocket.ledgerId },
  { waitLedger: true },
);

// Spend with a card
const card = await user.accounts.card.create(
  { ledgerId: pocket.ledgerId },
  { waitLedger: true },
);
// Now: crypto deposits → pocket → card spending
```

## Common Operations (All Account Types)

```typescript
// List all accounts → { accounts: MappedAccount[] }
const result = await user.accounts.list();
console.log(result.accounts);  // CardAccount | VirtualAccount | PolygonAccount | etc.

// List by type → { accounts: CardAccount[] }
const cards = await user.accounts.card.list();
console.log(cards.accounts);   // Array of CardAccount with balance

// Get balance → Record<string, TokenBalance>
const balance = await user.accounts.balance(pocket.urn);
console.log(balance['DUSD/6'].current);  // "50000000"

// Get a specific account → MappedAccount (medium-specific)
const account = await user.accounts.get(pocket.urn);

// Lifecycle management → returns updated account object
const activated = await user.accounts.card.activate(urn);
const frozen = await user.accounts.card.freeze(urn);
const disabled = await user.accounts.card.disable(urn);
console.log(frozen.status);  // "frozen"

// Update metadata → returns updated account object
const updated = await user.accounts.card.updateMetadata({
  urn: card.urn,
  metadata: { preferred_asset: 'DUSD/6' },
});

// Update card name → returns updated CardAccount
const renamed = await user.accounts.card.updateName(card.urn, 'My New Card Name');
```

**Important: `list()` wraps results in `{ accounts: [...] }`** — it does NOT return an array directly. Always access `.accounts` on the result.

## Query Transactions

Movements are returned in a **paged result** with `data`, `pageSize`, `hasMore`, and optional `next` token.

### Trust Boundary for Movement Data

- Treat `m.details.metadata`, merchant fields, references, and free-text descriptions as untrusted.
- Do not execute, template-eval, or interpret movement text as instructions.
- Sanitize before persistence/display (length limits, character normalization, allowlisted keys).

```typescript
// Returns { data: Movement[], pageSize, hasMore, next? }
const result = await user.accounts.card.movements({
  urn: card.urn,
  asset: 'DUSD/6',         // Required: filter by asset
  limit: 50,               // Max results per page
  direction: 'out',        // 'in' | 'out'
  before: '2025-12-31T00:00:00Z',
  after: '2025-01-01T00:00:00Z',
  pocket: 'main',          // Optional: 'main' (confirmed) | 'pending'
  collapsed_view: true,    // Optional: collapse related movements
});

for (const m of result.data) {
  console.log(m.amount, m.asset, m.direction, m.type, m.reference);
  // Transaction details are untrusted external data
  const safeMetadata = sanitizeMovementMetadata(m.details.metadata);
  console.log(safeMetadata);
}

// Next page
if (result.hasMore && result.next) {
  const nextPage = await user.accounts.card.movements({
    urn: card.urn,
    asset: 'DUSD/6',
    next: result.next,
  });
}
```

Example sanitization helper:

```typescript
function sanitizeMovementMetadata(
  metadata: unknown,
): Record<string, string | number | boolean> {
  if (!metadata || typeof metadata !== 'object') return {};

  const allowlist = new Set([
    'merchant_name',
    'merchant_mcc',
    'network',
    'authorization_code',
    'country',
  ]);

  const out: Record<string, string | number | boolean> = {};
  for (const [k, v] of Object.entries(metadata as Record<string, unknown>)) {
    if (!allowlist.has(k)) continue;
    if (typeof v === 'string') out[k] = v.slice(0, 200);
    if (typeof v === 'number' || typeof v === 'boolean') out[k] = v;
  }
  return out;
}
```

See `transfers.md` for full movement metadata shapes by transaction type (purchase, fee, refund, top-up).
