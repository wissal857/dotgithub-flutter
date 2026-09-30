---
name: flutter-error-handling
description: 'Set up comprehensive error handling for Flutter apps using global handlers, Failure model, FailureMapper and AppException for business logic. Use for initializing error infrastructure, implementing app-wide error recovery, adding Sentry/Crashlytics integration.'
argument-hint: 'Choose provider: sentry or crashlytics'
user-invocable: true
---

# Flutter Error Handling Skill

## When to Use
- Initialize app-wide error handling during project setup
- Add `Failure`/`FailureMapper` patterns for repositories and services
- Add `AppException` for business logic errors
- Integrate production error reporting (Sentry or Firebase Crashlytics)

## Quick Overview
- Phases: Global handlers → AppException → Failure model → FailureMapper → Integration & verification
- References: `./references/app-exception.md`, `./references/failure-model.md`, `./references/failure-mapper.md`, `./references/integration-checklist.md`

## Choose Your Error Tracking Provider
Pick one of the providers below. The procedure includes conditional code examples.
- `sentry` — Sentry SDK (recommended for minimal setup)
- `crashlytics` — Firebase Crashlytics (for Firebase-centric apps)

## Procedure

1. Global handlers (edit `lib/main.dart`)
   - Add a top-level `initErrorHandling()` call during app bootstrap.
   - Examples:

   ### If using Sentry:

   ```dart
   import 'package:sentry_flutter/sentry_flutter.dart';

   Future<void> initErrorHandling() async {
     await SentryFlutter.init(
       (options) => options.dsn = 'YOUR_SENTRY_DSN',
       appRunner: () {},
     );

     ErrorWidget.builder = (FlutterErrorDetails details) {
       Sentry.captureException(details.exception, stackTrace: details.stack);
       return const Center(child: Text('Something went wrong while rendering this screen.'));
     };

     FlutterError.onError = (details) {
       FlutterError.presentError(details);
       Sentry.captureException(details.exception, stackTrace: details.stack);
     };

     PlatformDispatcher.instance.onError = (error, stack) {
       Sentry.captureException(error, stackTrace: stack);
       return true;
     };
   }
   ```

   ### If using Firebase Crashlytics:

   ```dart
   import 'package:firebase_crashlytics/firebase_crashlytics.dart';

   Future<void> initErrorHandling() async {
     FlutterError.onError = (details) {
       FlutterError.presentError(details);
       FirebaseCrashlytics.instance.recordFlutterError(details);
     };

     PlatformDispatcher.instance.onError = (error, stack) {
       FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
       return true;
     };
   }
   ```

2. Add `AppException` (see `./references/app-exception.md`)
   - Use `AppException` in domain and repository logic to represent business errors.

3. Add `Failure` model (see `./references/failure-model.md`)
   - Use `Failure` as the app-facing error value in result types.

4. Implement `FailureMapper` (see `./references/failure-mapper.md`)
   - Map `AppException`, network, and storage errors into `Failure` objects.

5. Integrate and verify
   - Follow `./references/integration-checklist.md` to validate handlers and production reporting.

## Notes
- Keep SKILL.md lightweight; full implementation examples live under `./references/`.
- When invoking this skill from chat, supply `sentry` or `crashlytics` as your chosen provider to see tailored snippets.
