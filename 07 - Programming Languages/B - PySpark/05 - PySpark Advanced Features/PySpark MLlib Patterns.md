
#pyspark #mllib #machine-learning #pipeline #feature-engineering #classification #regression #clustering #tuning

## Overview

Comprehensive guide to PySpark's MLlib library for scalable machine learning. Covers the ML Pipeline API, feature engineering transformers, classification and regression models, clustering algorithms, model evaluation, hyperparameter tuning, and model persistence. These patterns enable data engineers to build reproducible, production-grade ML workflows on Spark. See also [[PySpark Core Concepts]], [[PySpark Advanced Features]], and [[PySpark Performance Optimization]].

---

## ML Pipeline API

### Core Abstractions

MLlib's Pipeline API is built on three core abstractions:

- **Transformer** — takes a DataFrame in, produces a DataFrame out (e.g. a trained model, a feature scaler)
- **Estimator** — takes a DataFrame in, produces a Transformer via `.fit()` (e.g. an untrained model, an unfitted scaler)
- **Pipeline** — chains multiple Estimators and Transformers into a single workflow

```python
from pyspark.ml import Pipeline, PipelineModel
from pyspark.ml.feature import VectorAssembler, StandardScaler
from pyspark.ml.classification import LogisticRegression

# A Pipeline is itself an Estimator
# Calling .fit() on it returns a PipelineModel (a Transformer)
pipeline = Pipeline(stages=[
    VectorAssembler(inputCols=["age", "income"], outputCol="raw_features"),
    StandardScaler(inputCol="raw_features", outputCol="features"),
    LogisticRegression(featuresCol="features", labelCol="label"),
])

# Fit the entire pipeline at once
model = pipeline.fit(train_df)   # returns PipelineModel

# Transform (predict) in a single call
predictions = model.transform(test_df)
```

### Accessing Individual Stages

```python
# Retrieve individual fitted stages from a PipelineModel
scaler_model = model.stages[1]       # fitted StandardScaler
lr_model = model.stages[2]           # fitted LogisticRegression

print(f"Coefficients: {lr_model.coefficients}")
print(f"Intercept: {lr_model.intercept}")
print(f"Scaler mean: {scaler_model.mean}")
```

### Custom Transformers

```python
from pyspark.ml import Transformer
from pyspark.ml.param.shared import HasInputCol, HasOutputCol
from pyspark import keyword_only

class LogTransformer(Transformer, HasInputCol, HasOutputCol):
    """Custom transformer that applies log1p to a column."""

    @keyword_only
    def __init__(self, inputCol=None, outputCol=None):
        super().__init__()
        kwargs = self._input_kwargs
        self.setParams(**kwargs)

    @keyword_only
    def setParams(self, inputCol=None, outputCol=None):
        kwargs = self._input_kwargs
        return self._set(**kwargs)

    def _transform(self, df):
        from pyspark.sql.functions import log1p
        return df.withColumn(self.getOutputCol(), log1p(df[self.getInputCol()]))

# Usage in a pipeline
log_tx = LogTransformer(inputCol="revenue", outputCol="log_revenue")
```

---

## Feature Engineering

### VectorAssembler

Combines multiple numeric columns into a single feature vector, which is the format MLlib models expect.

```python
from pyspark.ml.feature import VectorAssembler

assembler = VectorAssembler(
    inputCols=["age", "income", "credit_score", "tenure_months"],
    outputCol="features",
    handleInvalid="skip"  # skip rows with nulls; also "keep" or "error"
)

assembled_df = assembler.transform(df)
# features column: DenseVector([25.0, 55000.0, 720.0, 36.0])
```

### StringIndexer And OneHotEncoder

Convert categorical strings to numeric representations.

