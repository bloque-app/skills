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
    .origin("my-origin")                    // Your origin identifier
    .apiKey(System.getenv("BLOQUE_API_KEY")) // Never hardcode in production
    .mode(Mode.SANDBOX)                     // SANDBOX for testing
    .timeout(10000)                         // Request timeout (ms)
    .retry {
        maxRetries(3)
        initialDelay(1000)
    }
    .build()
```

## Auth Strategies

- **API Key** — For backend/server use. Pass via `apiKey()`.
- **Environment variable** — Prefer `System.getenv("BLOQUE_API_KEY")` over hardcoding.

## First Request

```kotlin
// Connect to a user (alias must match what was used in register)
val session = bloque.connect("@alice")

// Get balance
val balance = session.accounts.balance("did:bloque:account:virtual:...")
```

## Java Usage

The SDK is fully interoperable with Java:

```java
BloqueSDK bloque = BloqueSDK.builder()
    .origin("my-origin")
    .apiKey(System.getenv("BLOQUE_API_KEY"))
    .mode(Mode.SANDBOX)
    .build();

UserSession session = bloque.connect("@alice");
Map<String, TokenBalance> balance = session.getAccounts().balance("did:bloque:account:virtual:...");
```
