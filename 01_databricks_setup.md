# Phase 1 — Databricks Setup & Data Exploration

## Goal
Set up an Azure Databricks workspace, load the SKAB dataset, and explore the data so we understand what we're modelling.

---

## Step 1 — Create Azure Databricks Workspace

1. Go to [portal.azure.com](https://portal.azure.com)
2. Search → **Azure Databricks** → Create
3. Fill in:
   - Resource group: `skab-mlops-rg` (create new)
   - Workspace name: `skab-mlops-ws`
   - Region: pick the closest to you (e.g. Central India)
   - Pricing tier: **Standard** (free to start)
4. Click **Review + Create** → **Create**
5. Once deployed, click **Launch Workspace**

---

## Step 2 — Create a Cluster

1. In Databricks, go to **Compute** → **Create Cluster**
2. Settings:
   - Cluster name: `skab-cluster`
   - Cluster mode: **Single Node**
   - Databricks Runtime: **13.x ML** (includes MLflow pre-installed)
   - Node type: `Standard_DS3_v2` (4 cores, 14 GB RAM — cost-effective)
   - Terminate after: **30 minutes** of inactivity (saves cost)
3. Click **Create Cluster** and wait ~3 minutes

---

## Step 3 — Install Libraries on the Cluster

1. Click your cluster → **Libraries** → **Install New**
2. Source: **PyPI**
3. Install these one by one:
   - `scikit-learn`
   - `pandas`
   - `numpy`
   - `matplotlib`
   - `seaborn`

> MLflow is already included in the ML runtime — no need to install it separately.

---

## Step 4 — Download the SKAB Dataset

1. Go to: https://github.com/waico/SKAB
2. Click **Code** → **Download ZIP**
3. Extract it. You'll see folders like `valve1/`, `valve2/`, `other/`
4. We'll use `valve1/valve1.csv` to start

**Upload to Databricks:**
1. In Databricks, go to **Data** → **Add Data** → **Upload File**
2. Upload `valve1.csv`
3. Note the DBFS path shown — it will look like `/FileStore/tables/valve1.csv`

---

## Step 5 — Create a Notebook and Load Data

1. Go to **Workspace** → **Create** → **Notebook**
2. Name: `01_train_mlflow`
3. Language: **Python**
4. Attach to: `skab-cluster`

Paste this into the first cell and run:

```python
import pandas as pd
import matplotlib.pyplot as plt

# Load dataset
df = pd.read_csv("/dbfs/FileStore/tables/valve1.csv", sep=";", parse_dates=["datetime"], index_col="datetime")

print("Shape:", df.shape)
print("\nColumns:", df.columns.tolist())
print("\nFirst 5 rows:")
display(df.head())
```

Expected output: ~18,000 rows, multiple sensor columns, plus `anomaly` and `changepoint` label columns.

---

## Step 6 — Explore the Data

Add a new cell:

```python
# Check anomaly distribution
print("Anomaly label counts:")
print(df["anomaly"].value_counts())
print(f"\nAnomaly rate: {df['anomaly'].mean():.2%}")

# Plot one sensor over time
sensor_col = df.columns[0]   # first sensor column
fig, axes = plt.subplots(2, 1, figsize=(14, 6), sharex=True)

axes[0].plot(df.index, df[sensor_col], linewidth=0.6, color="steelblue")
axes[0].set_title(f"Sensor: {sensor_col}")

axes[1].fill_between(df.index, df["anomaly"], alpha=0.4, color="red")
axes[1].set_title("Anomaly labels (red = anomaly)")

plt.tight_layout()
plt.show()
```

---

## Step 7 — Feature Engineering

Add a new cell:

```python
# Pick the primary sensor column for modelling
sensor_col = df.columns[0]

# Rolling statistics — these are your features
# Mirrors the same pattern used in production pipelines
df["rolling_mean_5"]  = df[sensor_col].rolling(window=5).mean()
df["rolling_std_5"]   = df[sensor_col].rolling(window=5).std()
df["rolling_max_5"]   = df[sensor_col].rolling(window=5).max()
df["rolling_min_5"]   = df[sensor_col].rolling(window=5).min()

# Drop rows with NaN (first 4 rows after rolling)
df.dropna(inplace=True)

# Define feature set and labels
feature_cols = [sensor_col, "rolling_mean_5", "rolling_std_5", "rolling_max_5", "rolling_min_5"]
X = df[feature_cols]
y = df["anomaly"]   # used only for evaluation — models train unsupervised

print("Feature matrix shape:", X.shape)
print("Features:", feature_cols)
display(X.head())
```

---

## Checkpoint ✅

By the end of Phase 1 you should have:
- [ ] Databricks workspace running
- [ ] Cluster created and libraries installed
- [ ] SKAB data loaded and visible in a notebook
- [ ] Feature matrix `X` and label vector `y` defined

**Next:** [Phase 2 — MLflow Experiment Tracking](02_mlflow_tracking.md)
