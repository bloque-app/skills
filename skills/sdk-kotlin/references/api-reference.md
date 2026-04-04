# API Reference — Kotlin SDK

Every SDK method, its parameters, and the exact shape of the returned object.

---

## BloqueSDK (Top Level)

### `BloqueSDK.builder()`

```kotlin
val bloque = BloqueSDK.builder()
    .secretKey("sk_test_...")      // sk_ key — auto-exchanged for JWT (recommended)
    // OR:
    // .origin("my-origin")        // Required for originKey auth
    // .originKey("my-origin-key") // Legacy origin-scoped key
    .mode(Mode.SANDBOX)            // SANDBOX or PRODUCTION
    .timeout(10000)                // Optional, ms
    .retry { ... }                 // Optional retry config
    .build()
```

**Auth types:**
- `.secretKey(key, scopes?)` → `AuthConfig.ApiKey` — auto-exchanges sk_ key for JWT. Origin resolved via `/me`.
- `.originKey(key)` → `AuthConfig.OriginKey` — legacy. Requires `.origin()`. Uses `connect(alias)`.
- `.apiKey(key)` → **Deprecated.** Use `.secretKey()` or `.originKey()`.

### `bloque.register(alias, params)` → UserSession

Registers a new user identity and returns a connected session. **Only available for `originKey` auth.**

```kotlin
val session = bloque.register("@alice", IndividualRegisterParams(
    profile = IndividualProfile(
        firstName = "Alice",
        lastName = "Smith",
        email = "alice@example.com",
        phone = "+1234567890",
        birthdate = "1990-01-01",
        city = "Miami",
        state = "FL",
        postalCode = "33101",
        countryOfBirthCode = "USA",
        countryOfResidenceCode = "USA"
    )
))
```

**Returns:** `UserSession` with `accounts`, `compliance`, `identity`, `orgs`, `swap`

### `bloque.connect()` → UserSession

**For `apiKey` auth.** Auto-exchanges the sk_ key for a JWT, then calls `/me` to resolve origin and URN.

```kotlin
val session = bloque.connect()
```

Throws `BloqueConfigError` if auth is not `ApiKey`.

### `bloque.connect(alias)` → UserSession

**For `originKey` auth.** Connects to an existing user via the legacy `API_KEY` challenge flow.

**Critical:** The `alias` must be **exactly** the same string used in `register()`. `connect()` always returns a session — it does NOT validate identity existence. Errors surface later when calling account methods.

```kotlin
val session = bloque.connect("@alice")
```

Throws `BloqueConfigError` if auth is not `OriginKey`.

---

## IdentityClient (`session.identity`)

### `session.identity.me()` → IdentityMe

Returns the authenticated identity's own profile. Used internally by `connect()` for `apiKey` auth.

```kotlin
val me = session.identity.me()
// me.urn, me.origin, me.type, me.profile
```

### Identity — API Keys (`session.identity.apiKeys`)

These endpoints require an authenticated user session (JWT from connect/register).

#### `session.identity.apiKeys.create(params)` → CreateApiKeyResult

```kotlin
val result = session.identity.apiKeys.create(CreateApiKeyParams(
    name = "My API Key",
    scopes = listOf("accounts:read", "accounts:write"),
    domains = listOf("*.example.com"),
    expiration = "2025-12-31T23:59:59Z"
))
// result.keyId, result.secretKey, result.publishableKey
```

#### `session.identity.apiKeys.list()` → List\<ApiKeyInfo\>

```kotlin
val keys = session.identity.apiKeys.list()
// keys[0].id, keyId, publishableKey, name, scopes, domains, status, expiration, lastUsedAt, createdAt
```

#### `session.identity.apiKeys.get(id)` → ApiKeyInfo

```kotlin
val key = session.identity.apiKeys.get("key-id")
```

#### `session.identity.apiKeys.exchange(params)` → ExchangeApiKeyResult

Exchanges an sk_ key for a short-lived JWT. This is called automatically by the SDK when using `apiKey` auth.

```kotlin
val result = session.identity.apiKeys.exchange(ExchangeApiKeyParams(
    key = "sk_live_...",
    scopes = listOf("accounts:read")
))
// result.accessToken, result.expiresIn, result.tokenType
```

#### `session.identity.apiKeys.revoke(id)`

```kotlin
session.identity.apiKeys.revoke("key-id")
```

#### `session.identity.apiKeys.rotate(id)` → RotateApiKeyResult

```kotlin
val result = session.identity.apiKeys.rotate("key-id")
// result.keyId, result.secretKey, result.publishableKey
```

---

## AccountsClient (`session.accounts`)

### `session.accounts.balance(urn)` → `Map<String, TokenBalance>`

```kotlin
val balance = session.accounts.balance("did:bloque:account:virtual:...")
```

### `session.accounts.transfer(params)` → TransferResult

```kotlin
val result = session.accounts.transfer(TransferParams(
    sourceUrn = "did:bloque:account:...",
    destinationUrn = "did:bloque:account:...",
    amount = "10000000",
    asset = Asset.DUSD_6,
    metadata = null,
    idempotencyKey = null
))
```

### `session.accounts.virtual.create(params, config)` → VirtualAccount

### `session.accounts.polygon.create(params, config)` → PolygonAccount

### `session.accounts.bancolombia.create(params, config)` → BancolombiaAccount

### `session.accounts.card.list()` → List\<CardAccount\>

### `session.accounts.card.freeze(cardUrn)` → CardAccount

### `session.accounts.card.activate(cardUrn)` → CardAccount

### `session.accounts.movements(params)` → PagedMovements

Returns `PagedMovements` with `data`, `pageSize`, `hasMore`, `next`. Use `result.data` for the list of movements.

---

## SwapClient (`session.swap`)

### `session.swap.findRates(params)` → FindRatesResult

### `session.swap.listOrders(params)` → ListOrdersResult

### `session.swap.bankTransfer.create(params)` → BankTransferResult

---

## ColBankClient (`session.swap.colbank`)

Colombian bank withdrawal operations.

---

## ComplianceClient (`session.compliance`)

### `session.compliance.kyc.startVerification(params)` → KycVerificationResponse

Starts KYC/KYB verification. API path: `POST /api/compliance`. Params: `urn` (required), `type` (optional, default `"kyc"`), `accompliceType` (optional, default `"person"`), `webhookUrl` (optional).

```kotlin
val result = session.compliance.kyc.startVerification(
    KycVerificationParams(urn = "did:bloque:user:...", type = "kyc")
)
// result.url, result.status
```

### `session.compliance.kyc.getVerification(params)` → KycVerificationResponse

Gets verification status by URN. API path: `GET /api/compliance/:urn`.

```kotlin
val status = session.compliance.kyc.getVerification(
    GetKycVerificationParams(urn = "did:bloque:user:...")
)
```

---

## OrgsClient (`session.orgs`)

### `session.orgs.create(params)` → Organization

Creates an organization. API path: `POST /api/orgs`.

### `session.orgs.get(orgUrn)` → Organization

### `session.orgs.list()` → List\<Organization\>

---

*This reference is kept in sync with the Kotlin SDK via the `sync-sdk-kotlin` command. Update when API changes are applied.*