```python
from pyspark.ml.feature import StringIndexer, OneHotEncoder

# StringIndexer maps strings to frequency-ordered indices
indexer = StringIndexer(
    inputCol="department",
    outputCol="department_index",
    handleInvalid="keep"  # assign unseen labels to a new index
)

# OneHotEncoder converts indices to sparse binary vectors
encoder = OneHotEncoder(
    inputCol="department_index",
    outputCol="department_vector",
    dropLast=True  # avoid dummy variable trap
)

# Multi-column indexing (Spark 3.x)
multi_indexer = StringIndexer(
    inputCols=["department", "region", "job_title"],
    outputCols=["dept_idx", "region_idx", "job_idx"],
    handleInvalid="keep"
)
```

### StandardScaler

Normalises features to zero mean and unit variance.

```python
from pyspark.ml.feature import StandardScaler, MinMaxScaler, MaxAbsScaler

# StandardScaler — z-score normalisation
scaler = StandardScaler(
    inputCol="raw_features",
    outputCol="scaled_features",
    withStd=True,
    withMean=True  # requires dense vectors
)

# MinMaxScaler — scale to [0, 1]
min_max = MinMaxScaler(inputCol="raw_features", outputCol="minmax_features")

# MaxAbsScaler — scale by max absolute value, preserves sparsity
max_abs = MaxAbsScaler(inputCol="raw_features", outputCol="maxabs_features")
```

### Bucketizer

Discretise continuous features into bins.

```python
from pyspark.ml.feature import Bucketizer, QuantileDiscretizer

# Manual bucket boundaries
bucketizer = Bucketizer(
    splits=[float("-inf"), 18, 25, 35, 50, 65, float("inf")],
    inputCol="age",
    outputCol="age_bucket"
)

# Automatic quantile-based bucketing
quantile_disc = QuantileDiscretizer(
    numBuckets=5,
    inputCol="income",
    outputCol="income_bucket",
    relativeError=0.01
)
```

### Additional Feature Transformers

```python
from pyspark.ml.feature import (
    Imputer, PCA, Interaction,
    Tokenizer, StopWordsRemover, HashingTF, IDF
)

# Impute missing values with median
imputer = Imputer(
    inputCols=["age", "income"],
    outputCols=["age_imputed", "income_imputed"],
    strategy="median"  # also "mean" or "mode"
)

# Dimensionality reduction with PCA
pca = PCA(k=10, inputCol="features", outputCol="pca_features")

# Text feature pipeline
tokenizer = Tokenizer(inputCol="description", outputCol="words")
remover = StopWordsRemover(inputCol="words", outputCol="filtered_words")
hashing_tf = HashingTF(inputCol="filtered_words", outputCol="tf_features", numFeatures=10000)
idf = IDF(inputCol="tf_features", outputCol="tfidf_features")
```

### Complete Feature Engineering Pipeline

```python
from pyspark.ml import Pipeline
from pyspark.ml.feature import (
    StringIndexer, OneHotEncoder, VectorAssembler,
    StandardScaler, Imputer
)

def build_feature_pipeline(categorical_cols, numerical_cols):
    """Build a reusable feature engineering pipeline."""
    stages = []

    # Impute numerical columns
    imputer = Imputer(
        inputCols=numerical_cols,
        outputCols=[f"{c}_imputed" for c in numerical_cols],
        strategy="median"
    )
    stages.append(imputer)

    # Index and encode categoricals
    for col in categorical_cols:
        indexer = StringIndexer(
            inputCol=col, outputCol=f"{col}_idx", handleInvalid="keep"
        )
        encoder = OneHotEncoder(
            inputCol=f"{col}_idx", outputCol=f"{col}_vec"
        )
        stages.extend([indexer, encoder])

    # Assemble all features
    feature_cols = (
        [f"{c}_imputed" for c in numerical_cols]
        + [f"{c}_vec" for c in categorical_cols]
    )
    assembler = VectorAssembler(inputCols=feature_cols, outputCol="raw_features")
    scaler = StandardScaler(inputCol="raw_features", outputCol="features")

    stages.extend([assembler, scaler])
    return Pipeline(stages=stages)

# Usage
feat_pipeline = build_feature_pipeline(
    categorical_cols=["region", "product_type"],
    numerical_cols=["revenue", "quantity", "discount"]
)
```

