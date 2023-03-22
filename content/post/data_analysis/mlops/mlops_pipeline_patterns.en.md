---
title: "MLOps pipeline design patterns"
date: 2023-03-22T10:00:00+01:00
draft: false
tags:
  - "mlops"
  - "machine-learning"
  - "pipelines"
  - "kubeflow"
  - "mlflow"

categories:
  - "DataOps"

toc: true
description: "MLOps pipeline design patterns for implementing production ML systems, covering training, validation, deployment, and model monitoring."
---

## What Is MLOps and Why It Matters

MLOps is about deploying and maintaining ML models reliably and efficiently in production. It bridges data science experiments and production engineering. Without it, you hit the same problems repeatedly: models that work in notebooks but fail in production, no way to reproduce results, painful handoffs between teams, and zero visibility into how models perform once deployed.

The idea is simple: treat ML systems with the same rigor as software. Use version control, automated testing, continuous delivery, and monitoring. Just acknowledge that data and models introduce unique challenges.

## The ML Lifecycle

Before diving into patterns, understand the stages every ML system goes through:

1. **Data ingestion and validation** - Collect, clean, and validate input data
2. **Feature engineering** - Transform raw data into features the model can use
3. **Model training** - Run experiments, tune hyperparameters, pick algorithms
4. **Model evaluation** - Test model quality against held-out data and business metrics
5. **Model deployment** - Serve predictions in production (batch or real-time)
6. **Monitoring and feedback** - Track performance, detect drift, retrain when needed

Each stage has failure modes, and the patterns below help prevent them.

## Key design patterns

### Feature store

A feature store is a centralized repository for storing, sharing, and serving ML features. Instead of each team recomputing features from scratch, a feature store provides:

- **Consistency** between training and serving (avoiding training-serving skew).
- **Reusability** across teams and models.
- **Point-in-time correctness** for historical feature values.

Tools like **Feast**, **Tecton**, and **Hopsworks** implement this pattern. If you find multiple teams duplicating feature pipelines, a feature store is likely worth the investment.

### Model registry

A model registry acts as a versioned catalog for trained models. It stores model artifacts, metadata (hyperparameters, metrics, training data version), and lifecycle stage (staging, production, archived).

**MLflow Model Registry** is one of the most widely adopted solutions. It lets you promote models through stages with approval workflows and track lineage from experiment to production.

### CT/CI/CD for ML

Traditional CI/CD pipelines build and deploy code. ML pipelines need three loops:

- **Continuous Training (CT)** --- Automatically retrain models when data changes or performance degrades.
- **Continuous Integration (CI)** --- Validate not just code but also data schemas, feature expectations, and model quality thresholds.
- **Continuous Delivery (CD)** --- Deploy validated models to serving infrastructure automatically.

A typical pipeline trigger might be: new data lands in the data lake, CT kicks off retraining, CI runs validation tests, and CD pushes the model to production if all checks pass.

### A/B testing

A/B testing for models means routing a percentage of traffic to a new model while the rest continues hitting the current production model. You measure business metrics (conversion rate, click-through, revenue) rather than just ML metrics (accuracy, F1). This pattern is essential because a model that scores well offline can still perform poorly in production due to feedback loops, latency, or distribution differences.

### Shadow deployment

In shadow mode, the new model receives production traffic and generates predictions, but those predictions are **not** served to users. Instead, they are logged alongside the current model's predictions for offline comparison. This is a low-risk way to validate a model on real traffic before exposing it to users.

### Canary releases for models

Similar to canary deployments in software, you roll out a new model to a small fraction of traffic (say 5%), monitor key metrics, and gradually increase traffic if everything looks healthy. If metrics degrade, you roll back automatically. This combines well with A/B testing but focuses more on risk mitigation than experimentation.

## Tooling overview

