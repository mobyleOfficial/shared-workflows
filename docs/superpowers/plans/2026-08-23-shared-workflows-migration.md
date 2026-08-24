# Shared Workflows Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add six reusable GitHub Actions units to `mobyleOfficial/shared-workflows` (one PR each), workflow-first, without changing consumer repos yet.

**Architecture:** Each unit is a `workflow_call` reusable workflow or a single composite action. Callers stay thin. Prompts and docs live in this repo. PR4–PR6 include a short public-repo research pass before final YAML.

**Tech Stack:** GitHub Actions (`workflow_call`, composite actions), `actions/checkout`, `mikefarah/yq`, `anthropics/claude-code-action`, Flutter/iOS/Android CI actions as researched per task.

**Spec:** `docs/superpowers/specs/2026-08-23-shared-workflows-migration-design.md`

## Global Constraints

- One unit per pull request; README updated in the same PR.
- Reusable workflows: `on: workflow_call` only (no app `pull_request`/`push` on the shared workflow itself).
- Optional `*-caller.yml` examples allowed (same pattern as `check-ai-contribution-caller.yml`).
- Pin major action versions (`@v4`-style); least-privilege `permissions`.
- No hardcoded Muuvie secrets (`TMDB_*`, etc.).
- No consumer-repo (Muuvie/Abbay*) changes in this plan.
- Do not implement work on `main`; use a feature branch (or worktree) per PR.
- Pin guidance in README: `@main` OK early; recommend tag/SHA later.
- Verification: YAML parse / `actionlint` if installed; no live Flutter/Xcode/Gradle builds required in this repo.

## File map (end state after all tasks)

```
.github/
  workflows/
    check-ai-contribution.yml          # existing
    check-ai-contribution-caller.yml   # existing
    ai-review.yml                      # PR1
    ai-review-caller.yml               # PR1
    ai-review/
      security-reviewer.md             # PR1
      bug-finder.md                    # PR1
      architecture-reviewer.md         # PR1
    load-config.yml                    # PR3
    ios-ci.yml                         # PR5
    ios-ci-caller.yml                  # PR5
    android-ci.yml                     # PR6
    android-ci-caller.yml              # PR6
  actions/
    determine-flavor/action.yml        # PR2
    setup-flutter-environment/action.yml  # PR4
    setup-android-signing/action.yml   # PR4 (optional if kept generic)
docs/
  actions-schema.md                    # PR3 (schema_version: 1)
README.md                              # updated each PR
```

---

### Task 1: AI Review reusable workflow (PR1)

