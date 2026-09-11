# Phase 3 — FastAPI Serving Layer

## Goal
Build a FastAPI application that loads the registered MLflow model and exposes it as a REST API with two endpoints: `/health` and `/predict`.

> **Prerequisite:** Complete [Phase 2](02_mlflow_tracking.md) — model registered in MLflow as `skab-anomaly-detector/Staging`.

---

## What is FastAPI? (Quick mental model)

FastAPI is a Python library that lets you turn a function into an HTTP endpoint. Think of it like this:

- You have a model that takes sensor readings and returns "ANOMALY" or "NORMAL"
- FastAPI wraps that in a web server so any system can call it over HTTP
- You send JSON in → you get JSON back
- It also auto-generates a test UI (Swagger) so you can try it in a browser

**The pattern you already know:** In your production work you *call* REST APIs for deployment automation. Here, you're *building* one.

---

## Step 1 — Set Up Local Project Structure

On your laptop, open a terminal and run:

```bash
# Navigate to where you cloned the repo
cd skab-anomaly-mlops

# Create the src folder
mkdir -p src

# Create the files we'll fill in
touch src/schemas.py
touch src/model_loader.py
touch src/app.py
touch requirements.txt
```

---

## Step 2 — Install Dependencies

```bash
pip install fastapi==0.111.0 uvicorn==0.29.0 mlflow==2.12.2 scikit-learn==1.4.2 numpy==1.26.4 pydantic==2.7.1
```

---

## Step 3 — Define Input/Output Schemas (`src/schemas.py`)

Schemas tell FastAPI exactly what JSON shape to expect as input and what to return as output. Pydantic validates this automatically — if a caller sends wrong data, FastAPI rejects it with a clear error message.

```python
# src/schemas.py
from pydantic import BaseModel

class SensorInput(BaseModel):
    sensor_value:    float
    rolling_mean_5:  float
    rolling_std_5:   float
    rolling_max_5:   float
    rolling_min_5:   float

class PredictionOutput(BaseModel):
    anomaly: bool
    score:   float
    label:   str   # "ANOMALY" or "NORMAL"
```

---

## Step 4 — Model Loader (`src/model_loader.py`)

This file loads the model from the MLflow registry. Separating it from `app.py` keeps things clean and makes it easy to swap models later.

```python
# src/model_loader.py
import mlflow.sklearn
import os

def load_model():
    """Load the registered model from MLflow Model Registry."""
    model_uri = os.getenv(
        "MODEL_URI",
        "models:/skab-anomaly-detector/Staging"   # default: Staging version
    )
    print(f"Loading model from: {model_uri}")
    return mlflow.sklearn.load_model(model_uri)
```

---

## Step 5 — FastAPI Application (`src/app.py`)

```python
# src/app.py
from fastapi import FastAPI
from src.schemas import SensorInput, PredictionOutput
from src.model_loader import load_model
import numpy as np

# --- App setup ---
app = FastAPI(
    title="SKAB Anomaly Detection API",
    description="Detects anomalies in industrial sensor readings.",
    version="1.0.0"
)

# Load model once when server starts (not on every request)
model = load_model()


# --- Endpoints ---

@app.get("/health")
def health():
    """Liveness check — Kubernetes uses this to know the pod is ready."""
    return {"status": "ok"}


@app.post("/predict", response_model=PredictionOutput)
def predict(data: SensorInput):
    """
    Accepts sensor feature values, returns anomaly prediction.

    Example request body:
    {
        "sensor_value": 1.25,
        "rolling_mean_5": 1.18,
        "rolling_std_5": 0.06,
        "rolling_max_5": 1.31,
        "rolling_min_5": 1.10
    }
    """
    features = np.array([[
        data.sensor_value,
        data.rolling_mean_5,
        data.rolling_std_5,
        data.rolling_max_5,
        data.rolling_min_5,
    ]])

    raw_pred = model.predict(features)[0]          # 1 = normal, -1 = anomaly
    score    = float(model.score_samples(features)[0])  # lower = more anomalous
    is_anomaly = raw_pred == -1

    return PredictionOutput(
        anomaly=is_anomaly,
        score=round(score, 4),
        label="ANOMALY" if is_anomaly else "NORMAL"
    )
```

---

## Step 6 — `requirements.txt`

```
fastapi==0.111.0
uvicorn==0.29.0
mlflow==2.12.2
scikit-learn==1.4.2
numpy==1.26.4
pydantic==2.7.1
```

---

## Step 7 — Run Locally and Test

**Start the server:**
```bash
# From the repo root
uvicorn src.app:app --reload --port 8000
```

You should see:
```
INFO: Uvicorn running on http://127.0.0.1:8000
INFO: Application startup complete.
```

**Open the Swagger UI:**
Go to http://localhost:8000/docs in your browser.
You'll see the interactive API — click `/predict` → **Try it out** → fill in values → **Execute**.

**Test with curl:**
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "sensor_value": 1.25,
    "rolling_mean_5": 1.18,
    "rolling_std_5": 0.06,
    "rolling_max_5": 1.31,
    "rolling_min_5": 1.10
  }'
```

Expected response:
```json
{
  "anomaly": false,
  "score": -0.4821,
  "label": "NORMAL"
}
```

---

## Checkpoint ✅

By the end of Phase 3 you should have:
- [ ] `src/schemas.py`, `src/model_loader.py`, `src/app.py` created
- [ ] Server running locally on port 8000
- [ ] `/health` returning `{"status": "ok"}`
- [ ] `/predict` returning a valid prediction
- [ ] Swagger UI visible at http://localhost:8000/docs

**Next:** [Phase 4 — Docker Packaging](04_docker_packaging.md)
