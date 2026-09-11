# Phase 6 — Kubernetes + AKS Deployment

## Goal
Create an AKS cluster, write the Kubernetes manifests, and deploy the anomaly detection API so it's accessible via a public endpoint.

> **Prerequisite:** Complete [Phase 5](05_cicd_github_actions.md) — CI/CD pipeline configured.

---

## What is Kubernetes? (Quick mental model)

You already understand pipeline orchestration. Kubernetes (K8s) is orchestration for *containers*:

- You tell it "I want 2 copies of my API running at all times"
- It starts them, monitors them, restarts them if they crash, and distributes traffic across them
- AKS = Azure's managed Kubernetes — Azure handles the control plane, you just manage your apps

**Key concepts for interviews:**

| Term | What it does |
|---|---|
| `Pod` | One running container (the smallest unit) |
| `Deployment` | Manages a set of identical pods — handles restarts, rolling updates |
| `Service` | Gives the pods a stable network address + load balancing |
| `LoadBalancer` | Service type that provisions an Azure public IP |
| `readinessProbe` | K8s checks this endpoint before sending traffic — if it fails, pod is removed from rotation |
| `resources` | CPU/memory limits per pod — prevents one pod starving others |

---

## Step 1 — Create the AKS Cluster

```bash
# Create AKS cluster (1 node to keep costs low for portfolio)
az aks create \
  --resource-group skab-mlops-rg \
  --name skab-mlops-aks \
  --node-count 1 \
  --node-vm-size Standard_DS2_v2 \
  --enable-managed-identity \
  --attach-acr skabmlopsacr \
  --generate-ssh-keys

# Get credentials (configures kubectl to talk to this cluster)
az aks get-credentials \
  --resource-group skab-mlops-rg \
  --name skab-mlops-aks

# Verify connection
kubectl get nodes
# Should show 1 node in "Ready" state
```

> `--attach-acr skabmlopsacr` gives AKS permission to pull images from your ACR — no separate auth needed.

---

## Step 2 — Deployment Manifest

Create `deployment/k8s-deployment.yaml`:

```yaml
# deployment/k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: anomaly-api
  labels:
    app: anomaly-api
spec:
  replicas: 2                        # run 2 copies of the pod
  selector:
    matchLabels:
      app: anomaly-api
  template:
    metadata:
      labels:
        app: anomaly-api
    spec:
      containers:
        - name: anomaly-api
          image: IMAGE_PLACEHOLDER   # replaced by CI/CD pipeline (sed command)
          ports:
            - containerPort: 8000
          env:
            - name: MODEL_URI
              value: "models:/skab-anomaly-detector/Staging"
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health          # FastAPI /health endpoint
              port: 8000
            initialDelaySeconds: 15  # wait 15s before first check (model loading time)
            periodSeconds: 10        # check every 10s
            failureThreshold: 3      # fail 3 times before removing from rotation
```

---

## Step 3 — Service Manifest

Create `deployment/k8s-service.yaml`:

```yaml
# deployment/k8s-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: anomaly-api-service
spec:
  type: LoadBalancer          # Azure provisions a public IP for this
  selector:
    app: anomaly-api          # routes traffic to pods with this label
  ports:
    - protocol: TCP
      port: 80                # external port (public-facing)
      targetPort: 8000        # internal port (FastAPI inside container)
```

---

## Step 4 — First Manual Deploy (to test before CI/CD takes over)

```bash
# Replace placeholder with your actual image
export IMAGE="skabmlopsacr.azurecr.io/anomaly-api:latest"
sed "s|IMAGE_PLACEHOLDER|$IMAGE|g" deployment/k8s-deployment.yaml | kubectl apply -f -
kubectl apply -f deployment/k8s-service.yaml

# Watch pods come up
kubectl get pods -w
# Wait until STATUS = Running, READY = 1/1

# Get the public IP (takes ~2 minutes to provision)
kubectl get service anomaly-api-service
# Look for EXTERNAL-IP — initially shows <pending>, then an IP
```

---

## Step 5 — Test the Live Endpoint

Once you have the EXTERNAL-IP:

```bash
export EXTERNAL_IP="<paste your EXTERNAL-IP here>"

# Health check
curl http://$EXTERNAL_IP/health

# Prediction
curl -X POST http://$EXTERNAL_IP/predict \
  -H "Content-Type: application/json" \
  -d '{
    "sensor_value": 1.25,
    "rolling_mean_5": 1.18,
    "rolling_std_5": 0.06,
    "rolling_max_5": 1.31,
    "rolling_min_5": 1.10
  }'
```

---

## Step 6 — Useful kubectl Commands

```bash
# See all running pods
kubectl get pods

# See logs from a pod (replace <pod-name> with actual name from above)
kubectl logs <pod-name>

# Describe a pod (useful for debugging startup issues)
kubectl describe pod <pod-name>

# Scale up/down replicas manually
kubectl scale deployment anomaly-api --replicas=3

# See rollout history
kubectl rollout history deployment/anomaly-api

# Roll back to previous version
kubectl rollout undo deployment/anomaly-api
```

---

## Step 7 — Push a Code Change to Test the Full CI/CD Loop

```bash
# Make any small change (e.g. update the version in app.py)
# Then push
git add .
git commit -m "test: trigger CI/CD rollout"
git push origin main
```

Go to GitHub → **Actions** → watch the full pipeline run end to end → verify new pods roll out in AKS.

---

## Checkpoint ✅

By the end of Phase 6 you should have:
- [ ] AKS cluster running: `skab-mlops-aks`
- [ ] `deployment/k8s-deployment.yaml` and `k8s-service.yaml` created
- [ ] Pods running: `kubectl get pods` shows 2 pods in `Running` state
- [ ] Public endpoint responding to `/health` and `/predict`
- [ ] Full CI/CD loop tested: `git push` → new image → AKS rollout

**Next:** [Interview Talking Points](07_interview_talking_points.md)
