# Phase 2 — MLflow Experiment Tracking & Model Registry

## Goal
Train 3 anomaly detection models inside Databricks, log every run to MLflow, compare results, and register the best model to the MLflow Model Registry.

> **Prerequisite:** Complete [Phase 1](01_databricks_setup.md) — your notebook should have `X` and `y` defined.

---

## What is MLflow? (Quick mental model)

MLflow is a logbook for ML experiments. Every time you train a model:
- You open a **run** (like starting a new page in the logbook)
- You **log parameters** (what settings you used)
- You **log metrics** (how well it performed)
- You **log the model** (save the actual model object)

Later, you open the MLflow UI to compare all runs side by side and pick the winner.

---

## Step 1 — Set Up MLflow Experiment

Add a new cell in your Databricks notebook:

```python
import mlflow
import mlflow.sklearn

# Create (or reuse) a named experiment
mlflow.set_experiment("/skab-anomaly-detection")

print("MLflow tracking URI:", mlflow.get_tracking_uri())
```

---

## Step 2 — Train Model 1: Isolation Forest

```python
from sklearn.ensemble import IsolationForest
from sklearn.metrics import f1_score, precision_score, recall_score
import numpy as np

with mlflow.start_run(run_name="isolation_forest"):

    # --- Parameters ---
    contamination = 0.05
    random_state  = 42

    # --- Train ---
    model = IsolationForest(contamination=contamination, random_state=random_state)
    model.fit(X)

    # --- Predict ---
    raw_preds    = model.predict(X)           # returns 1 (normal) or -1 (anomaly)
    preds_binary = (raw_preds == -1).astype(int)
    scores       = model.score_samples(X)     # lower = more anomalous

    # --- Evaluate ---
    f1   = f1_score(y, preds_binary)
    prec = precision_score(y, preds_binary, zero_division=0)
    rec  = recall_score(y, preds_binary, zero_division=0)

    # --- Log to MLflow ---
    mlflow.log_param("model_type",    "IsolationForest")
    mlflow.log_param("contamination", contamination)
    mlflow.log_param("random_state",  random_state)

    mlflow.log_metric("f1_score",  f1)
    mlflow.log_metric("precision", prec)
    mlflow.log_metric("recall",    rec)

    mlflow.sklearn.log_model(model, artifact_path="model")

    print(f"Isolation Forest → F1: {f1:.3f} | Precision: {prec:.3f} | Recall: {rec:.3f}")
```

---

## Step 3 — Train Model 2: One-Class SVM

```python
from sklearn.svm import OneClassSVM

with mlflow.start_run(run_name="one_class_svm"):

    nu     = 0.05
    kernel = "rbf"

    model = OneClassSVM(nu=nu, kernel=kernel)
    model.fit(X)

    raw_preds    = model.predict(X)
    preds_binary = (raw_preds == -1).astype(int)

    f1   = f1_score(y, preds_binary)
    prec = precision_score(y, preds_binary, zero_division=0)
    rec  = recall_score(y, preds_binary, zero_division=0)

    mlflow.log_param("model_type", "OneClassSVM")
    mlflow.log_param("nu",         nu)
    mlflow.log_param("kernel",     kernel)

    mlflow.log_metric("f1_score",  f1)
    mlflow.log_metric("precision", prec)
    mlflow.log_metric("recall",    rec)

    mlflow.sklearn.log_model(model, artifact_path="model")

    print(f"One-Class SVM   → F1: {f1:.3f} | Precision: {prec:.3f} | Recall: {rec:.3f}")
```

---

## Step 4 — Train Model 3: Autoencoder (via reconstruction error)

```python
from sklearn.neural_network import MLPRegressor
from sklearn.preprocessing import StandardScaler

with mlflow.start_run(run_name="autoencoder_mlp"):

    hidden_layer = (32, 16, 32)
    max_iter     = 200

    # Scale features (important for neural networks)
    scaler  = StandardScaler()
    X_scaled = scaler.fit_transform(X)

    # Train MLP to reconstruct its own input (autoencoder behaviour)
    model = MLPRegressor(hidden_layer_sizes=hidden_layer, max_iter=max_iter, random_state=42)
    model.fit(X_scaled, X_scaled)

    # Reconstruction error = anomaly score
    X_reconstructed = model.predict(X_scaled)
    recon_error = np.mean((X_scaled - X_reconstructed) ** 2, axis=1)

    # Threshold: top `contamination` fraction = anomaly
    threshold    = np.percentile(recon_error, 95)
    preds_binary = (recon_error > threshold).astype(int)

    f1   = f1_score(y, preds_binary)
    prec = precision_score(y, preds_binary, zero_division=0)
    rec  = recall_score(y, preds_binary, zero_division=0)

    mlflow.log_param("model_type",   "Autoencoder_MLP")
    mlflow.log_param("hidden_layer", str(hidden_layer))
    mlflow.log_param("max_iter",     max_iter)
    mlflow.log_param("threshold_pct", 95)

    mlflow.log_metric("f1_score",  f1)
    mlflow.log_metric("precision", prec)
    mlflow.log_metric("recall",    rec)

    mlflow.sklearn.log_model(model, artifact_path="model")

    print(f"Autoencoder MLP → F1: {f1:.3f} | Precision: {prec:.3f} | Recall: {rec:.3f}")
```

---

## Step 5 — Compare Runs in MLflow UI

1. In Databricks left sidebar, click **Experiments**
2. Find `/skab-anomaly-detection`
3. You'll see all 3 runs listed with their metrics
4. Click **Compare** (select all 3 runs) → see a side-by-side chart

**Pick the model with the best F1 score.** F1 balances precision and recall — best single metric for imbalanced anomaly detection.

---

## Step 6 — Register the Best Model

Back in the notebook, register the winner:

```python
# Paste the run ID from the MLflow UI (click the run → copy the Run ID)
best_run_id = "PASTE_YOUR_RUN_ID_HERE"

model_uri = f"runs:/{best_run_id}/model"

registered = mlflow.register_model(
    model_uri=model_uri,
    name="skab-anomaly-detector"
)

print(f"Registered: {registered.name} — version {registered.version}")
```

**Promote to Staging:**
1. In MLflow UI → **Models** → `skab-anomaly-detector`
2. Click version 1 → **Stage** → **Transition to Staging**
3. Confirm

---

## Step 7 — Export Notebook from Databricks

1. File → **Export** → **IPython Notebook (.ipynb)**
2. Save the file as `01_train_mlflow.ipynb`
3. This goes into the `notebooks/` folder in your GitHub repo

---

## Checkpoint ✅

By the end of Phase 2 you should have:
- [ ] 3 MLflow runs logged under `/skab-anomaly-detection`
- [ ] Best model registered as `skab-anomaly-detector` version 1
- [ ] Model promoted to **Staging** in the registry
- [ ] Notebook exported as `.ipynb`

**Next:** [Phase 3 — FastAPI Serving](03_fastapi_serving.md)
