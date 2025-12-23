# GKE + Argo CD Guide (GCP)

This document shows the recommended steps to deploy OSSN to Google Kubernetes Engine (GKE) and install Argo CD for GitOps. It covers Workload Identity, Artifact Registry, Cloud SQL connectivity, and Argo CD installation and bootstrapping.

Replace placeholders: `YOUR_PROJECT`, `YOUR_REGION`, `YOUR_CLUSTER`, `YOUR_ARTIFACT_REPO`, and `YOUR_GCP_SA` with your values.

## 1. Prerequisites
- Install `gcloud`, `kubectl`, `docker` (or `podman`), and optionally `helm` and `argocd` CLI.
- Authenticate: `gcloud auth login` and `gcloud config set project YOUR_PROJECT`.

## 2. Enable required APIs

```bash
gcloud services enable container.googleapis.com sqladmin.googleapis.com artifactregistry.googleapis.com iam.googleapis.com
```

## 3. Create GKE cluster (Workload Identity)

Workload Identity lets Kubernetes service accounts act as Google service accounts without storing JSON keys.

```bash
# Standard cluster example
gcloud container clusters create YOUR_CLUSTER \
  --region YOUR_REGION \
  --num-nodes=3 \
  --workload-pool=YOUR_PROJECT.svc.id.goog

# Or Autopilot (managed nodes)
gcloud container clusters create-auto YOUR_CLUSTER --region YOUR_REGION --workload-pool=YOUR_PROJECT.svc.id.goog

# Get credentials for kubectl
gcloud container clusters get-credentials YOUR_CLUSTER --region YOUR_REGION
kubectl get nodes
```

## 4. Artifact Registry: build & push OSSN image

Create an Artifact Registry Docker repo (one-time):

```bash
gcloud artifacts repositories create ossn-repo --repository-format=docker --location=YOUR_REGION
```

Build, tag and push the image:

```bash
# from docker/ directory
docker build -t YOUR_REGION-docker.pkg.dev/YOUR_PROJECT/ossn-repo/ossn:latest .
docker push YOUR_REGION-docker.pkg.dev/YOUR_PROJECT/ossn-repo/ossn:latest
```

Update `docker/k8s/ossn-k8s.yaml` `image:` field to this Artifact Registry path.

## 5. Cloud SQL setup (MySQL)

- Create a Cloud SQL MySQL instance in your project (via Console or `gcloud sql instances create`).
- Note the instance connection name: `PROJECT:REGION:INSTANCE`.
- Decide on connection method:
  - Workload Identity (recommended): grant `roles/cloudsql.client` to a GCP service account and bind it to a Kubernetes service account.
  - JSON key (less recommended): create a service account JSON key and store it in a Kubernetes secret (not recommended for production).

### Workload Identity (recommended)

```bash
# create GCP SA
gcloud iam service-accounts create ossn-cloudsql-sa --display-name="OSSN Cloud SQL SA"

gcloud projects add-iam-policy-binding YOUR_PROJECT \
  --member="serviceAccount:ossn-cloudsql-sa@YOUR_PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# create k8s namespace and KSA
kubectl create namespace ossn
kubectl create serviceaccount ossn-ksa -n ossn

# allow the KSA to impersonate the GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  ossn-cloudsql-sa@YOUR_PROJECT.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:YOUR_PROJECT.svc.id.goog[ossn/ossn-ksa]"

# annotate the KSA
kubectl annotate serviceaccount --namespace ossn ossn-ksa \
  iam.gke.io/gcp-service-account=ossn-cloudsql-sa@YOUR_PROJECT.iam.gserviceaccount.com
```

If you cannot use Workload Identity, create a secret with the JSON key and mount it into the Cloud SQL Proxy sidecar as `cloudsql-sa-key`.

```bash
kubectl create secret generic cloudsql-sa-key --from-file=credentials.json=/path/to/key.json -n ossn
```

## 6. Update `ossn-k8s.yaml`

- Set `CLOUDSQL_CONNECTION_NAME` to your instance connection name (format: `PROJECT:REGION:INSTANCE`).
- Ensure `serviceAccountName: ossn-ksa` is present under `spec.template.spec` for the `ossn-web` Deployment.
- If using Workload Identity, remove the `cloudsql-sa-key` secret volume and let the proxy use the KSA.

## 7. Create namespace and apply manifests

```bash
kubectl create namespace ossn
kubectl apply -f docker/k8s/ossn-k8s.yaml -n ossn
kubectl rollout status deployment/ossn-web -n ossn
kubectl get pods,svc -n ossn
```

## 8. Install Argo CD

Install Argo CD into the cluster to manage deployments via GitOps.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose the server (quick test using LoadBalancer):

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc -n argocd argocd-server
```

Port-forward (local test):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# open https://localhost:8080
```

Get the initial admin password and install `argocd` CLI:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
# macOS: brew install argocd
argocd login <ARGOCD_SERVER> --username admin --password <PASSWORD> --insecure
```

## 9. Bootstrap an Argo CD Application

Create an `Application` that points to your Git repository and path with k8s manifests (for example `k8s/` or `docker/k8s`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ossn
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/YOUR_USER/YOUR_REPO.git'
    targetRevision: HEAD
    path: docker/k8s
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: ossn
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply it:

```bash
kubectl apply -f app.yaml
```

## 10. Notes and best practices
- Use Workload Identity instead of JSON keys where possible.
- For production, expose Argo CD with Ingress + TLS and lock down access (SSO/OIDC).
- Use private IP for Cloud SQL where possible and ensure VPC peering between GKE and Cloud SQL.
- Store sensitive config in Kubernetes `Secrets` or use a secret manager.
- For multi-environment deployments, use `kustomize` or Helm charts and separate overlays.

---
If you want, I can:
- update `docker/k8s/ossn-k8s.yaml` to remove the JSON-key secret and use Workload Identity annotations,
- or generate a sample `app.yaml` ready to apply to Argo CD with the correct repo URL.
