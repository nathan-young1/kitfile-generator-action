# KitOps Kitfile Generator

This GitHub Action scans your repo, generates a `Kitfile` with the [Kit CLI](https://github.com/kitops-ml/kitops), validates it with `kit pack`, and opens a pull request adding the Kitfile to your repo.

## What it does

1. Installs the Kit CLI on the runner
2. Runs `kit init .` against your repo to auto-generate a `Kitfile`
3. Validates the generated `Kitfile` by packing it locally (this is just a local sanity check, no registry or login is required)
4. Opens a pull request adding the new or updated `Kitfile` to your repo, if one doesn't already exist or your repo contents has changed. If nothing changed, the action finishes successfully with no PR opened.

## Usage

Create `.github/workflows/kitfile-scan.yml` in your repo:

```yaml
name: Generate Kitfile

on:
  workflow_dispatch:
  push:
    branches: [main]
    paths-ignore:
      - 'Kitfile'

concurrency:
  group: kitfile-generator-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: write
  pull-requests: write

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: nathan-young1/kitfile-generator-action@v1
```

Notes:
- `workflow_dispatch` is included so you can also trigger a Kitfile generation manually.
- `paths-ignore: ['Kitfile']` stops the workflow from re-triggering itself when one of this action's own PRs gets merged back into `main`. Since that merge only touches the `Kitfile`, there's nothing new to scan for.

## One-time repo setup (required)
 
Before this action can open pull requests, two settings need to be enabled on the repo you're installing it into. Go to **Settings → Actions → General → Workflow permissions**:
 
1. Select **"Read and write permissions"** (the default on many repos is read-only, which silently overrides the `permissions:` block in the workflow above)
2. Check **"Allow GitHub Actions to create and approve pull requests"**
3. Click **Save**
Without both of these, the action will fail at the "Open pull request" step.

## Inputs

| Name | Description | Required | Default |
|---|---|---|---|
| `kit-version` | Kit CLI version to install | No | `latest` |
| `pack-tag` | Local tag used to validate the Kitfile via `kit pack` | No | `local/validate:latest` |
| `pr-branch` | Branch name used for the pull request | No | `kitops/add-kitfile` |
| `github-token` | Token used to open the PR | No | `${{ github.token }}` |

### Example with custom inputs

```yaml
- uses: nathan-young1/kitfile-generator-action@v1
  with:
    kit-version: v0.4.0
    pr-branch: kitops/update-kitfile
```

## Permissions

Your workflow needs:

```yaml
permissions:
  contents: write
  pull-requests: write
```

Without these, the action can generate the `Kitfile` but will fail to open the PR. (See also the one-time repo setup section above as the workflow-level `permissions:` block alone isn't enough on repos with read-only defaults.)

## What "validate" means here

The `kit pack` step in this action packs to a throwaway **local** tag (`local/validate:latest` by default). It never contacts a registry and requires no credentials. Its only job is to confirm the generated `Kitfile` is well-formed enough to actually be packed.

If you want to publish a ModelKit to a registry (e.g. Jozu Hub) as part of your CI/CD, see the companion action: [`kit-pack-push-action`](https://github.com/nathan-young1/kit-pack-push-action).

## Concurrency

The example workflow above includes a `concurrency` block. This matters because rapid successive pushes to the same branch can race multiple runs which generates a `Kitfile` and pushes to the same PR branch (`kitops/add-kitfile`) at the same time causing conflict. `cancel-in-progress: true` ensures only the most recent push's state matters, which is what you would want here as you only care about the current repo contents.

## Troubleshooting

- **No PR opens on a run:** this is expected if `kit init .` produced no changes to the `Kitfile` (i.e. it's already up to date). [`create-pull-request`](https://github.com/peter-evans/create-pull-request), used internally, only opens a PR when there's an actual diff.
- **`Error: GitHub Actions is not permitted to create or approve pull requests`:** see the "One-time repo setup" section above on how to enable "Allow GitHub Actions to create and approve pull requests" in repo settings.
- **PR step fails even with `permissions: pull-requests: write` set:** the repo-level default under Settings → Actions → General is likely set to read-only, which overrides the workflow file's `permissions:` block. Switch it to "Read and write permissions."
- **`kit pack` validation fails:** this usually means the generated `Kitfile` references paths or artifacts `kit init` couldn't fully resolve. Run `kit init .` and `kit pack .` locally to debug.

## License

[Put the licence Kitops or Jozu wants to use here]

## Feedback / Issues

Please file issues in this repository. For questions about the Kit CLI itself, see the [main KitOps repository](https://github.com/kitops-ml/kitops).