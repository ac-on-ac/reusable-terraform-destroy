# reusable-terraform-destroy

A reusable GitHub Actions workflow that runs `terraform destroy` with an optional manual approval gate before any infrastructure is removed.

## How it works

The workflow runs two jobs:

1. **`terraform-plan-destroy`** — Authenticates to providers, initialises Terraform, produces a destroy plan (`terraform plan -destroy`), writes the plan output to the job summary, and uploads the plan file as a workflow artifact.
2. **`terraform-destroy`** — Downloads the saved plan and applies it (`terraform apply tfplan`). When `environment` is provided, this job is blocked by GitHub's environment protection rules until a required reviewer approves.

Because the destroy job applies the saved plan file rather than recalculating it, the scope of the destroy is locked at approval time. Terraform will error rather than proceed if state has drifted between planning and applying.

## Approval gate behaviour

| `environment` input | Environment has Required reviewers | Behaviour |
|---|---|---|
| `''` (default / omitted) | N/A | Destroys immediately after plan — no gate |
| Name provided | No protection rules configured | **Destroys immediately** — the environment is targeted but provides no gate |
| Name provided | Required reviewers configured | Pauses before destroy — reviewers are notified and must approve |

> **Important:** Simply providing an `environment` name is not enough to create a gate. The environment must exist in the repository and have **Required reviewers** enabled in **Settings → Environments**. If the name is provided but the environment does not exist, the workflow fails immediately with a clear error before any infrastructure is touched.

## Setting up the approval environment

1. Go to the repository's **Settings → Environments → New environment**
2. Name it (e.g., `destroy-approval` or `production-destroy`)
3. Under **Environment protection rules**, enable **Required reviewers** and add the users or teams who must approve
4. Optionally set a **Wait timer** to enforce a minimum delay after the plan is posted
5. Optionally configure a **Deployment branch policy** to restrict which branches can trigger the destroy

Pass this exact environment name as the `environment` input when calling the workflow.

## Usage

### With approval gate (recommended for production)

```yaml
on:
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  destroy:
    uses: ac-on-ac/reusable-terraform-destroy/.github/workflows/destroy.yml@main
    with:
      environment: destroy-approval
      working_directory: infra/
    secrets: inherit
```

### Without approval gate

```yaml
on:
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  destroy:
    uses: ac-on-ac/reusable-terraform-destroy/.github/workflows/destroy.yml@main
    with:
      working_directory: infra/
    secrets: inherit
```

### With Azure OIDC and S3 backend

```yaml
jobs:
  destroy:
    uses: ac-on-ac/reusable-terraform-destroy/.github/workflows/destroy.yml@main
    with:
      environment: destroy-approval
      working_directory: infra/environments/prod
      arm_client_id: ${{ vars.ARM_CLIENT_ID }}
      arm_tenant_id: ${{ vars.ARM_TENANT_ID }}
      arm_subscription_id: ${{ vars.ARM_SUBSCRIPTION_ID }}
      s3_backend_bucket: my-tfstate-bucket
      s3_backend_key: prod/terraform.tfstate
    secrets: inherit
```

## Inputs

### General

| Input | Description | Required | Default |
|---|---|---|---|
| `working_directory` | Directory containing the Terraform configuration | No | `.` |
| `terraform_version` | Terraform version to install (e.g. `1.11.0`) | No | `latest` |
| `terraform_vars` | Newline-separated `KEY=VALUE` pairs. Only `TF_VAR_*` keys are permitted. | No | `''` |
| `aws_region` | AWS region for login and S3 backend default | No | `us-east-1` |
| `environment` | GitHub environment name with required reviewers. Leave empty to skip the approval gate. Must already exist in the repository if provided. | No | `''` |

### Azure OIDC

Supply `arm_client_id`, `arm_tenant_id`, and `arm_subscription_id` to authenticate via OIDC instead of a service principal JSON credential. The calling workflow must also declare `permissions: id-token: write`.

> **Two federated credentials are required when using an approval environment.**
> The plan job (`terraform-plan-destroy`) runs without an environment, so its OIDC subject is `repo:<org>/<repo>:ref:refs/heads/<branch>`. The destroy job (`terraform-destroy`) runs with `environment: <name>`, so its OIDC subject is `repo:<org>/<repo>:environment:<name>`. Azure must be able to match both subjects, which means you need one federated credential scoped to the branch **and** one scoped to the environment. If no `environment` input is provided, only the branch-scoped federated credential is needed.

| Input | Description | Required | Default |
|---|---|---|---|
| `arm_client_id` | Azure client (application) ID | No | `''` |
| `arm_tenant_id` | Azure tenant ID | No | `''` |
| `arm_subscription_id` | Azure subscription ID | No | `''` |

### Azure blob storage backend

