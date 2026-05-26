# Grauss infrastructure repository

This repository manages cloud infrastructure using the [Grauss](https://github.com/your-org/grauss-engine) engine — Terraform/OpenTofu packaged as Docker images, one per cloud provider.

Infrastructure is described as **JSON files** under `data/`. No `.tf` editing required for day-to-day operations.

## Supported clouds

| Cloud | Provider |
|-------|----------|
| Azure | AzureRM + AzureAD |
| AWS | hashicorp/aws |
| GCP | hashicorp/google |
| OCI | oracle/oci |
| OVH | ovh/ovh |

## Getting started

### 1. Use this template

Click **Use this template → Create a new repository** on GitHub.

### 2. Set the image source variable

Go to **Settings → Secrets and variables → Actions → Variables** and create:

| Variable | Value |
|----------|-------|
| `GRAUSS_IMAGE_ORG` | GitHub org/user that publishes the Grauss engine images (e.g. `my-org`) |

Images are pulled as `ghcr.io/<GRAUSS_IMAGE_ORG>/grauss-<cloud>-terraform:<version>`.

### 3. Configure cloud credential secrets

Go to **Settings → Secrets and variables → Actions → Secrets** and add the secrets for your cloud.

#### Azure

| Secret | Description |
|--------|-------------|
| `ARM_CLIENT_ID` | Service principal application (client) ID |
| `ARM_CLIENT_SECRET` | Service principal client secret |
| `ARM_TENANT_ID` | Azure AD tenant ID |
| `ARM_SUBSCRIPTION_ID` | Target subscription ID |
| `ARM_USE_AZUREAD` | `true` to use Azure AD auth for state storage (recommended) |

#### AWS

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | IAM access key ID |
| `AWS_SECRET_ACCESS_KEY` | IAM secret access key |
| `AWS_SESSION_TOKEN` | Session token (optional — temporary/assumed-role credentials only) |
| `AWS_DEFAULT_REGION` | Default region (e.g. `eu-west-1`) |

#### GCP

| Secret | Description |
|--------|-------------|
| `GOOGLE_CREDENTIALS` | Service account key JSON (full file content) |
| `GOOGLE_PROJECT` | GCP project ID |
| `GOOGLE_REGION` | Default region (e.g. `europe-west1`) |
| `GOOGLE_ZONE` | Default zone (e.g. `europe-west1-b`) |

#### OCI

| Secret | Description |
|--------|-------------|
| `OCI_TENANCY_OCID` | Tenancy OCID |
| `OCI_USER_OCID` | User OCID |
| `OCI_FINGERPRINT` | API key fingerprint |
| `OCI_REGION` | Region identifier (e.g. `eu-frankfurt-1`) |
| `OCI_PRIVATE_KEY` | PEM private key (full `-----BEGIN...-----END...` block) |

#### OVH

| Secret | Description |
|--------|-------------|
| `OVH_ENDPOINT` | API endpoint (`ovh-eu`, `ovh-ca`, or `ovh-us`) |
| `OVH_APPLICATION_KEY` | Application key from [OVH token console](https://api.ovh.com/createToken) |
| `OVH_APPLICATION_SECRET` | Application secret |
| `OVH_CONSUMER_KEY` | Consumer key |
| `OVH_SERVICE_NAME` | Public Cloud project ID (UUID from OVH Control Panel → Public Cloud → Settings) |
| `OVH_S3_REGION` | Object Storage region slug (e.g. `gra`) — add after bootstrap |
| `OVH_S3_ENDPOINT` | Object Storage endpoint URL — add after bootstrap |
| `OVH_S3_ACCESS_KEY` | S3-compatible access key — add after bootstrap |
| `OVH_S3_SECRET_KEY` | S3-compatible secret key — add after bootstrap |

### 4. Run the setup workflow

Go to **Actions → Grauss setup → Run workflow** and fill in:

| Input | Description |
|-------|-------------|
| `cloud` | Target cloud: `azure`, `aws`, `gcp`, `oci`, or `ovh` |
| `project` | Project name (lowercase alphanumeric + hyphens, e.g. `networking`) |
| `image_version` | Engine image version tag (e.g. `v0.1.0` or `latest`) |

The workflow writes `grauss.json` with your choices and commits it.

### 5. Add your infrastructure

With `grauss.json` configured, start adding JSON resource files under `data/`. The recommended layout is:

```
data/
├── _bootstrap/     ← remote state infrastructure (apply once, local state)
│   └── <resource_type>/
│       └── <name>.json
└── <project>/      ← your infrastructure project
    ├── backend.tf.json
    └── <resource_type>/
        └── <name>.json
```

One JSON file = one resource instance. Add a file to create the resource; delete it to destroy it.

#### Bootstrap remote state

The `_bootstrap` project provisions the storage backend that holds Terraform state. It runs with **local state** (no `backend.tf.json`) and is applied once. Once done, add `backend.tf.json` to your project pointing to the resources it created.

**Azure** — `data/<project>/backend.tf.json`:
```json
{
  "terraform": [{"backend": {"azurerm": {
    "resource_group_name": "rg-<project>-tfstate",
    "storage_account_name": "<storage-account-name>",
    "container_name": "tfstate",
    "key": "<project>/terraform.tfstate",
    "use_azuread_auth": true
  }}}]
}
```

**AWS** — `data/<project>/backend.tf.json`:
```json
{
  "terraform": [{"backend": {"s3": {
    "bucket": "<project>-tfstate-bootstrap",
    "key": "<project>/terraform.tfstate",
    "region": "<region>",
    "encrypt": true,
    "dynamodb_table": "<project>-tfstate-locks"
  }}}]
}
```

**GCP** — `data/<project>/backend.tf.json`:
```json
{
  "terraform": [{"backend": {"gcs": {
    "bucket": "<project>-tfstate-bootstrap",
    "prefix": "<project>/terraform"
  }}}]
}
```

**OCI** — `data/<project>/backend.tf.json`:
```json
{
  "terraform": [{"backend": {"oci": {
    "bucket": "<project>-tfstate-bootstrap",
    "namespace": "<object-storage-namespace>",
    "region": "<region>",
    "prefix": "<project>"
  }}}]
}
```

**OVH** — `data/<project>/backend.tf.json`:
```json
{
  "terraform": [{"backend": {"s3": {
    "bucket": "<project>-tfstate",
    "key": "<project>/terraform.tfstate",
    "region": "<s3-region>",
    "endpoint": "https://s3.<region>.io.cloud.ovh.net",
    "skip_credentials_validation": true,
    "skip_region_validation": true,
    "skip_requesting_account_id": true,
    "skip_s3_checksum": true
  }}}]
}
```

> **OVH bootstrap is two-phase:** the first apply creates the `cloud_project_user`; take the user ID from the output, add it to the `cloud_project_user_s3_credential` JSON file, then apply again to create the S3 credential and bucket.

## Repository structure

```
grauss.json          # cloud, project, image version, template version
data/                # JSON resource files (add your own)
```

## Terraform vs OpenTofu

To use OpenTofu instead of Terraform, change the image name in the workflow files from `grauss-<cloud>-terraform` to `grauss-<cloud>-opentofu`. The JSON files under `data/` are identical for both engines.
