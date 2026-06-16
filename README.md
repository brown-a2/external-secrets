# external-secrets

Kubernetes manifests that sync secrets from **GCP Secret Manager** into in-cluster `Secret` objects using the [External Secrets Operator](https://external-secrets.io/) (ESO). Deployed via ArgoCD.

## What it does

For each app, ESO watches an `ExternalSecret` CR, pulls the listed keys from GCP Secret Manager every hour, and materializes them as a Kubernetes `Secret` that workloads consume as env vars or mounted files. No plaintext secrets live in this repo — only references to remote keys.

## Structure

| Path | Scope | GCP project | Target Secret(s) |
|------|-------|-------------|------------------|
| `event-tracker/` | namespace `event-tracker` | `event-tracker-474419` | `event-tracker-secrets` (DB, Google OAuth, Supabase, cookie/allowlist) |
| `wordpress-ms-prod/` | namespace `wordpress-ms-prod` | `wordpress-ms-prod-444322` | `mariadb-secrets`, `wordpress-ms-secrets` (WP multisite, S3 uploads, SMTP, admin) |
| `totoro-media/` | namespace `totoro-media` | `totoro-media` | `totoro-media-secrets` (DB, Google OAuth, Cloudflare R2, Dropbox, allowlist) — bundled: one GSM secret `totoro_media_secrets` (JSON) via `dataFrom.extract` |
| `totoro-finance/` | namespace `totoro-finance` | `totoro-finance` | `totoro-finance-secrets` (DB, `TOKEN_ENC_KEY`, `SUPERADMIN_*`, `MONZO_*`) — bundled: one GSM secret `totoro_finance_secrets` (JSON) via `dataFrom.extract` |
| `totoro-ox-k3s-cluster/` | cluster-wide | `totoro-ox-k3s-cluster` | `cloudflare-api-token` in `certmanager` ns (used by cert-manager for DNS-01) |
| `applications/` | ArgoCD | — | Application source spec(s) pointing back at this repo |

Each app directory has:
- `secretstore.yaml` — `SecretStore` (or `ClusterSecretStore`) with `gcpsm` provider config, project ID, and auth reference.
- `external-secrets*.yaml` — one or more `ExternalSecret` manifests mapping remote GCP keys → k8s Secret keys.

## Auth

Each `SecretStore` references a pre-existing Kubernetes Secret holding a GCP service account JSON key. These auth secrets are **not tracked in this repo** and must be created out-of-band before ArgoCD syncs:

| Store | Auth Secret | Key |
|-------|-------------|-----|
| event-tracker | `gcp-credentials-event-tracker` | `key.json` |
| wordpress-ms-prod | `gcp-credentials-wordpress-ms-prod` | `key.json` |
| totoro-media | `gcp-credentials-totoro-media` | `key.json` |
| totoro-finance | `gcp-credentials-totoro-finance` | `key.json` |
| totoro-ox-k3s-cluster (cluster) | `gcp-cluster-secret-manager-credentials` | `credentials.json` |

## Conventions

- API version: `external-secrets.io/v1beta1`.
- GCP Secret Manager keys: `lowercase_snake_case`.
- K8s env var keys: `UPPER_SNAKE_CASE`.
- ExternalSecrets: `refreshInterval: 1h`, `creationPolicy: Owner`, `deletionPolicy: Retain`.

## Adding a secret

1. Create the secret in GCP Secret Manager under the correct project (`lowercase_snake_case` key name).
2. Add a `data` entry to the relevant `external-secrets*.yaml`:
   ```yaml
   - secretKey: MY_NEW_VAR
     remoteRef:
       key: my_new_var
       version: latest
   ```
3. Commit + push. ArgoCD auto-syncs.

## Adding a new app

1. Create GCP project + Secret Manager secrets.
2. Create a GCP service account with `roles/secretmanager.secretAccessor`, download key JSON.
3. In-cluster: `kubectl create secret generic <auth-secret-name> --from-file=key.json=...` in the target namespace.
4. Add a new directory here with `secretstore.yaml` + `external-secrets*.yaml`.
5. Wire up an ArgoCD Application pointing at the new path.
