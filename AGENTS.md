# Agent Instructions

## GCP Authentication

Use Application Default Credentials (ADC) for local Terraform operations. The
Google Cloud project name is `cyber-analyzer`, but its project ID is
`x-catwalk-509916-q8`. Always use the project ID in commands and Terraform
variables.

Run these commands in order:

```bash
export TF_VAR_project_id=x-catwalk-509916-q8
gcloud auth application-default login \
  --scopes="https://www.googleapis.com/auth/cloud-platform"
gcloud config set project "$TF_VAR_project_id"
gcloud auth application-default set-quota-project "$TF_VAR_project_id"

# Configure Docker credential helpers for Google Container Registry.
gcloud auth configure-docker
```

Set `TF_VAR_project_id` in every new shell before running `gcloud` or Terraform.
Do not set it to `cyber-analyzer`: that is the display name, and using it as the
quota project fails with a `serviceusage.services.use` permission error.

On the Google consent page, explicitly approve the requested Google Cloud
Platform access. Do not store credentials or OAuth callback URLs in the
repository.

The unqualified Docker command configures the `gcr.io` registry hosts. The
Terraform configuration pushes to Artifact Registry at
`us-central1-docker.pkg.dev`, which requires that hostname to be configured
separately before pushing an image.

Verify authentication and project access before running Terraform:

```bash
gcloud config list
gcloud auth application-default print-access-token >/dev/null
gcloud projects describe "$(gcloud config get-value project 2>/dev/null)" \
  --format="table(projectId,name,lifecycleState)"
```

The expected project is:

```text
PROJECT_ID           NAME            LIFECYCLE_STATE
x-catwalk-509916-q8  cyber-analyzer  ACTIVE
```

Pass the same ID to Terraform, for example:

```bash
export TF_VAR_project_id=x-catwalk-509916-q8
terraform plan
```

Load the local secrets and deploy from `terraform/gcp` with the project ID
passed explicitly. This prevents a stale shell variable from targeting the
display name `cyber-analyzer`:

```bash
export TF_VAR_project_id=x-catwalk-509916-q8
set -a
source ../../.env
set +a
terraform apply \
  -var="project_id=$TF_VAR_project_id" \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -var="semgrep_app_token=$SEMGREP_APP_TOKEN"
```

Only when explicitly requested, destroy the GCP deployment from
`terraform/gcp` with the same project ID and local secrets:

```bash
export TF_VAR_project_id=x-catwalk-509916-q8
set -a
source ../../.env
set +a
terraform destroy \
  -var="project_id=$TF_VAR_project_id" \
  -var="openai_api_key=$OPENAI_API_KEY" \
  -var="semgrep_app_token=$SEMGREP_APP_TOKEN"
```

The Docker provider build must use the default buildx builder and the full
Dockerfile path. Keep these settings in `terraform/gcp/main.tf`; without them,
recent Docker Engine versions can fail while reading the build context with an
`unpigz: corrupted` error:

```hcl
build {
  context    = "${path.module}/../.."
  dockerfile = "${path.module}/../../Dockerfile"
  builder    = "default"
}
```

If `gcloud auth application-default login` reports that the `cloud-platform`
scope was not consented, rerun the scoped login command above and approve the
access. Use `--no-browser` only in a genuinely headless environment; its
`--remote-bootstrap` command must run in a separate terminal, and the resulting
callback URL is entered into the original prompt.