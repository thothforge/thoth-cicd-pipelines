# fullstack-cicd-pipelines-gha

Transversal, organization-shared **GitHub Actions reusable workflows and composite
actions** for the `thothforge` DevSecOps platform — the GitHub Actions counterpart of
the Azure DevOps `fullstack-cicd-pipelines-azdo` templates.

Projects **reuse** these centrally instead of copying pipeline logic. Public so every
`thothforge` repo (and the community) can consume them.

## What's here

| Path | Purpose |
|------|---------|
| `.github/workflows/iac-pipeline.yml` | Reusable IaC pipeline (Terragrunt/OpenTofu, ThothCTL-governed): plan → tfplan summary → cost → blast-radius → checkov → OPA → optional apply. |
| `.github/actions/assume-spoke-role/` | Composite action: chain-assume a hub-spoke IAM role into a named AWS CLI profile (no OIDC; runs under the ambient CodeBuild service role). |

## Execution model

- Runs on **CodeBuild-hosted GitHub Actions runners** (Option C: one ORG-scoped BUILD
  runner + one DEPLOY runner in the DevSecOps hub account).
- Runner tools (`tofu`, `terragrunt`, `thothctl`, `checkov`, `aws`) are **preinstalled**
  in the shared DevSecOps IaC CodeBuild image — the workflow installs nothing.
- **Hub-spoke credentials, no OIDC:** the job runs as the ambient CodeBuild service role
  and chain-assumes the per-account spoke roles:
  - `pipeline/read/hub-<hub>-read-<spoke>` for plan/scan,
  - `pipeline/deploy/hub-<hub>-deploy-<spoke>` for apply.

## Consume from a project

```yaml
# .github/workflows/deploy.yml in a consuming repo
name: deploy-iac
on:
  push: { branches: [main], paths: ["stacks/**", "modules/**", "environments/**"] }
  pull_request: { paths: ["stacks/**", "modules/**", "environments/**"] }
  workflow_dispatch:
    inputs:
      environment: { type: choice, options: [dev, qa, prd], default: dev }

permissions:
  contents: read

jobs:
  iac:
    uses: thothforge/fullstack-cicd-pipelines-gha/.github/workflows/iac-pipeline.yml@v1
    with:
      environment: ${{ inputs.environment || 'dev' }}
      hub_account_id: "225311078840"
      deploy_account_id: "225311078840"
      region: us-east-2
      pac_enforcement: soft
      pipelines_ref: v1           # match the @ref above
      do_apply: ${{ github.event_name != 'pull_request' }}
```

## Inputs (iac-pipeline.yml)

| Input | Req | Default | Notes |
|-------|-----|---------|-------|
| `environment` | yes | — | `TF_WORKSPACE` (dev/qa/prd) |
| `deploy_account_id` | yes | — | target spoke account |
| `hub_account_id` | yes | — | DevSecOps hub account |
| `runner_project` | no | `github-actions-deploy-runner` | CodeBuild runs-on label |
| `region` | no | `us-east-2` | |
| `spoke_read_role` / `spoke_deploy_role` | no | computed | full-ARN overrides |
| `pipelines_ref` | no | `main` | ref for the composite-action checkout; match your `@ref` |
| `pac_enforcement` | no | `soft` | OPA gate (`soft`/`hard`) |
| `do_apply` | no | `false` | apply after plan+scan (PRs should stay false) |

## Versioning

Pin consumers to a tag (`@v1`), not `@main`. Tag releases with semantic versions; keep a
moving major tag (`v1`) pointing at the latest compatible release.

## License

Apache-2.0.