| Input | Description | Required | Default |
|---|---|---|---|
| `azure_backend_resource_group_name` | Resource group of the storage account | No | `''` |
| `azure_backend_storage_account_name` | Storage account name | No | `''` |
| `azure_backend_container_name` | Blob container name | No | `''` |
| `azure_backend_key` | Blob path for the state file | No | `''` |

### S3 backend

| Input | Description | Required | Default |
|---|---|---|---|
| `s3_backend_bucket` | S3 bucket name | No | `''` |
| `s3_backend_key` | S3 object key (path) for the state file | No | `''` |
| `s3_backend_region` | Region of the S3 bucket. Defaults to `aws_region`. | No | `''` |
| `s3_backend_dynamodb_table` | DynamoDB table for state locking | No | `''` |

## Secrets

### Azure

| Secret | Description |
|---|---|
| `azure_credentials` | JSON service principal credentials. Use this **or** the three OIDC inputs above, not both. |
| `databricks_host` | Databricks workspace URL |
| `databricks_azure_workspace_resource_id` | Azure resource ID of the Databricks workspace |

### Terraform Cloud / Enterprise

| Secret | Description |
|---|---|
| `terraform_token` | API token for Terraform Cloud or Enterprise |

### AWS

| Secret | Description |
|---|---|
| `aws_access_key_id` | AWS access key ID |
| `aws_secret_access_key` | AWS secret access key |
| `aws_role_to_assume` | IAM role ARN to assume (for role-based auth) |

### GitHub

| Secret | Description |
|---|---|
| `gh_token` | Personal access token or fine-grained token for the GitHub Terraform provider. Must not be named `github_token` — that name is reserved by GitHub Actions. Mapped to `GITHUB_TOKEN_PROVIDER` internally. |

### HashiCorp Vault

| Secret | Description |
|---|---|
| `vault_addr` | Vault server URL (e.g. `https://vault.example.com`). Maps to `VAULT_ADDR`. |
| `vault_token` | Vault token (token auth method) |
| `vault_role_id` | Vault role ID (AppRole auth method) |
| `vault_secret_id` | Vault secret ID (AppRole auth method) |

### Snowflake

| Secret | Description |
|---|---|
| `snowflake_account` | Snowflake account identifier. Maps to `SNOWFLAKE_ACCOUNT`. |
| `snowflake_user` | Snowflake username. Maps to `SNOWFLAKE_USER`. |
| `snowflake_password` | Snowflake password. Maps to `SNOWFLAKE_PASSWORD`. |
| `snowflake_role` | Snowflake role. Maps to `SNOWFLAKE_ROLE`. |

### Grafana

| Secret | Description |
|---|---|
| `grafana_url` | Grafana instance URL. Maps to `GRAFANA_URL`. |
| `grafana_auth` | Grafana API key or basic auth string. Maps to `GRAFANA_AUTH`. |

## Provider authentication

### Azure

Provide either:
- `azure_credentials` (service principal JSON) for static credential auth, or
- `arm_client_id` + `arm_tenant_id` + `arm_subscription_id` (inputs) for OIDC — also requires `permissions: id-token: write` in the calling workflow

When using OIDC with an approval environment, configure **two** federated credentials on the Azure app registration or managed identity:

| Federated credential | Entity type | Value |
|---|---|---|
| Plan job | Branch | `main` (or whichever branch the calling workflow runs on) |
| Destroy job | Environment | The exact name passed as the `environment` input (e.g. `destroy-approval`) |

If no `environment` input is provided, only the branch-scoped federated credential is needed.

### AWS

Provide `aws_access_key_id` + `aws_secret_access_key` for static credentials, or `aws_role_to_assume` alone for OIDC role assumption. Any combination that `aws-actions/configure-aws-credentials` accepts is valid.

### HashiCorp Vault

Provide either `vault_token` (token auth) or `vault_role_id` + `vault_secret_id` (AppRole auth). `vault_addr` is required in both cases.

### Snowflake and Grafana

No login action is used. Credentials are exported as environment variables (`SNOWFLAKE_*`, `GRAFANA_URL`, `GRAFANA_AUTH`) which the respective Terraform providers read automatically.

## Notes

- **The plan artifact is retained for 1 day.** The destroy must be approved and complete within that window or the artifact will expire and the apply will fail with a missing plan error.
- **Plan drift protection.** Terraform validates the saved plan against current state before applying. If state has changed between the plan and the approval, Terraform will error rather than apply a stale plan.
- **Backend partial config validation.** If any Azure backend input is provided, all four must be present. If `s3_backend_bucket` is set, `s3_backend_key` must also be set. The workflow fails early with a clear error otherwise.
- **`TF_VAR_*` enforcement.** The `terraform_vars` input only accepts keys prefixed with `TF_VAR_`. Any other key causes an immediate failure before Terraform runs.

