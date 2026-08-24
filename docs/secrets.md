# Secrets and environment variables

Configure these in **each consumer repository** (Settings → Secrets and variables → Actions). Shared workflows do not store app credentials in `mobyleOfficial/shared-workflows`.

## How secrets reach workflows

| Mechanism | Used by |
|-----------|---------|
| `secrets:` on the caller job + `secrets: inherit` on `uses:` | Reusable workflows (`flutter-publish`, `android-publish`, …) |
| Job `env:` mapping `secrets.*` → env var names | Composite actions (`setup-android-signing`) |
| Built-in `GITHUB_TOKEN` | PR comments, GitHub Releases |

Composite actions **cannot** read `secrets.*` dynamically. The caller must map secrets into **exact env var names** the action expects (see Android signing below).

Use [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) (`dev`, `staging`, `prod`) for environment-scoped secrets when a workflow sets `environment:` on the job.

---

## Quick reference by workflow

| Workflow / action | Required secrets | Optional / app-specific |
|-------------------|------------------|-------------------------|
| `check-ai-contribution` | — | `GITHUB_TOKEN` (automatic) |
| `ai-review` | `ANTHROPIC_API_KEY` | — |
| `load-config` | — | — |
| `determine-flavor` | — | — |
| `setup-flutter-environment` | — | Any vars your `ci-env-command` / Fastlane lane reads |
| `setup-android-signing` | Signing env vars per environment (see below) | — |
| `ios-ci` | — | — |
| `android-ci` | — | `GOOGLE_SERVICES_JSON` when `write-google-services: true` |
| `android-publish` | Signing (build mode) + Play API (Play target) | — |
| `ios-publish` | ASC API (TestFlight) or `GITHUB_TOKEN` (GitHub Release) | Match/signing for native build mode |
| `flutter-ci` | — | App CI env vars if `setup-ci-env: true` |
| `flutter-publish` | All of the above for enabled platforms | Same as Android + iOS publish |

---

## AI Code Review

