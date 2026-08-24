# shared-workflows

Reusable GitHub Actions workflows for the [mobyleOfficial](https://github.com/mobyleOfficial) organization.

## Available workflows

### Check AI Contribution

Fails pull requests that include AI ownership attribution in commit messages or the PR body (for example `Co-Authored-By`, `Generated with …`, `Made with …`).

**Caller (copy into each repo):**

```yaml
# .github/workflows/check-ai-contribution.yml
name: Check AI Contribution

on:
  pull_request_target:
    types: [opened, edited, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  check-ai-contribution:
    uses: mobyleOfficial/shared-workflows/.github/workflows/check-ai-contribution.yml@main
    permissions:
      contents: read
      pull-requests: write
```

Pin `@main` to a tag or commit SHA if you want stricter versioning.

### AI Code Review

Runs a Claude-powered multi-reviewer pass (security, bugs, architecture) on pull requests and posts inline review comments.

**Required secret:** `ANTHROPIC_API_KEY`

Reviewer prompts live in this repository under `.github/workflows/ai-review/`. The reusable workflow checks them out at runtime into `.shared-workflows/` so caller repos do not need local copies.

**Caller (copy into each repo):**

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

permissions:
  contents: read
  pull-requests: write
  issues: read
  id-token: write

jobs:
  ai-review:
    uses: mobyleOfficial/shared-workflows/.github/workflows/ai-review.yml@main
    permissions:
      contents: read
      pull-requests: write
      issues: read
      id-token: write
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Pin `@main` to a tag or commit SHA if you want stricter versioning.

### Determine Flavor

Composite action that maps the GitHub event to a deployment flavor: `push` → `prod`, otherwise the `environment` input (`dev` | `staging` | `prod`).

**Inputs:** `environment` (optional, default `dev`)  
**Outputs:** `flavor`

**Usage:**

```yaml
- name: Determine flavor
  id: flavor
  uses: mobyleOfficial/shared-workflows/.github/actions/determine-flavor@main
  with:
    environment: ${{ inputs.environment }}
```

### Load Configuration

Reusable workflow that reads the caller repo’s `.actions.yml` (schema v1) and exposes feature flags / tool versions as job outputs.

See [`docs/actions-schema.md`](docs/actions-schema.md) for the full key list.

**Caller example:**

```yaml
jobs:
  config:
    uses: mobyleOfficial/shared-workflows/.github/workflows/load-config.yml@main

  lint:
    needs: config
    if: needs.config.outputs.run_linter == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # use needs.config.outputs.flutter_version, etc.
```

### Setup Flutter Environment

Composite action that installs Flutter (pinned version + cache), Ruby (Bundler cache), optional Java/Xcode, runs `flutter pub get`, and optionally runs codegen / CI-env shell commands.

No app-specific secrets are hardcoded — pass any needed values via the job `env` when `setup-ci-env` is enabled.

**Research notes:** [`docs/research/flutter-ci.md`](docs/research/flutter-ci.md)

**Usage:**

```yaml
- uses: mobyleOfficial/shared-workflows/.github/actions/setup-flutter-environment@main
  with:
    flutter-version: ${{ needs.config.outputs.flutter_version }}
    ruby-version: ${{ needs.config.outputs.ruby_version }}
    setup-java: 'true'
    java-version: ${{ needs.config.outputs.android_java_version }}
    run-codegen: 'true'
    setup-ci-env: 'true'
  env:
    # optional vars consumed by your ci-env-command / Fastlane lane
    BACKEND_URL: ${{ secrets.BACKEND_URL }}
```

### Setup Android Signing

Writes a keystore and `key.properties` from env vars named `ANDROID_KEYSTORE_BASE64_<ENV>`, `ANDROID_KEYSTORE_PASSWORD_<ENV>`, `ANDROID_KEY_ALIAS_<ENV>`, and `ANDROID_KEY_PASSWORD_<ENV>` (`ENV` = `DEV`/`STAGING`/`PROD`).

Map GitHub secrets into those env vars on the job. Delete the written files in a later `if: always()` step after the build/deploy.

```yaml
- uses: mobyleOfficial/shared-workflows/.github/actions/setup-android-signing@main
  with:
    environment: prod
  env:
    ANDROID_KEYSTORE_BASE64_PROD: ${{ secrets.ANDROID_KEYSTORE_BASE64_PROD }}
    ANDROID_KEYSTORE_PASSWORD_PROD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD_PROD }}
    ANDROID_KEY_ALIAS_PROD: ${{ secrets.ANDROID_KEY_ALIAS_PROD }}
    ANDROID_KEY_PASSWORD_PROD: ${{ secrets.ANDROID_KEY_PASSWORD_PROD }}
```

### iOS CI

Opinionated native iOS CI reusable workflow: optional SwiftLint (ubuntu container), then macOS Xcode + Ruby/Fastlane tests with xcresult artifact/report. Wire flags from `load-config` outputs `testing_ios_run_swiftlint` and `testing_ios_run_tests`.

**Research notes:** [`docs/research/ios-ci.md`](docs/research/ios-ci.md)

**Caller example:**

```yaml
jobs:
  config:
    uses: mobyleOfficial/shared-workflows/.github/workflows/load-config.yml@main

  ios-ci:
    needs: config
    uses: mobyleOfficial/shared-workflows/.github/workflows/ios-ci.yml@main
    with:
      macos-version: ${{ needs.config.outputs.macos_version }}
      xcode-version: ${{ needs.config.outputs.xcode_version }}
      run-swiftlint: ${{ needs.config.outputs.testing_ios_run_swiftlint == 'true' }}
      run-tests: ${{ needs.config.outputs.testing_ios_run_tests == 'true' }}
    permissions:
      contents: read
      checks: write
```

### Android CI

Opinionated native Android CI reusable workflow: JDK + Gradle setup, optional `google-services.json` from a base64 secret, unit tests, lint, and report artifacts. Wire flags from `load-config` outputs `testing_android_run_unit_tests` and `testing_android_run_lint`.

**Research notes:** [`docs/research/android-ci.md`](docs/research/android-ci.md)

**Caller example:**

```yaml
jobs:
  config:
    uses: mobyleOfficial/shared-workflows/.github/workflows/load-config.yml@main

  android-ci:
    needs: config
    uses: mobyleOfficial/shared-workflows/.github/workflows/android-ci.yml@main
    with:
      java-version: ${{ needs.config.outputs.android_java_version }}
      run-unit-tests: ${{ needs.config.outputs.testing_android_run_unit_tests == 'true' }}
      run-lint: ${{ needs.config.outputs.testing_android_run_lint == 'true' }}
      write-google-services: true
    secrets:
      GOOGLE_SERVICES_JSON: ${{ secrets.GOOGLE_SERVICES_JSON }}
```

### Android Publish

Native Android publish: **build-and-publish** (sign + build + Play upload) or **distribute-artifact** (upload a pre-built AAB/APK). Used directly by native apps or by `flutter-publish` after Flutter builds the binary.

**Modes:** `build-and-publish` | `distribute-artifact`  
**Targets:** `play_store` | `artifact-only`

Map signing secrets to job `env` (`ANDROID_KEYSTORE_BASE64_<ENV>`, etc.) and Play credentials for `distribute-command` (typically Fastlane).

```yaml
jobs:
  publish:
    uses: mobyleOfficial/shared-workflows/.github/workflows/android-publish.yml@main
    with:
      mode: build-and-publish
      environment: prod
      build-command: bundle exec fastlane android build
      distribute-command: bundle exec fastlane android deploy_play env:prod dry_run:false
    secrets: inherit
```

### iOS Publish

Native iOS publish: **build-and-publish** or **distribute-artifact**. Targets: **TestFlight** or **GitHub Release** (Abbay-style `.ipa`/`.zip` asset).

```yaml
jobs:
  publish:
    uses: mobyleOfficial/shared-workflows/.github/workflows/ios-publish.yml@main
    with:
      mode: distribute-artifact
      target: testflight
      artifact-name: flutter-ios-release
      distribute-command: bundle exec fastlane ios deploy_testflight env:prod dry_run:false
    secrets: inherit
```

### Flutter CI

PR testing workflow: Dart analyze, iOS/Android test + simulator/debug build jobs, optional LCOV report. Pass values from `load-config` (`testing.flutter.*` / legacy v1 outputs).

```yaml
jobs:
  config:
    uses: mobyleOfficial/shared-workflows/.github/workflows/load-config.yml@main

  flutter-ci:
    needs: config
    uses: mobyleOfficial/shared-workflows/.github/workflows/flutter-ci.yml@main
    with:
      run-linter: ${{ needs.config.outputs.run_linter == 'true' }}
      run-ios-tests: ${{ needs.config.outputs.run_ios_tests == 'true' }}
      run-android-tests: ${{ needs.config.outputs.run_android_tests == 'true' }}
      flutter-version: ${{ needs.config.outputs.flutter_version }}
      ruby-version: ${{ needs.config.outputs.ruby_version }}
      macos-version: ${{ needs.config.outputs.macos_version }}
      xcode-version: ${{ needs.config.outputs.xcode_version }}
      test-coverage-minimum: ${{ needs.config.outputs.test_coverage_minimum }}
      ios-test-device: ${{ needs.config.outputs.ios_test_device }}
    secrets: inherit
```

### Flutter Publish

Builds signed Flutter release artifacts, then calls **native publish** workflows in `distribute-artifact` mode (Play + TestFlight). Configure via `publishing.flutter.deploy_ios` / `deploy_android` in schema v2.

```yaml
jobs:
  config:
    uses: mobyleOfficial/shared-workflows/.github/workflows/load-config.yml@main

  flutter-publish:
    needs: config
    if: needs.config.outputs.deploy_ios == 'true' || needs.config.outputs.deploy_android == 'true'
    uses: mobyleOfficial/shared-workflows/.github/workflows/flutter-publish.yml@main
    with:
      environment: prod
      deploy-android: ${{ needs.config.outputs.deploy_android == 'true' }}
      deploy-ios: ${{ needs.config.outputs.deploy_ios == 'true' }}
      flutter-version: ${{ needs.config.outputs.flutter_version }}
      ruby-version: ${{ needs.config.outputs.ruby_version }}
      macos-version: ${{ needs.config.outputs.macos_version }}
      xcode-version: ${{ needs.config.outputs.xcode_version }}
    secrets: inherit
```

See [`docs/actions-schema.md`](docs/actions-schema.md) for schema v2 (`testing` / `publishing` sections).

