# Failure Model

Simple `Failure` model used as an app-facing error value.

```dart
class Failure {
  final String code;
  final String message;

  Failure({required this.code, required this.message});

  @override
  String toString() => 'Failure(code: $code, message: $message)';
}

// Examples:
// Failure(code: 'network_error', message: 'Please check your internet connection')
// Failure(code: 'unauthorized', message: 'You must sign in to continue')
```

Guidance:
- Keep messages user-friendly; map technical errors via `FailureMapper`.
- `code` should be stable and machine-readable.