| Secret | Required | Description |
|--------|----------|-------------|
| `ANTHROPIC_API_KEY` | yes | API key for [Claude Code Action](https://github.com/anthropics/claude-code-action) |

```yaml
jobs:
  ai-review:
    uses: mobyleOfficial/shared-workflows/.github/workflows/ai-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

---

## Android signing (`setup-android-signing`)

Required when building **signed** Android release binaries. Set **four env vars per deployment environment** on the job (map from repository secrets).

| Env var | Description | How to produce |
|---------|-------------|----------------|
| `ANDROID_KEYSTORE_BASE64_<ENV>` | Base64-encoded `.jks` / keystore file | `base64 -i upload-keystore.jks \| pbcopy` (macOS) |
| `ANDROID_KEYSTORE_PASSWORD_<ENV>` | Keystore password | From your signing config |
| `ANDROID_KEY_ALIAS_<ENV>` | Key alias | From `key.properties` |
| `ANDROID_KEY_PASSWORD_<ENV>` | Key password | From `key.properties` |

`<ENV>` is uppercase: `DEV`, `STAGING`, or `PROD` (matches `environment` input: `dev` → `DEV`).

**Example repository secrets** (names are your choice; map them in `env:`):

| Repository secret | Maps to env |
|-----------------|-------------|
| `ANDROID_KEYSTORE_BASE64_PROD` | `ANDROID_KEYSTORE_BASE64_PROD` |
| `ANDROID_KEYSTORE_PASSWORD_PROD` | `ANDROID_KEYSTORE_PASSWORD_PROD` |
| `ANDROID_KEY_ALIAS_PROD` | `ANDROID_KEY_ALIAS_PROD` |
| `ANDROID_KEY_PASSWORD_PROD` | `ANDROID_KEY_PASSWORD_PROD` |

Repeat for `_DEV` and `_STAGING` if you deploy those flavors.

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

When using `secrets: inherit` on `flutter-publish`, define the same repository secrets and map them on the **top-level caller job** `env:` so all nested jobs inherit them.

---

## Google Play (`android-publish`, `flutter-publish`)

Upload uses your **`distribute-command`** (typically Fastlane `deploy_play`). Fastlane reads JSON service account credentials from env — convention in Mobyle apps:

| Secret | Required when | Description |
|--------|---------------|-------------|
| `GOOGLE_PLAY_API_JSON_DEV` | Play upload to dev/internal | Full JSON key for Play Console API (dev track) |
| `GOOGLE_PLAY_API_JSON_STAGING` | Play upload to staging | Same for staging |
| `GOOGLE_PLAY_API_JSON_PROD` | Play upload to production | Same for production |

Create keys in [Google Cloud Console](https://console.cloud.google.com/) → IAM → Service accounts, grant Play Console access in Play Console → Users and permissions.

Your Fastlane lane selects the secret by environment (e.g. `GOOGLE_PLAY_API_JSON_${ENV}`). Pass via caller job `env:` or ensure `secrets: inherit` makes them available to the publish job.

**Not required** when `target: artifact-only` (no store upload).

---

## TestFlight / App Store Connect (`ios-publish`, `flutter-publish`)

Upload uses your **`distribute-command`** (typically Fastlane `deploy_testflight`). Mobyle convention — **per environment**:

| Secret | Description |
|--------|-------------|
| `APP_STORE_CONNECT_API_KEY_ID_<ENV>` | Key ID from App Store Connect → Users and Access → Integrations → App Store Connect API |
| `APP_STORE_CONNECT_API_KEY_ISSUER_ID_<ENV>` | Issuer ID (same page) |
| `APP_STORE_CONNECT_API_KEY_CONTENT_<ENV>` | Contents of the `.p8` private key file (paste as secret, or base64) |

`<ENV>` = `DEV`, `STAGING`, or `PROD`.

Prefer API keys over Apple ID + app-specific password in CI.

**Not required** when `target: github_release` (see below).

### iOS code signing (build-and-publish mode)

If `ios-publish` or native Fastlane **builds** on the runner (not `distribute-artifact` only), signing material is usually managed by **Fastlane Match**. Typical secrets (app/Fastfile specific):

| Secret | Description |
|--------|-------------|
| `MATCH_PASSWORD` | Passphrase for Match encrypted repo |
| `MATCH_GIT_BASIC_AUTHORIZATION` | Base64 of `username:personal_access_token` for Match cert repo |
| `MATCH_GIT_URL` | Git URL of certificates repo (if not in Fastfile) |

Flutter iOS release builds in `flutter-publish` assume your **`ios-build-command`** / Fastlane lane handles signing (Match or project config). Document the exact names your `Fastfile` expects alongside these shared defaults.

---

## GitHub Release (`ios-publish`, `target: github_release`)

| Secret / token | Required | Description |
|----------------|----------|-------------|
| `GITHUB_TOKEN` | yes (automatic) | Default Actions token; workflow requests `contents: write` |

No store credentials. Tag and release name come from workflow inputs (`release-tag`, `release-name`).

---

## Firebase / Google Services (Android CI)

| Secret | Required when | Description |
|--------|---------------|-------------|
| `GOOGLE_SERVICES_JSON` | `android-ci` with `write-google-services: true` | Base64-encoded `app/google-services.json` |

```bash
base64 -i app/google-services.json | pbcopy   # macOS
```

```yaml
jobs:
  android-ci:
    uses: mobyleOfficial/shared-workflows/.github/workflows/android-ci.yml@main
    secrets:
      GOOGLE_SERVICES_JSON: ${{ secrets.GOOGLE_SERVICES_JSON }}
```

---

## Flutter CI / app runtime config

Shared workflows do **not** define app-specific secret names. If you enable `setup-ci-env: true`, your Fastlane lane (e.g. `setup_ci_env`) decides which variables are required.

**Example (Muuvie-style)** — add to repository secrets and map on the job:

| Secret | Typical use |
|--------|-------------|
| `TMDB_API_KEY` | Injected into `.env` / dart-defines for CI builds |
| `BACKEND_URL` | API base URL for CI/test builds |

```yaml
jobs:
  flutter-ci:
    uses: mobyleOfficial/shared-workflows/.github/workflows/flutter-ci.yml@main
    env:
      TMDB_API_KEY: ${{ secrets.TMDB_API_KEY }}
      BACKEND_URL: ${{ secrets.BACKEND_URL }}
    with:
      setup-ci-env: true
      # ...
    secrets: inherit
```

Name and count of secrets depend on your app; list them in the consumer repo README when you adopt the workflow.

---

## `flutter-publish` — combined checklist

When both Android and iOS deploy are enabled (`deploy-android` + `deploy-ios`), configure at minimum:

### Android leg
- [ ] `ANDROID_KEYSTORE_BASE64_<ENV>` (+ password, alias, key password) for build signing
- [ ] `GOOGLE_PLAY_API_JSON_<ENV>` for Play upload (unless artifact-only)

### iOS leg
- [ ] Signing handled by your Fastlane build lane (often Match secrets)
- [ ] `APP_STORE_CONNECT_API_KEY_ID_<ENV>` (+ issuer, content) for TestFlight upload

### Optional (both)
- [ ] App CI env secrets if `setup-ci-env: true` (e.g. `TMDB_API_KEY`, `BACKEND_URL`)

Use GitHub **environment** protection rules on `prod` for approval gates before publish jobs run.

---

## Security notes

- Never commit keystores, `.p8`, `google-services.json`, or Play JSON to git.
- Prefer environment-scoped secrets for production (`prod` environment in GitHub).
- Signing files written to disk (`upload-keystore.jks`, `key.properties`) are deleted in cleanup steps; verify your caller workflow does not cache those paths.
- Rotate API keys if exposed; Play and ASC keys can be revoked independently.
- `secrets: inherit` passes **all** repository secrets to the reusable workflow — only enable on jobs that need them.

---

## Org prerequisite

Reusable workflows from a private `shared-workflows` repo require org/repo setting: **Actions → General → Access** → allow repositories in the organization to use workflows from this repo.
