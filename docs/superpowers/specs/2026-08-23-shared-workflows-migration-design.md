# Shared Workflows Migration Design

Date: 2026-08-23  
Updated: 2026-08-24  
Status: implemented in `mobyleOfficial/shared-workflows` (consumer cutovers still later)  
Source repos audited: Muuvie, AbbayiOS, AbbayAndroid

## Goal

Migrate **generic** and **reusable building-block** GitHub Actions from Mobyle app repos into `mobyleOfficial/shared-workflows`. Create everything here first; update consumers later. **One unit per pull request.**

## Scope bar (“generic”)

Include:

- Repo-agnostic workflows (e.g. AI review)
- Reusable Flutter / iOS / Android **building blocks and opinionated CI workflows** that other Mobyle mobile repos can call

Exclude from the original migration phase (still true for consumer repos):

- Consumer-repo caller cutovers (Muuvie / Abbay*)
- Replacing Fastlane lane implementations inside apps

Publish workflows were later added **in this repo** as generic reusable units (`ios-publish`, `android-publish`, `flutter-publish`) with `build-and-publish` and `distribute-artifact` modes. App Fastfiles stay in consumer repos.

## Architecture

**Workflow-first:** each PR ships one `workflow_call` reusable workflow (or one composite action when that is the unit). Extract composite actions under `.github/actions/` only when shared by two or more workflows.

```
.github/
  workflows/     # reusable workflow_call YAMLs (+ optional *-caller.yml examples)
  actions/       # composites only when reused by 2+ workflows
docs/
  examples/      # copy-paste .actions.yml templates
  secrets.md
  actions-schema.md
README.md        # one section per shipped unit: purpose, inputs, caller snippet
```

### Conventions

- Reusable workflows use `on: workflow_call` only (no direct app `pull_request`/`push` triggers in shared workflows themselves), except this repo’s own gates (`check-ai-contribution-caller`, `lint`).
- Consumer callers stay thin; copy-paste snippets live in README (same pattern as `check-ai-contribution`).
- Prefer inputs with opinionated defaults; pass secrets via `secrets:` / `secrets: inherit`.
- Pin guidance: recommend a release tag (`@v1.0.0`) or commit SHA; `@main` is acceptable for early dogfood.
- One unit per PR; README updated in the same PR (later enhancement PRs may batch related polish).
- No Muuvie/Abbay changes in this phase.

## Already done

| Muuvie | shared-workflows |
|--------|------------------|
| `check-coauthors.yml` | Superseded by `check-ai-contribution` (also checks PR body + Made/Generated with) |

## Implementation status

