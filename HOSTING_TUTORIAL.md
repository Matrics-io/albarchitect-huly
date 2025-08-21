# Huly Self‑Hosting (GKE + Kubernetes)

This guide shows the simplest, reliable path to get Huly running on Google Kubernetes Engine (GKE) using the provided GitHub Actions workflow. It builds from source, pushes images to Google Artifact Registry, provisions CockroachDB and MongoDB in‑cluster, pulls app secrets from Google Secret Manager (GSM) with GitHub Secrets as fallback, and deploys via kustomize.

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

6. Create the Kubernetes namespace (workflow will create it if missing, but doing it once is fine):

```bash
kubectl create namespace huly || true
```

## Secrets management

Use GSM for app secrets; keep GitHub Secrets minimal. The workflow tries GSM first and falls back to GitHub Secrets.

1. In Google Secret Manager, create (as needed) the following secrets. Start with these minimum ones; you can add more later:

- SERVER_SECRET
- OPENAI_API_KEY (optional)
- SMTP_HOST, SMTP_PORT, SMTP_USERNAME, SMTP_PASSWORD (if sending email)
- Any other app vars you need from `.github/workflows/gke-deploy.yml` under the “Fetch secrets from Google Secret Manager” step

For CockroachDB and MongoDB credentials (workflow will auto‑generate if missing):

- CRDB_USER (optional)
- CRDB_PASSWORD (optional)
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

## Deploy flow (CI/CD)

Push to `main` or manually trigger the workflow “Build and Deploy to GKE”. The workflow will:

1. Build and bundle from source using Rush/PNPM
2. Authenticate to Google via WIF
3. Build Docker images for the core pods and optional services
4. Push images to Artifact Registry
5. Install Helm and deploy in‑cluster databases:
   - CockroachDB (3 replicas, TLS enabled, self‑signed certs) and creates a DB user
   - MongoDB (auth enabled)
6. Pull secrets from GSM (fallback to GitHub Secrets), and create/update `huly-secrets` in namespace `huly`
7. Apply kustomize overlay if present, or skip if none

No manual kubectl commands are needed beyond initial setup.

## Verify

After the workflow completes:

```bash
kubectl -n huly get pods
kubectl -n huly get svc
```

Look for CockroachDB statefulset pods (`huly-crdb-cockroachdb-0/1/2`), MongoDB pod, and your application deployments. The workflow sets image tags automatically.

## Exposing the app (optional)

If you have a `kube/overlays/prod` or `kube/base` with Services/Ingress, the workflow applies it. If not, you can expose the front end with a simple LoadBalancer service:

```bash
kubectl -n huly expose deploy/front --type=LoadBalancer --name=front-lb --port=80 --target-port=8080
kubectl -n huly get svc front-lb -w
```

Then map your domain’s DNS to the external IP.

## Updating configuration

- Add or change secrets in GSM; re‑run the workflow to sync them to Kubernetes.
- To scale CockroachDB:

```bash
helm -n huly upgrade --install huly-crdb cockroachdb/cockroachdb \
  --set statefulset.replicas=5 \
  --set tls.enabled=true \
  --set tls.certs.selfSigner.enabled=true
```

## Troubleshooting

- Workflow fails at GKE credentials: verify cluster name/zone and that your WIF service account has permission.
- Images fail to push: ensure roles/artifactregistry.writer on the service account, and that GAR repo/location match.
- Secrets not found: create them in GSM or define GitHub fallbacks; re‑run the workflow.
- App cannot connect to DB: the workflow injects CockroachDB CA into `POSTGRES_OPTIONS` and uses `sslmode=require`. Confirm pods can resolve `huly-crdb-cockroachdb-public:26257` and that `DB_URL` is present in `huly-secrets`.

## That’s it

Commit and push. The pipeline does the rest—builds from source, provisions secure in‑cluster DBs, syncs secrets, and deploys to your GKE cluster.
