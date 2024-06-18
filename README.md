# GitHub Workflows for Terraform 

## Usage

**The default configuration validates the terraform code:**

Can be used for modules and workspaces for continuous integration.

- lint via `tflint`
- check formatting via `terraform fmt`
- validate configuration via `terraform validate`
- check docs via `terraform-docs`

By default, changes due to incorrect formatting or outdated `README.md` will be 
pushed back to PRs and the validate job succeeds if the push was successful.

On all other events, validate fails on any error.
```yaml
jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
```

**To plan changes:**

```yaml
jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      environment: 'service-test'
      mode: 'plan'
```

If `environment` is set to `all`, all environments will be processed. (`ls *-*.tfvars`)

```yaml
jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      environment: 'all'
      mode: 'plan'
```

**To apply changes:**

Set `mode` to `apply` to automatically apply the changes if the plan was successfully created. For security reasons, 
the `prod` stage is always excluded in apply jobs. 

```yaml
jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      environment: 'all'  
      mode: 'apply' # 'service-prod', 'main-prod', etc. will only run plan even if apply is set
```

By default, only pushes to the main branch will trigger the apply job. If you want to apply changes for PRs, set
`allow-apply-on-pr` to `yes`.

```yaml
jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      environment: 'service-test'
      mode: 'apply'
      allow-apply-on-pr: 'yes'
```

**Terraform output:**

To store the terraform output as an artifact, set `mode` to `output`.

```yaml
jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      mode: 'output'
```

For detailed example configurations for different use-cases, see the [examples](examples) directory.

**The different jobs behave differently depending on the event:**

- `validate`:
  - skipped if `run-id` is set (`workflow_dispatch` event), as the previous run already performed the validation 
- `plan`
  - skipped if `run-id` is set (`workflow_dispatch` event), as the previous run already created the plan
  - plan result will be reported on `pull_request`
- `apply`
  - if `run-id` is set (`workflow_dispatch` event), the plan from the given run will be fetched and applied
  - if `run-id` is empty, the plan from the current run will be applied
  - skipped if `mode` is not set to `apply`
  - skipped on `pull_request` if `allow-apply-on-pr` is set to `no`
  - skipped on `push` events if not on the default branch
- `output`
  - only runs if `mode` is set to `output`
  - runs if the `apply`job was successful

### Artifacts

- `<environment>-plan`
  - `<environment>-plan.tfvars`: The terraform plan binary
  - `<environment>-plan-result.txt`: Contains the [detailed result](https://developer.hashicorp.com/terraform/cli/commands/plan#detailed-exitcode) of the terraform plan (success, error or changes)
- `<environment>-show`
  - `<environment>-show.txt`: The output of `terraform show`
  - `<environment>-show-no-color.txt`: The output of `terraform show -no-color`
  - `<environment>-show.json`: The output of `terraform show -json`
- `<environment>-apply`
    - `<environment>-apply-result.txt`: The result of the apply job (success or failure)
- `<environment>-output`
  - `<environment>-output.txt`: The output of `terraform output`
  - `<environment>-output.json`: The output of `terraform output -json`

### Inputs

The action supports the following inputs:

- `environment` - (optional) The name of the environment to process. Instead of an environment name, you 
can also specify `all` to process all environments. Defaults to `vars.DEFAULT_ENVIRONMENT`.
- `exclude` - (optional) A list of regex patterns to exclude matching environments. Ignored if
`environment` is not set to `all`. Defaults to empty string.
- `exclude-from-apply` - (optional) A list of regex patterns to exclude matching environments from apply. Ignored
if event is `workflow_dispatch` Defaults to `.*-prod`.
- `mode` - (optional) Supported modes are `validate`, `plan`, `apply` and `output`. Defaults to `validate`.
- `apply-on-pr` - (optional) A list of regex patterns to include matching environments. Defaults to empty string.
- `run-id` - (optional) If set, the apply job will load the plan from the given run. Defaults to an empty string.
- `comment-mode` - (optional) A PR comment can be updated (upsert) or a new comment can be created on every 
run (create). Defaults to `upsert`.
- `report-plan-result` - (optional) Weather to report plan result to slack. Defaults to `no`.
- `report-apply-result` - (optional) Weather to report apply result to slack. Defaults to `no`.
- `report-output` - (optional) Weather to report terraform output to slack. Defaults to `no`.
- `skip-validate` - (optional) Set to `yes` to skip validation. Defaults to `no`.
- `disable-auto-commit` - (optional) Set to `yes` to disable code change pushes. Defaults to `no`.

Checkout [`terraform.yaml`](.github/workflows/terraform.yaml) for a full list of supported inputs.

### Variables / Secrets

**Global Variables:**
- `TF_VERSION`: the terraform version to use (required, default set in organisation variables)
- `TFLINT_VERSION`: the tflint version to use (required, default set in organisation variables)
- `AWS_DEFAULT_REGION`: The default region to use (required, default set in organisation variables)

**Secrets:**
- `SLACK_WEBHOOK`: the slack webhook to send notifications to slack (optional: if not set, no slack notifications will be sent)

### Permissions

The following permissions are required by the workflow:

```yaml
permissions:
  id-token: write       # For aws-actions/configure-aws-credentials
  contents: write       # To commit format/docs changes
  pull-requests: write  # To write PR comments
```

### Dependencies

- actions/checkout
- hashicorp/setup-terraform
- actions/download-artifact
- actions/upload-artifact
- aws-actions/configure-aws-credentials
- webfactory/ssh-agent
- slackapi/slack-github-action

## Scenarios

### Plan for the default environment on PR and create a plan for all environments on push to the main branch

This is the behavior that you already know. The plan results will be reported as PR comments.
```yaml
on:
  # perform CI (including tf plan) on PRs
  pull_request:

jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      environment: ${{ vars.DEFAULT_ENVIRONMENT }}
      mode: 'plan'
```

When the PR has been merged, this workflow will trigger a plan job for each environment. The plan results are stored as
artifacts ad can be applied manually by using the run id.
```yaml
on:
  # perform CI (including tf plan) on PRs
  push:
    branches:
      - main

jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      environment: all
      mode: 'plan'
```

### Continuous deployment to the default environment

This workflow will create a plan and immediately apply it for the default environment.
```yaml
on:
  # perform CI (including tf plan) on PRs
  push:
    branches:
      - main

jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      environment: ${{ vars.DEFAULT_ENVIRONMENT }}
      mode: 'apply'
```

### Deliver the terraform output for all environments

```yaml
on:
  workflow_dispatch:

jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      environment: all
      mode: 'output'
      skip-validate: 'yes'
```

### Schedule plans for all environments to detect drift

This will schedule plans every day and report the result to Slack:
```yaml
on:
  schedule:
    - cron: "15 3 * * MON-FRI"

jobs:
  terraform:
    uses: Schumann-IT/terraform-gh-workflows/.github/workflows/terraform.yaml@main
    secrets: inherit
    with:
      environment: all
      mode: plan
      skip-validate: yes
      report-plan-result: yes
```
