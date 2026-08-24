# Example `.actions.yml` files

Copy one of these into the **root** of a consumer repo as `.actions.yml`. Schema details: [`../actions-schema.md`](../actions-schema.md). Secrets: [`../secrets.md`](../secrets.md).

| File | Use when |
|------|----------|
| [`actions.v1.yml`](actions.v1.yml) | Legacy flat schema (still supported by `load-config`) |
| [`actions.v2.flutter.yml`](actions.v2.flutter.yml) | Flutter app (Muuvie-style): Flutter CI + Play/TestFlight via `flutter-publish` |
| [`actions.v2.ios.yml`](actions.v2.ios.yml) | Native iOS app: `ios-ci` + `ios-publish` |
| [`actions.v2.android.yml`](actions.v2.android.yml) | Native Android app: `android-ci` + `android-publish` |

Pin reusable workflow refs to a release tag (for example `@v1.0.0`) instead of `@main` when you want a frozen version.
