# Align Decision Check

A GitHub Action that checks PR alignment against your team's decisions tracked in [Align](https://align.tech). Add it to your CI pipeline to catch decision drift before it ships.

## What It Does

When a developer opens or updates a PR, this action:

1. Extracts the PR diff (title, description, changed files, patches)
2. Sends it to the Align API for analysis against your tracked decisions
3. Creates a GitHub check run showing the results (aligned, conflicts, or no relevant decisions)
4. Optionally fails the CI pipeline if conflicts exceed your threshold

No GitHub App installation required - just two secrets and a workflow file.

## Setup

### 1. Get your credentials

In [Align](https://app.align.tech), go to **Settings > API Keys**:
- Generate an **API key**
- Copy your **Tenant ID**

### 2. Add secrets to your repo

In your GitHub repo, go to **Settings > Secrets and variables > Actions** and add:

| Secret | Value |
|--------|-------|
| `ALIGN_API_KEY` | Your Align API key |
| `ALIGN_TENANT_ID` | Your Align tenant ID |

### 3. Add the workflow

Create `.github/workflows/decision-check.yml`:

```yaml
name: Decision Check
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  checks: write
  pull-requests: read
  contents: read

jobs:
  check-alignment:
    runs-on: ubuntu-latest
    steps:
      - uses: aligndottech/decision-check@v1
        with:
          align-api-key: ${{ secrets.ALIGN_API_KEY }}
          align-tenant-id: ${{ secrets.ALIGN_TENANT_ID }}
```

That's it. Every PR now gets checked against your team's decisions.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `align-api-key` | Yes | - | Align API key for authentication |
| `align-tenant-id` | Yes | - | Your Align tenant ID |
| `align-api-url` | No | `https://api.align.tech` | Align API URL (override for self-hosted) |
| `fail-on` | No | `critical` | When to fail: `critical`, `any`, or `none` |
| `github-token` | No | `${{ github.token }}` | GitHub token for creating check runs |

## Outputs

| Output | Description |
|--------|-------------|
| `status` | Alignment status: `aligned`, `conflicting`, or `no-context` |
| `conflicts-count` | Total number of conflicts found |
| `critical-count` | Number of critical conflicts |

## Configuration Examples

### Fail only on critical conflicts (default)

The action fails the check only when a PR directly contradicts a decision with high confidence.
Warnings are reported but don't block the PR.

```yaml
- uses: aligndottech/decision-check@v1
  with:
    align-api-key: ${{ secrets.ALIGN_API_KEY }}
    align-tenant-id: ${{ secrets.ALIGN_TENANT_ID }}
    fail-on: critical
```

### Fail on any conflict

Stricter mode - any conflict (including warnings) fails the check.

```yaml
- uses: aligndottech/decision-check@v1
  with:
    align-api-key: ${{ secrets.ALIGN_API_KEY }}
    align-tenant-id: ${{ secrets.ALIGN_TENANT_ID }}
    fail-on: any
```

### Report only (never fail)

Shows results in the check run but never blocks the PR. Good for rollout.

```yaml
- uses: aligndottech/decision-check@v1
  with:
    align-api-key: ${{ secrets.ALIGN_API_KEY }}
    align-tenant-id: ${{ secrets.ALIGN_TENANT_ID }}
    fail-on: none
```

### Use outputs in follow-up steps

```yaml
- uses: aligndottech/decision-check@v1
  id: decision-check
  with:
    align-api-key: ${{ secrets.ALIGN_API_KEY }}
    align-tenant-id: ${{ secrets.ALIGN_TENANT_ID }}
    fail-on: none

- name: Comment on PR if conflicts found
  if: steps.decision-check.outputs.conflicts-count != '0'
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        owner: context.repo.owner,
        repo: context.repo.repo,
        issue_number: context.issue.number,
        body: `Decision check found ${{ steps.decision-check.outputs.conflicts-count }} conflict(s). Review the check run for details.`
      })
```

### Self-hosted Align

If you're running Align on your own infrastructure:

```yaml
- uses: aligndottech/decision-check@v1
  with:
    align-api-key: ${{ secrets.ALIGN_API_KEY }}
    align-tenant-id: ${{ secrets.ALIGN_TENANT_ID }}
    align-api-url: https://api.align.yourcompany.com
```

## Check Run Results

The action creates a GitHub check run called "Align Decision Check":

| Result | Meaning |
|--------|---------|
| **Pass** | PR aligns with tracked decisions. Shows related decisions with match percentages. |
| **Neutral** | No relevant decisions found, or conflicts are below your `fail-on` threshold. |
| **Fail** | PR conflicts with tracked decisions. Shows conflict details, severity, and suggested resolutions. |

## How It Differs from the Align GitHub App

| | Decision Check (this action) | Align GitHub App |
|---|---|---|
| **Install** | Workflow file + 2 secrets | GitHub App install (org admin) |
| **Scope** | PR alignment checking only | Full integration: decision capture, PR linking, bidirectional sync |
| **Runs on** | Your CI runner | Align's infrastructure |
| **Best for** | Quick adoption, specific repos | Organization-wide deep integration |

Both can coexist - use this action on repos that aren't connected to the GitHub App yet.

## Permissions

The action needs these GitHub token permissions:
- `checks: write` - to create check runs on the PR
- `pull-requests: read` - to read PR diff and metadata
- `contents: read` - to access repository content

## Troubleshooting

**"Align API error: 401 Unauthorized"** - Check that `ALIGN_API_KEY` is correct and not expired.

**"Align API error: 403 Forbidden"** - Check that `ALIGN_TENANT_ID` matches the tenant the API key belongs to.

**"This action only works on pull_request events"** - The action must be triggered by `pull_request` events. Check your workflow `on:` trigger.

**Check run shows "No relevant decisions"** - Your Align tenant may not have decisions tracked yet, or the PR doesn't relate to any tracked decisions.
