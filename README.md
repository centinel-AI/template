# Grauss infrastructure repository

> **Template version:** see [`VERSION`](VERSION).

This repository manages cloud infrastructure using the [Grauss](https://github.com/your-org/grauss-engine) engine — Terraform/OpenTofu packaged as Docker images, one per cloud provider and engine combination.

Infrastructure is described as **JSON files** under `data/`. No `.tf` editing required for day-to-day operations. Opening a pull request triggers a `terraform plan`; merging to `main` triggers `terraform apply`.

## Supported clouds

| Cloud | Image name |
|-------|------------|
| Azure (AzureRM + AzureAD) | `grauss-azure-terraform` / `grauss-azure-opentofu` |
| AWS | `grauss-aws-terraform` / `grauss-aws-opentofu` |
| GCP | `grauss-gcp-terraform` / `grauss-gcp-opentofu` |
| OCI (Oracle Cloud Infrastructure) | `grauss-oci-terraform` / `grauss-oci-opentofu` |
| OVH Public Cloud | `grauss-ovh-terraform` / `grauss-ovh-opentofu` |

## Template version

The `VERSION` file records which version of the Grauss template this repository was created from. When the template is updated, the version is bumped so you can compare your instance against the latest template and pick up any workflow or structural improvements.

## Getting started

### 1. Use this template

Click **Use this template → Create a new repository** on GitHub. Give it a meaningful name — this will be your infrastructure repository.

### 2. Set the image source variable

Grauss engine images are published to GitHub Container Registry by the organisation or user that maintains the engine. Go to:

**Settings → Secrets and variables → Actions → Variables → New repository variable**

| Variable | Value |
|----------|-------|
| `GRAUSS_IMAGE_ORG` | GitHub org/user that publishes the engine images (e.g. `my-org`) |

Images are pulled as `ghcr.io/<GRAUSS_IMAGE_ORG>/grauss-<cloud>-terraform:<version>`.

### 3. Configure cloud credential secrets

Go to **Settings → Secrets and variables → Actions → Secrets → New repository secret** and add the secrets for your cloud.

#### Azure

| Secret | Description |
|--------|-------------|
| `ARM_CLIENT_ID` | Service principal application (client) ID |
| `ARM_CLIENT_SECRET` | Service principal client secret |
| `ARM_TENANT_ID` | Azure AD tenant ID |
| `ARM_SUBSCRIPTION_ID` | Target subscription ID |
| `ARM_USE_AZUREAD` | Set to `true` to authenticate against the storage account with Azure AD (recommended) |

#### AWS

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | IAM access key ID |
| `AWS_SECRET_ACCESS_KEY` | IAM secret access key |
| `AWS_SESSION_TOKEN` | Session token (optional — only for temporary/assumed-role credentials) |
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
| `OCI_TENANCY_OCID` | Tenancy OCID (`ocid1.tenancy.oc1..`) |
| `OCI_USER_OCID` | User OCID (`ocid1.user.oc1..`) |
| `OCI_FINGERPRINT` | API key fingerprint (`xx:xx:xx:...`) |
| `OCI_REGION` | Region identifier (e.g. `eu-frankfurt-1`) |
| `OCI_PRIVATE_KEY` | PEM private key content (full `-----BEGIN...-----END...` block) |

#### OVH

| Secret | Description |
|--------|-------------|
| `OVH_ENDPOINT` | API endpoint (e.g. `ovh-eu`, `ovh-ca`, `ovh-us`) |
| `OVH_APPLICATION_KEY` | Application key from [OVH token console](https://api.ovh.com/createToken) |
| `OVH_APPLICATION_SECRET` | Application secret |
| `OVH_CONSUMER_KEY` | Consumer key |
| `OVH_SERVICE_NAME` | Public Cloud project ID (UUID shown in OVH Control Panel → Public Cloud → Settings) |
| `OVH_S3_REGION` | Object Storage region slug for state backend (e.g. `gra`) — add after bootstrap |
| `OVH_S3_ENDPOINT` | Object Storage endpoint URL (e.g. `https://s3.gra.io.cloud.ovh.net`) — add after bootstrap |
| `OVH_S3_ACCESS_KEY` | S3-compatible access key — add after bootstrap |
| `OVH_S3_SECRET_KEY` | S3-compatible secret key — add after bootstrap |

> **OVH note:** The four `OVH_S3_*` secrets are only required when using remote Terraform state (after the bootstrap project has been applied). They are safe to leave empty until then.

### 4. Run the setup workflow

Go to **Actions → Grauss setup → Run workflow** and fill in:

| Input | Description |
|-------|-------------|
| `cloud` | Target cloud: `azure`, `aws`, `gcp`, `oci`, or `ovh` |
| `project` | Project name (lowercase alphanumeric + hyphens, e.g. `networking`). Becomes `data/<cloud>/<project>/` and `TF_VAR_project`. |
| `image_version` | Engine image version tag (e.g. `v1.0.0` or `latest`) |

The workflow creates two directory trees and commits them:

- `data/<cloud>/_bootstrap/` — starter files for provisioning the remote state backend (see next step).
- `data/<cloud>/<project>/` — starter files for your first infrastructure project.

### 5. Set up remote state (recommended for teams)

By default, Terraform uses **local state** stored inside the container, which is discarded after each run. For shared, durable state, provision a remote backend first.

#### How bootstrap works

The `_bootstrap` project creates the cloud storage infrastructure needed to hold Terraform state. It deliberately has **no `backend.tf.json`** — it runs with local state just once, then its output becomes the backend for all subsequent projects.

**Apply the bootstrap project:**

1. Set `TF_VAR_project=_bootstrap` in `grauss.json` (temporarily) or run the plan/apply workflow with a manual dispatch after editing `grauss.json`:
   ```json
   { "cloud": "azure", "project": "_bootstrap", "image_version": "latest" }
   ```
2. Fill in the `REPLACE_WITH_*` placeholders in `data/<cloud>/_bootstrap/*.json` with your actual values.
3. Push the changes and trigger the apply workflow (or run apply manually).
4. Restore `grauss.json` to your real project name once bootstrap completes.

> **OVH bootstrap is two-phase:** first apply creates the `cloud_project_user`; take the user ID from the Terraform output, fill it into `data/ovh/_bootstrap/cloud_project_user_s3_credential/<file>.json`, then apply again to create the S3 credential and bucket.

#### Add `backend.tf.json` to your project

Once bootstrap has run, add a `backend.tf.json` file to `data/<cloud>/<project>/` pointing to the resources it created:

**Azure** — `data/azure/<project>/backend.tf.json`:
```json
{
  "terraform": [{
    "backend": {
      "azurerm": {
        "resource_group_name": "rg-<project>-tfstate",
        "storage_account_name": "<your-storage-account-name>",
        "container_name": "tfstate",
        "key": "<project>/terraform.tfstate",
        "use_azuread_auth": true
      }
    }
  }]
}
```

**AWS** — `data/aws/<project>/backend.tf.json`:
```json
{
  "terraform": [{
    "backend": {
      "s3": {
        "bucket": "<project>-tfstate-bootstrap",
        "key": "<project>/terraform.tfstate",
        "region": "<your-region>",
        "encrypt": true,
        "dynamodb_table": "<project>-tfstate-locks"
      }
    }
  }]
}
```

**GCP** — `data/gcp/<project>/backend.tf.json`:
```json
{
  "terraform": [{
    "backend": {
      "gcs": {
        "bucket": "<project>-tfstate-bootstrap",
        "prefix": "<project>/terraform"
      }
    }
  }]
}
```

**OCI** — `data/oci/<project>/backend.tf.json`:
```json
{
  "terraform": [{
    "backend": {
      "oci": {
        "bucket": "<project>-tfstate-bootstrap",
        "namespace": "<your-object-storage-namespace>",
        "region": "<your-region>",
        "prefix": "<project>"
      }
    }
  }]
}
```

**OVH** — `data/ovh/<project>/backend.tf.json`:
```json
{
  "terraform": [{
    "backend": {
      "s3": {
        "bucket": "<project>-tfstate",
        "key": "<project>/terraform.tfstate",
        "region": "<your-s3-region>",
        "endpoint": "https://s3.<region>.io.cloud.ovh.net",
        "skip_credentials_validation": true,
        "skip_region_validation": true,
        "skip_requesting_account_id": true,
        "skip_s3_checksum": true
      }
    }
  }]
}
```

### 6. Configure the production environment (recommended)

Go to **Settings → Environments → New environment** and create `production`. Add required reviewers so every apply to `main` must be approved before it runs.

### 7. Add infrastructure and open a pull request

Edit the JSON files under `data/<cloud>/<project>/`. Each file represents one resource instance. Opening a pull request triggers a `terraform plan` and posts the output as a comment. Merging to `main` triggers `terraform apply`.

## Repository structure

```
VERSION                                   # template version this repo was created from
grauss.json                               # cloud, project name, and image version
data/
└── <cloud>/
    ├── _bootstrap/                       # remote state infrastructure (apply once)
    │   └── <resource_type>/
    │       └── <name>.json
    └── <project>/
        ├── backend.tf.json               # optional — remote state config (add after bootstrap)
        └── <resource_type>/
            └── <resource-name>.json      # one file per resource instance
```

**One JSON file = one resource instance.** To add a resource, create a JSON file in the matching `<resource_type>/` directory. To destroy it, delete the file.

## Day-to-day workflow

```
Edit JSON files under data/<cloud>/<project>/
        ↓
git checkout -b my-change && git push
        ↓
Open pull request → plan runs automatically → output posted as PR comment
        ↓
Review plan, get approval, merge to main
        ↓
Apply runs automatically (with production environment approval if configured)
```

## Terraform vs OpenTofu

Both engines accept the same JSON inputs and provider. The only difference is the binary inside the Docker image. To switch:

1. Change `image_version` in `grauss.json` to a version that has OpenTofu images published.
2. Edit the workflow files to replace `grauss-<cloud>-terraform` with `grauss-<cloud>-opentofu` in the image name.

The JSON files under `data/` do not change when switching engines.