---

## Classification

### Logistic Regression

```python
from pyspark.ml.classification import LogisticRegression

lr = LogisticRegression(
    featuresCol="features",
    labelCol="label",
    maxIter=100,
    regParam=0.01,       # L2 regularisation strength
    elasticNetParam=0.5, # 0=L2, 1=L1, between=elastic net
    threshold=0.5
)

lr_model = lr.fit(train_df)

# Model summary (binary classification)
summary = lr_model.summary
print(f"AUC: {summary.areaUnderROC}")
print(f"Accuracy: {summary.accuracy}")
```

### Random Forest Classifier

```python
from pyspark.ml.classification import RandomForestClassifier

rf = RandomForestClassifier(
    featuresCol="features",
    labelCol="label",
    numTrees=100,
    maxDepth=10,
    featureSubsetStrategy="sqrt",  # also "log2", "all", "onethird"
    impurity="gini",               # also "entropy"
    seed=42
)

rf_model = rf.fit(train_df)
print(f"Feature importances: {rf_model.featureImportances}")
```

### Gradient-Boosted Trees (GBT)

```python
from pyspark.ml.classification import GBTClassifier

gbt = GBTClassifier(
    featuresCol="features",
    labelCol="label",
    maxIter=50,
    maxDepth=5,
    stepSize=0.1,   # learning rate
    seed=42
)

gbt_model = gbt.fit(train_df)
print(f"Feature importances: {gbt_model.featureImportances}")
print(f"Number of trees: {gbt_model.getNumTrees}")
```

### Multi-Class Classification

```python
from pyspark.ml.classification import (
    LogisticRegression, RandomForestClassifier,
    OneVsRest
)

# Native multi-class with Logistic Regression
lr_multi = LogisticRegression(
    featuresCol="features",
    labelCol="label",
    maxIter=100,
    family="multinomial"  # explicit multi-class
)

# One-vs-Rest wrapper for binary classifiers
ovr = OneVsRest(
    classifier=LogisticRegression(maxIter=100),
    featuresCol="features",
    labelCol="label"
)
```

---

## Regression

```python
from pyspark.ml.regression import (
    LinearRegression,
    RandomForestRegressor,
    GBTRegressor,
    GeneralizedLinearRegression
)

# Linear Regression
lr_reg = LinearRegression(
    featuresCol="features",
    labelCol="price",
    maxIter=100,
    regParam=0.01,
    elasticNetParam=0.5
)

lr_reg_model = lr_reg.fit(train_df)
print(f"RMSE: {lr_reg_model.summary.rootMeanSquaredError}")
print(f"R2: {lr_reg_model.summary.r2}")

# Random Forest Regressor
rf_reg = RandomForestRegressor(
    featuresCol="features",
    labelCol="price",
    numTrees=100,
    maxDepth=10
)

# GBT Regressor
gbt_reg = GBTRegressor(
    featuresCol="features",
    labelCol="price",
    maxIter=50,
    maxDepth=5,
    stepSize=0.1
)

# Generalised Linear Model (Poisson, Gamma, etc.)
glm = GeneralizedLinearRegression(
    family="poisson",
    link="log",
    featuresCol="features",
    labelCol="count"
)
```

---

## Clustering

### KMeans

```python
from pyspark.ml.clustering import KMeans, KMeansModel

kmeans = KMeans(
    featuresCol="features",
    predictionCol="cluster",
    k=5,
    seed=42,
    maxIter=50,
    initMode="k-means||"  # also "random"
)

kmeans_model = kmeans.fit(df)

# Cluster centres
centres = kmeans_model.clusterCenters()
for i, centre in enumerate(centres):
    print(f"Cluster {i}: {centre}")

# Within-cluster sum of squared errors (inertia)
wssse = kmeans_model.summary.trainingCost
print(f"WSSSE: {wssse}")

# Cluster sizes
print(f"Cluster sizes: {kmeans_model.summary.clusterSizes}")
```

