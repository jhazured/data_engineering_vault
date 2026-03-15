
#pyspark #mlops #mlflow #model-registry #feature-store #model-serving #cicd #monitoring #drift

## Overview

MLOps (Machine Learning Operations) patterns for PySpark workloads: experiment tracking with MLflow, model versioning and registry, model serving strategies, feature stores, A/B testing, model monitoring, and CI/CD pipelines for ML. These practices bridge the gap between model development and production deployment. See also [[PySpark Core Concepts]], [[PySpark MLlib Patterns]], [[PySpark Advanced Features]], [[Databricks & Delta Lake]], and [[Databricks Modern Patterns (2025)]].

---

## MLflow Integration

### Tracking Experiments

MLflow Tracking records parameters, metrics, and artifacts for each training run.

```python
import mlflow
import mlflow.spark
from pyspark.ml.classification import RandomForestClassifier
from pyspark.ml.evaluation import BinaryClassificationEvaluator

# Set the tracking URI (Databricks sets this automatically)
mlflow.set_tracking_uri("http://mlflow-server:5000")
mlflow.set_experiment("/experiments/churn-prediction")

with mlflow.start_run(run_name="rf_baseline_v1") as run:
    # Log parameters
    num_trees = 100
    max_depth = 10
    mlflow.log_param("num_trees", num_trees)
    mlflow.log_param("max_depth", max_depth)
    mlflow.log_param("training_rows", train_df.count())
    mlflow.log_param("feature_count", len(feature_cols))

    # Train model
    rf = RandomForestClassifier(
        featuresCol="features", labelCol="churned",
        numTrees=num_trees, maxDepth=max_depth, seed=42
    )
    model = rf.fit(train_df)

    # Evaluate
    predictions = model.transform(test_df)
    evaluator = BinaryClassificationEvaluator(labelCol="churned")
    auc = evaluator.evaluate(predictions, {evaluator.metricName: "areaUnderROC"})

    # Log metrics
    mlflow.log_metric("auc_roc", auc)
    mlflow.log_metric("test_rows", test_df.count())

    # Log the Spark ML model
    mlflow.spark.log_model(model, "spark-model")

    # Log additional artifacts
    importance_pdf = feature_importance_df(model, feature_cols, spark).toPandas()
    importance_pdf.to_csv("/tmp/feature_importances.csv", index=False)
    mlflow.log_artifact("/tmp/feature_importances.csv")

    print(f"Run ID: {run.info.run_id}")
    print(f"AUC: {auc:.4f}")
```

### Logging PySpark Pipeline Models

```python
from pyspark.ml import Pipeline

with mlflow.start_run(run_name="full_pipeline_v2"):
    pipeline = Pipeline(stages=[
        imputer, indexer, encoder, assembler, scaler, classifier
    ])

    pipeline_model = pipeline.fit(train_df)

    # Log the entire pipeline model
    mlflow.spark.log_model(
        pipeline_model,
        "pipeline-model",
        registered_model_name="churn-predictor"  # auto-register
    )

    # Log pipeline stage names for transparency
    stage_names = [type(s).__name__ for s in pipeline.getStages()]
    mlflow.log_param("pipeline_stages", str(stage_names))
```

### Comparing Experiments

```python
# Query runs programmatically
import mlflow

experiment = mlflow.get_experiment_by_name("/experiments/churn-prediction")
runs = mlflow.search_runs(
    experiment_ids=[experiment.experiment_id],
    filter_string="metrics.auc_roc > 0.8",
    order_by=["metrics.auc_roc DESC"],
    max_results=10
)

print(runs[["run_id", "params.num_trees", "params.max_depth", "metrics.auc_roc"]])
```

---

## Model Registry

### Model Versioning Workflow

```python
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Register a model from a run
model_uri = f"runs:/{run_id}/spark-model"
registered = mlflow.register_model(model_uri, "churn-predictor")
print(f"Version: {registered.version}")

# Add description and tags
client.update_model_version(
    name="churn-predictor",
    version=registered.version,
    description="Random Forest baseline with 15 features, AUC 0.87"
)
client.set_model_version_tag(
    name="churn-predictor",
    version=registered.version,
    key="validation_status",
    value="pending"
)
```

### Stage Transitions

