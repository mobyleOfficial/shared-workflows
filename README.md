# shared-workflows

Reusable GitHub Actions workflows for the [mobyleOfficial](https://github.com/mobyleOfficial) organization.

## Available workflows

### Check Co-Authors

Fails pull requests that include AI ownership attribution in commit messages or the PR body (for example `Co-Authored-By`, `Generated with …`, `Made with …`).

**Caller (copy into each repo):**

```yaml
# .github/workflows/check-coauthors.yml
name: Check Co-Authors

on:
  pull_request_target:
    types: [opened, edited, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  check-coauthors:
    uses: mobyleOfficial/shared-workflows/.github/workflows/check-coauthors.yml@main
    permissions:
      contents: read
      pull-requests: write
```

Pin `@main` to a tag or commit SHA if you want stricter versioning.
