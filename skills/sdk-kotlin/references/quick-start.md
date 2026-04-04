# Quick Start — Bloque Kotlin SDK

## Installation

Add the Bloque SDK to your Gradle project:

```kotlin
// build.gradle.kts
dependencies {
    implementation("app.bloque.sdk:sdk:0.0.22")
}
```

Or use individual modules:

```kotlin
implementation("app.bloque.sdk:sdk-accounts:0.0.22")
implementation("app.bloque.sdk:sdk-identity:0.0.22")
implementation("app.bloque.sdk:sdk-swap:0.0.22")
// ...
```

## Configuration

```kotlin
import app.bloque.sdk.BloqueSDK
import app.bloque.sdk.core.Mode

val bloque = BloqueSDK.builder()
    .secretKey(System.getenv("SECRET_KEY"))  // sk_ key — auto-exchanged for JWT
    // OR: .originKey(System.getenv("ORIGIN_KEY"))  // legacy origin-scoped key (requires .origin())
    .mode(Mode.SANDBOX)
    .timeout(10000)
    .retry {
        maxRetries(3)
        initialDelay(1000)
    }
    .build()
```

## Auth Strategies

- **API Key (recommended)** — For server-side use with `sk_` secret keys. Pass via `.secretKey()`. The SDK auto-exchanges the key for a short-lived JWT and refreshes it before expiry. Origin is not needed — resolved via `/me`.
- **Origin Key (legacy)** — Origin-scoped keys. Pass via `.originKey()` and `.origin()`. Requires `connect(alias)` to authenticate.

### Option A: API Key auth (recommended)

```kotlin
val bloque = BloqueSDK.builder()
    .secretKey(System.getenv("SECRET_KEY"))
    .mode(Mode.SANDBOX)
    .build()

val session = bloque.connect()  // no alias — identity resolved via /me
```

### Option B: Origin Key auth (legacy)

```kotlin
val bloque = BloqueSDK.builder()
    .origin("my-origin")
    .originKey(System.getenv("ORIGIN_KEY"))
    .mode(Mode.SANDBOX)
    .build()

val session = bloque.connect("@alice")
```

## First Request

```kotlin
val session = bloque.connect()  // or bloque.connect("@alice") for originKey

val balance = session.accounts.balance("did:bloque:account:virtual:...")
```

## Java Usage

The SDK is fully interoperable with Java:

```java
// API Key auth
BloqueSDK bloque = BloqueSDK.builder()
    .secretKey(System.getenv("SECRET_KEY"))
    .mode(Mode.SANDBOX)
    .build();

UserSession session = bloque.connect();
```

```java
// Origin Key auth (legacy)
BloqueSDK bloque = BloqueSDK.builder()
    .origin("my-origin")
    .originKey(System.getenv("ORIGIN_KEY"))
    .mode(Mode.SANDBOX)
    .build();

UserSession session = bloque.connect("@alice");
Map<String, TokenBalance> balance = session.getAccounts().balance("did:bloque:account:virtual:...");
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `SECRET_KEY` | `sk_test_...` or `sk_live_...` — server-side secret key for API Key auth (recommended) |
| `ORIGIN_KEY` | Origin-scoped key for legacy Origin Key auth |