```python
# Transition through stages: None -> Staging -> Production -> Archived

# Promote to Staging
client.transition_model_version_stage(
    name="churn-predictor",
    version=3,
    stage="Staging",
    archive_existing_versions=False
)

# After validation, promote to Production
client.transition_model_version_stage(
    name="churn-predictor",
    version=3,
    stage="Production",
    archive_existing_versions=True  # auto-archive previous Production version
)

# Load a model by stage
staging_model = mlflow.spark.load_model("models:/churn-predictor/Staging")
prod_model = mlflow.spark.load_model("models:/churn-predictor/Production")

# Load a specific version
v2_model = mlflow.spark.load_model("models:/churn-predictor/2")
```

### Aliases (MLflow 2.x / Databricks Unity Catalog)

```python
# Aliases replace stages in newer MLflow versions
client.set_registered_model_alias("churn-predictor", "champion", version=3)
client.set_registered_model_alias("churn-predictor", "challenger", version=4)

# Load by alias
champion = mlflow.spark.load_model("models:/churn-predictor@champion")
challenger = mlflow.spark.load_model("models:/churn-predictor@challenger")
```

---

## Model Serving

### Batch Scoring With PySpark

```python
def batch_score(model_name, input_path, output_path, stage="Production"):
    """Load a registered model and score a dataset in batch."""
    model = mlflow.spark.load_model(f"models:/{model_name}/{stage}")
    input_df = spark.read.parquet(input_path)
    predictions = model.transform(input_df)

    (
        predictions
        .select("customer_id", "prediction", "probability")
        .write.mode("overwrite")
        .parquet(output_path)
    )
    return predictions.count()

scored = batch_score(
    "churn-predictor",
    "/mnt/data/customers/latest/",
    "/mnt/predictions/churn/latest/"
)
print(f"Scored {scored} records")
```

### Databricks Model Serving

```python
# Databricks Model Serving provides REST API endpoints for registered models
# Enable via UI or API

import requests
import json

def score_via_endpoint(endpoint_url, records, token):
    """Score records via a Databricks Model Serving endpoint."""
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    }
    payload = {"dataframe_records": records}
    response = requests.post(endpoint_url, headers=headers, json=payload)
    response.raise_for_status()
    return response.json()

# Example call
results = score_via_endpoint(
    "https://my-workspace.databricks.com/serving-endpoints/churn-predictor/invocations",
    records=[{"age": 35, "income": 75000, "tenure_months": 24}],
    token="dapi_xxxxx"
)
```

### SageMaker Deployment

```python
import mlflow.sagemaker

# Deploy an MLflow model to SageMaker
mlflow.sagemaker.deploy(
    app_name="churn-predictor",
    model_uri="models:/churn-predictor/Production",
    region_name="eu-west-1",
    mode="replace",  # also "create" or "add"
    instance_type="ml.m5.xlarge",
    instance_count=1,
    image_url="<ecr-image-uri>",  # custom container if needed
)

# Score via SageMaker endpoint
import boto3

runtime = boto3.client("sagemaker-runtime", region_name="eu-west-1")
response = runtime.invoke_endpoint(
    EndpointName="churn-predictor",
    ContentType="application/json",
    Body=json.dumps({"columns": ["age", "income"], "data": [[35, 75000]]})
)
prediction = json.loads(response["Body"].read())
```

---

## Feature Stores

### Databricks Feature Store

```python
from databricks.feature_store import FeatureStoreClient

fs = FeatureStoreClient()

# Create a feature table
customer_features = (
    spark.read.parquet("/mnt/data/customers/")
    .select(
        "customer_id",
        "avg_order_value",
        "total_orders",
        "days_since_last_order",
        "lifetime_value",
    )
)

fs.create_table(
    name="main.features.customer_features",
    primary_keys=["customer_id"],
    df=customer_features,
    description="Aggregated customer features for ML models"
)

# Update features (write new values)
updated_features = compute_latest_features()
fs.write_table(
    name="main.features.customer_features",
    df=updated_features,
    mode="merge"  # also "overwrite"
)

# Training: look up features by key
from databricks.feature_store import FeatureLookup

training_set = fs.create_training_set(
    df=labels_df,  # must have customer_id and label columns
    feature_lookups=[
        FeatureLookup(
            table_name="main.features.customer_features",
            lookup_key="customer_id"
        ),
        FeatureLookup(
            table_name="main.features.product_features",
            lookup_key="product_id"
        ),
    ],
    label="churned",
    exclude_columns=["customer_id", "product_id"]
)

training_df = training_set.load_df()

# Log model with feature store lineage
fs.log_model(
    model=pipeline_model,
    artifact_path="model",
    flavor=mlflow.spark,
    training_set=training_set,
    registered_model_name="churn-predictor-fs"
)
```

