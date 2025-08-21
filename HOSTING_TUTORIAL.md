# Huly Self‑Hosting (GKE + Kubernetes)

This guide shows the simplest, reliable path to get Huly running on Google Kubernetes Engine (GKE) using the provided GitHub Actions workflow. It builds from source, pushes images to Google Artifact Registry, provisions MongoDB (TLS enabled, external access via LoadBalancer), pulls app secrets from Google Secret Manager (GSM) with GitHub Secrets as fallback, and deploys via kustomize. The workflow also provisions MinIO (for storage) and Elasticsearch (for full‑text) in‑cluster by default.

Follow the steps exactly; it’s designed to work end‑to‑end.

## Prerequisites

- A Google Cloud project (Project ID noted as <PROJECT_ID>)
- Owner/Editor on the project or equivalent permissions
- Billing enabled; GKE API enabled
- A GitHub repository with this code
- A domain (optional) and an Ingress controller if you want public access

## One‑time Google Cloud setup

1. Install tools locally (once):

- gcloud SDK: https://cloud.google.com/sdk
- kubectl: https://kubernetes.io/docs/tasks/tools/
- helm: https://helm.sh/docs/intro/install/

2. Enable required services:

```bash
PROJECT_ID=<PROJECT_ID>
gcloud config set project $PROJECT_ID

gcloud services enable container.googleapis.com \
  artifactregistry.googleapis.com \
  iamcredentials.googleapis.com \
  secretmanager.googleapis.com
```

3. Create a GKE cluster (example single zone):

```bash
REGION=europe-west1
ZONE=europe-west1-b
CLUSTER=huly-prod
MACHINE=e2-standard-4

gcloud container clusters create "$CLUSTER" \
  --zone "$ZONE" \
  --machine-type "$MACHINE" \
  --num-nodes 3 \
  --workload-pool=$(gcloud config get-value project).svc.id.goog

gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
```

4. Create an Artifact Registry repo for Docker images:

```bash
LOCATION=europe
REPO=huly

gcloud artifacts repositories create "$REPO" \
  --repository-format=docker \
  --location="$LOCATION" \
  --description="Huly images"
```

5. Create a Workload Identity Pool + Provider for GitHub Actions (WIF)

- Follow Google’s official guide: https://github.com/google-github-actions/auth#setting-up-workload-identity-federation
- Outcome needed:
  - A Workload Identity Provider resource name (WORKLOAD_IDENTITY_PROVIDER)
  - A Service Account email (SERVICE_ACCOUNT) with required roles:
    - roles/artifactregistry.writer
    - roles/container.admin (or granular k8s access if you prefer RBAC)
    - roles/secretmanager.secretAccessor

Example commands (replace owner/repo):

```bash
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')

# Vars
POOL_ID=github-pool
PROVIDER_ID=github-oidc
SA_NAME=gh-actions-deploy
REPO="owner/repo"   # e.g., erzenkrasniqi/albarchitect-huly

# Create Workload Identity Pool and Provider
gcloud iam workload-identity-pools create "$POOL_ID" \
  --project="$PROJECT_ID" --location=global \
  --display-name="GitHub Actions Pool"

gcloud iam workload-identity-pools providers create-oidc "$PROVIDER_ID" \
  --project="$PROJECT_ID" --location=global \
  --workload-identity-pool="$POOL_ID" \
  --display-name="GitHub OIDC" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref"

# Create Service Account
gcloud iam service-accounts create "$SA_NAME" \
  --project="$PROJECT_ID" \
  --display-name="GitHub Actions Deploy"

SA_EMAIL="$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com"

# Grant deploy roles to the SA
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SA_EMAIL" --role="roles/artifactregistry.writer"
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SA_EMAIL" --role="roles/container.admin"
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SA_EMAIL" --role="roles/secretmanager.secretAccessor"

# Allow GitHub repo identities from the pool to impersonate the SA
gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" \
  --project="$PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$POOL_ID/attribute.repository/$REPO"

# Output provider resource name for GitHub Secret GCP_WORKLOAD_ID_PROVIDER
echo "projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/$POOL_ID/providers/$PROVIDER_ID"

# Output SA email for GitHub Secret GCP_SERVICE_ACCOUNT
echo "$SA_EMAIL"
```

6. Create the Kubernetes namespace (workflow will create it if missing, but doing it once is fine):

```bash
kubectl create namespace huly || true
```

## Secrets management

Use GSM for app secrets; keep GitHub Secrets minimal. The workflow tries GSM first and falls back to GitHub Secrets. Non‑sensitive config is also written to a ConfigMap `huly-config`; sensitive values are stored in the Secret `huly-secrets`. If a referenced GSM secret does not exist, the workflow creates it and adds a version.

1. In Google Secret Manager, create (as needed) the following secrets. Start with these minimum ones; you can add more later:

- SERVER_SECRET
- OPENAI_API_KEY (optional)
- SMTP_HOST, SMTP_PORT, SMTP_USERNAME, SMTP_PASSWORD (if sending email)
- Any other app vars you need from `.github/workflows/gke-deploy.yml` under the “Fetch secrets from Google Secret Manager” step

