# `.actions.yml` schema

Used by the `load-config` reusable workflow.

## Schema version 1 (legacy flat keys)

Required: `schema_version: 1`, `run_ios_tests`, `run_android_tests`, `flutter_version`, `ruby_version`, `macos_version`.

See git history or Muuvie `.actions.yml` for the flat key list. Still supported.

## Schema version 2 (recommended)

Structured **testing** and **publishing** sections plus shared **versions**.

### `versions` (required for Flutter / shared toolchains)

| Key | Required | Description |
|-----|----------|-------------|
| `flutter` | yes* | Flutter SDK version |
| `ruby` | yes* | Ruby version |
| `macos` | yes* | GitHub-hosted macOS image label |
| `xcode` | no | Xcode version |
| `android_java` | no | JDK version (default `17`) |
| `android_java_distribution` | no | JDK distribution (default `temurin`) |

\* Required when any Flutter testing or publishing is enabled.

### `testing.flutter`

| Key | Default | Description |
|-----|---------|-------------|
| `run_linter` | `false` | Dart analyze |
| `run_ios_tests` | `false` | iOS test + simulator build job |
| `run_android_tests` | `false` | Android test + debug build job |
| `coverage_minimum` | `""` | LCOV minimum for PR report |
| `ios_test_device` | `""` | Simulator/device for Fastlane test lane |

### `testing.ios` (native)

| Key | Default | Description |
|-----|---------|-------------|
| `run_swiftlint` | `true` | SwiftLint on ubuntu |
| `run_tests` | `true` | macOS test job |

### `testing.android` (native)

| Key | Default | Description |
|-----|---------|-------------|
| `run_unit_tests` | `true` | Gradle unit tests |
| `run_lint` | `true` | Android lint |
| `run_emulator_tests` | `false` | Reserved (not implemented in shared CI v1) |

### `publishing.flutter`

| Key | Default | Description |
|-----|---------|-------------|
| `deploy_ios` | `false` | Enable iOS leg of `flutter-publish` |
| `deploy_android` | `false` | Enable Android leg of `flutter-publish` |

### `publishing.ios`

| Key | Default | Description |
|-----|---------|-------------|
| `enabled` | `false` | Run `ios-publish` |
| `mode` | `build-and-publish` | `build-and-publish` or `distribute-artifact` |
| `target` | `testflight` | `testflight` or `github_release` |

### `publishing.android`

| Key | Default | Description |
|-----|---------|-------------|
| `enabled` | `false` | Run `android-publish` |
| `mode` | `build-and-publish` | `build-and-publish` or `distribute-artifact` |
| `target` | `play_store` | `play_store` or `artifact-only` |

Unknown keys are ignored.

### Examples

Copy-paste files live in [`docs/examples/`](examples/README.md):

- [`actions.v1.yml`](examples/actions.v1.yml) — legacy flat schema
- [`actions.v2.flutter.yml`](examples/actions.v2.flutter.yml)
- [`actions.v2.ios.yml`](examples/actions.v2.ios.yml)
- [`actions.v2.android.yml`](examples/actions.v2.android.yml)

## Wiring config to workflows

| Config | Workflow |
|--------|----------|
| `testing.flutter.*` | `flutter-ci.yml` |
| `testing.ios.*` | `ios-ci.yml` |
| `testing.android.*` | `android-ci.yml` |
| `publishing.flutter.*` | `flutter-publish.yml` |
| `publishing.ios.*` | `ios-publish.yml` |
| `publishing.android.*` | `android-publish.yml` |

Flutter publish builds signed artifacts, then calls native publish workflows in `distribute-artifact` mode.