### Feast Integration

```python
from feast import FeatureStore
from datetime import datetime

# Feast connects to an offline store (e.g. Parquet/BigQuery) and an online store (e.g. Redis)
store = FeatureStore(repo_path="/path/to/feast_repo")

# Retrieve historical features for training (point-in-time join)
entity_df = spark.read.parquet("/mnt/data/training_entities/").toPandas()

training_features = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "customer_features:avg_order_value",
        "customer_features:total_orders",
        "customer_features:days_since_last_order",
    ],
).to_df()

# Materialise features to the online store for real-time serving
store.materialize(
    start_date=datetime(2025, 1, 1),
    end_date=datetime.now()
)

# Online serving (low-latency lookup)
online_features = store.get_online_features(
    features=[
        "customer_features:avg_order_value",
        "customer_features:total_orders",
    ],
    entity_rows=[{"customer_id": "C001"}, {"customer_id": "C002"}],
).to_dict()
```

---

## A/B Testing Patterns

### Champion/Challenger Scoring

```python
import random
from pyspark.sql import functions as F

def ab_score(df, champion_model, challenger_model, traffic_split=0.1, seed=42):
    """Score with champion and challenger models based on traffic split."""
    # Assign each row to a variant deterministically
    df_with_variant = df.withColumn(
        "variant",
        F.when(F.rand(seed) < traffic_split, F.lit("challenger"))
         .otherwise(F.lit("champion"))
    )

    champion_df = champion_model.transform(
        df_with_variant.filter(F.col("variant") == "champion")
    )
    challenger_df = challenger_model.transform(
        df_with_variant.filter(F.col("variant") == "challenger")
    )

    return champion_df.union(challenger_df)

champion = mlflow.spark.load_model("models:/churn-predictor@champion")
challenger = mlflow.spark.load_model("models:/churn-predictor@challenger")

scored = ab_score(input_df, champion, challenger, traffic_split=0.2)
scored.groupBy("variant").agg(
    F.avg("prediction").alias("avg_prediction"),
    F.count("*").alias("count")
).show()
```

### Logging A/B Results For Analysis

```python
def log_ab_metrics(scored_df, actuals_df, experiment_name):
    """Log per-variant metrics for A/B comparison."""
    results = (
        scored_df
        .join(actuals_df, "customer_id")
        .groupBy("variant")
        .agg(
            F.avg(F.when(F.col("prediction") == F.col("actual"), 1).otherwise(0)).alias("accuracy"),
            F.count("*").alias("sample_size"),
        )
    )

    for row in results.collect():
        with mlflow.start_run(run_name=f"ab_test_{row['variant']}",
                              experiment_id=mlflow.get_experiment_by_name(experiment_name).experiment_id):
            mlflow.log_metric("accuracy", row["accuracy"])
            mlflow.log_metric("sample_size", row["sample_size"])
            mlflow.log_param("variant", row["variant"])
```

---

## Model Monitoring

### Data Drift Detection

```python
from pyspark.sql import functions as F

def compute_feature_stats(df, feature_cols):
    """Compute summary statistics for drift comparison."""
    stats = {}
    for col in feature_cols:
        agg_result = df.select(
            F.mean(col).alias("mean"),
            F.stddev(col).alias("stddev"),
            F.min(col).alias("min"),
            F.max(col).alias("max"),
            F.expr(f"percentile_approx({col}, 0.5)").alias("median"),
            F.count(F.when(F.col(col).isNull(), 1)).alias("null_count"),
        ).collect()[0]
        stats[col] = agg_result.asDict()
    return stats

def detect_drift(baseline_stats, current_stats, threshold=2.0):
    """Detect drift by comparing z-scores of current means against baseline."""
    alerts = []
    for feature, baseline in baseline_stats.items():
        current = current_stats.get(feature)
        if current is None:
            alerts.append({"feature": feature, "issue": "missing_feature"})
            continue

        if baseline["stddev"] and baseline["stddev"] > 0:
            z_score = abs(current["mean"] - baseline["mean"]) / baseline["stddev"]
            if z_score > threshold:
                alerts.append({
                    "feature": feature,
                    "issue": "mean_drift",
                    "z_score": z_score,
                    "baseline_mean": baseline["mean"],
                    "current_mean": current["mean"],
                })

        # Check for null rate changes
        if current["null_count"] > baseline["null_count"] * 2:
            alerts.append({
                "feature": feature,
                "issue": "null_rate_increase",
                "baseline_nulls": baseline["null_count"],
                "current_nulls": current["null_count"],
            })

    return alerts

# Usage
baseline = compute_feature_stats(training_df, feature_cols)
current = compute_feature_stats(production_df, feature_cols)
drift_alerts = detect_drift(baseline, current, threshold=2.0)

for alert in drift_alerts:
    print(f"DRIFT ALERT: {alert}")
```

