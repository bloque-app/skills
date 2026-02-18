# Cards and Spending Controls

## Overview

Cards are the spending medium. Every card is linked to a pocket (virtual account) via `ledgerId`. Spending controls determine *how* the card debits funds when a purchase happens.

Two modes:

| Mode | Config | Behavior |
|------|--------|----------|
| `default` | Single pocket | All merchants accepted. One pocket debited. |
| `smart` | Multi-pocket + MCC routing | Route transactions to pockets by merchant category. |

**Card Details URL:** Every card has a `detailsUrl` — a PCI-compliant signed URL to view the full card number, CVV, and expiry date. This URL **expires**. To get a fresh one, always call `user.accounts.get(card.urn)` right before displaying it. Do NOT cache `detailsUrl`.

## Default Spending Control

The simplest setup. One pocket, one card, all merchants accepted. Supports automatic currency conversion when the transaction currency differs from the card's asset.

```typescript
const pocket = await user.accounts.virtual.create({}, { waitLedger: true });

const card = await user.accounts.card.create(
  {
    name: 'My Everyday Card',
    ledgerId: pocket.ledgerId,
    metadata: {
      spending_control: 'default',   // Optional — this is the default
      preferred_asset: 'DUSD/6',
      default_asset: 'DUSD/6',
      fallback_asset: 'KSM/12',     // Used when default asset is unavailable
      currency_asset_map: {         // Maps transaction currencies to preferred assets
        USD: ['DUSD/6'],
        COP: ['COPM/2'],
      },
    },
  },
  { waitLedger: true },
);
```

**What happens on purchase**: Card debits the full amount (+ fees) from the linked pocket. Any merchant accepted. If the transaction currency matches an entry in `currency_asset_map`, that asset is preferred; otherwise `default_asset` and then `fallback_asset` are used.

## Smart Spending Control (MCC Routing)

Route transactions to different pockets based on Merchant Category Codes (MCC).

### How It Works

1. A purchase comes in with a Merchant Category Code (MCC)
2. The system checks each pocket in `priority_mcc` order
3. If the MCC matches a pocket's whitelist, debit from that pocket
4. If no match or insufficient funds, try the next pocket
5. Pockets without a whitelist entry are "catch-all" (accept any MCC)

### Basic Setup (Two Pockets)

```typescript
// Category pocket — only food purchases
const foodPocket = await user.accounts.virtual.create(
  { name: 'Food Budget' },
  { waitLedger: true },
);

// Catch-all pocket — everything else
const mainWallet = await user.accounts.virtual.create(
  { name: 'Main Wallet' },
  { waitLedger: true },
);

const card = await user.accounts.card.create(
  {
    name: 'Smart Card',
    ledgerId: mainWallet.ledgerId,
    metadata: {
      spending_control: 'smart',
      preferred_asset: 'DUSD/6',
      default_asset: 'DUSD/6',
      fallback_asset: 'KSM/12',
      mcc_whitelist: {
        [foodPocket.urn]: ['5411', '5812', '5814'],
      },
      priority_mcc: [foodPocket.urn, mainWallet.urn],
    },
  },
  { waitLedger: true },
);
```

**MCC whitelist as URL**: You can pass a URL that returns a JSON array of MCC codes instead of an inline array. That makes it easy to update whitelists without changing card metadata.

```typescript
mcc_whitelist: {
  [foodPocket.urn]: 'https://api.example.com/mcc/food',
  [transportPocket.urn]: ['4111', '4121', '4131'],
}
```

### Multi-Category Setup

```typescript
const foodPocket = await user.accounts.virtual.create({ name: 'Food' }, { waitLedger: true });
const transportPocket = await user.accounts.virtual.create({ name: 'Transport' }, { waitLedger: true });
const generalPocket = await user.accounts.virtual.create({ name: 'General' }, { waitLedger: true });

const card = await user.accounts.card.create(
  {
    name: 'Budget Master',
    ledgerId: generalPocket.ledgerId,
    metadata: {
      spending_control: 'smart',
      preferred_asset: 'DUSD/6',
      default_asset: 'DUSD/6',
      mcc_whitelist: {
        [foodPocket.urn]: ['5411', '5812', '5814'],
        [transportPocket.urn]: ['4111', '4121', '4131', '5541', '5542'],
        // generalPocket has NO entry → catch-all
      },
      priority_mcc: [foodPocket.urn, transportPocket.urn, generalPocket.urn],
    },
  },
  { waitLedger: true },
);
```

