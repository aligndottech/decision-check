# Align Decision Check

A GitHub Action that checks PR alignment against your team's decisions tracked in [Align](https://align.tech). Add it to your CI pipeline to catch decision drift before it ships.

## What It Does

When a developer opens or updates a PR, this action:

1. Extracts the PR diff (title, description, changed files, patches)
2. Sends it to the Align API for analysis against your tracked decisions
3. Creates a GitHub check run showing the results (aligned, conflicts, or no relevant decisions)
4. Optionally fails the CI pipeline if conflicts exceed your threshold

No GitHub App installation required - just two secrets and a workflow file.

## Distribution

This action is published at [github.com/aligndottech/decision-check](https://github.com/aligndottech/decision-check).

### `@v1` or a pinned SHA?

`v1` is a **moving major tag**: it is re-pointed at each 1.x release, so you pick up fixes without doing anything. That is the recommended default, and it is deliberate. This action's job is to be honest about its own degradation, and the failure mode of pinning is that a fix to exactly that behaviour never reaches you - which is how a build that reported success when the analysis service was unreachable stayed on this tag from 2026-03-29 to 2026-08-04.

```yaml
uses: aligndottech/decision-check@v1                        # recommended: gets fixes
uses: aligndottech/decision-check@81edf6a218d73963c4b340b2b4e40c21a785bd86   # pinned: fully reproducible
```

Pin a SHA if your supply-chain policy requires that no third-party action can change under you between runs. If you do, put the bump on whatever cadence you already use for dependency review - a pinned action never updates itself.


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
| `status` | Alignment status: `aligned`, `conflicting`, `no-context`, or `unknown` (the check could not run - see below) |
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
| **Neutral, titled "could not run"** | The check did not complete. This is **not** a pass - see below. |

## When the check cannot run

A check that reports success when it never ran is worse than no check, so this action never does that. Three things can stop it completing, and all three produce the **same neutral check run titled "Alignment check could not run", whose summary says "This is not a pass"**:

| What happened | What you see |
|---|---|
| Align reached the API, but its analysis service was degraded | `status` output is `unknown`. The step does **not** fail: your install is fine and the outage is not a conflict. |
| The API was unreachable, returned an error, timed out (120s ceiling), or answered with something unparseable | `status` output is `unknown`, and the summary names the underlying error. The step **fails**, unless `fail-on: none`. |
| The API answered with a status this action does not recognise | Same as above. |

The distinction in the second row is deliberate. A degraded backend is Align's problem and should not block your PR. An unreachable API is indistinguishable from a broken install - a wrong `align-api-url`, a revoked key, a tenant mismatch - and a broken install that quietly goes green is exactly the failure this action exists to avoid. Set `fail-on: none` if you want the action purely advisory; the neutral record is still written either way.

**If you make this check required**, note that a run cannot report at all if the job itself never starts (for example, on a fork PR with no secrets - see Limitations). A required check that never reports leaves the PR pending, not passing.

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

## Limitations

**Fork PRs**: GitHub does not expose repository secrets to workflows triggered by `pull_request` events from forks. The action will fail with a missing API key error on fork PRs. This is a GitHub security feature and applies to all actions that use secrets. For open-source repos accepting external contributions, consider using the Align GitHub App instead, which runs on Align's infrastructure and doesn't depend on the contributor's workflow context.

## Troubleshooting

**"Align API error: 401 Unauthorized"** - Check that `ALIGN_API_KEY` is correct and not expired.

**"Align API error: 403 Forbidden"** - Check that `ALIGN_TENANT_ID` matches the tenant the API key belongs to.

**"This action only works on pull_request events"** - The action must be triggered by `pull_request` events. Check your workflow `on:` trigger.

**Check run shows "No relevant decisions"** - Your Align tenant may not have decisions tracked yet, or the PR doesn't relate to any tracked decisions.
