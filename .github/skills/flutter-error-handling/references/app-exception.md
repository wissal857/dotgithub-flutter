# AppException

Use `AppException` to represent business-logic errors thrown from domain and repository layers.

Example implementation:

```dart
class AppException implements Exception {
  final String code;
  final String message;

  AppException(this.code, this.message);

  @override
  String toString() => 'AppException(code: $code, message: $message)';
}

// Usage:
// throw AppException('item_not_found', 'Requested item was not found');
```

Guidance:
- Keep `code` stable (use snake_case) for telemetry and translations.
- Use `AppException` for recoverable, user-facing business errors.
