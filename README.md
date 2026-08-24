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