Shipped on `main` (see GitHub PRs #3–#11 plus polish for examples/lint/v1 tag):

| Unit | Location |
|------|----------|
| Check AI contribution | `.github/workflows/check-ai-contribution.yml` |
| AI Code Review | `.github/workflows/ai-review.yml` + prompts |
| Determine flavor | `.github/actions/determine-flavor/` |
| Load config (schema v1 + v2) | `.github/workflows/load-config.yml`, `docs/actions-schema.md` |
| Flutter setup / Android signing | `.github/actions/setup-flutter-environment/`, `setup-android-signing/` |
| Native iOS CI | `.github/workflows/ios-ci.yml` |
| Native Android CI | `.github/workflows/android-ci.yml` |
| Native iOS/Android publish | `.github/workflows/ios-publish.yml`, `android-publish.yml` |
| Flutter CI / publish | `.github/workflows/flutter-ci.yml`, `flutter-publish.yml` |
| Secrets reference | `docs/secrets.md` |
| Example `.actions.yml` | `docs/examples/` |
| Workflow lint | `.github/workflows/lint.yml` (actionlint) |

Android emulator UI tests remain reserved (`run-emulator-tests` fails closed).

## Original PR sequence (completed)

### PR1 — `ai-review`

- Reusable workflow ported from Muuvie AI Code Review (Claude multi-reviewer orchestration).
- Prompts live in this repo (e.g. under `.github/workflows/ai-review/` or `prompts/`).
- Secret: `ANTHROPIC_API_KEY`.
- No Muuvie-specific filesystem assumptions beyond checkout + prompt paths resolved in shared repo or via documented inputs.

### PR2 — `determine-flavor`

- Composite action: on `push` → `prod`; otherwise use `environment` input (`dev` \| `staging` \| `prod`).
- Same behavior as Muuvie `.github/actions/determine-flavor`.

### PR3 — `load-config`

- Reusable workflow reading `.actions.yml` with `yq`.
- Document a **versioned schema** (`schema_version: 1`, later **v2** with `testing` / `publishing`).
- Start with Flutter-oriented keys used today; extend documented keys as iOS/Android PRs land.
- Missing required keys → fail with a clear error; unknown keys ignored.
- Outputs mirror the useful Muuvie set (feature flags + tool versions).

### PR4 — Flutter building blocks

- Research public Flutter CI practices before finalizing (see Research bar).
- Composite `setup-flutter-environment`: Flutter + Ruby (+ optional Java / Xcode), `pub get`, optional codegen / CI-env steps behind inputs.
- No hardcoded Muuvie secrets (`TMDB_*`, etc.); use generic inputs or skippable steps.
- Optional same-track or follow-up: generic `setup-android-signing` (env-suffixed secrets; configurable android paths with defaults).

### PR5 — Native iOS CI

- Opinionated `workflow_call` reusable workflow.
- Baseline: AbbayiOS `validate-branch` (SwiftLint optional, Xcode, Fastlane/tests, xcresult artifact + report).
- Harden with researched public best practices (action pins, caching, permissions, simulator/device inputs).
- Publish/release workflows stay out of scope for this PR.

### PR6 — Native Android CI

- Opinionated `workflow_call` reusable workflow.
- Complete what AbbayAndroid’s stub started (JDK, Gradle cache, optional `google-services.json` from secret, unit tests, lint, artifacts).
- Harden with researched public best practices.
- Emulator UI tests optional via input (default off or documented separately if costly).

## Later work in this repo (completed)

- Native-first publish workflows with `distribute-artifact` so Flutter can hand off signed AAB/IPA.
- `flutter-ci` / `flutter-publish` composing native publish.
- Secrets documentation; example configs; actionlint CI; release tag for caller pinning.

## Research bar

Before implementing PR4–PR6, research and cite references in the PR description and/or README:

| Track | Research targets |
|-------|------------------|
| Flutter | Official Flutter CI docs, `subosito/flutter-action` patterns, pub/Gradle caching, analyze + test + lcov coverage, channel/platform matrix; Fastlane-in-Flutter pipelines. Muuvie CI is a **baseline**, not the gold standard. |
| iOS | `actions/starter-workflows` Xcode samples, Fastlane CI docs, `setup-xcode`, xcresult reporting, simulator selection, SPM/CocoaPods/Bundler caching |
| Android | Android starter workflows, `setup-java` + Gradle caching, unit-test vs emulator split, lint, injecting `google-services.json` from secrets |

## Quality bar

- Pin major action versions (`@v4`-style); least-privilege `permissions`.
- Clear inputs with defaults; fail fast with actionable errors.
- Upload useful artifacts on failure (logs, test reports).
- README section per unit: purpose, inputs/outputs/secrets, copy-paste caller.
- Never log secrets; clean up keystores / signing files written to disk.

## Verification (this repo)

- Structural YAML correctness via `actionlint` (`.github/workflows/lint.yml`).
- No requirement to run real Xcode/Gradle/Flutter app builds against this repo for v1.
- Optional `*-caller.yml` examples only (not live gates against empty app code).

## Rollout

### Now (shared-workflows)

Original PRs 1 → 6 plus publish/config/docs polish are on `main`. Callers should pin a `v1` release tag when available.

### Later (consumers)

1. Add thin callers pointing at `mobyleOfficial/shared-workflows/...@v1.0.0` (or `@main` while dogfooding).
2. Remove duplicated local workflow/action bodies once proven on a PR.
3. Muuvie: switch `check-coauthors` → `check-ai-contribution` when convenient; adopt `flutter-ci` / `flutter-publish`.
4. AbbayAndroid: replace incomplete `android-ci.yml` with shared Android CI / publish.
5. AbbayiOS: adopt shared iOS CI; use `ios-publish` (`testflight` or `github_release`) instead of keeping release YAML local.

### Org prerequisite

This repository is public, so other org repos can call its reusable workflows without extra Actions access settings. If it is ever made private, enable org/repo Actions access (“accessible from repositories in the organization”) before consumer cutover.

## Explicit non-goals (this phase)

- Updating Muuvie / AbbayiOS / AbbayAndroid callers
- Replacing Fastlane lane implementations inside apps

## Source inventory (reference)

| Source | Item | Disposition |
|--------|------|-------------|
| Muuvie | `check-coauthors.yml` | Already superseded here |
| Muuvie | `ai-review.yml` + prompts | Shipped (PR1) |
| Muuvie | `determine-flavor` | Shipped (PR2) |
| Muuvie | `load-config.yml` | Shipped (PR3; schema later extended to v2) |
| Muuvie | `setup-environment` / signing | Shipped (PR4, generalized) |
| Muuvie | `ci.yml`, publish YAMLs | Informed Flutter CI/publish; generalized in this repo |
| AbbayiOS | `validate-branch.yml` | Informs `ios-ci` |
| AbbayiOS | `release-app.yml` | Informs `ios-publish` `github_release` target |
| AbbayAndroid | `android-ci.yml` (incomplete) | Informs `android-ci` / `android-publish` |