### Prediction Drift Monitoring

```python
def monitor_prediction_distribution(historical_predictions, current_predictions,
                                     label_col="prediction"):
    """Compare prediction distributions between historical and current batches."""
    hist_dist = (
        historical_predictions
        .groupBy(label_col)
        .agg((F.count("*") / historical_predictions.count()).alias("hist_ratio"))
    )
    curr_dist = (
        current_predictions
        .groupBy(label_col)
        .agg((F.count("*") / current_predictions.count()).alias("curr_ratio"))
    )

    comparison = hist_dist.join(curr_dist, label_col, "full_outer").fillna(0)
    comparison = comparison.withColumn(
        "ratio_change",
        F.abs(F.col("curr_ratio") - F.col("hist_ratio"))
    )

    comparison.show()
    return comparison

def log_monitoring_metrics(drift_alerts, prediction_comparison, run_date):
    """Log monitoring results to MLflow for dashboarding."""
    with mlflow.start_run(run_name=f"monitoring_{run_date}"):
        mlflow.log_metric("drift_alert_count", len(drift_alerts))
        mlflow.log_metric("features_monitored", len(feature_cols))

        # Save alerts as artifact
        import json
        with open("/tmp/drift_alerts.json", "w") as f:
            json.dump(drift_alerts, f, indent=2, default=str)
        mlflow.log_artifact("/tmp/drift_alerts.json")
```

### Performance Decay Detection

```python
def check_model_performance(predictions_with_actuals, evaluator, threshold=0.05):
    """Alert if model performance drops below baseline minus threshold."""
    current_metric = evaluator.evaluate(predictions_with_actuals)

    # Load baseline metric from model registry
    client = MlflowClient()
    prod_version = client.get_latest_versions("churn-predictor", stages=["Production"])[0]
    baseline_run = client.get_run(prod_version.run_id)
    baseline_metric = float(baseline_run.data.metrics.get("auc_roc", 0))

    degradation = baseline_metric - current_metric
    if degradation > threshold:
        print(f"ALERT: Model performance degraded by {degradation:.4f}")
        print(f"Baseline AUC: {baseline_metric:.4f}, Current AUC: {current_metric:.4f}")
        return True
    return False
```

---

## CI/CD For ML Pipelines

### Train-Test-Deploy Pipeline

```python
# Typically orchestrated by Airflow, Databricks Workflows, or GitHub Actions
# Each stage is a separate task/job

# Stage 1: Data Validation
def validate_training_data(data_path):
    """Validate training data before model training."""
    df = spark.read.parquet(data_path)
    assert df.count() > 1000, f"Insufficient training data: {df.count()} rows"
    assert df.filter(F.col("label").isNull()).count() == 0, "Null labels found"

    # Check for data drift against reference
    baseline = compute_feature_stats(reference_df, feature_cols)
    current = compute_feature_stats(df, feature_cols)
    alerts = detect_drift(baseline, current)
    assert len(alerts) == 0, f"Data drift detected: {alerts}"

    return df

# Stage 2: Train and Evaluate
def train_and_evaluate(train_df, test_df, experiment_name):
    """Train model and log to MLflow."""
    mlflow.set_experiment(experiment_name)

    with mlflow.start_run() as run:
        model = pipeline.fit(train_df)
        predictions = model.transform(test_df)

        auc = evaluator.evaluate(predictions)
        mlflow.log_metric("auc_roc", auc)
        mlflow.spark.log_model(model, "model", registered_model_name="churn-predictor")

        return run.info.run_id, auc

# Stage 3: Model Validation Gate
def validate_model(run_id, min_auc=0.80):
    """Check model meets minimum quality bar before promotion."""
    client = MlflowClient()
    run = client.get_run(run_id)
    auc = float(run.data.metrics["auc_roc"])

    if auc < min_auc:
        raise ValueError(f"Model AUC {auc:.4f} below threshold {min_auc}")

    # Compare against current production model
    prod_versions = client.get_latest_versions("churn-predictor", stages=["Production"])
    if prod_versions:
        prod_run = client.get_run(prod_versions[0].run_id)
        prod_auc = float(prod_run.data.metrics["auc_roc"])
        if auc < prod_auc:
            print(f"WARNING: New model AUC {auc:.4f} < production AUC {prod_auc:.4f}")
            return False

    return True

# Stage 4: Promote to Production
def promote_model(model_name, version):
    """Promote a validated model to Production stage."""
    client = MlflowClient()
    client.transition_model_version_stage(
        name=model_name,
        version=version,
        stage="Production",
        archive_existing_versions=True
    )
    print(f"Model {model_name} v{version} promoted to Production")
```

