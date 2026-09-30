---
name: detect-race-conditions
description: Analyze a Flutter/Dart project to identify potential race conditions in concurrent code, async state management, and database operations. Generates a prioritized list of detected race conditions with test coverage recommendations and a comprehensive test plan.
argument-hint: Workspace-scoped analysis tool for Riverpod, async/await, Drift, and state management patterns
user-invocable: true
---

# Race Condition Detection Skill

## Purpose

Automatically scan a Flutter/Dart codebase to identify common race condition patterns in:

- State management (Riverpod notifiers with concurrent state updates)
- Async/await code (missing cancellation, concurrent operations)
- Database operations (Drift transactions, concurrent reads/writes)
- Network requests (overlapping mutations, orphaned futures)
- Timer and stream management
- Shared resource access without locks or atomicity

## When to Use

- Initial audit of a new or existing Flutter project for concurrency issues
- Before implementing critical features (payments, data sync, mutations)
- Refactoring legacy async code to identify latent race conditions
- Setting up test coverage for concurrent scenarios
- Performance optimization of state management
- Preparation for production deployment

## Quick Overview

### Detection Phases

1. **Codebase Scanning** — Analyze patterns in Riverpod providers, async methods, database operations
2. **Pattern Matching** — Identify common race condition anti-patterns
3. **Risk Assessment** — Categorize by severity (critical, high, medium, low)
4. **Test Plan Generation** — Create reproducible test cases for each condition

### Common Race Condition Patterns Detected

- **State Management Race Conditions**
  - Concurrent state mutations in Riverpod notifiers without coalescing
  - Multiple async operations updating the same state
  - State listeners triggering cascading state updates
- **Database Race Conditions**
  - Concurrent INSERT/UPDATE without transactions
  - Read-after-write consistency issues in sync engine
  - Missing ETag validation in conflict scenarios
- **Network Race Conditions**
  - Overlapping mutation requests with same idempotency key not handled
  - Response handler races with state cleanup
  - Cancelled requests not properly cleaned up
- **Async Lifecycle Race Conditions**
  - Missing cancellation tokens on long-running futures
  - Timer leaks without proper disposal
  - Stream subscriptions not unsubscribed before state disposal
- **Sync Engine Race Conditions**
  - Concurrent reconcile calls without coalescing
  - Mutation queue updates racing with sync operations
  - Staleness detection not atomic

## How It Works

### 1. Scan the Project

The skill analyzes:

- **Provider files** (`*_provider.dart`, `*_notifier.dart`)
  - Detects multiple concurrent `state =` assignments
  - Finds async operations without proper error handling
  - Identifies missing `try-finally` cleanup blocks
- **Database files** (`*_dao.dart`, `*_datasource.dart`)
  - Detects non-transactional multi-step operations
  - Identifies ETag validation logic gaps
  - Flags missing conflict resolution checks
- **Network/Remote files** (`*_remote_datasource.dart`)
  - Finds overlapping requests without deduplication
  - Detects response handlers without null-safety checks
  - Identifies mutation operations without coalescing
- **Sync engine files** (`sync_engine.dart`, `sync_coordinator.dart`)
  - Detects concurrent sync calls
  - Identifies timer management issues
  - Flags mutation queue races

### 2. Generate Findings Report

For each detected race condition, the report includes:

- **Location** — File path and line number(s)
- **Pattern Type** — Classification (state mutation, database, network, lifecycle, sync)
- **Severity** — Critical/High/Medium/Low
- **Description** — Concise explanation of the race condition
- **Impact** — Potential consequences (data loss, crashes, inconsistency, etc.)
- **Reproduction Steps** — High-level scenario to trigger the condition

### 3. Create Test Plan

The skill generates a comprehensive test plan with:

- **Unit Tests** — Isolated testing of each component
- **Integration Tests** — Cross-component interaction scenarios
- **Concurrency Tests** — Stress tests with simultaneous operations
- **Timing Tests** — Race condition triggering scenarios