In‑cluster services (installed automatically by the workflow):

- MinIO: Credentials are generated and used to compute `STORAGE_CONFIG`
- Elasticsearch: Exposed within cluster at `http://elastic:9200` for `ELASTIC_URL`

For MongoDB credentials (workflow will auto‑generate if missing):

- MONGO_ROOT_USER (optional)
- MONGO_ROOT_PASSWORD (optional)

2. In GitHub repository settings → Secrets and variables → Actions, set these GitHub Secrets (minimal):

- GCP_PROJECT_ID = <PROJECT_ID>
- GAR_LOCATION = <LOCATION> (e.g., europe)
- GAR_REPOSITORY = <REPO> (e.g., huly)
- GKE_CLUSTER = <CLUSTER>
- GKE_ZONE = <ZONE>
- GCP_WORKLOAD_ID_PROVIDER = <WORKLOAD_IDENTITY_PROVIDER> (resource name)
- GCP_SERVICE_ACCOUNT = <SERVICE_ACCOUNT> (email)

Optional GitHub fallbacks (if you didn’t put them in GSM):

- SERVER*SECRET, OPENAI_API_KEY, SMTP*\* etc.

Note: The workflow also pushes a Kubernetes Secret `huly-secrets` that the app consumes.

The workflow also creates a ConfigMap `huly-config` with commonly used non‑sensitive keys (e.g., `MONGO_URL`, `DB_URL`, `TRANSACTOR_URL`, `FRONT_URL`, `ACCOUNTS_URL`, `STATS_URL`, `ELASTIC_URL`).

## Deploy flow (CI/CD)

Push to `main` or manually trigger the workflow “Build and Deploy to GKE”. The workflow will:

1. Build and bundle from source using Rush/PNPM
2. Authenticate to Google via WIF
3. Build Docker images for the core pods and optional services
4. Push images to Artifact Registry
5. Install Helm and deploy in‑cluster database:
   - MongoDB (auth enabled)
6. Pull secrets from GSM (fallback to GitHub Secrets), and create/update `huly-secrets` in namespace `huly`
7. Create/update `huly-config` ConfigMap with standard non‑sensitive settings
8. Ensure required GSM secrets exist (create + add versions if missing)
9. Apply kustomize overlay if present, or skip if none

No manual kubectl commands are needed beyond initial setup.

## Verify

After the workflow completes:

```bash
kubectl -n huly get pods
kubectl -n huly get svc
```

Look for the MongoDB pod and your application deployments. The workflow sets image tags automatically.

About the transactor endpoint (`TRANSACTOR_URL`): upstream expects a semicolon‑separated list of endpoints, typically an internal WebSocket endpoint and a public one used by the browser. If `FRONT_URL` is set, the workflow defaults `TRANSACTOR_URL` to `ws://server:3333;wss://<your-front-host>/ws/transactor`. You can override via GSM or GitHub Secrets.

MongoDB: The workflow enables TLS (`requireTLS`) and exposes the service via a `LoadBalancer` for external access, while also providing the internal cluster DNS name `huly-mongo-mongodb:27017`. The workflow computes and stores connection strings with TLS parameters. For external tools, prefer the external URL stored in GSM/Kubernetes secrets if needed.

## Exposing the app (optional)

If you have a `kube/overlays/prod` or `kube/base` with Services/Ingress, the workflow applies it. If not, you can expose the front end with a simple LoadBalancer service:

```bash
kubectl -n huly expose deploy/front --type=LoadBalancer --name=front-lb --port=80 --target-port=8080
kubectl -n huly get svc front-lb -w
```

Then map your domain’s DNS to the external IP.

## Updating configuration

- Add or change secrets in GSM; re‑run the workflow to sync them to Kubernetes.

## Troubleshooting

- Workflow fails at GKE credentials: verify cluster name/zone and that your WIF service account has permission.
- Images fail to push: ensure roles/artifactregistry.writer on the service account, and that GAR repo/location match.
- Secrets not found: create them in GSM or define GitHub fallbacks; re‑run the workflow.
- App cannot connect to DB (MongoDB): confirm pods can resolve `huly-mongo-mongodb:27017`, that `MONGO_URL`/`DB_URL` are present in `huly-secrets`/`huly-config`, and that auth credentials match the generated values (stored in GSM as `MONGO_ROOT_USER`/`MONGO_ROOT_PASSWORD`).
- Full‑text or attachments not working: ensure `ELASTIC_URL` and `STORAGE_CONFIG` are set. If you enabled `INSTALL_ELASTIC` or `INSTALL_MINIO`, the workflow will set reasonable defaults (`http://elastic:9200` and `minio|minio?...`). Otherwise, point these to your managed services.

## That’s it

Commit and push. The pipeline does the rest—builds from source, provisions secure in‑cluster DBs, syncs secrets, and deploys to your GKE cluster.
