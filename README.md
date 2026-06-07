# terraform-google-litellm

> **This is a read-only mirror.** Source of truth is [`BerriAI/litellm//terraform/litellm/gcp/`](https://github.com/BerriAI/litellm/tree/main/terraform/litellm/gcp). Open issues and PRs there — anything pushed directly to this repo will be overwritten on the next release.

Terraform module that deploys [LiteLLM](https://github.com/BerriAI/litellm) on GCP — gateway, backend, and UI on Cloud Run, Cloud SQL Postgres (writer + reader), Memorystore Redis, GCS, and an external HTTPS Load Balancer with URL-map routing. Each release tag here is mirrored from a `BerriAI/litellm` release.

## Usage

```hcl
module "litellm" {
  source  = "BerriAI/litellm/google"
  version = "~> 1.89"

  project_id = "my-gcp-project"
  region     = "us-central1"
  tenant     = "acme"
  env        = "prod"

  # Cloud Run rejects ghcr.io. Mirror through Artifact Registry first:
  #   gcloud artifacts repositories create litellm \
  #     --location=us-central1 --repository-format=docker \
  #     --mode=remote-repository \
  #     --remote-docker-repo=https://ghcr.io
  image_registry = "us-central1-docker.pkg.dev/my-gcp-project/litellm/berriai"
}
```

Then:

```bash
terraform init
terraform apply
```

### Required inputs

| Name | Description |
|---|---|
| `project_id` | GCP project to deploy into. |
| `region` | GCP region for Cloud Run / Cloud SQL / Memorystore. |
| `tenant` | Identifier prefix for every resource (`${tenant}-litellm-${env}`). |
| `env` | Environment slug (`prod`, `stage`, `dev`, ...). |

**Plus one of**: `image_registry` pointing at an Artifact Registry **remote** repository backed by `https://ghcr.io`, **or** all four per-component `*_image` overrides pointing at images you've mirrored into a regular Artifact Registry repo. Cloud Run only authenticates against Artifact Registry / `[region.]gcr.io` / `docker.io` — the public `ghcr.io/berriai/...` defaults will fail admission.

All other inputs have safe defaults — full list in [`variables.tf`](./variables.tf) or the auto-generated [registry page](https://registry.terraform.io/modules/BerriAI/litellm/google/latest?tab=inputs).

### Common knobs

- **TLS**: pass DNS names in `lb_domains` and the LB will use Google-managed certs (issuance takes 15–60 min after DNS points at the LB IP). Fall back to HTTP only when `allow_plaintext_lb = true` (dev/trial use only).
- **Proxy config**: pass YAML as a typed map via `proxy_config` (mirrors the [Helm chart](https://github.com/BerriAI/litellm/tree/main/helm/litellm)'s `gateway.config.proxy_config`).
- **Provider API keys**: store in Secret Manager, reference resource IDs via `gateway_extra_secrets` / `backend_extra_secrets` (e.g. `projects/<id>/secrets/openai-api-key`).
- **Data retention**: `cloudsql_deletion_protection` (default `true`) refuses `terraform destroy` on the DB; `gcs_force_destroy` (default `false`) refuses on a non-empty bucket. Flip both to `false`/`true` only for ephemeral stacks.

## Architecture

```
External HTTPS LB (global, URL map)
  ├── LLM data-plane prefixes (/v1/chat/*, /v1/embeddings, ...) → Cloud Run (gateway)
  ├── UI assets (/, /_next/*, /litellm-asset-prefix/*)          → Cloud Run (ui)
  └── everything else (management: /key/*, /user/*, ...)         → Cloud Run (backend)
     (via Serverless NEGs, one per service)

Private VPC (Serverless VPC Access connector):
  Cloud SQL Postgres (writer + reader)
  Memorystore Redis
  GCS bucket (versioned)
  Secret Manager (LITELLM_MASTER_KEY, DB password, your API keys)
  One-off Cloud Run Job: prisma migrate deploy
```

## Migration job

The proxy runs `prisma migrate deploy` at startup, but on first apply the
gateway/backend can race the empty DB. The module exposes a one-off
Cloud Run Job (`litellm-migrations`) that runs the migration;
`terraform apply` auto-runs it via `local-exec` (requires the `gcloud`
CLI on the apply machine). To run manually, see the
`migration_run_command` output.

## Issues & contributions

File issues at [BerriAI/litellm](https://github.com/BerriAI/litellm/issues). PRs against the Terraform module go to [`BerriAI/litellm`](https://github.com/BerriAI/litellm) under `terraform/litellm/gcp/`.

## License

Same as [BerriAI/litellm](https://github.com/BerriAI/litellm/blob/main/LICENSE).