## Detection Patterns & Fixes

### Pattern: Concurrent State Mutations in Riverpod Notifier

**Red Flags:**

```dart
// ❌ Race condition: multiple concurrent state updates
class UserNotifier extends Notifier<User> {
  @override
  User build() => User.empty();

  Future<void> updateName(String name) async {
    state = state.copyWith(name: name); // Update 1
    final result = await api.updateName(name);
    state = state.copyWith(name: result.name); // Update 2 - could race with other updates
  }

  Future<void> updateEmail(String email) async {
    state = state.copyWith(email: email); // Update 3 - may interleave with updateName
  }
}
```

**Fix:**

```dart
// ✅ Properly coalesced state updates
class UserNotifier extends Notifier<User> {
  UserNotifier() : _updating = false;
  bool _updating = false;

  Future<void> updateName(String name) async {
    if (_updating) return; // Coalesce: ignore concurrent calls
    _updating = true;
    try {
      state = state.copyWith(name: name);
      final result = await api.updateName(name);
      state = state.copyWith(name: result.name);
    } finally {
      _updating = false;
    }
  }
}
```

### Pattern: Concurrent Database Operations

**Red Flags:**

```dart
// ❌ Race condition: non-atomic multi-step DB operation
Future<void> syncUsers() async {
  final localUsers = await db.getUsersDao().getAll();
  await api.uploadUsers(localUsers);
  await db.getUsersDao().deleteAll(); // May race with other DB writes
}
```

**Fix:**

```dart
// ✅ Atomic transaction
Future<void> syncUsers() async {
  await db.transaction(() async {
    final localUsers = await db.getUsersDao().getAll();
    await api.uploadUsers(localUsers);
    await db.getUsersDao().deleteAll();
  });
}
```

### Pattern: Uncancelled Async Operations

**Red Flags:**

```dart
// ❌ Race condition: orphaned futures
class ProductNotifier extends StateNotifier<ProductState> {
  void loadProduct(String id) {
    _fetchProduct(id).then((product) {
      state = ProductLoaded(product); // May update after notifier disposal
    });
  }
}
```

**Fix:**

```dart
// ✅ Proper cancellation with mounted check
class ProductNotifier extends StateNotifier<ProductState> {
  void loadProduct(String id) async {
    try {
      final product = await _fetchProduct(id);
      if (mounted) {
        state = ProductLoaded(product);
      }
    } catch (e) {
      if (mounted) {
        state = ProductError(e);
      }
    }
  }
}
```

### Pattern: Timer Leaks

**Red Flags:**

```dart
// ❌ Race condition: timer not cancelled
class SyncNotifier extends Notifier<SyncState> {
  @override
  SyncState build() {
    Timer.periodic(Duration(minutes: 1), (_) {
      reconcile(); // Timer continues even after notifier disposal
    });
    return SyncState.ready;
  }
}
```

**Fix:**

```dart
// ✅ Proper timer lifecycle management
class SyncNotifier extends Notifier<SyncState> {
  Timer? _syncTimer;

  @override
  SyncState build() {
    ref.onDispose(() {
      _syncTimer?.cancel();
    });
    _syncTimer = Timer.periodic(Duration(minutes: 1), (_) {
      reconcile();
    });
    return SyncState.ready;
  }
}
```

## Detailed Test Plan Template

### Unit Tests

1. **State Mutation Isolation**

   ```dart
   test('UserNotifier: concurrent name/email updates do not interleave', () async {
     final container = ProviderContainer();
     final notifier = container.read(userNotifierProvider.notifier);

     final nameUpdate = notifier.updateName('John');
     final emailUpdate = notifier.updateEmail('john@example.com');

     await Future.wait([nameUpdate, emailUpdate]);

     expect(notifier.state.name, 'John');
     expect(notifier.state.email, 'john@example.com');
   });
   ```

