# Phase 4 — Docker Packaging

## Goal
Package the FastAPI app into a Docker image, test it locally, then push it to Azure Container Registry (ACR).

> **Prerequisite:** Complete [Phase 3](03_fastapi_serving.md) — FastAPI app running locally.

---

## What is Docker? (Quick mental model)

Think of Docker as a shipping container for software.

- Your laptop has Python 3.11, the server might have Python 3.9 — without Docker, things break
- A Docker image packages your app + Python version + all libraries + the OS layer into one sealed unit
- That image runs identically on your laptop, the CI server, and AKS
- "Build once, run anywhere" — this is exactly what eliminates the "works on my machine" problem

**Key terms:**
| Term | Meaning |
|---|---|
| `Dockerfile` | The recipe — instructions for building the image |
| Image | The built package (like a ZIP file of your whole app) |
| Container | A running instance of an image |
| Registry (ACR) | A storage location for images in the cloud |

---

## Step 1 — Create the Dockerfile

In the repo root, create `deployment/Dockerfile`:

```bash
mkdir -p deployment
touch deployment/Dockerfile
```

```dockerfile
# deployment/Dockerfile

# Base image: slim Python 3.10 (smaller than full image)
FROM python:3.10-slim

# Set working directory inside the container
WORKDIR /app

# Copy and install dependencies first (Docker caches this layer)
# If only app code changes, Docker reuses this cached layer → faster builds
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application source code
COPY src/ ./src/

# Expose the port FastAPI runs on
EXPOSE 8000

# Command to start the server when the container starts
CMD ["uvicorn", "src.app:app", "--host", "0.0.0.0", "--port", "8000"]
```

> **Why `--host 0.0.0.0`?**
> By default uvicorn only listens on `127.0.0.1` (localhost inside the container). `0.0.0.0` means "accept connections from anywhere" — needed so traffic from outside the container can reach it.

---

## Step 2 — Build the Image Locally

```bash
# From repo root (where Dockerfile is NOT — it's in deployment/)
docker build -f deployment/Dockerfile -t anomaly-api:latest .
```

What's happening:
- `-f deployment/Dockerfile` — tells Docker where the Dockerfile is
- `-t anomaly-api:latest` — names the image `anomaly-api` with tag `latest`
- `.` — the build context (sends all files in current dir to Docker)

Watch for output ending in:
```
Successfully built <image_id>
Successfully tagged anomaly-api:latest
```

---

## Step 3 — Run and Test the Container Locally

```bash
docker run -p 8000:8000 \
  -e MODEL_URI="models:/skab-anomaly-detector/Staging" \
  anomaly-api:latest
```

What's happening:
- `-p 8000:8000` — maps port 8000 on your laptop to port 8000 inside the container
- `-e MODEL_URI="..."` — passes an environment variable into the container
- `anomaly-api:latest` — which image to run

Test it (same as Phase 3):
```bash
curl http://localhost:8000/health
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"sensor_value": 1.25, "rolling_mean_5": 1.18, "rolling_std_5": 0.06, "rolling_max_5": 1.31, "rolling_min_5": 1.10}'
```

Stop the container: press `Ctrl+C`

---

## Step 4 — Create Azure Container Registry (ACR)

```bash
# Login to Azure
az login

# Create resource group (if not already done)
az group create --name skab-mlops-rg --location centralindia

# Create ACR (name must be globally unique, lowercase, no hyphens)
az acr create \
  --resource-group skab-mlops-rg \
  --name skabmlopsacr \
  --sku Basic

# Login to ACR
az acr login --name skabmlopsacr
```

---

## Step 5 — Tag and Push Image to ACR

```bash
# Tag the local image with the ACR URL
docker tag anomaly-api:latest skabmlopsacr.azurecr.io/anomaly-api:latest

# Push to ACR
docker push skabmlopsacr.azurecr.io/anomaly-api:latest
```

Verify it's there:
```bash
az acr repository list --name skabmlopsacr --output table
# Should show: anomaly-api
```

---

## Checkpoint ✅

By the end of Phase 4 you should have:
- [ ] `deployment/Dockerfile` created
- [ ] Image built locally: `anomaly-api:latest`
- [ ] Container running and responding to requests locally
- [ ] ACR created: `skabmlopsacr`
- [ ] Image pushed to ACR: `skabmlopsacr.azurecr.io/anomaly-api:latest`

**Next:** [Phase 5 — CI/CD with GitHub Actions](05_cicd_github_actions.md)