**Result**:
- Grocery (MCC 5411) → Food pocket
- Uber (MCC 4121) → Transport pocket
- Online shopping → General pocket (catch-all)
- If Food pocket is empty → falls through to Transport (no match) → General (catch-all)

**Refunds**: Refunds are routed back to the **original pockets** that were debited, proportionally.

## Update Spending Controls

Upgrade from default to smart, add new categories, or change routing — all via `updateMetadata`. No need to recreate the card.

```typescript
// Upgrade a default card to smart
await user.accounts.card.updateMetadata({
  urn: card.urn,
  metadata: {
    spending_control: 'smart',
    mcc_whitelist: {
      [foodPocket.urn]: ['5411', '5812', '5814'],
    },
    priority_mcc: [foodPocket.urn, mainWallet.urn],
  },
});

// Add a new category pocket
const entertainmentPocket = await user.accounts.virtual.create(
  { name: 'Entertainment' },
  { waitLedger: true },
);

// Send the FULL config (not a partial update)
await user.accounts.card.updateMetadata({
  urn: card.urn,
  metadata: {
    spending_control: 'smart',
    mcc_whitelist: {
      [foodPocket.urn]: ['5411', '5812', '5814'],
      [entertainmentPocket.urn]: ['7832', '7841', '7911'],
    },
    priority_mcc: [foodPocket.urn, entertainmentPocket.urn, mainWallet.urn],
  },
});
```

**Important**: `updateMetadata` replaces the full metadata object. Always send the complete spending control config, not just the changed fields.

## Fee Configuration (Spending Fees and Rules)

Fees are configured with the `spending_fees` array in metadata. Each fee can be unconditional or gated by a **rule** that evaluates the transaction at authorization time. Configurable at **origin** (all cards) or **card** (per card).

### Resolution Priority (Merge by Name)

Fees are **merged** across three layers by `fee_name`. Higher layers override or add; lower-layer fees are never removed.

| Layer | Source | Description |
|-------|--------|-------------|
| Base | **Defaults** | Built-in fees (e.g. 1.44% interchange + 2% FX). Always present unless overridden by name. |
| + | **Origin metadata** | `spending_fees` in the origin's metadata. Override defaults and add fees for all cards. |
| + | **Card metadata** | `spending_fees` in the card's `metadata`. Override origin/default for this card only. |

**Example**: Defaults have `interchange` and `fx`; origin adds `premium`; card overrides `fx` to 1.5%. Final: `interchange` (default), `fx` (1.5% from card), `premium` (from origin).

**Important**: Base fees cannot be removed. You can change `value`, `type`, `account_urn`, or `rule` by redefining the same `fee_name`, but you cannot delete a fee entirely.

### SpendingFee Shape

```typescript
type SpendingFeeCategory = 'fx' | 'interchange' | 'custom';

interface SpendingFee {
  fee_name: string;           // Unique id (e.g. "interchange", "fx")
  account_urn: string;       // Destination account for the fee
  type: 'percentage' | 'flat';
  value: number;             // Rate (0.0144 = 1.44%) or flat amount (scaled)
  category?: SpendingFeeCategory;  // Purpose: 'fx' | 'interchange' | 'custom' (defaults to 'custom')
  rule?: string;             // Optional: conditional rule name
  rule_params?: Record<string, unknown>;  // Optional: parameters for the rule
}
```

| Type | `value` meaning | Example |
|------|------------------|---------|
| `percentage` | Fraction of transaction amount | `0.0144` = 1.44% |
| `flat` | Fixed amount in the transaction's asset (scaled) | `100000` |

| Category | Purpose |
|----------|---------|
| `fx` | Drives the exchange rate spread in currency conversions. Multiple `fx` fees are summed. |
| `interchange` | Standard interchange fee. |
| `custom` | Default for any other fee. |

**FX Category and Conversion Spread**: Fees with `category: "fx"` control the exchange rate spread applied during currency conversions. For example, an `fx` fee with `value: 0.02` results in a 2% spread (rate multiplied by 0.98). If multiple fees have `category: "fx"`, their values are summed. If no `fx` category fees are configured, the default 2% spread is used.

### Defining Fees at the Card Level

Set `metadata.spending_fees` when creating or updating the card. Only include fees you want to override or add; others from origin/default are preserved.