**Files:**
- Create: `.github/workflows/ai-review.yml`
- Create: `.github/workflows/ai-review-caller.yml`
- Create: `.github/workflows/ai-review/security-reviewer.md`
- Create: `.github/workflows/ai-review/bug-finder.md`
- Create: `.github/workflows/ai-review/architecture-reviewer.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: `secrets.ANTHROPIC_API_KEY`; caller PR event context (`github.event.pull_request`)
- Produces: reusable workflow `mobyleOfficial/shared-workflows/.github/workflows/ai-review.yml@<ref>`

**Design notes (must follow):**
- Do **not** depend on `load-config` (that is PR3). When called, the job runs.
- Checkout the **caller** repo (default), then checkout `mobyleOfficial/shared-workflows` into `.shared-workflows` so Claude can read prompt files from this repo (caller repos will not have those paths).
- Update orchestration prompt paths to `.shared-workflows/.github/workflows/ai-review/*.md`.
- Grant `pull-requests: write` (and keep contents/issues/id-token as needed) so the Claude action can post review comments.
- Port prompt markdown verbatim from Muuvie.
- Port orchestration prompt logic from Muuvie `ai-review.yml` (steps 1–8).

- [ ] **Step 1: Create feature branch**

```bash
git checkout main && git pull origin main
git checkout -b add-ai-review-workflow
```

- [ ] **Step 2: Add the three reviewer prompt files**

Copy Muuvie contents into:
- `.github/workflows/ai-review/security-reviewer.md`
- `.github/workflows/ai-review/bug-finder.md`
- `.github/workflows/ai-review/architecture-reviewer.md`

Source: `mobyleOfficial/Muuvie` paths under `.github/workflows/ai-review/`.

- [ ] **Step 3: Write `.github/workflows/ai-review.yml`**

```yaml
name: AI Code Review

on:
  workflow_call:
    secrets:
      ANTHROPIC_API_KEY:
        required: true

permissions:
  contents: read
  pull-requests: write
  issues: read
  id-token: write

jobs:
  ai-review:
    name: AI Code Review
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      issues: read
      id-token: write

    steps:
      - name: Checkout caller repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Checkout shared prompts
        uses: actions/checkout@v4
        with:
          repository: mobyleOfficial/shared-workflows
          path: .shared-workflows
          sparse-checkout: |
            .github/workflows/ai-review
          sparse-checkout-cone-mode: false

      - name: Run AI Code Review
        id: ai-review
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Provide a comprehensive code review for PR #${{ github.event.pull_request.number }} in ${{ github.repository }}.

            You will coordinate multiple specialized reviewers and produce inline review comments.

            Follow these steps EXACTLY:

            ---

            Step 1 — Understand the context
            - Read PR title, description, and diff
            - Identify intent of the change
            - Do NOT review yet

            ---

            Step 2 — Run independent reviewers

            Run:
            1. Security Reviewer (use: .shared-workflows/.github/workflows/ai-review/security-reviewer.md)
            2. Bug Finder (use: .shared-workflows/.github/workflows/ai-review/bug-finder.md)
            3. Architecture Reviewer (use: .shared-workflows/.github/workflows/ai-review/architecture-reviewer.md)

            Each must:
            - Work independently
            - Return STRICT JSON
            - Focus only on their domain

            ---

            Step 3 — Aggregate findings
            - Combine all findings into a single list
            - Tag each with:
              - category (security | bug | architecture)
              - source reviewer

            ---

            Step 4 — Validate findings (CRITICAL)

            For EACH finding:
            - Re-evaluate from scratch
            - Attempt to DISPROVE it
            - Classify:
              - VALID
              - UNCERTAIN
              - INVALID

            Assign:
            - confidence score (0.0 → 1.0)

            Rules:
            - Keep only VALID or strong UNCERTAIN (confidence ≥ 0.6)
            - Drop everything else
            - Prefer missing an issue over false positives

            ---

            Step 5 — Assign risk level

            For each validated finding, compute:

            Risk = severity × likelihood × impact

            Then classify:
            - CRITICAL → exploitable or breaks core functionality
            - HIGH → serious issue, likely in real usage
            - MEDIUM → edge cases or limited scope
            - LOW → minor impact

            ---

            Step 6 — Deduplicate
            - Merge overlapping findings
            - Keep the clearest explanation
            - Preserve strongest evidence

            ---

            Step 7 — Map to diff lines (IMPORTANT)

            For EACH finding:
            - Identify exact file path
            - Identify the closest changed line in the diff
            - If exact line is unclear, pick the most relevant nearby line
            - NEVER leave location empty

            ---

            Step 8 — Generate inline PR comments

            Each finding becomes ONE inline comment.

            Output MUST be STRICT JSON:

            {
              "summary": {
                "overall_risk": "Low | Medium | High | Critical",
                "total_findings": number,
                "posted_comments": number
              },
              "comments": [
                {
                  "path": "file/path.ext",
                  "line": 123,
                  "side": "RIGHT",
                  "category": "security | bug | architecture",
                  "risk": "CRITICAL | HIGH | MEDIUM | LOW",
                  "confidence": 0.0,
                  "title": "Short issue title",
                  "body": "Clear explanation of the issue, why it matters, and how to fix it."
                }
              ]
            }

            ---

            Strict rules:
            - ONLY include validated findings
            - Confidence MUST reflect real certainty (no fake 0.9 everywhere)
            - Keep comments concise and actionable
            - No duplicate comments
            - If no issues → return empty comments array
```

If sparse-checkout fails on the runner’s checkout version, drop sparse-checkout and checkout the full shared-workflows repo into `.shared-workflows` instead (same path contract).

- [ ] **Step 4: Write `.github/workflows/ai-review-caller.yml`**

```yaml
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

Note: enabling this caller on *this* repo is optional; if `ANTHROPIC_API_KEY` is not configured here, either omit the caller file and document only in README, or keep the caller and accept it will fail until the secret exists. Prefer shipping the caller (matches existing coauthors pattern) and document the secret requirement.

- [ ] **Step 5: Update README.md**

Add a section **AI Code Review** after Check AI Contribution:
- One-paragraph purpose
- Required secret `ANTHROPIC_API_KEY`
- Copy-paste caller snippet (same as Step 4 job body / thin caller)
- Note that reviewer prompts live in this repo and are checked out at runtime into `.shared-workflows/`

- [ ] **Step 6: Validate YAML**

```bash
python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/ai-review.yml')); yaml.safe_load(open('.github/workflows/ai-review-caller.yml')); print('ok')"
# if actionlint is installed:
actionlint .github/workflows/ai-review.yml .github/workflows/ai-review-caller.yml || true
```

Expected: `ok` (and actionlint clean if available).

- [ ] **Step 7: Commit, push, open PR**

```bash
git add .github/workflows/ai-review.yml \
  .github/workflows/ai-review-caller.yml \
  .github/workflows/ai-review \
  README.md
git commit -m "$(cat <<'EOF'
Add reusable AI Code Review workflow.

Port Muuvie's Claude multi-reviewer flow into a workflow_call unit with prompts hosted here.
EOF
)"
git push -u origin HEAD
gh pr create --title "Add reusable AI Code Review workflow" --body "$(cat <<'EOF'
## Summary
- Add `workflow_call` AI Code Review workflow ported from Muuvie
- Host security / bug / architecture reviewer prompts in this repo
- Document thin caller + `ANTHROPIC_API_KEY` requirement

## Test plan
- [ ] YAML loads / actionlint (if available)
- [ ] Caller snippet matches README
- [ ] Prompt paths reference `.shared-workflows/...` (not caller `.github/...`)
EOF
)"
```

---

### Task 2: `determine-flavor` composite action (PR2)

**Files:**
- Create: `.github/actions/determine-flavor/action.yml`
- Modify: `README.md`

**Interfaces:**
- Consumes: input `environment` (optional, default `dev`); `github.event_name`
- Produces: output `flavor` (`dev` | `staging` | `prod`)

- [ ] **Step 1: Branch from latest main**

```bash
git checkout main && git pull origin main
git checkout -b add-determine-flavor-action
```

- [ ] **Step 2: Create `.github/actions/determine-flavor/action.yml`**

Use Muuvie’s action verbatim (composite bash step: push → `prod`, else validate `environment`).

- [ ] **Step 3: Document in README** (inputs/outputs + example `uses:` snippet).

- [ ] **Step 4: Commit, push, open PR** titled `Add determine-flavor composite action`.

---

### Task 3: `load-config` reusable workflow (PR3)

**Files:**
- Create: `.github/workflows/load-config.yml`
- Create: `docs/actions-schema.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: caller checkout’s `.actions.yml`
- Produces: workflow outputs listed in Muuvie `load-config.yml` plus require `schema_version`

- [ ] **Step 1: Branch `add-load-config-workflow` from main.**

- [ ] **Step 2: Write `docs/actions-schema.md`** documenting `schema_version: 1` and Flutter-oriented keys from Muuvie (flags + tool versions). State that iOS/Android keys will be added when PR5/PR6 land.

- [ ] **Step 3: Write `load-config.yml` as `workflow_call` with outputs.** Read `.actions.yml` via `mikefarah/yq@v4`. Fail if file missing or required keys null. Require `schema_version` == `1` (fail with clear message otherwise).

- [ ] **Step 4: README section + example caller that maps outputs.**

- [ ] **Step 5: YAML validate, commit, push, open PR** `Add load-config reusable workflow`.

---

### Task 4: Flutter building blocks (PR4)

**Files:**
- Create: `.github/actions/setup-flutter-environment/action.yml`
- Create (if generic): `.github/actions/setup-android-signing/action.yml`
- Modify: `README.md`
- Optionally add: `docs/research/flutter-ci.md` (short citations)

**Interfaces:**
- Consumes: inputs `flutter-version`, `ruby-version`, optional `xcode-version`, `setup-java`, `java-version`, `java-distribution`, `run-codegen` (bool), `setup-ci-env` (bool), generic env map or skippable CI-env command input
- Produces: configured runner (Flutter/Ruby/optional Java/Xcode); no workflow outputs required

- [ ] **Step 1: Research (write 5–15 line notes in PR body or `docs/research/flutter-ci.md`)** covering official Flutter CI, `subosito/flutter-action`, caching, analyze/test/lcov. Cite 2–4 links. Treat Muuvie as baseline only.

- [ ] **Step 2: Branch `add-setup-flutter-environment` from main.**

- [ ] **Step 3: Implement `setup-flutter-environment`** generalized from Muuvie `setup-environment`:
  - Optional Java / Xcode
  - Flutter + Ruby + `flutter pub get`
  - Codegen / CI-env behind inputs; **no** `TMDB_API_KEY` / `BACKEND_URL` hardcoding — use optional `ci-env-command` input defaulting to empty/skip, or pass-through env file pattern documented in README

- [ ] **Step 4 (optional same PR if still generic):** `setup-android-signing` with env-suffixed secret *names documented for the caller to export into `env:`*, or document required env vars the composite reads. Prefer reading from process env (caller maps secrets) to avoid impossible dynamic `secrets.*` lookups inside composites.

- [ ] **Step 5: README + validate + PR** `Add Flutter setup composite actions`.

---

### Task 5: Native iOS CI workflow (PR5)

**Files:**
- Create: `.github/workflows/ios-ci.yml`
- Create: `.github/workflows/ios-ci-caller.yml`
- Modify: `README.md`, `docs/actions-schema.md` (add iOS keys if any)
- Optionally: `docs/research/ios-ci.md`

**Interfaces:**
- Consumes: inputs e.g. `xcode-version`, `macos-version`, `run-swiftlint`, `run-tests`, `swiftlint-version`/`image`, `test-command` (default Fastlane), `working-directory`
- Produces: reusable `ios-ci.yml`; artifacts for xcresult / test reports

- [ ] **Step 1: Research** starter Xcode workflows, Fastlane CI, setup-xcode, xcresult tools, caching. Cite links in PR/`docs/research/ios-ci.md`. Baseline AbbayiOS `validate-branch.yml`.

- [ ] **Step 2: Branch `add-ios-ci-workflow`.**

- [ ] **Step 3: Implement opinionated `ios-ci.yml`** with jobs gated by inputs (SwiftLint optional; test job on macOS; upload xcresult; report action). Pin current major action versions. Least-privilege permissions.

- [ ] **Step 4: Caller example + README + schema updates + PR** `Add reusable native iOS CI workflow`.

---

### Task 6: Native Android CI workflow (PR6)

**Files:**
- Create: `.github/workflows/android-ci.yml`
- Create: `.github/workflows/android-ci-caller.yml`
- Modify: `README.md`, `docs/actions-schema.md`
- Optionally: `docs/research/android-ci.md`

**Interfaces:**
- Consumes: inputs e.g. `java-version`, `java-distribution`, `run-lint`, `run-unit-tests`, `write-google-services` (bool), secrets `GOOGLE_SERVICES_JSON` (optional)
- Produces: reusable `android-ci.yml`; test/lint artifacts

- [ ] **Step 1: Research** Android starter workflows, setup-java, Gradle cache, unit vs emulator, google-services injection. Cite links. Baseline AbbayAndroid stub (complete the missing test/lint steps).

- [ ] **Step 2: Branch `add-android-ci-workflow`.**

- [ ] **Step 3: Implement `android-ci.yml`** — JDK, Gradle cache, optional google-services from secret, `./gradlew` test + lint, artifacts. Emulator UI tests **default off** (input to enable later).

- [ ] **Step 4: Caller example + README + schema + PR** `Add reusable native Android CI workflow`.

---

## Spec coverage checklist

| Spec item | Task |
|-----------|------|
| PR1 ai-review + prompts here | Task 1 |
| PR2 determine-flavor | Task 2 |
| PR3 load-config + schema_version | Task 3 |
| PR4 Flutter blocks + research | Task 4 |
| PR5 iOS CI + research | Task 5 |
| PR6 Android CI + research | Task 6 |
| One unit per PR | Each task opens its own PR |
| No consumer updates | All tasks |
| Workflow-first | Tasks 1,3,5,6 workflows; 2,4 composites |

## Execution order

Always merge (or at least open) Task N before starting Task N+1 only when needed for dependencies. **Hard dependency:** none require prior merges to *function*, but README conflicts are easier if PRs merge in order 1→6. Prefer sequential open→merge for cleanliness.