2. **Database Transaction Atomicity**

   ```dart
   test('Sync: database updates are atomic', () async {
     await db.transaction(() async {
       await usersDao.insert(user1);
       await usersDao.insert(user2);
     });

     final users = await usersDao.getAll();
     expect(users.length, 2); // Both or neither inserted
   });
   ```

3. **Async Cancellation**

   ```dart
   test('ProductNotifier: does not update after disposal', () async {
     final container = ProviderContainer();
     final notifier = container.read(productNotifierProvider.notifier);

     notifier.loadProduct('1');
     container.invalidateAll(); // Simulate disposal

     await Future.delayed(Duration(milliseconds: 100));

     // Verify no exception or leaked state update
   });
   ```

### Integration Tests

1. **Concurrent Mutations with Sync**

   ```dart
   test('SyncEngine: mutations during sync are queued, not lost', () async {
     syncEngine.start();

     final mutation1 = enqueueMutation(UserMutation(name: 'John'));
     await Future.delayed(Duration(milliseconds: 50));
     final mutation2 = enqueueMutation(UserMutation(email: 'john@example.com'));

     await syncEngine.reconcile();

     expect(mutations.length, 2); // Both mutations processed
   });
   ```

2. **ETag Conflict Handling**

   ```dart
   test('Repository: resolves ETag conflict with last-write-wins', () async {
     // Local: version 1, Remote: version 2
     final result = await repository.getUser(id, etagVersion: '1');

     expect(result, isA<Result<User>>());
     // Verify conflict resolution applied
   });
   ```

### Concurrency Stress Tests

1. **High-Frequency State Updates**

   ```dart
   test('UserNotifier: handles 100 concurrent updates without data loss', () async {
     final futures = List.generate(100, (i) =>
       notifier.incrementCounter()
     );
     await Future.wait(futures);
     expect(notifier.state.counter, 100); // Or verify coalescing behavior
   });
   ```

2. **Timer + State Update Race**

   ```dart
   test('SyncNotifier: timer reconcile does not race with manual reconcile', () async {
     syncEngine.start();

     // Trigger manual sync while timer tick is pending
     final manualSync = syncEngine.reconcile();
     await Future.delayed(Duration(milliseconds: 10));

     expect(await manualSync, isNotNull); // Completes without conflict
   });
   ```

## Integration Checklist

- [ ] All Riverpod notifiers use `mounted` or disposal guards for async updates
- [ ] All Drift database operations in sync paths use transactions
- [ ] All timers in notifiers are cancelled in `ref.onDispose()`
- [ ] All network requests have proper cancellation token handling
- [ ] Mutation operations use idempotency keys or deduplication
- [ ] Sync engine coalesces concurrent `reconcile()` calls
- [ ] State updates in error handlers check `mounted` before updating
- [ ] Stream subscriptions are cancelled in `ref.onDispose()`
- [ ] All futures from API calls are awaited, not orphaned
- [ ] Database migrations include conflict resolution logic

## Verification & Tests

- Run: `dart analyze` and scan for static analysis hints
- Generate: concurrency test suite from template
- Execute: `flutter test --coverage` with concurrency tests enabled
- Monitor: production crash reports for race condition symptoms (data inconsistency, orphaned state)

## Prompts & Examples

- "Scan my Flutter project for race conditions in Riverpod notifiers and Drift operations. Prioritize by severity."
- "Generate a test plan for the race conditions detected in my sync engine implementation."
- "Create isolated reproduction tests for each race condition found in the codebase with timing controls."
- "Suggest fixes for the detected race conditions and generate updated code."

## References

See also:

- [Riverpod Async Patterns](https://riverpod.dev/)
- [Drift Transactions](https://drift.simonbinder.eu/docs/advanced-features/transactions/)
- [Dart Concurrency](https://dart.dev/guides/language/concurrency)
- [Flutter State Management Best Practices](https://flutter.dev/docs/development/data-and-backend/state-mgmt/intro)
