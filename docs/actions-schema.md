# `.actions.yml` schema

Used by the `load-config` reusable workflow.

## `schema_version: 1`

Required. Currently only `1` is supported.

### Flutter-oriented keys (v1)

| Key | Required | Description |
|-----|----------|-------------|
| `schema_version` | yes | Must be `1` |
| `run_ios_tests` | yes | Whether to run iOS tests |
| `run_android_tests` | yes | Whether to run Android tests |
| `flutter_version` | yes | Flutter SDK version |
| `ruby_version` | yes | Ruby version (Bundler/Fastlane) |
| `macos_version` | yes | GitHub-hosted macOS image label (e.g. `14`) |
| `run_linter` | no | Whether to run the linter job |
| `run_ai_review` | no | Whether to run AI code review |
| `deploy_ios` | no | Whether to deploy iOS |
| `deploy_android` | no | Whether to deploy Android |
| `android_java_version` | no | JDK version for Android |
| `android_java_distribution` | no | JDK distribution (e.g. `temurin`) |
| `test_coverage_minimum` | no | Minimum coverage percent for reporting |
| `ios_test_device` | no | Simulator/device name for iOS tests |
| `android_test_emulator` | no | Android emulator name |
| `xcode_version` | no | Xcode version for macOS jobs |

Unknown keys are ignored.

Native iOS / Android CI-specific keys may be added in later schema revisions when those shared workflows land.

### Example

```yaml
schema_version: 1
run_ios_tests: true
run_android_tests: true
run_linter: true
run_ai_review: true
deploy_ios: false
deploy_android: false
flutter_version: "3.24.0"
ruby_version: "3.3.0"
macos_version: "14"
android_java_version: "17"
android_java_distribution: temurin
test_coverage_minimum: "80"
ios_test_device: "iPhone 16"
android_test_emulator: "Pixel_6"
xcode_version: "16.2"
```
