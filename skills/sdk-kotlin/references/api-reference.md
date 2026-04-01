# API Reference — Kotlin SDK

Every SDK method, its parameters, and the exact shape of the returned object.

---

## BloqueSDK (Top Level)

### `BloqueSDK.builder()`

```kotlin
val bloque = BloqueSDK.builder()
    .origin("my-origin")           // Required
    .apiKey("sk_test_...")         // Required for backend
    .mode(Mode.SANDBOX)            // SANDBOX or PRODUCTION
    .timeout(10000)                // Optional, ms
    .retry { ... }                 // Optional retry config
    .build()
```

### `bloque.register(alias, params)` → UserSession

Registers a new user identity and returns a connected session.

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

### `bloque.connect(alias)` → UserSession

Connects to an existing user. Returns the same `UserSession` shape as `register`.

**Critical:** The `alias` must be **exactly** the same string used in `register()`. `connect()` always returns a session — it does NOT validate identity existence. Errors surface later when calling account methods.

```kotlin
val user = bloque.connect("@alice")
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

### `session.accounts.card.list()` → List<CardAccount>

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

### `session.orgs.list()` → List<Organization>

---

*This reference is kept in sync with the Kotlin SDK via the `sync-sdk-kotlin` command. Update when API changes are applied.*
