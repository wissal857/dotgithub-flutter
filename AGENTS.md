# AI Instructions: Offline-First Mobile App

## 🏗 Architecture & Patterns
- **Architecture**: Layered Architecture combined with MVVM.
    - **Data Layer**: Repository implementations, Data sources (Local/Remote), and DTOs.
    - **Domain Layer**: Business entities, Repository interfaces, and Use cases.
    - **UI Layer**: ViewModels (implemented via Riverpod providers) and Views (Widgets).
- **Pattern**: MVVM (Model-View-ViewModel).
    - Views observe ViewModels.
    - ViewModels interact with the Domain layer.
- **Folder Structure**: Feature-first.
    - **Core**: `lib/core/` — Shared code across features.
      - `lib/core/theming/` — app theme definitions, color palette, typography, custom theme extensions, and reusable style helpers.
      - `lib/core/error/` — `Failure` model, failure mapping, global error handling utilities, and error reporting helpers.
      - `lib/core/network/` — `dio` client setup, interceptors, API helpers, connectivity checks, and retry/idempotency support.
      - `lib/core/caching/` — core caching components (cache store, eviction, revalidation, mutation queue).
      - `lib/core/utils/` — common utility helpers, extensions, validators, formatters, and lightweight shared widgets.
      - `lib/core/routing/` — `go_router` configuration, route definitions, navigation helpers, and deep link handling.
    - **Features**: `lib/features/{feature_name}/{data|domain|ui}` — Feature-specific code following layered architecture.

## 🛠 Tech Stack
- **State Management & DI**: `riverpod`
- **Immutability**: `freezed`
- **Routing**: `go_router`
- **HTTP Client**: `dio`
- **Persistence**: Local SQL database for persistent storage.

## 🔐 Authentication
- **Strategy**: Stateless authentication using **JWT (JSON Web Tokens)**.
- **Implementation**: Tokens should be stored securely and attached to requests via `dio` interceptors.

## 🛡 Error Handling
- **Failure Model**: Use a `Failure` model containing a `code` (String) and a `message` (String).
- **Mapping**: Use a `FailureMapper` to map technical exceptions (e.g., `SocketException`, `SqliteException`) to user-friendly `Failure` objects.
- **Safety**: Implement a global fallback for all failures to ensure no error is swallowed silently.
- **Skills**: Use `flutter-error-handling` skill for global handlers, `AppException`, `FailureMapper`, and Sentry/Crashlytics integration.

## 💾 Caching & Sync Strategy
- **In-Memory**: Use Riverpod providers for fast, in-memory caching (default).
- **Optional Two-Layer**: Enable `mode: two-layer` in `offline-caching` skill for in-memory + SQLite persistence.
- **Eviction**: LFU default (configurable: LRU, TTL_ONLY, SIZE) with `maxEntries` limit.
- **Revalidation**: Stale-while-revalidate with `ETag`/`max-age` and client-side jitter to avoid stampedes.
- **Write Flow**: Write-behind with optimistic updates, idempotency keys, and background queue drain.
- **Skills**: Use `offline-caching` skill for implementation patterns and `http-idempotency` for write reliability.
<!--- **Persistence**: Use a local SQL database for long-term offline storage. -->
- **Flow**: UI → ViewModel → Repository → In-Memory Cache


## 🌐 API & Reliability
- **Idempotency**: Use **idempotency keys** for all write operations (POST, PUT, PATCH) to prevent duplicate processing and ensure reliability during retries.

## 📝 Coding Conventions
- Use `freezed` for all state and event classes.
- Ensure Domain layer remains independent of the Data and UI layers.
- All repository implementations must reside in the Data layer and implement interfaces defined in the Domain layer.