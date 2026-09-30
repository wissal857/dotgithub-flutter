# Integration & Verification Checklist

Use this checklist to verify the error handling setup.

- [ ] `initErrorHandling()` wired into `main()` before `runApp()`
- [ ] `ErrorWidget.builder` returns fallback UI and reports to provider
- [ ] `FlutterError.onError` sends framework errors to provider
- [ ] `PlatformDispatcher.instance.onError` captures uncaught async errors
- [ ] Repositories throw `AppException` for business errors
- [ ] `FailureMapper` maps exceptions into `Failure` consistently
- [ ] Production DSN/API key configured and tested in staging
- [ ] Crash reports visible in selected provider dashboard
- [ ] Automated tests cover failure mapping and key error flows

Tips:
- Use stable `code` values for telemetry.
- Verify offline scenarios (network errors) and idempotent retries separately.