| Tool | Primary Use | Key Strength |
|------|------------|--------------|
| **MLflow** | Experiment tracking, model registry | Flexible, vendor-neutral |
| **Kubeflow** | End-to-end ML pipelines on Kubernetes | Scalable, cloud-native |
| **DVC** | Data and model versioning | Git-like workflow for data |
| **Weights & Biases** | Experiment tracking, visualization | Excellent UI and collaboration |
| **Feast** | Feature store | Open-source, production-ready |
| **Seldon Core** | Model serving on Kubernetes | Advanced deployment strategies |

There is no single tool that covers everything. Most production setups combine several of these, choosing based on team expertise and infrastructure constraints.

## Example: MLflow experiment tracking

Here is a minimal example of tracking an experiment with MLflow:

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, f1_score
from sklearn.model_selection import train_test_split

# Start an MLflow run
with mlflow.start_run(run_name="rf-baseline"):
    # Log parameters
    n_estimators = 100
    max_depth = 10
    mlflow.log_param("n_estimators", n_estimators)
    mlflow.log_param("max_depth", max_depth)

    # Train model
    model = RandomForestClassifier(
        n_estimators=n_estimators,
        max_depth=max_depth,
        random_state=42
    )
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
    model.fit(X_train, y_train)

    # Log metrics
    predictions = model.predict(X_test)
    mlflow.log_metric("accuracy", accuracy_score(y_test, predictions))
    mlflow.log_metric("f1_score", f1_score(y_test, predictions, average="weighted"))

    # Log model artifact
    mlflow.sklearn.log_model(model, "random-forest-model")
```

Every run is tracked with its parameters, metrics, and artifacts, making it straightforward to compare experiments and reproduce results.

## Anti-patterns to avoid

**No versioning of data or models.** If you cannot reproduce a training run from six months ago, you have a problem. Version everything: code, data, configuration, and model artifacts.

**Training-serving skew.** When the feature computation logic differs between training and serving, predictions silently degrade. A feature store or shared feature computation library helps eliminate this.

**Manual deployment.** Copy-pasting model files to a server is a recipe for incidents. Automate deployment through pipelines with proper validation gates.

**Ignoring model monitoring.** Models degrade over time as input distributions shift. Without monitoring, you only discover this when a user complains or a business metric drops. Set up alerts for prediction distribution changes, latency, and data quality.

**Monolithic pipelines.** A single pipeline that does everything from data ingestion to model serving is fragile and hard to debug. Break pipelines into modular, independently testable stages.

**Over-engineering too early.** Not every ML project needs Kubeflow and a feature store on day one. Start simple, identify bottlenecks, and adopt patterns as the complexity of your system grows.

## MLOps maturity levels

Organizations typically progress through several maturity levels:

### Level 0: manual

- Models trained in notebooks.
- Manual deployment (file copy, manual API restart).
- No experiment tracking.
- No monitoring.

### Level 1: ML pipeline automation

- Automated training pipelines.
- Experiment tracking with tools like MLflow.
- Basic model validation before deployment.
- Some monitoring of model predictions.

### Level 2: CI/CD for ML

- Automated testing of data, features, and model quality.
- Continuous training triggered by data changes or schedule.
- Automated deployment with canary or shadow releases.
- Comprehensive monitoring with alerting and automated rollback.

### Level 3: Full MLOps

- Feature store for consistent feature management.
- Model registry with governance and approval workflows.
- A/B testing integrated into the deployment process.
- Data and model lineage tracked end-to-end.
- Self-healing pipelines that detect and respond to drift automatically.

Most teams are somewhere between Level 0 and Level 1. The goal is not to jump to Level 3 immediately but to progress incrementally, addressing the most painful bottlenecks first.

## Conclusion

MLOps is about applying engineering patterns to ML's unique challenges. Start with experiment tracking and basic automation, then add feature stores, model registries, and advanced deployment strategies as you scale. The key: treat models like first-class production artifacts. Version them, test them, monitor them, improve them.