### GitHub Actions Workflow (Example)

```yaml
# .github/workflows/ml_pipeline.yml
name: ML Training Pipeline

on:
  schedule:
    - cron: '0 6 * * 1'  # Weekly retrain
  workflow_dispatch:

jobs:
  train:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - name: Validate Data
        run: |
          databricks jobs run-now --job-id 123 \
            --notebook-params '{"stage": "validate"}'

      - name: Train Model
        run: |
          databricks jobs run-now --job-id 123 \
            --notebook-params '{"stage": "train"}'

      - name: Evaluate and Gate
        run: |
          databricks jobs run-now --job-id 123 \
            --notebook-params '{"stage": "evaluate"}'

      - name: Promote Model
        if: success()
        run: |
          databricks jobs run-now --job-id 123 \
            --notebook-params '{"stage": "promote"}'
```

### Databricks Workflows Orchestration

```python
# Define a multi-task job via the Databricks SDK
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.jobs import (
    Task, NotebookTask, TaskDependency, JobCluster
)

w = WorkspaceClient()

w.jobs.create(
    name="ml-retrain-pipeline",
    tasks=[
        Task(
            task_key="validate_data",
            notebook_task=NotebookTask(
                notebook_path="/Repos/ml/notebooks/01_validate_data"
            ),
        ),
        Task(
            task_key="train_model",
            depends_on=[TaskDependency(task_key="validate_data")],
            notebook_task=NotebookTask(
                notebook_path="/Repos/ml/notebooks/02_train_model"
            ),
        ),
        Task(
            task_key="evaluate_gate",
            depends_on=[TaskDependency(task_key="train_model")],
            notebook_task=NotebookTask(
                notebook_path="/Repos/ml/notebooks/03_evaluate_gate"
            ),
        ),
        Task(
            task_key="promote_model",
            depends_on=[TaskDependency(task_key="evaluate_gate")],
            notebook_task=NotebookTask(
                notebook_path="/Repos/ml/notebooks/04_promote_model"
            ),
        ),
    ],
    schedule={
        "quartz_cron_expression": "0 0 6 ? * MON",
        "timezone_id": "Europe/London",
    },
)
```

---

## End-To-End MLOps Architecture

```text
+-------------------+     +-------------------+     +-------------------+
|  Feature Store    |     |  Training Pipeline |     |  Model Registry   |
|  (Databricks/    |---->|  (Spark + MLflow)  |---->|  (MLflow)         |
|   Feast)          |     |                   |     |  Staging/Prod     |
+-------------------+     +-------------------+     +-------------------+
        ^                         |                         |
        |                         v                         v
+-------------------+     +-------------------+     +-------------------+
|  Data Sources     |     |  Experiment        |     |  Model Serving    |
|  (Delta Lake,     |     |  Tracking          |     |  (Batch / REST)   |
|   S3, etc.)       |     |  (MLflow UI)       |     |                   |
+-------------------+     +-------------------+     +-------------------+
                                                          |
                                                          v
                                                  +-------------------+
                                                  |  Monitoring       |
                                                  |  (Drift, Perf,    |
                                                  |   A/B results)    |
                                                  +-------------------+
```

---

## Related Notes

- [[PySpark Core Concepts]] — SparkSession, DataFrame fundamentals
- [[PySpark MLlib Patterns]] — ML Pipeline API, model training details
- [[PySpark Advanced Features]] — broader advanced feature coverage
- [[PySpark Performance Optimization]] — tuning Spark for ML workloads
- [[Databricks & Delta Lake]] — managed platform for MLOps
- [[Databricks Modern Patterns (2025)]] — Unity Catalog model governance
- [[PySpark Production Engineering]] — production deployment patterns
