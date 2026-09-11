# SKAB Anomaly Detection — MLOps Pipeline

End-to-end MLOps project: train anomaly detection models on industrial sensor data, track experiments with MLflow, serve via FastAPI, containerise with Docker, and deploy to Azure Kubernetes Service (AKS) through a GitHub Actions CI/CD pipeline.

## Architecture

```
SKAB Dataset (CSV)
     ↓
Databricks (EDA + Feature Engineering)
     ↓
MLflow Experiment Tracking (Isolation Forest · One-Class SVM · Autoencoder)
     ↓
MLflow Model Registry → best model → Staging
     ↓
GitHub (code + notebooks)
     ↓
GitHub Actions CI/CD
     ↓
Docker image → Azure Container Registry (ACR)
     ↓
Azure Kubernetes Service (AKS) — 2 replicas
     ↓
REST endpoint: POST /predict → anomaly score + label
```

## Tech Stack

| Layer | Tool |
|---|---|
| Experimentation | Azure Databricks |
| Experiment tracking | MLflow (built into Databricks) |
| Model registry | MLflow Model Registry |
| Serving | FastAPI + Uvicorn |
| Containerisation | Docker |
| Container registry | Azure Container Registry (ACR) |
| Orchestration | Azure Kubernetes Service (AKS) |
| CI/CD | GitHub Actions |

## Dataset

**SKAB — Skoltech Anomaly Benchmark**
Time-series sensor data from a water pump testbed. Pre-labelled anomalies. Publicly available.
Source: https://github.com/waico/SKAB

## Project Phases

| Phase | Doc | Status |
|---|---|---|
| 1 — Databricks setup | [docs/01_databricks_setup.md](docs/01_databricks_setup.md) | 🔲 |
| 2 — MLflow tracking | [docs/02_mlflow_tracking.md](docs/02_mlflow_tracking.md) | 🔲 |
| 3 — FastAPI serving | [docs/03_fastapi_serving.md](docs/03_fastapi_serving.md) | 🔲 |
| 4 — Docker packaging | [docs/04_docker_packaging.md](docs/04_docker_packaging.md) | 🔲 |
| 5 — CI/CD pipeline | [docs/05_cicd_github_actions.md](docs/05_cicd_github_actions.md) | 🔲 |
| 6 — Kubernetes + AKS | [docs/06_kubernetes_aks.md](docs/06_kubernetes_aks.md) | 🔲 |
| 7 — Interview notes | [docs/07_interview_talking_points.md](docs/07_interview_talking_points.md) | 🔲 |

## Repo Structure

```
skab-anomaly-mlops/
├── README.md
├── requirements.txt
├── docs/
│   ├── 01_databricks_setup.md
│   ├── 02_mlflow_tracking.md
│   ├── 03_fastapi_serving.md
│   ├── 04_docker_packaging.md
│   ├── 05_cicd_github_actions.md
│   ├── 06_kubernetes_aks.md
│   └── 07_interview_talking_points.md
├── notebooks/
│   └── 01_train_mlflow.ipynb       ← exported from Databricks
├── src/
│   ├── app.py                      ← FastAPI app
│   ├── model_loader.py             ← loads model from MLflow registry
│   └── schemas.py                  ← Pydantic input/output models
└── deployment/
    ├── Dockerfile
    ├── k8s-deployment.yaml
    └── k8s-service.yaml
```
