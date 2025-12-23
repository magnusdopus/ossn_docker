# OSSN Kubernetes deployment

This directory contains Kubernetes manifests to deploy OSSN (web) and persistent storage. It is configured to use Cloud SQL for MySQL via the Cloud SQL Auth Proxy sidecar.

Quick steps
1. Build the web image locally:

```bash
# from the docker/ folder
docker build -t ossn:latest .
```

2. Load or push the image (pick one):

- For `kind`:

```bash
kind load docker-image ossn:latest
```

- For `minikube`:

```bash
minikube image load ossn:latest
```

- To push to a registry (then update the image in the manifest):

```bash
docker tag ossn:latest your-registry/ossn:latest
docker push your-registry/ossn:latest
# edit docker/k8s/ossn-k8s.yaml and set the image field to your-registry/ossn:latest
```

3. Create Cloud SQL service account and Kubernetes secret

- Create a Google Cloud service account with the `Cloud SQL Client` role and download the JSON key.
- Create the Kubernetes secret (replace `/path/to/key.json`):

```bash
kubectl create secret generic cloudsql-sa-key --from-file=credentials.json=/path/to/key.json
```

4. Update the manifest

- Edit `docker/k8s/ossn-k8s.yaml` and set the `CLOUDSQL_CONNECTION_NAME` value to your instance connection name (format: `PROJECT:REGION:INSTANCE`).

5. Apply manifests

```bash
kubectl apply -f docker/k8s/ossn-k8s.yaml
kubectl get pods,svc
```

Notes and troubleshooting
- The Cloud SQL Auth Proxy sidecar exposes MySQL on `127.0.0.1:3306` inside the pod; `MYSQL_HOST` is already set to `127.0.0.1` in the manifest.
- Ensure the service account used for `cloudsql-sa-key` has the `roles/cloudsql.client` role and that the Cloud SQL Admin API is enabled for your GCP project.
- If you use a private IP Cloud SQL instance, additional VPC configuration may be required.
- If using GKE with Workload Identity, prefer a Kubernetes Service Account mapped to the GCP service account instead of a JSON key; replace the `cloudsql-sa-key` secret usage accordingly.

Contact
- If you want, I can update the manifest to use a registry image name, add Workload Identity examples, or generate a `kustomization.yaml` for overlays.
