# Tasky on DigitalOcean Kubernetes (DOKS)
 
A containerized deployment of [Tasky](https://github.com/jeffthorne/tasky) on DigitalOcean Kubernetes, with a DigitalOcean Load Balancer, Horizontal Pod Autoscaling, and a GitHub Actions CI/CD pipeline.
 
## Architecture
 
```
Internet / Users
      |
      v
DigitalOcean Load Balancer  (auto-provisioned via K8s LoadBalancer service)
      |
      v
DOKS Cluster — tasky-cluster (nyc1, 2x s-2vcpu-2gb nodes, free control plane)
      |
      v
Horizontal Pod Autoscaler  (min: 2 pods, max: 5, CPU threshold: 50%)
      |
      v
Tasky Pods  (Go + Gin, port 8080, /health liveness + readiness probes)
      ^
      |
DO Container Registry — tasky-registry  (free up to 500MiB)
      ^
      |
GitHub Actions CI/CD  (builds + pushes image on every push to main)
```
 
## Repository Structure
 
```
.
├── .github/
│   └── workflows/
│       └── deploy.yml        # CI/CD pipeline — builds and pushes image to DOCR
├── k8s/
│   ├── deployment.yaml       # Kubernetes Deployment (2 replicas, resource limits, health probes)
│   ├── service.yaml          # LoadBalancer Service (port 80 → 8080)
│   └── hpa.yaml              # Horizontal Pod Autoscaler (CPU-based, 2–5 replicas)
├── Dockerfile                # Multi-stage Go build (golang:1.19 → alpine:3.17)
├── main.go                   # App entrypoint — includes /health endpoint
└── README.md
```
 
## Prerequisites
 
- [DigitalOcean account](https://cloud.digitalocean.com) with an API token (read/write)
- [`doctl`](https://docs.digitalocean.com/reference/doctl/how-to/install/) installed and authenticated
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/) installed
- Docker (for local builds; CI/CD handles this automatically via GitHub Actions)
## Deployment Instructions
 
### 1. Authenticate doctl
 
```bash
doctl auth init
# paste your DO API token when prompted
 
doctl account get
# verify connection
```
 
### 2. Create the DOKS Cluster
 
```bash
doctl kubernetes cluster create tasky-cluster \
  --region nyc1 \
  --node-pool "name=tasky-pool;size=s-2vcpu-2gb;count=2" \
  --wait
 
kubectl get nodes
# expect: 2 nodes in Ready state
```
 
### 3. Create the Container Registry
 
```bash
doctl registry create tasky-registry --subscription-tier basic
 
# Grant the cluster pull access
doctl registry kubernetes-manifest > registry-secret.yaml
kubectl create -f registry-secret.yaml
```
 
### 4. Configure GitHub Actions CI/CD
 
Add the following secret to your GitHub repository under **Settings → Secrets and variables → Actions**:
 
| Secret Name | Value |
|---|---|
| `DIGITALOCEAN_ACCESS_TOKEN` | Your DO API token |
 
Push to `main` to trigger the first build. Monitor progress under the **Actions** tab. On success, the image will appear in your DOCR registry.
 
### 5. Deploy to Kubernetes
 
```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml
```
 
### 6. Install the Metrics Server
 
Required for the HPA to read CPU metrics:
 
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
 
# Verify after ~60 seconds
kubectl get deployment metrics-server -n kube-system
# expect: READY 1/1
 
kubectl get hpa
# expect: TARGETS shows a real CPU % (e.g. 1%/50%), not <unknown>
```
 
### 7. Verify the Deployment
 
```bash
# Check pods
kubectl get pods
# expect: 2 pods in Running state
 
# Get the Load Balancer IP
kubectl get service tasky
# wait for EXTERNAL-IP to show a real IP (1-2 minutes)
```
 
Navigate to `http://[EXTERNAL-IP]` in your browser. You should see the Tasky login page.
 
## Environment Variables
 
| Variable | Purpose | Value in this deployment |
|---|---|---|
| `SECRET_KEY` | JWT signing secret | Set in deployment.yaml |
| `MONGODB_URI` | MongoDB connection string | Set in deployment.yaml |
 
## Kubernetes Manifests
 
| File | Description |
|---|---|
| `k8s/deployment.yaml` | Runs 2 Tasky pods; sets resource requests/limits and health probes |
| `k8s/service.yaml` | Exposes app on port 80 via a DigitalOcean Load Balancer |
| `k8s/hpa.yaml` | Scales pods between 2–5 replicas when CPU exceeds 50% |
 
## Cost Breakdown
 
| Component | Monthly Cost |
|---|---|
| DOKS control plane | $0 (free) |
| 2× `s-2vcpu-2gb` worker nodes | ~$24 |
| DigitalOcean Load Balancer | ~$12 |
| Container Registry (under 500MiB) | $0 (free) |
| Bandwidth (within 2,000 GiB/node) | $0 |
| **Total** | **~$36/month** |
 
## Troubleshooting
 
| Symptom | Likely Cause | Resolution |
|---|---|---|
| Pods in `CrashLoopBackOff` | App startup error | `kubectl logs <pod-name>` |
| `EXTERNAL-IP` stuck on `<pending>` | LB still provisioning | Wait 2–3 min and retry |
| HPA shows `<unknown>` CPU | Metrics Server not ready | Wait 3–5 min after install |
| GitHub Actions pipeline fails | Token missing or expired | Re-check `DIGITALOCEAN_ACCESS_TOKEN` secret |
| `kubectl get nodes` fails | kubeconfig not configured | `doctl kubernetes cluster kubeconfig save tasky-cluster` |
 
## Teardown
 
To avoid ongoing charges, delete the cluster and registry when done:
 
```bash
doctl kubernetes cluster delete tasky-cluster
doctl registry delete tasky-registry
```
 
The DigitalOcean Load Balancer is deleted automatically when the cluster is removed.

# Docker
A Dockerfile has been provided to run this application.  The default port exposed is 8080.

# Environment Variables
The following environment variables are needed.
|Variable|Purpose|example|
|---|---|---|
|`MONGODB_URI`|Address to mongo server|`mongodb://servername:27017` or `mongodb://username:password@hostname:port` or `mongodb+srv://` schema|
|`SECRET_KEY`|Secret key for JWT tokens|`secret123`|

Alternatively, you can create a `.env` file and load it up with the environment variables.

# Running with Go

Clone the repository into a directory of your choice Run the command `go mod tidy` to download the necessary packages.

You'll need to add a .env file and add a MongoDB connection string with the name `MONGODB_URI` to access your collection for task and user storage.
You'll also need to add `SECRET_KEY` to the .env file for JWT Authentication.

Run the command `go run main.go` and the project should run on `locahost:8080`

# License

This project is licensed under the terms of the MIT license.

Original project: https://github.com/dogukanozdemir/golang-todo-mongodb
