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
