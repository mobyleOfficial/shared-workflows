# Flutter CI research notes

Sources consulted while designing `setup-flutter-environment` / signing helpers:

1. [subosito/flutter-action](https://github.com/subosito/flutter-action) — pin `flutter-version`, use `channel: stable`, enable `cache: true` (SDK + pub cache).
2. Production CI write-ups emphasizing pinned Flutter versions (not floating `stable` alone) and sequential `pub get` → analyze → test → build.
3. Fastlane + GitHub Actions pipelines pairing `ruby/setup-ruby` (`bundler-cache: true`) with Flutter for mobile lanes.
4. Muuvie `.github/actions/setup-environment` as a **baseline** only — generalized here (no app-specific secrets; optional codegen/CI-env commands).

Decisions reflected in the composites:

- Require explicit `flutter-version`; default channel `stable`.
- Optional Java / Xcode behind inputs.
- Codegen and CI-env are opt-in commands (defaults match Fastlane lane names common in Mobyle apps).
- Android signing reads env vars mapped by the caller (composites cannot access `secrets.*` dynamically); caller must delete keystore/`key.properties` after use.
