# Native iOS CI research notes

Sources consulted for `ios-ci.yml`:

1. [fastlane GitHub Actions docs](https://docs.fastlane.tools/best-practices/continuous-integration/github/) — macOS runners, Ruby + Bundler cache, Fastlane test lanes.
2. Community iOS CI guides — pin Xcode via `maxim-lobanov/setup-xcode` / `xcode-select`; run SwiftLint on cheaper `ubuntu-latest` before macOS tests.
3. AbbayiOS `validate-branch.yml` baseline — SwiftLint container, Fastlane tests, xcresult zip + `slidoapp/xcresulttool` report.
4. Publish/TestFlight/Match signing left out of this workflow (CI only).

Decisions:

- Gated jobs via `run-swiftlint` / `run-tests` inputs.
- Configurable `test-command` and `xcresult-path` so apps without Fastlane can override.
- Artifact upload + xcresult report with `continue-on-error` on the reporter so missing bundles do not mask test failures incorrectly.