```typescript
const card = await user.accounts.card.create(
  {
    name: 'Premium Card',
    ledgerId: pocket.ledgerId,
    metadata: {
      spending_control: 'default',
      default_asset: 'DUSD/6',
      spending_fees: [
        {
          fee_name: 'interchange',
          account_urn: 'urn:bloque:treasury:interchange',
          type: 'percentage',
          value: 0.01,         // 1% for this card
        },
        {
          fee_name: 'fx',
          account_urn: 'urn:bloque:treasury:fx',
          type: 'percentage',
          value: 0.015,
          rule: 'fx_conversion',   // Only when currency conversion occurred
        },
      ],
    },
  },
  { waitLedger: true },
);
```

Origin-level fees are configured via the Bloque dashboard or API (origin metadata `spending_fees`). Card-level entries override by `fee_name`.

### Conditional Fee Rules

Fees without `rule` always apply. With `rule`, the fee is applied only when the rule matches the transaction at authorization time.

| Rule | Description | Parameters |
|------|-------------|------------|
| `fx_conversion` | Applies when the transaction required currency conversion (spending asset ≠ transaction currency). | None |
| `amount_range_usd` | Applies when the USD settlement amount is within a range. | `min?: number`, `max?: number` |
| `wallet` | Applies when the transaction was made through a specific tokenization wallet (case-insensitive partial match). | `wallet_name: string` |

```typescript
// Example: higher FX fee for high-ticket USD transactions
{
  fee_name: 'high_ticket_fx',
  account_urn: 'urn:bloque:treasury:fx',
  type: 'percentage',
  value: 0.035,
  rule: 'amount_range_usd',
  rule_params: { min: 500, max: 5000 },
}
```

Fees are calculated with BigInt; on partial refunds, fees are recalculated proportionally.

### Fee Breakdown in Movements and Webhooks

Every successful card transaction includes a `fee_breakdown` in movement metadata and in webhook payloads (see `transfers.md` and `webhooks.md`):

```typescript
interface FeeBreakdown {
  total_fees: string;          // Total fees as scaled BigInt string
  fees: Array<{
    fee_name: string;          // Fee identifier
    amount: string;            // Fee amount as scaled BigInt string
    rate: number;              // The rate or flat value used
  }>;
}
```

## MCC Code Reference

| Category | MCCs | Description |
|----------|------|-------------|
| **Food** | `5411` | Grocery stores |
| | `5812` | Restaurants |
| | `5814` | Fast food |
| **Transport** | `4111` | Local commuter transport |
| | `4121` | Taxis and rideshares |
| | `4131` | Bus lines |
| | `5541` | Gas stations |
| | `5542` | Fuel dealers |
| **Entertainment** | `7832` | Movies |
| | `7841` | Streaming services |
| | `7911` | Entertainment events |
| **Health** | `5912` | Pharmacies |
| | `8011` | Doctors |
| | `8021` | Dentists |
| **Shopping** | `5311` | Department stores |
| | `5651` | Clothing stores |
| | `5691` | Clothing accessories |

## Metadata Fields Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spending_control` | `'default' \| 'smart'` | No | Control type. Defaults to `'default'`. |
| `preferred_asset` | `string` | No | Preferred asset for transactions (e.g., `'DUSD/6'`). |
| `default_asset` | `string` | No | Primary asset. Defaults to `'DUSD/6'`. |
| `fallback_asset` | `string` | No | Fallback when default is unavailable. |
| `currency_asset_map` | `Record<string, string[]>` | No | Maps ISO 4217 currency codes to preferred asset arrays. |
| `mcc_whitelist` | `Record<pocketUrn, string[] \| string>` | Smart only | Pocket URN → array of MCC codes, or URL returning a JSON array. |
| `priority_mcc` | `string[]` | Smart only | Ordered list of pocket URNs. Checked top-to-bottom. |
| `spending_fees` | `SpendingFee[]` | No | Fee overrides/additions for this card. Merged with origin/default by `fee_name`. See Fee Configuration. |

## Processing Details

### Multi-Currency

The system supports multi-currency transactions:

1. **Direct match**: If the transaction currency matches an available asset (e.g., COP with COPM balance), no conversion.
2. **Conversion**: If no direct match, the system converts using exchange rates (e.g., COP → USD debit).
3. **Asset preference**: Use `currency_asset_map` to map currency codes to preferred assets; otherwise `default_asset` and `fallback_asset` apply.

### Concurrency (Smart Spending)

Smart spending uses Redis locks to prevent race conditions when multiple transactions hit the same pockets simultaneously. Locks are acquired in sorted order to prevent deadlocks and released after the batch executes.
