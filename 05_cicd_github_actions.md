# Phase 5 — CI/CD with GitHub Actions

## Goal
Set up a GitHub Actions pipeline that automatically builds a new Docker image and deploys it to AKS every time you push to the `main` branch.

> **Prerequisite:** Complete [Phase 4](04_docker_packaging.md) — image pushed to ACR manually at least once.

---

## What is CI/CD? (Quick mental model)

You already do release governance (Dev → QA → Prod) manually in your current role. CI/CD automates this:

- **CI (Continuous Integration):** Every push to `main` triggers an automated build + test
- **CD (Continuous Deployment):** If the build passes, it automatically deploys to AKS

The pipeline replaces the manual steps: "build image → push to ACR → update AKS deployment" with a single `git push`.

---

## Step 1 — Create the Workflow File

```bash
mkdir -p .github/workflows
touch .github/workflows/ci-cd.yaml
```

```yaml
# .github/workflows/ci-cd.yaml
name: MLOps CI/CD Pipeline

on:
  push:
    branches: [main]       # triggers on every push to main
  workflow_dispatch:       # also allows manual trigger from GitHub UI

env:
  ACR_NAME: skabmlopsacr
  IMAGE_NAME: anomaly-api
  RESOURCE_GROUP: skab-mlops-rg
  AKS_CLUSTER: skab-mlops-aks    # we'll create this in Phase 6

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # 1. Check out the code
      - name: Checkout code
        uses: actions/checkout@v4

      # 2. Log in to Azure using a Service Principal (stored as a secret)
      - name: Azure login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      # 3. Log in to ACR
      - name: Login to ACR
        run: az acr login --name $ACR_NAME

      # 4. Build Docker image tagged with the Git commit SHA (unique per push)
      - name: Build Docker image
        run: |
          docker build -f deployment/Dockerfile \
            -t $ACR_NAME.azurecr.io/$IMAGE_NAME:$GITHUB_SHA \
            -t $ACR_NAME.azurecr.io/$IMAGE_NAME:latest \
            .

      # 5. Push both tags to ACR
      - name: Push image to ACR
        run: |
          docker push $ACR_NAME.azurecr.io/$IMAGE_NAME:$GITHUB_SHA
          docker push $ACR_NAME.azurecr.io/$IMAGE_NAME:latest

      # 6. Set kubectl context to your AKS cluster
      - name: Set AKS context
        uses: azure/aks-set-context@v3
        with:
          resource-group: ${{ env.RESOURCE_GROUP }}
          cluster-name: ${{ env.AKS_CLUSTER }}

      # 7. Replace placeholder in k8s manifest with the actual image tag
      - name: Update image in deployment manifest
        run: |
          sed -i "s|IMAGE_PLACEHOLDER|$ACR_NAME.azurecr.io/$IMAGE_NAME:$GITHUB_SHA|g" \
            deployment/k8s-deployment.yaml

      # 8. Apply the manifests to AKS
      - name: Deploy to AKS
        run: |
          kubectl apply -f deployment/k8s-deployment.yaml
          kubectl apply -f deployment/k8s-service.yaml

      # 9. Wait for rollout to complete (fails pipeline if pods don't come up)
      - name: Verify rollout
        run: kubectl rollout status deployment/anomaly-api --timeout=120s
```

---

## Step 2 — Create Azure Service Principal (for GitHub to authenticate)

Run this in your terminal:

```bash
# Get your subscription ID
az account show --query id -o tsv

# Create service principal with contributor access on the resource group
az ad sp create-for-rbac \
  --name "skab-mlops-github-sp" \
  --role contributor \
  --scopes /subscriptions/<YOUR_SUBSCRIPTION_ID>/resourceGroups/skab-mlops-rg \
  --sdk-auth
```

This outputs a JSON block. **Copy the entire JSON output** — you'll need it in the next step.

---

## Step 3 — Add Secrets to GitHub

1. Go to your GitHub repo → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret** for each:

| Secret name | Value |
|---|---|
| `AZURE_CREDENTIALS` | The entire JSON output from Step 2 |
| `ACR_NAME` | `skabmlopsacr` |
| `RESOURCE_GROUP` | `skab-mlops-rg` |
| `AKS_CLUSTER` | `skab-mlops-aks` (we create this in Phase 6) |

---

## Step 4 — Commit and Push to Trigger the Pipeline

```bash
git add .github/workflows/ci-cd.yaml
git commit -m "add: GitHub Actions CI/CD pipeline"
git push origin main
```

Go to your GitHub repo → **Actions** tab → watch the pipeline run.

> **Note:** The deploy step will fail until AKS is set up in Phase 6. The build and push steps should succeed. That's expected at this stage.

---

## Understanding the Pipeline Flow

```
git push to main
      ↓
GitHub Actions triggered
      ↓
[Step 1] Checkout code
      ↓
[Step 2] az login (using AZURE_CREDENTIALS secret)
      ↓
[Step 3] az acr login
      ↓
[Step 4] docker build → image tagged with commit SHA
      ↓
[Step 5] docker push → ACR
      ↓
[Step 6] kubectl config → points to AKS cluster
      ↓
[Step 7] sed replaces IMAGE_PLACEHOLDER in k8s-deployment.yaml
      ↓
[Step 8] kubectl apply → updates the running deployment
      ↓
[Step 9] kubectl rollout status → waits for pods to become ready
```

---

## Checkpoint ✅

By the end of Phase 5 you should have:
- [ ] `.github/workflows/ci-cd.yaml` created and committed
- [ ] Service principal created in Azure
- [ ] `AZURE_CREDENTIALS` and other secrets added to GitHub
- [ ] Pipeline runs on push (build + push steps succeed)

**Next:** [Phase 6 — Kubernetes + AKS Deployment](06_kubernetes_aks.md)
