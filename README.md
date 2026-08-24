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

Opinionated native iOS CI reusable workflow: optional SwiftLint (ubuntu container), then macOS Xcode + Ruby/Fastlane tests with xcresult artifact/report.

**Research notes:** [`docs/research/ios-ci.md`](docs/research/ios-ci.md)

**Caller example:**

```yaml
jobs:
  ios-ci:
    uses: mobyleOfficial/shared-workflows/.github/workflows/ios-ci.yml@main
    with:
      macos-version: '14'
      xcode-version: '16.2'
      run-swiftlint: true
      run-tests: true
      test-command: bundle exec fastlane test_project open:false
    permissions:
      contents: read
      checks: write
```

### Android CI

Opinionated native Android CI reusable workflow: JDK + Gradle setup, optional `google-services.json` from a base64 secret, unit tests, lint, and report artifacts. Emulator UI tests are not enabled in v1 (`run-emulator-tests` fails closed if set true).

**Research notes:** [`docs/research/android-ci.md`](docs/research/android-ci.md)

**Caller example:**

```yaml
jobs:
  android-ci:
    uses: mobyleOfficial/shared-workflows/.github/workflows/android-ci.yml@main
    with:
      java-version: '17'
      write-google-services: true
      run-unit-tests: true
      run-lint: true
    secrets:
      GOOGLE_SERVICES_JSON: ${{ secrets.GOOGLE_SERVICES_JSON }}
```

