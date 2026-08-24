# Shared Workflows Migration Design

Date: 2026-08-23  
Status: approved (in-chat); pending implementation plan  
Source repos audited: Muuvie, AbbayiOS, AbbayAndroid

## Goal

Migrate **generic** and **reusable building-block** GitHub Actions from Mobyle app repos into `mobyleOfficial/shared-workflows`. Create everything here first; update consumers later. **One unit per pull request.**

## Scope bar (“generic”)

Include:

- Repo-agnostic workflows (e.g. AI review)
- Reusable Flutter / iOS / Android **building blocks and opinionated CI workflows** that other Mobyle mobile repos can call

Exclude (for this phase):

- App-specific publish/deploy pipelines (Muuvie Play/TestFlight, AbbayiOS GitHub Release `.app` zip) unless later generalized
- Consumer-repo caller cutovers (Muuvie / Abbay*)

## Architecture

**Workflow-first:** each PR ships one `workflow_call` reusable workflow (or one composite action when that is the unit). Extract composite actions under `.github/actions/` only when shared by two or more workflows.

```
.github/
  workflows/     # reusable workflow_call YAMLs (+ optional *-caller.yml examples)
  actions/       # composites only when reused by 2+ workflows
README.md        # one section per shipped unit: purpose, inputs, caller snippet
```

### Conventions

- Reusable workflows use `on: workflow_call` only (no direct app `pull_request`/`push` triggers in shared workflows themselves).
- Consumer callers stay thin; copy-paste snippets live in README (same pattern as `check-ai-contribution`).
- Prefer inputs with opinionated defaults; pass secrets via `secrets:` / `secrets: inherit`.
- Pin guidance: `@main` acceptable early; recommend tag or commit SHA for stricter versioning.
- One unit per PR; README updated in the same PR.
- No Muuvie/Abbay changes in this phase.

## Already done

| Muuvie | shared-workflows |
|--------|------------------|
| `check-coauthors.yml` | Superseded by `check-ai-contribution` (also checks PR body + Made/Generated with) |

## PR sequence

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
- Document a **versioned schema** (`schema_version: 1`).
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

- Structural YAML correctness; optional workflow-lint can be added later.
- No requirement to run real Xcode/Gradle/Flutter app builds against this repo for v1.
- Optional `*-caller.yml` examples only (not live gates against empty app code).

## Rollout

### Now (shared-workflows)

Ship PRs 1 → 6 in order; each merges independently.

### Later (consumers)

1. Add thin callers pointing at `mobyleOfficial/shared-workflows/...@main` (or a tag).
2. Remove duplicated local workflow/action bodies once proven on a PR.
3. Muuvie: switch `check-coauthors` → `check-ai-contribution` when convenient.
4. AbbayAndroid: replace incomplete `android-ci.yml` with shared Android CI.
5. AbbayiOS: adopt shared iOS CI; keep release/publish local until a shared publish workflow exists.

### Org prerequisite

Private reusable workflows require org/repo Actions access (“accessible from repositories in the organization”). Confirm before first consumer cutover.

## Explicit non-goals (this phase)

- Updating Muuvie / AbbayiOS / AbbayAndroid callers
- Shared Play Store / TestFlight / GitHub Release publish workflows
- Replacing Fastlane lane implementations inside apps

## Source inventory (reference)

| Source | Item | Disposition |
|--------|------|-------------|
| Muuvie | `check-coauthors.yml` | Already superseded here |
| Muuvie | `ai-review.yml` + prompts | PR1 |
| Muuvie | `determine-flavor` | PR2 |
| Muuvie | `load-config.yml` | PR3 |
| Muuvie | `setup-environment` / signing | PR4 (generalized) |
| Muuvie | `ci.yml`, publish YAMLs | Out of scope (app-specific); inform Flutter research only |
| AbbayiOS | `validate-branch.yml` | Informs PR5 |
| AbbayiOS | `release-app.yml` | Out of scope (publish) |
| AbbayAndroid | `android-ci.yml` (incomplete) | Informs PR6 |
