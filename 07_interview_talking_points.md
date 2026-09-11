# Interview Talking Points — Project 1

Use these when asked about this project or the technologies involved.

---

## The Project Story (30-second version)

> "I built an end-to-end MLOps pipeline for industrial anomaly detection using the SKAB benchmark dataset — it mirrors the production system I've maintained for 5+ years. I trained three models in Databricks with MLflow tracking, registered the best one, wrapped it in a FastAPI service, containerised it with Docker, and deployed it to AKS through a GitHub Actions CI/CD pipeline. The whole flow — from a code push to a live rolling update — takes about 3 minutes."

---

## On MLflow

**Q: How did you use MLflow in this project?**
> "I used MLflow for two things: experiment tracking and model registry. During training, I logged parameters, metrics, and the model artifact for each run. Then I used the MLflow UI to compare F1 scores across all three models, registered the winner to the Model Registry, and promoted it to Staging. The FastAPI app then loads the model directly from the registry using its URI — so swapping models is just a registry promotion, no code change needed."

---

## On FastAPI

**Q: Why FastAPI over Flask or Django?**
> "FastAPI gives you automatic input validation via Pydantic schemas, auto-generated Swagger documentation, and async support out of the box. For an ML serving endpoint, this matters — I can define exactly what shape the input JSON must be, and FastAPI rejects malformed requests before they ever reach the model. Flask would require me to write that validation manually."

---

## On Docker

**Q: What does Docker solve in an MLOps context?**
> "It eliminates environment inconsistency. In my production work, I've seen models that run fine in development fail in production because of library version mismatches. With Docker, the image that passes CI is byte-for-byte identical to what runs in AKS — the model, its dependencies, the Python version, everything is locked."

---

## On Kubernetes / AKS

**Q: Walk me through how you deployed to AKS.**
> "I wrote two manifests: a Deployment and a Service. The Deployment specifies 2 replicas, resource limits (256Mi RAM, 250m CPU per pod), and a readinessProbe that calls the /health endpoint — Kubernetes won't route traffic to a pod until that probe passes. The Service is a LoadBalancer type, which tells Azure to provision a public IP and distribute traffic across the healthy pods. When a new image is pushed through CI/CD, Kubernetes does a rolling update — it brings up new pods first, waits for them to pass the readiness check, then terminates the old ones. Zero downtime."

---

## On CI/CD

**Q: What does your CI/CD pipeline do?**
> "On every push to main, GitHub Actions: logs into Azure, builds the Docker image tagged with the commit SHA, pushes it to ACR, updates the image reference in the K8s deployment manifest, applies it to AKS, and waits for the rollout to complete — if the new pods don't become ready within 2 minutes, the pipeline fails and the old version stays running. The commit SHA as image tag means every deployment is fully traceable back to the exact code commit."

---

## On the anomaly detection model itself

**Q: Why Isolation Forest over the other models?**
> "Isolation Forest tends to work well on high-dimensional tabular sensor data because it isolates anomalies by randomly partitioning the feature space — anomalies are isolated in fewer splits than normal points. One-Class SVM can be slower on larger datasets and more sensitive to the nu hyperparameter. The autoencoder approach is flexible but harder to tune without a validation set. I compared all three using F1 on the labelled SKAB data and picked the one with the best balance of precision and recall — which in practice is often Isolation Forest for this kind of time-series sensor anomaly scenario."

---

## Connecting to your production experience

**Q: How does this differ from your real production pipeline?**
> "Scale and platform. In production I'm running 7,000+ sensor groups on RapidMiner, with Hadoop-to-SQL-Server migration behind us. This portfolio project replicates the core logic — feature engineering from rolling statistics, unsupervised anomaly detection, Dev-QA-Prod release governance — but on a modern cloud-native stack: Databricks, MLflow, Docker, AKS. The production work gave me the domain understanding; this project demonstrates the infrastructure and deployment patterns the market currently expects."