### Elbow Method For Optimal K

```python
def find_optimal_k(df, feature_col="features", k_range=range(2, 15)):
    """Run KMeans for multiple k values and return costs."""
    costs = []
    for k in k_range:
        km = KMeans(featuresCol=feature_col, k=k, seed=42, maxIter=50)
        model = km.fit(df)
        costs.append((k, model.summary.trainingCost))
    return costs

# Silhouette score evaluation
from pyspark.ml.evaluation import ClusteringEvaluator

evaluator = ClusteringEvaluator(
    featuresCol="features",
    predictionCol="cluster",
    metricName="silhouette"
)

silhouette = evaluator.evaluate(predictions)
print(f"Silhouette score: {silhouette}")
```

### BisectingKMeans

A hierarchical variant that recursively splits the largest cluster.

```python
from pyspark.ml.clustering import BisectingKMeans

bisecting = BisectingKMeans(
    featuresCol="features",
    predictionCol="cluster",
    k=5,
    seed=42,
    maxIter=20,
    minDivisibleClusterSize=1.0  # minimum points in a cluster to split
)

bisecting_model = bisecting.fit(df)
print(f"Cluster centres: {bisecting_model.clusterCenters()}")
```

---

## Model Evaluation

### Binary Classification Evaluator

```python
from pyspark.ml.evaluation import BinaryClassificationEvaluator

binary_eval = BinaryClassificationEvaluator(
    rawPredictionCol="rawPrediction",
    labelCol="label"
)

# Area under ROC (default)
auc_roc = binary_eval.evaluate(predictions, {binary_eval.metricName: "areaUnderROC"})

# Area under PR curve
auc_pr = binary_eval.evaluate(predictions, {binary_eval.metricName: "areaUnderPR"})

print(f"AUC-ROC: {auc_roc:.4f}")
print(f"AUC-PR:  {auc_pr:.4f}")
```

### Multi-Class Classification Evaluator

```python
from pyspark.ml.evaluation import MulticlassClassificationEvaluator

multi_eval = MulticlassClassificationEvaluator(
    predictionCol="prediction",
    labelCol="label"
)

accuracy = multi_eval.evaluate(predictions, {multi_eval.metricName: "accuracy"})
f1 = multi_eval.evaluate(predictions, {multi_eval.metricName: "f1"})
precision = multi_eval.evaluate(predictions, {multi_eval.metricName: "weightedPrecision"})
recall = multi_eval.evaluate(predictions, {multi_eval.metricName: "weightedRecall"})

print(f"Accuracy:  {accuracy:.4f}")
print(f"F1:        {f1:.4f}")
print(f"Precision: {precision:.4f}")
print(f"Recall:    {recall:.4f}")
```

### Regression Evaluator

```python
from pyspark.ml.evaluation import RegressionEvaluator

reg_eval = RegressionEvaluator(predictionCol="prediction", labelCol="price")

rmse = reg_eval.evaluate(predictions, {reg_eval.metricName: "rmse"})
mae = reg_eval.evaluate(predictions, {reg_eval.metricName: "mae"})
r2 = reg_eval.evaluate(predictions, {reg_eval.metricName: "r2"})

print(f"RMSE: {rmse:.4f}")
print(f"MAE:  {mae:.4f}")
print(f"R2:   {r2:.4f}")
```

---

## Hyperparameter Tuning

### ParamGridBuilder

```python
from pyspark.ml.tuning import ParamGridBuilder

rf = RandomForestClassifier(featuresCol="features", labelCol="label")

param_grid = (
    ParamGridBuilder()
    .addGrid(rf.numTrees, [50, 100, 200])
    .addGrid(rf.maxDepth, [5, 10, 15])
    .addGrid(rf.featureSubsetStrategy, ["sqrt", "log2"])
    .build()
)

print(f"Total parameter combinations: {len(param_grid)}")
# 3 x 3 x 2 = 18 combinations
```

### CrossValidator

