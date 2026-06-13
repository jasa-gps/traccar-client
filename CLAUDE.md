# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Traccar Client — a Flutter (Dart SDK `^3.7.2`) GPS tracking app for Android and iOS. It runs in the background and POSTs periodic location reports to a user-configured Traccar server (or any compatible OsmAnd-protocol endpoint). All low-level tracking — geolocation, background execution, buffering, server upload — lives in the external `traccar_client_sdk` package; this repo is the Flutter UI and configuration layer around it.

## Commands

```bash
flutter pub get                  # install deps; also regenerates l10n (generate: true)
flutter analyze                  # lint/static analysis — the CI gate (.github/workflows/analyze.yml)
flutter run                      # run on a connected device/emulator
flutter build apk --release      # Android release build
flutter build ios --release      # iOS release build
flutter gen-l10n                 # regenerate lib/l10n/app_localizations.dart from .arb files
graphify update .                # refresh the knowledge graph after code changes (AST-only, no API cost)
```

There is no `test/` directory and no test suite. `flutter analyze` is the only automated check on PRs.

## Architecture

The app uses a **static-class service pattern** — there is no DI/state-management framework. Services are classes with only `static` members, initialized once in `main()`. Trace any feature through these:

- **`main.dart`** — boot sequence (order matters): `Firebase.initializeApp()` → `Preferences.init()` → `PasswordService.migrate()` → `PushService.init()` → `runApp()`. Also wires deep links via `app_links` and the rate-my-app prompt. Holds global `messengerKey`/`navigatorKey`.
- **`preferences.dart`** — single source of truth for all settings. `Preferences.instance` is a `SharedPreferencesWithCache` with a fixed `allowList` of keys (id, url, accuracy, distance, interval, angle, buffer, wakelock, stop_detection, password). First-run seeds defaults (note the default server `http://demo.traccar.org:5055`). **`buildConfig()` converts stored prefs into the SDK's `Config` object** — this is the bridge between app settings and the tracking engine. Any new setting must be added to the key constants, the `allowList`, and `buildConfig()`.
- **`geolocation_service.dart`** — owns the single `TraccarClientSdk()` instance as `GeolocationService.tracker`. All tracking control (`start`/`stop`/`isTracking`/`requestPosition`) goes through it. `restartIfTracking()` applies new config to a live session.
- **`configuration_service.dart`** — applies remote configuration from a deep-link `Uri` (query params map onto preference keys), then calls `restartIfTracking()`.
- **`push_service.dart`** — Firebase Cloud Messaging. Handles remote commands (`positionSingle`, `positionPeriodic`, `positionStop`, `factoryReset`) in both foreground and a `@pragma('vm:entry-point')` background isolate handler. Uploads the FCM token to the server (`id` + `notificationToken` form post) so the server can push commands to the device.
- **`password_service.dart`** — optional password gate for the settings screen and tracking toggle. `migrate()` moves a legacy password out of `flutter_secure_storage` into preferences on startup.
- **`quick_actions.dart`** — Android/iOS home-screen shortcuts (`start`/`stop`/`sos`) that drive the tracker directly.

**Entry points that trigger tracking** (keep behavior consistent across all of them): the UI toggle in `main_screen.dart`, deep links handled in `main.dart` (`host == 'action'`, paths `start`/`stop`), FCM commands in `push_service.dart`, and home-screen shortcuts in `quick_actions.dart`. They all funnel through `GeolocationService.tracker` + `Preferences.buildConfig()`.

UI is three screens: `main_screen.dart` (tracking + settings cards), `settings_screen.dart` (edit all preferences), `status_screen.dart` (live log), plus `qr_code_screen.dart` (scan a config QR → deep link).

## Localization

- Strings live in `lib/l10n/app_*.arb`. **`app_en.arb` is the only file to edit by hand** — it is the template (`l10n.yaml`).
- Translations are managed by Transifex (`.tx/config`) and pulled in by the `Update Translations` workflow. **Do not hand-edit non-English `.arb` files** — changes will be overwritten.
- `lib/l10n/app_localizations.dart` is generated; never edit it.

## Branding / white-labeling

`tool/brand.dart` is a one-shot rebranding script (run `dart run tool/brand.dart`). Edit the constants at the top first (`appName`, `packageId`, `version`, `url`, `iconPath`, keystore settings); it rewrites app name, bundle/package IDs, version, the default server URL in `preferences.dart`, regenerates launcher icons, and creates a signing keystore. It expects `flutterfire configure` to be run afterward. The committed values are real Traccar identifiers (`org.traccar.client`); the constants in the file are placeholders.

## Firebase

Firebase config is **committed**, not gitignored: `firebase.json`, `android/app/google-services.json`, `ios/Runner/GoogleService-Info.plist`, and the generated `lib/firebase_options.dart` — all bound to the Firebase project `traccar-client-app`. The app won't build/run without them (`main()` calls `Firebase.initializeApp()`). Don't delete them. When rebranding, run `flutterfire configure` to regenerate them against a new project. Firebase powers Crashlytics, Analytics, and the FCM remote commands in `push_service.dart`.

## CI/CD (`.github/workflows/`)

- **analyze.yml** — `flutter analyze` on every push/PR to `main`.
- **release.yml** — triggered by `v*` tags (or manual dispatch). Builds + signs Android (→ Google Play production) and iOS (→ App Store), with an Android emulator smoke test that fails if the app crashes on launch. App version comes from `pubspec.yaml` (`version: X.Y.Z+BUILD`).
- **translation.yml** — manual; pulls translations from Transifex and commits them.

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
