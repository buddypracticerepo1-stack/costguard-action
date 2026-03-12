# CostGuard GitHub Action

![CostGuard](https://img.shields.io/badge/CostGuard-shift--left%20cost%20governance-brightgreen)
![GitHub Action](https://img.shields.io/badge/GitHub%20Action-composite-blue)
![Version](https://img.shields.io/badge/version-v1-blue)

> Know your infrastructure costs **before** you merge.

CostGuard reviews every PR for cost impact. It works with both **Terraform** (`plan.json`) and **CloudFormation** (`changeset.json`) — auto-detected, no extra config.

On every PR push, CostGuard:
- Posts a **cost breakdown** as a PR comment (per-resource, with regions)
- Returns a **decision** — ALLOW, WARN, or BLOCK — that gates your merge
- Validates against your **budget** and **guardrails**
- Provides **AI-powered** recommendations to optimize costs
- Saves an **HTML report** as a workflow artifact

---

## Quick Start

### 1. Set Repository Secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret | Description |
|--------|-------------|
| `COSTGUARD_API_KEY` | Your CostGuard API key |
| `COSTGUARD_BUDGET_CODE` | Your budget code from CostGuard dashboard (e.g. `CS-FY2026-BU105-M03`). Optional — omit to skip budget validation |

> `GITHUB_TOKEN` is provided automatically — no setup needed.

### 2. Create workflow

> **Important:** The `permissions:` block with `pull-requests: write` is required for CostGuard to post PR comments. Without it, the default `GITHUB_TOKEN` has read-only access and comment posting will fail with a 403 error.

**Terraform** — `.github/workflows/costguard.yml`:

```yaml
name: CostGuard
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      # Add your AWS credentials step here, e.g.:
      # - uses: aws-actions/configure-aws-credentials@v4
      #   with:
      #     role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
      #     aws-region: us-east-1
      - run: |
          terraform init
          terraform plan -out=tfplan
          terraform show -json tfplan > plan.json
      - uses: actions/upload-artifact@v4
        with:
          name: plan
          path: plan.json

  cost-review:
    needs: terraform-plan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: plan
      - uses: buddypracticerepo1-stack/costguard-action@v1
        with:
          plan-path: plan.json
          api-key: ${{ secrets.COSTGUARD_API_KEY }}
          budget-code: ${{ secrets.COSTGUARD_BUDGET_CODE }}
```

**CloudFormation** — If your pipeline already generates `changeset.json`, just add the cost-review job:

```yaml
  cost-review:
    needs: cfn-changeset           # your job that outputs changeset.json
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: changeset
      - uses: buddypracticerepo1-stack/costguard-action@v1
        with:
          plan-path: changeset.json
          api-key: ${{ secrets.COSTGUARD_API_KEY }}
          budget-code: ${{ secrets.COSTGUARD_BUDGET_CODE }}
```

> Generate `changeset.json` with: `aws cloudformation describe-change-set --change-set-name <arn> > changeset.json`

### 3. Protect your main branch

**Settings → Branches → Branch protection rules → main:**

| Setting | Value |
|---------|-------|
| Require status checks to pass | Yes |
| Required checks | `cost-review` |
| Require PR before merging | Yes |

**That's it.** Push a PR and CostGuard will post a cost review comment.

---

## How It Works

```
Developer opens PR
  │
  ├─ Your existing job produces:
  │     Terraform:      terraform show -json tfplan > plan.json
  │     CloudFormation: aws cloudformation describe-change-set > changeset.json
  │
  └─ CostGuard action
       │
       ├─ Reads plan.json or changeset.json (auto-detected)
       ├─ Sends to CostGuard API for pricing + budget + guardrails + AI analysis
       ├─ Posts PR comment with cost breakdown
       ├─ Saves HTML report as workflow artifact
       │
       └─ Exit code controls your workflow:
            0 → ALLOW  → job passes → merge allowed
            1 → BLOCK  → job fails  → merge blocked
            2 → WARN   → job passes → review recommended (GitHub warning annotation)
```

CostGuard posts **one comment per PR** and updates it on every push. No duplicate comments.

---

## What You'll See

On every PR, CostGuard posts a comment with:
- **Decision badge** — ALLOW (green), WARN (yellow), or BLOCK (red)
- **Total monthly cost** — estimated infrastructure spend
- **Per-resource breakdown** — cost per resource with region and pricing details
- **Budget status** — within budget, near limit, or over budget
- **AI recommendations** — optimization suggestions to reduce costs

The comment updates automatically on each push — no duplicates.

---

## Supported IaC Types

| Type | Input File | How to generate |
|------|-----------|-----------------|
| **Terraform** | `plan.json` | `terraform show -json tfplan > plan.json` |
| **CloudFormation Changeset** | `changeset.json` | `aws cloudformation describe-change-set > changeset.json` |

CostGuard auto-detects the type from file content.

---

## Inputs

| Input | Required | Default | Description |
|-------|:--------:|---------|-------------|
| `plan-path` | No | `plan.json` | Path to plan or changeset file |
| `api-key` | **Yes** | | CostGuard API key |
| `budget-code` | No | `""` | Budget code for cost validation |
| `github-token` | No | `${{ github.token }}` | Token for PR comments (auto-provided) |
| `skip-budget` | No | `false` | Skip budget check (pricing-only mode) |
| `skip-narrative` | No | `false` | Skip AI analysis (faster runs) |
| `skip-guardrails` | No | `false` | Skip guardrail evaluation |
| `auto-approve` | No | `false` | Pass when no budget found |
| `format` | No | `markdown` | Comment format: `markdown` or `terminal` |
| `post-comment` | No | `true` | Post cost breakdown as PR comment |
| `api-url` | No | `https://api.skyxops.com` | CostGuard API URL (for custom deployments) |
| `extra-args` | No | `""` | Extra CLI flags (escape hatch) |

## Outputs

| Output | Description |
|--------|-------------|
| `decision` | `ALLOW`, `WARN`, or `BLOCK` |
| `monthly-cost` | Estimated monthly cost |
| `result-file` | Path to cached result JSON |
| `report-file` | Path to HTML report |

Use outputs in subsequent steps:

```yaml
- uses: buddypracticerepo1-stack/costguard-action@v1
  id: costguard
  with:
    plan-path: plan.json
    api-key: ${{ secrets.COSTGUARD_API_KEY }}

- run: echo "Decision: ${{ steps.costguard.outputs.decision }}"
- run: echo "Monthly cost: ${{ steps.costguard.outputs.monthly-cost }}"
```

---

## Examples

See the [examples/](./examples/) folder for complete workflows:

| Example | Description |
|---------|-------------|
| [terraform-basic](./examples/terraform-basic.yml) | Simplest setup — plan.json from Terraform |
| [terraform-subdirectory](./examples/terraform-subdirectory.yml) | Terraform in a nested directory |
| [cloudformation-changeset](./examples/cloudformation-changeset.yml) | Cost review from a change set |
| [skip-options](./examples/skip-options.yml) | Skip budget, AI narrative, guardrails |
| [save-artifacts](./examples/save-artifacts.yml) | Upload HTML report as workflow artifact |

---

## Versioning

| Tag | Recommended | Description |
|-----|:-----------:|-------------|
| `v1` | Yes | Floating tag — auto-receives bug fixes |

```yaml
# Recommended
- uses: buddypracticerepo1-stack/costguard-action@v1

# Pinned to exact commit (most stable)
- uses: buddypracticerepo1-stack/costguard-action@abc1234
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| PR comment not posted | Token missing permissions | Use default `${{ github.token }}` or a PAT with `pull_requests: write` |
| BLOCK doesn't prevent merge | Branch not protected | Settings → Branches → require `cost-review` status check |
| Plan file not found | Wrong path | Check `plan-path` matches your artifact download path |
| API timeout on first run | Cold start | Re-run workflow — subsequent runs complete in ~5 seconds |

---

*[SKYXOPS](https://skyxops.com) — shift-left cost governance for cloud infrastructure*