```python
from pyspark.ml.tuning import CrossValidator

cv = CrossValidator(
    estimator=pipeline,       # can be a Pipeline or single Estimator
    estimatorParamMaps=param_grid,
    evaluator=BinaryClassificationEvaluator(metricName="areaUnderROC"),
    numFolds=5,
    parallelism=4,            # fit models in parallel
    seed=42,
    collectSubModels=False    # set True to inspect all fold models
)

cv_model = cv.fit(train_df)

# Best model and its metrics
best_model = cv_model.bestModel
avg_metrics = cv_model.avgMetrics  # average metric per param combo
best_idx = avg_metrics.index(max(avg_metrics))
print(f"Best AUC: {max(avg_metrics):.4f}")
print(f"Best params: {param_grid[best_idx]}")
```

### TrainValidationSplit (Faster Alternative)

```python
from pyspark.ml.tuning import TrainValidationSplit

tvs = TrainValidationSplit(
    estimator=pipeline,
    estimatorParamMaps=param_grid,
    evaluator=BinaryClassificationEvaluator(metricName="areaUnderROC"),
    trainRatio=0.8,    # 80% train, 20% validation
    parallelism=4,
    seed=42
)

tvs_model = tvs.fit(train_df)
best_model = tvs_model.bestModel
```

---

## Model Persistence

### Save And Load

```python
# Save a fitted PipelineModel
model.save("/mnt/models/churn_predictor/v1")

# Load it back
from pyspark.ml import PipelineModel
loaded_model = PipelineModel.load("/mnt/models/churn_predictor/v1")

# Save/load a CrossValidatorModel
cv_model.save("/mnt/models/churn_cv/v1")

from pyspark.ml.tuning import CrossValidatorModel
loaded_cv = CrossValidatorModel.load("/mnt/models/churn_cv/v1")

# Save individual stages
rf_model.save("/mnt/models/rf_classifier/v1")

from pyspark.ml.classification import RandomForestClassificationModel
loaded_rf = RandomForestClassificationModel.load("/mnt/models/rf_classifier/v1")
```

### Model Versioning Pattern

```python
from datetime import datetime

def save_model_versioned(model, base_path, model_name, metadata=None):
    """Save a model with timestamp-based versioning."""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    version_path = f"{base_path}/{model_name}/{timestamp}"

    model.save(version_path)

    # Save metadata alongside the model
    if metadata:
        metadata_df = spark.createDataFrame([metadata])
        metadata_df.write.mode("overwrite").json(f"{version_path}/_metadata")

    # Update a latest symlink / pointer
    pointer_df = spark.createDataFrame([{"latest_version": version_path}])
    pointer_df.write.mode("overwrite").json(f"{base_path}/{model_name}/_latest")

    return version_path

save_model_versioned(
    model=cv_model.bestModel,
    base_path="/mnt/models",
    model_name="churn_predictor",
    metadata={
        "auc": float(max(cv_model.avgMetrics)),
        "training_date": datetime.now().isoformat(),
        "num_features": 15,
        "framework": "pyspark_mllib"
    }
)
```

---

## Feature Importance Extraction

### Tree-Based Feature Importances

```python
from pyspark.ml.feature import VectorAssembler

def extract_feature_importances(model, feature_names):
    """Extract and sort feature importances from a tree-based model."""
    importances = model.featureImportances.toArray()
    feature_imp = list(zip(feature_names, importances))
    feature_imp.sort(key=lambda x: x[1], reverse=True)
    return feature_imp

# Get feature names from the VectorAssembler stage
assembler_stage = pipeline_model.stages[0]  # adjust index as needed
feature_names = assembler_stage.getInputCols()

# Extract from Random Forest
rf_model = pipeline_model.stages[-1]
importances = extract_feature_importances(rf_model, feature_names)

for name, imp in importances[:10]:
    print(f"{name:30s} {imp:.4f}")
```

### Logistic Regression Coefficients

