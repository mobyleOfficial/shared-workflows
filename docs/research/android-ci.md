# Native Android CI research notes

Sources consulted for `android-ci.yml`:

1. [actions/setup-java](https://github.com/actions/setup-java) — Temurin JDK + `cache: gradle`.
2. [gradle/actions/setup-gradle](https://github.com/gradle/actions) — dedicated Gradle cache/configuration caching vs JDK cache alone.
3. Android sample / community CI — `./gradlew testDebugUnitTest` + `lintDebug`, upload reports with `if: always()`.
4. Common pattern for Firebase: store base64 `google-services.json` as a secret and decode before Gradle.
5. AbbayAndroid `android-ci.yml` stub (JDK + google-services + chmod only) completed into a real unit-test/lint pipeline.
6. Emulator / `connectedDebugAndroidTest` left **opt-in but unimplemented** in v1 (`run-emulator-tests` fails closed) to avoid expensive flaky defaults.

Decisions:

- Single job for unit test + lint (simpler than Abbay’s incomplete stub).
- Emulator tests default off and explicitly unimplemented until a follow-up.
- Configurable Gradle commands and `working-directory` for multi-module layouts.
