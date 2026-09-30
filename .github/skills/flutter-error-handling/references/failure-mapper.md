# FailureMapper

`FailureMapper` converts exceptions into `Failure` objects for app consumption.

Example pattern:

```dart
import 'failure-model.dart';
import 'app-exception.md';

class FailureMapper {
  Failure map(Object error) {
    if (error is AppException) {
      return Failure(code: error.code, message: error.message);
    }

    // Add network/client-specific mappings here
    // Example pseudocode:
    // if (error is SocketException) return Failure(code: 'network_error', message: 'No internet');

    // Global fallback
    return Failure(code: 'unknown_error', message: 'Something went wrong');
  }
}
```

Integration notes:
- Use `FailureMapper` in repository catch blocks to translate caught exceptions before returning results to the UI.
- Keep mapping deterministic for analytics and translation.