```python
import numpy as np

def extract_lr_coefficients(lr_model, feature_names):
    """Extract coefficients from a logistic regression model."""
    coefficients = lr_model.coefficients.toArray()
    coef_map = list(zip(feature_names, coefficients))
    # Sort by absolute value for importance ranking
    coef_map.sort(key=lambda x: abs(x[1]), reverse=True)
    return coef_map

coefs = extract_lr_coefficients(lr_model, feature_names)
for name, coef in coefs[:10]:
    print(f"{name:30s} {coef:+.4f}")
```

### Feature Importance As DataFrame

```python
def feature_importance_df(model, feature_names, spark):
    """Return feature importances as a Spark DataFrame for downstream use."""
    importances = model.featureImportances.toArray()
    rows = [
        {"feature": name, "importance": float(imp)}
        for name, imp in zip(feature_names, importances)
    ]
    return (
        spark.createDataFrame(rows)
        .orderBy("importance", ascending=False)
    )

importance_df = feature_importance_df(rf_model, feature_names, spark)
importance_df.show(10)

# Persist for reporting
importance_df.write.mode("overwrite").parquet("/mnt/analytics/feature_importances/")
```

---

## End-To-End Example

```python
from pyspark.ml import Pipeline
from pyspark.ml.feature import (
    StringIndexer, OneHotEncoder, VectorAssembler,
    StandardScaler, Imputer
)
from pyspark.ml.classification import RandomForestClassifier
from pyspark.ml.evaluation import BinaryClassificationEvaluator
from pyspark.ml.tuning import CrossValidator, ParamGridBuilder

# 1. Load and split
df = spark.read.parquet("/mnt/data/customers/")
train_df, test_df = df.randomSplit([0.8, 0.2], seed=42)

# 2. Build feature pipeline
categorical_cols = ["region", "product_type", "channel"]
numerical_cols = ["revenue", "num_orders", "days_since_last_order"]

stages = []
stages.append(Imputer(
    inputCols=numerical_cols,
    outputCols=[f"{c}_imp" for c in numerical_cols],
    strategy="median"
))
for col in categorical_cols:
    stages.append(StringIndexer(inputCol=col, outputCol=f"{col}_idx", handleInvalid="keep"))
    stages.append(OneHotEncoder(inputCol=f"{col}_idx", outputCol=f"{col}_vec"))

all_features = [f"{c}_imp" for c in numerical_cols] + [f"{c}_vec" for c in categorical_cols]
stages.append(VectorAssembler(inputCols=all_features, outputCol="raw_features"))
stages.append(StandardScaler(inputCol="raw_features", outputCol="features"))

# 3. Add classifier
rf = RandomForestClassifier(featuresCol="features", labelCol="churned", seed=42)
stages.append(rf)

pipeline = Pipeline(stages=stages)

# 4. Hyperparameter tuning
param_grid = (
    ParamGridBuilder()
    .addGrid(rf.numTrees, [100, 200])
    .addGrid(rf.maxDepth, [5, 10])
    .build()
)

evaluator = BinaryClassificationEvaluator(
    labelCol="churned", metricName="areaUnderROC"
)

cv = CrossValidator(
    estimator=pipeline,
    estimatorParamMaps=param_grid,
    evaluator=evaluator,
    numFolds=3,
    parallelism=4,
    seed=42
)

# 5. Train and evaluate
cv_model = cv.fit(train_df)
predictions = cv_model.transform(test_df)

auc = evaluator.evaluate(predictions)
print(f"Test AUC: {auc:.4f}")

# 6. Save best model
cv_model.bestModel.save("/mnt/models/churn_predictor/latest")
```

---

## Related Notes

- [[PySpark Core Concepts]] — SparkSession, DataFrame fundamentals
- [[PySpark Advanced Features]] — broader advanced feature coverage
- [[PySpark Performance Optimization]] — tuning Spark for ML workloads
- [[PySpark MLOps Patterns]] — MLflow integration, model serving, monitoring
- [[Databricks & Delta Lake]] — managed ML on Databricks
