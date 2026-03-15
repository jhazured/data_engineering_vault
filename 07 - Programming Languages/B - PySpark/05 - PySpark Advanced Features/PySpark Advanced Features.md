
#pyspark #advanced #ml #graph #streaming #delta-lake #catalyst #rdd #optimization #enterprise

## Overview

This note covers advanced PySpark features that go beyond basic data processing, including machine learning integration, graph processing, advanced RDD operations, Catalyst optimizer internals, Delta Lake integration, and enterprise-grade features. These capabilities enable sophisticated data science workflows and production-scale analytics.

---

## Machine Learning with MLlib

### MLlib Pipeline Architecture

```python
from pyspark.ml import Pipeline, PipelineModel
from pyspark.ml.feature import *
from pyspark.ml.classification import *
from pyspark.ml.regression import *
from pyspark.ml.clustering import *
from pyspark.ml.evaluation import *
from pyspark.ml.tuning import *

# Create a comprehensive ML pipeline
def create_ml_pipeline():
    """Create a complete ML pipeline with feature engineering and model training"""
    
    # Feature engineering stages
    string_indexer = StringIndexer(inputCol="category", outputCol="category_index")
    one_hot_encoder = OneHotEncoder(inputCol="category_index", outputCol="category_vector")
    
    # Numerical feature processing
    assembler_numerical = VectorAssembler(
        inputCols=["age", "income", "score"],
        outputCol="numerical_features"
    )
    
    scaler = StandardScaler(
        inputCol="numerical_features",
        outputCol="scaled_numerical_features",
        withStd=True,
        withMean=True
    )
    
    # Text processing pipeline
    tokenizer = Tokenizer(inputCol="description", outputCol="words")
    remover = StopWordsRemover(inputCol="words", outputCol="filtered_words")
    hashing_tf = HashingTF(inputCol="filtered_words", outputCol="text_features", numFeatures=10000)
    idf = IDF(inputCol="text_features", outputCol="tfidf_features")
    
    # Final feature assembly
    final_assembler = VectorAssembler(
        inputCols=["category_vector", "scaled_numerical_features", "tfidf_features"],
        outputCol="features"
    )
    
    # Model
    rf = RandomForestClassifier(
        featuresCol="features",
        labelCol="label",
        numTrees=100,
        maxDepth=10,
        seed=42
    )
    
    # Create pipeline
    pipeline = Pipeline(stages=[
        string_indexer,
        one_hot_encoder,
        assembler_numerical,
        scaler,
        tokenizer,
        remover,
        hashing_tf,
        idf,
        final_assembler,
        rf
    ])
    
    return pipeline

# Advanced pipeline with feature selection
def create_advanced_pipeline_with_feature_selection():
    """Pipeline with automatic feature selection and hyperparameter tuning"""
    
    # Basic preprocessing
    string_indexer = StringIndexer(inputCol="category", outputCol="category_index")
    encoder = OneHotEncoder(inputCol="category_index", outputCol="category_vector")
    
    # Assemble all features
    assembler = VectorAssembler(
        inputCols=["category_vector", "feature1", "feature2", "feature3", "feature4", "feature5"],
        outputCol="raw_features"
    )
    
    # Feature selection
    selector = ChiSqSelector(
        featuresCol="raw_features",
        outputCol="selected_features",
        labelCol="label",
        numTopFeatures=10
    )
    
    # Model with hyperparameter tuning
    lr = LogisticRegression(
        featuresCol="selected_features",
        labelCol="label",
        maxIter=100
    )
    
    # Create pipeline
    pipeline = Pipeline(stages=[string_indexer, encoder, assembler, selector, lr])
    
    # Hyperparameter tuning
    param_grid = ParamGridBuilder() \
        .addGrid(selector.numTopFeatures, [5, 10, 15]) \
        .addGrid(lr.regParam, [0.01, 0.1, 1.0]) \
        .addGrid(lr.elasticNetParam, [0.0, 0.5, 1.0]) \
        .build()
    
    # Cross validation
    evaluator = BinaryClassificationEvaluator(
        labelCol="label",
        rawPredictionCol="rawPrediction",
        metricName="areaUnderROC"
    )
    
    cv = CrossValidator(
        estimator=pipeline,
        estimatorParamMaps=param_grid,
        evaluator=evaluator,
        numFolds=5,
        seed=42
    )
    
    return cv

# Usage example
# Create sample data
sample_data = spark.createDataFrame([
    (0, "electronics", 25, 50000, 0.8, "high quality product"),
    (1, "clothing", 30, 60000, 0.9, "comfortable and stylish"),
    (0, "books", 45, 40000, 0.7, "educational content"),
    (1, "electronics", 35, 70000, 0.95, "latest technology")
], ["label", "category", "age", "income", "score", "description"])

# Train pipeline
pipeline = create_ml_pipeline()
model = pipeline.fit(sample_data)

# Make predictions
predictions = model.transform(sample_data)
predictions.select("label", "prediction", "probability").show()
```

### Advanced Model Evaluation and Selection

```python
class MLModelEvaluator:
    """Comprehensive model evaluation framework"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.models = {}
        self.evaluation_results = {}
    
    def evaluate_classification_models(self, models_dict, test_data, label_col="label"):
        """Evaluate multiple classification models"""
        
        evaluators = {
            "auc": BinaryClassificationEvaluator(
                labelCol=label_col, 
                rawPredictionCol="rawPrediction",
                metricName="areaUnderROC"
            ),
            "accuracy": MulticlassClassificationEvaluator(
                labelCol=label_col,
                predictionCol="prediction",
                metricName="accuracy"
            ),
            "f1": MulticlassClassificationEvaluator(
                labelCol=label_col,
                predictionCol="prediction",
                metricName="f1"
            ),
            "precision": MulticlassClassificationEvaluator(
                labelCol=label_col,
                predictionCol="prediction",
                metricName="weightedPrecision"
            ),
            "recall": MulticlassClassificationEvaluator(
                labelCol=label_col,
                predictionCol="prediction",
                metricName="weightedRecall"
            )
        }
        
        results = {}
        
        for model_name, model in models_dict.items():
            print(f"Evaluating {model_name}...")
            
            # Make predictions
            predictions = model.transform(test_data)
            
            # Calculate metrics
            model_metrics = {}
            for metric_name, evaluator in evaluators.items():
                try:
                    score = evaluator.evaluate(predictions)
                    model_metrics[metric_name] = score
                except Exception as e:
                    print(f"Could not calculate {metric_name} for {model_name}: {e}")
                    model_metrics[metric_name] = None
            
            # Additional metrics
            model_metrics.update(self._calculate_additional_metrics(predictions, label_col))
            
            results[model_name] = model_metrics
            
        self.evaluation_results = results
        return results
    
    def _calculate_additional_metrics(self, predictions, label_col):
        """Calculate additional custom metrics"""
        
        # Confusion matrix elements
        tp = predictions.filter((col(label_col) == 1) & (col("prediction") == 1)).count()
        tn = predictions.filter((col(label_col) == 0) & (col("prediction") == 0)).count()
        fp = predictions.filter((col(label_col) == 0) & (col("prediction") == 1)).count()
        fn = predictions.filter((col(label_col) == 1) & (col("prediction") == 0)).count()
        
        # Calculate metrics
        precision = tp / (tp + fp) if (tp + fp) > 0 else 0
        recall = tp / (tp + fn) if (tp + fn) > 0 else 0
        specificity = tn / (tn + fp) if (tn + fp) > 0 else 0
        
        return {
            "true_positives": tp,
            "true_negatives": tn,
            "false_positives": fp,
            "false_negatives": fn,
            "precision_manual": precision,
            "recall_manual": recall,
            "specificity": specificity
        }
    
    def compare_models(self, primary_metric="auc"):
        """Compare models and rank by primary metric"""
        
        if not self.evaluation_results:
            print("No evaluation results available. Run evaluate_classification_models first.")
            return None
        
        # Sort models by primary metric
        sorted_models = sorted(
            self.evaluation_results.items(),
            key=lambda x: x[1].get(primary_metric, 0),
            reverse=True
        )
        
        print(f"\nModel Comparison (ranked by {primary_metric}):")
        print("-" * 80)
        print(f"{'Model':<20} {primary_metric.upper():<10} {'Accuracy':<10} {'F1':<10} {'Precision':<12} {'Recall':<10}")
        print("-" * 80)
        
        for model_name, metrics in sorted_models:
            print(f"{model_name:<20} "
                  f"{metrics.get(primary_metric, 0):<10.4f} "
                  f"{metrics.get('accuracy', 0):<10.4f} "
                  f"{metrics.get('f1', 0):<10.4f} "
                  f"{metrics.get('precision', 0):<12.4f} "
                  f"{metrics.get('recall', 0):<10.4f}")
        
        return sorted_models
    
    def feature_importance_analysis(self, model, feature_names):
        """Analyze feature importance for tree-based models"""
        
        try:
            if hasattr(model, 'featureImportances'):
                importances = model.featureImportances.toArray()
                
                feature_importance_df = self.spark.createDataFrame([
                    (feature_names[i], float(importances[i]))
                    for i in range(len(feature_names))
                ], ["feature", "importance"])
                
                return feature_importance_df.orderBy(col("importance").desc())
            else:
                print("Model does not support feature importance analysis")
                return None
                
        except Exception as e:
            print(f"Error analyzing feature importance: {e}")
            return None

# Usage example
evaluator = MLModelEvaluator(spark)

# Train multiple models
models = {
    "RandomForest": RandomForestClassifier(numTrees=100).fit(train_data),
    "LogisticRegression": LogisticRegression().fit(train_data),
    "GBT": GBTClassifier(maxIter=50).fit(train_data)
}

# Evaluate all models
results = evaluator.evaluate_classification_models(models, test_data)

# Compare models
best_models = evaluator.compare_models("auc")

# Analyze feature importance
feature_names = ["feature1", "feature2", "feature3", "feature4"]
importance_df = evaluator.feature_importance_analysis(models["RandomForest"], feature_names)
if importance_df:
    importance_df.show()
```

### Online Learning and Model Updates

```python
class OnlineLearningFramework:
    """Framework for online learning with streaming data"""
    
    def __init__(self, spark_session, model_path, checkpoint_path):
        self.spark = spark_session
        self.model_path = model_path
        self.checkpoint_path = checkpoint_path
        self.current_model = None
        self.model_version = 0
    
    def initialize_model(self, initial_training_data):
        """Initialize model with historical data"""
        
        # Create initial pipeline
        pipeline = Pipeline(stages=[
            VectorAssembler(inputCols=["feature1", "feature2", "feature3"], outputCol="features"),
            LogisticRegression(featuresCol="features", labelCol="label")
        ])
        
        # Train initial model
        self.current_model = pipeline.fit(initial_training_data)
        self.save_model()
        
        print(f"Initialized model version {self.model_version}")
        return self.current_model
    
    def update_model_streaming(self, streaming_data):
        """Update model with streaming data using micro-batches"""
        
        def update_model_batch(batch_df, batch_id):
            """Update model for each micro-batch"""
            
            if batch_df.count() > 0:
                print(f"Updating model with batch {batch_id}, records: {batch_df.count()}")
                
                try:
                    # Combine with historical data (simplified approach)
                    # In production, you might use more sophisticated incremental learning
                    
                    # For demonstration, retrain on recent batches
                    if batch_df.count() >= 100:  # Minimum batch size for retraining
                        
                        # Create new pipeline (same structure)
                        pipeline = Pipeline(stages=[
                            VectorAssembler(inputCols=["feature1", "feature2", "feature3"], outputCol="features"),
                            LogisticRegression(featuresCol="features", labelCol="label")
                        ])
                        
                        # Retrain model
                        updated_model = pipeline.fit(batch_df)
                        
                        # Evaluate model performance
                        if self._validate_model_performance(updated_model, batch_df):
                            self.current_model = updated_model
                            self.model_version += 1
                            self.save_model()
                            print(f"Model updated to version {self.model_version}")
                        else:
                            print("Model update rejected due to performance degradation")
                    
                except Exception as e:
                    print(f"Error updating model: {e}")
        
        # Start streaming query for model updates
        query = streaming_data.writeStream \
            .foreachBatch(update_model_batch) \
            .option("checkpointLocation", f"{self.checkpoint_path}/model_updates") \
            .trigger(processingTime="5 minutes") \
            .start()
        
        return query
    
    def _validate_model_performance(self, new_model, validation_data):
        """Validate new model performance against current model"""
        
        if self.current_model is None:
            return True  # First model
        
        try:
            # Evaluate current model
            current_predictions = self.current_model.transform(validation_data)
            current_evaluator = BinaryClassificationEvaluator(
                labelCol="label", 
                rawPredictionCol="rawPrediction"
            )
            current_auc = current_evaluator.evaluate(current_predictions)
            
            # Evaluate new model
            new_predictions = new_model.transform(validation_data)
            new_auc = current_evaluator.evaluate(new_predictions)
            
            # Accept new model if it's better or within acceptable degradation
            improvement_threshold = -0.05  # Allow 5% degradation
            performance_change = new_auc - current_auc
            
            print(f"Current AUC: {current_auc:.4f}, New AUC: {new_auc:.4f}, Change: {performance_change:.4f}")
            
            return performance_change >= improvement_threshold
            
        except Exception as e:
            print(f"Error validating model: {e}")
            return False
    
    def save_model(self):
        """Save current model to persistent storage"""
        
        if self.current_model:
            model_version_path = f"{self.model_path}/version_{self.model_version}"
            self.current_model.write().overwrite().save(model_version_path)
            
            # Save metadata
            metadata = {
                "version": self.model_version,
                "timestamp": datetime.now().isoformat(),
                "model_path": model_version_path
            }
            
            metadata_df = self.spark.createDataFrame([metadata])
            metadata_df.write.mode("append").json(f"{self.model_path}/metadata")
    
    def load_model(self, version=None):
        """Load model from persistent storage"""
        
        if version is None:
            version = self.model_version
        
        try:
            model_path = f"{self.model_path}/version_{version}"
            self.current_model = PipelineModel.load(model_path)
            self.model_version = version
            print(f"Loaded model version {version}")
            return self.current_model
        except Exception as e:
            print(f"Error loading model: {e}")
            return None
    
    def predict_streaming(self, streaming_data):
        """Make predictions on streaming data"""
        
        def make_predictions(batch_df, batch_id):
            """Make predictions for each batch"""
            
            if batch_df.count() > 0 and self.current_model:
                predictions = self.current_model.transform(batch_df)
                
                # Save predictions
                predictions.select("id", "features", "prediction", "probability") \
                    .write \
                    .mode("append") \
                    .parquet(f"{self.model_path}/predictions")
                
                print(f"Made predictions for batch {batch_id}: {batch_df.count()} records")
        
        query = streaming_data.writeStream \
            .foreachBatch(make_predictions) \
            .option("checkpointLocation", f"{self.checkpoint_path}/predictions") \
            .trigger(processingTime="1 minute") \
            .start()
        
        return query

# Usage example
online_learner = OnlineLearningFramework(
    spark, 
    "/models/online_learning", 
    "/checkpoints/online_learning"
)

# Initialize with historical data
# initial_model = online_learner.initialize_model(historical_data)

# Start online learning with streaming data
# update_query = online_learner.update_model_streaming(streaming_training_data)

# Start making predictions
# prediction_query = online_learner.predict_streaming(streaming_inference_data)
```

---

## Graph Processing with GraphFrames

### GraphFrames Fundamentals

```python
# Install GraphFrames if needed
# spark.sparkContext.addPyFile("graphframes-0.8.2-spark3.0-s_2.12.jar")

from graphframes import GraphFrame
from pyspark.sql.functions import *

def create_sample_graph():
    """Create a sample social network graph"""
    
    # Vertices (users)
    vertices = spark.createDataFrame([
        ("1", "Alice", 25, "Engineer"),
        ("2", "Bob", 30, "Manager"),
        ("3", "Charlie", 28, "Designer"),
        ("4", "Diana", 32, "Analyst"),
        ("5", "Eve", 27, "Developer"),
        ("6", "Frank", 35, "Director")
    ], ["id", "name", "age", "role"])
    
    # Edges (relationships)
    edges = spark.createDataFrame([
        ("1", "2", "friend", 0.8),
        ("1", "3", "colleague", 0.6),
        ("2", "3", "friend", 0.9),
        ("2", "4", "manager", 1.0),
        ("3", "5", "friend", 0.7),
        ("4", "5", "colleague", 0.5),
        ("4", "6", "reports_to", 1.0),
        ("5", "6", "colleague", 0.4),
        ("1", "5", "friend", 0.8)
    ], ["src", "dst", "relationship", "weight"])
    
    # Create GraphFrame
    graph = GraphFrame(vertices, edges)
    return graph

# Advanced graph analytics
class GraphAnalytics:
    """Advanced graph analytics using GraphFrames"""
    
    def __init__(self, graph):
        self.graph = graph
    
    def basic_graph_statistics(self):
        """Calculate basic graph statistics"""
        
        stats = {
            "num_vertices": self.graph.vertices.count(),
            "num_edges": self.graph.edges.count(),
            "avg_degree": self.graph.degrees.agg(avg("degree")).collect()[0][0]
        }
        
        print(f"Graph Statistics:")
        print(f"  Vertices: {stats['num_vertices']}")
        print(f"  Edges: {stats['num_edges']}")
        print(f"  Average Degree: {stats['avg_degree']:.2f}")
        
        return stats
    
    def find_influential_users(self, algorithm="pagerank"):
        """Find influential users using centrality measures"""
        
        if algorithm == "pagerank":
            # PageRank algorithm
            pagerank_results = self.graph.pageRank(resetProbability=0.15, maxIter=10)
            
            influential_users = pagerank_results.vertices \
                .select("id", "name", "pagerank") \
                .orderBy(col("pagerank").desc()) \
                .limit(10)
            
            print("Top 10 Influential Users (PageRank):")
            influential_users.show()
            
            return influential_users
        
        elif algorithm == "degree_centrality":
            # Degree centrality
            degree_centrality = self.graph.degrees \
                .join(self.graph.vertices, "id") \
                .select("id", "name", "degree") \
                .orderBy(col("degree").desc())
            
            print("Users by Degree Centrality:")
            degree_centrality.show()
            
            return degree_centrality
    
    def community_detection(self):
        """Detect communities using Label Propagation Algorithm"""
        
        # Label Propagation Algorithm
        communities = self.graph.labelPropagation(maxIter=5)
        
        # Analyze communities
        community_stats = communities.groupBy("label") \
            .agg(
                count("*").alias("community_size"),
                collect_list("name").alias("members")
            ) \
            .orderBy(col("community_size").desc())
        
        print("Detected Communities:")
        community_stats.show(truncate=False)
        
        return communities, community_stats
    
    def shortest_paths_analysis(self, landmarks):
        """Calculate shortest paths from landmarks"""
        
        # Shortest paths to landmarks
        shortest_paths = self.graph.shortestPaths(landmarks=landmarks)
        
        print(f"Shortest Paths from landmarks {landmarks}:")
        shortest_paths.select("id", "name", "distances").show(truncate=False)
        
        return shortest_paths
    
    def motif_analysis(self):
        """Find common motifs/patterns in the graph"""
        
        # Find triangles (3-cycles)
        triangles = self.graph.find("(a)-[e1]->(b); (b)-[e2]->(c); (c)-[e3]->(a)")
        
        print("Triangles in the graph:")
        triangles.select(
            col("a.name").alias("person1"),
            col("b.name").alias("person2"),
            col("c.name").alias("person3")
        ).show()
        
        # Find mutual friends pattern
        mutual_friends = self.graph.find("(a)-[e1]->(b); (b)-[e2]->(a)")
        
        print("Mutual connections:")
        mutual_friends.select(
            col("a.name").alias("person1"),
            col("b.name").alias("person2"),
            col("e1.relationship").alias("relationship1"),
            col("e2.relationship").alias("relationship2")
        ).show()
        
        return triangles, mutual_friends
    
    def recommendation_engine(self, user_id, max_recommendations=5):
        """Simple friend recommendation based on mutual connections"""
        
        # Find friends of friends
        friends_of_friends = self.graph.find(f"""
            (user)-[e1]->(friend1);
            (friend1)-[e2]->(friend2);
            (friend2)-[e3]->(potential_friend)
        """).filter(f"user.id = '{user_id}' AND potential_friend.id != '{user_id}'")
        
        # Count mutual connections and recommend
        recommendations = friends_of_friends \
            .groupBy("potential_friend.id", "potential_friend.name") \
            .agg(count("*").alias("mutual_connections")) \
            .orderBy(col("mutual_connections").desc()) \
            .limit(max_recommendations)
        
        print(f"Friend recommendations for user {user_id}:")
        recommendations.show()
        
        return recommendations

# Usage example
graph = create_sample_graph()
analytics = GraphAnalytics(graph)

# Perform various analyses
stats = analytics.basic_graph_statistics()
influential = analytics.find_influential_users("pagerank")
communities, community_stats = analytics.community_detection()
recommendations = analytics.recommendation_engine("1")
```

### Advanced Graph Algorithms

```python
class AdvancedGraphAlgorithms:
    """Advanced graph algorithms for complex analytics"""
    
    def __init__(self, graph):
        self.graph = graph
    
    def collaborative_filtering_graph(self, user_item_interactions):
        """Build collaborative filtering using graph-based approach"""
        
        # Create bipartite graph of users and items
        user_vertices = user_item_interactions.select("user_id").distinct() \
            .withColumnRenamed("user_id", "id") \
            .withColumn("type", lit("user"))
        
        item_vertices = user_item_interactions.select("item_id").distinct() \
            .withColumnRenamed("item_id", "id") \
            .withColumn("type", lit("item"))
        
        bipartite_vertices = user_vertices.union(item_vertices)
        
        # Create edges from interactions
        bipartite_edges = user_item_interactions.select(
            col("user_id").alias("src"),
            col("item_id").alias("dst"),
            col("rating").alias("weight")
        ).withColumn("relationship", lit("rates"))
        
        # Create bipartite graph
        bipartite_graph = GraphFrame(bipartite_vertices, bipartite_edges)
        
        # Find item-item similarity through user connections
        item_similarity = bipartite_graph.find("""
            (item1)-[e1]-(user)-[e2]-(item2)
        """).filter("item1.type = 'item' AND item2.type = 'item' AND item1.id != item2.id")
        
        # Calculate similarity scores
        similarity_scores = item_similarity \
            .groupBy("item1.id", "item2.id") \
            .agg(
                count("*").alias("common_users"),
                avg((col("e1.weight") + col("e2.weight")) / 2).alias("avg_rating")
            ) \
            .withColumn("similarity_score", 
                       col("common_users") * col("avg_rating"))
        
        return bipartite_graph, similarity_scores
    
    def fraud_detection_graph(self, transaction_data):
        """Fraud detection using graph analysis"""
        
        # Create transaction graph
        account_vertices = transaction_data.select("from_account").distinct() \
            .withColumnRenamed("from_account", "id") \
            .withColumn("type", lit("account"))
        
        transaction_edges = transaction_data.select(
            col("from_account").alias("src"),
            col("to_account").alias("dst"),
            col("amount").alias("weight"),
            col("timestamp")
        ).withColumn("relationship", lit("transfer"))
        
        transaction_graph = GraphFrame(account_vertices, transaction_edges)
        
        # Detect suspicious patterns
        
        # 1. Circular transfers (money laundering pattern)
        circular_transfers = transaction_graph.find("""
            (a)-[e1]->(b); (b)-[e2]->(c); (c)-[e3]->(a)
        """).filter("""
            e1.timestamp < e2.timestamp AND 
            e2.timestamp < e3.timestamp AND
            abs(e1.weight - e3.weight) / e1.weight < 0.1
        """)  # Similar amounts
        
        # 2. Hub accounts (accounts with many connections)
        hub_accounts = transaction_graph.degrees \
            .filter(col("degree") > 10) \
            .orderBy(col("degree").desc())
        
        # 3. Burst activity (many transactions in short time)
        burst_activity = transaction_edges \
            .groupBy("src", window(col("timestamp"), "1 hour")) \
            .agg(
                count("*").alias("transactions_per_hour"),
                sum("weight").alias("total_amount")
            ) \
            .filter(col("transactions_per_hour") > 20)
        
        return {
            "graph": transaction_graph,
            "circular_transfers": circular_transfers,
            "hub_accounts": hub_accounts,
            "burst_activity": burst_activity
        }
    
    def supply_chain_analysis(self, supply_chain_data):
        """Analyze supply chain networks for optimization"""
        
        # Create supply chain graph
        supplier_vertices = supply_chain_data.select("supplier_id", "supplier_name", "location") \
            .distinct() \
            .withColumnRenamed("supplier_id", "id") \
            .withColumn("type", lit("supplier"))
        
        supply_edges = supply_chain_data.select(
            col("supplier_id").alias("src"),
            col("customer_id").alias("dst"),
            col("delivery_time").alias("weight"),
            col("cost"),
            col("quantity")
        ).withColumn("relationship", lit("supplies"))
        
        supply_graph = GraphFrame(supplier_vertices, supply_edges)
        
        # Analysis
        
        # 1. Critical suppliers (high degree centrality)
        critical_suppliers = supply_graph.degrees \
            .join(supplier_vertices.select("id", "supplier_name"), "id") \
            .orderBy(col("degree").desc()) \
            .limit(10)
        
        # 2. Supply chain paths and bottlenecks
        paths_analysis = supply_graph.shortestPaths(landmarks=["SUPPLIER_001", "SUPPLIER_002"])
        
        # 3. Risk analysis - single points of failure
        risk_analysis = supply_edges \
            .groupBy("dst") \
            .agg(
                countDistinct("src").alias("supplier_count"),
                avg("weight").alias("avg_delivery_time"),
                sum("quantity").alias("total_quantity")
            ) \
            .filter(col("supplier_count") == 1)  # Single supplier dependency
        
        return {
            "graph": supply_graph,
            "critical_suppliers": critical_suppliers,
            "paths_analysis": paths_analysis,
            "risk_analysis": risk_analysis
        }

# Usage examples
advanced_algorithms = AdvancedGraphAlgorithms(graph)

# Example: Collaborative filtering

# user_item_data = spark.createDataFrame([

# ("user1", "item1", 4.5),

# ("user1", "item2", 3.0),

# ("user2", "item1", 5.0),

# ("user2", "item3", 4.0)

# ], ["user_id", "item_id", "rating"])

# bipartite_graph, similarity = advanced_algorithms.collaborative_filtering_graph(user_item_data)
```

---

## Advanced RDD Operations

### Custom Partitioners and Advanced RDD Transformations

```python
from pyspark import Partitioner
import hashlib

class CustomHashPartitioner(Partitioner):
    """Custom partitioner for specific data distribution needs"""
    
    def __init__(self, num_partitions, partition_func=None):
        self.num_partitions = num_partitions
        self.partition_func = partition_func or self._default_partition_func
    
    def _default_partition_func(self, key):
        """Default partitioning function"""
        return hash(key) % self.num_partitions
    
    def numPartitions(self):
        return self.num_partitions
    
    def getPartition(self, key):
        return self.partition_func(key)

class GeographicPartitioner(Partitioner):
    """Geographic-based partitioner for location data"""
    
    def __init__(self, regions):
        self.regions = regions
        self.num_partitions = len(regions)
    
    def numPartitions(self):
        return self.num_partitions
    
    def getPartition(self, key):
        # Assume key is (lat, lon) tuple
        lat, lon = key
        
        # Simple geographic partitioning
        for i, (region_name, bounds) in enumerate(self.regions.items()):
            min_lat, max_lat, min_lon, max_lon = bounds
            if min_lat <= lat <= max_lat and min_lon <= lon <= max_lon:
                return i
        
        return 0  # Default partition

class AdvancedRDDOperations:
    """Advanced RDD operations and optimizations"""
    
    def __init__(self, spark_context):
        self.sc = spark_context
    
    def advanced_join_strategies(self, rdd1, rdd2):
        """Demonstrate advanced join strategies"""
        
        # 1. Broadcast hash join for small RDD
        if rdd2.count() < 10000:  # Small RDD threshold
            small_rdd_dict = rdd2.collectAsMap()
            broadcast_dict = self.sc.broadcast(small_rdd_dict)
            
            def broadcast_join(iterator):
                lookup = broadcast_dict.value
                for key, value in iterator:
                    if key in lookup:
                        yield (key, (value, lookup[key]))
            
            result = rdd1.mapPartitions(broadcast_join)
            return result
        
        # 2. Sort-merge join with custom partitioner
        else:
            partitioner = CustomHashPartitioner(200)
            
            partitioned_rdd1 = rdd1.partitionBy(partitioner)
            partitioned_rdd2 = rdd2.partitionBy(partitioner)
            
            # Now join will be much more efficient
            result = partitioned_rdd1.join(partitioned_rdd2)
            return result
    
    def advanced_aggregations(self, rdd):
        """Advanced aggregation patterns"""
        
        # 1. Custom combiner function for complex aggregations
        def create_combiner(value):
            return {
                'sum': value,
                'count': 1,
                'min': value,
                'max': value,
                'values': [value]
            }
        
        def merge_value(acc, value):
            return {
                'sum': acc['sum'] + value,
                'count': acc['count'] + 1,
                'min': min(acc['min'], value),
                'max': max(acc['max'], value),
                'values': acc['values'] + [value]
            }
        
        def merge_combiners(acc1, acc2):
            return {
                'sum': acc1['sum'] + acc2['sum'],
                'count': acc1['count'] + acc2['count'],
                'min': min(acc1['min'], acc2['min']),
                'max': max(acc1['max'], acc2['max']),
                'values': acc1['values'] + acc2['values']
            }
        
        # Apply custom aggregation
        aggregated = rdd.combineByKey(
            create_combiner,
            merge_value,
            merge_combiners
        )
        
        # Calculate final statistics
        def calculate_stats(acc):
            values = acc['values']
            mean = acc['sum'] / acc['count']
            variance = sum((x - mean) ** 2 for x in values) / acc['count']
            std_dev = variance ** 0.5
            
            return {
                'count': acc['count'],
                'sum': acc['sum'],
                'mean': mean,
                'min': acc['min'],
                'max': acc['max'],
                'std_dev': std_dev,
                'median': sorted(values)[len(values) // 2]
            }
        
        final_stats = aggregated.mapValues(calculate_stats)
        return final_stats
    
    def sliding_window_operations(self, rdd, window_size=3):
        """Implement sliding window operations on RDD"""
        
        def sliding_window(iterator):
            """Create sliding windows from iterator"""
            items = list(iterator)
            for i in range(len(items) - window_size + 1):
                yield items[i:i + window_size]
        
        # Apply sliding window to each partition
        windowed_rdd = rdd.mapPartitions(sliding_window)
        
        # Example: Calculate moving average
        def calculate_moving_average(window):
            values = [item[1] for item in window if isinstance(item, tuple)]
            if values:
                return (window[0][0], sum(values) / len(values))
            return (window[0][0], 0)
        
        moving_averages = windowed_rdd.map(calculate_moving_average)
        return moving_averages
    
    def custom_sampling_strategies(self, rdd):
        """Advanced sampling techniques"""
        
        # 1. Stratified sampling
        def stratified_sample(rdd, strata_col_index=0, sample_size_per_stratum=100):
            """Stratified sampling by key"""
            
            # Group by strata
            stratified = rdd.groupByKey()
            
            # Sample from each stratum
            def sample_stratum(kv_pair):
                key, values = kv_pair
                values_list = list(values)
                sample_size = min(sample_size_per_stratum, len(values_list))
                
                # Random sampling without replacement
                import random
                sampled = random.sample(values_list, sample_size)
                return [(key, value) for value in sampled]
            
            sampled_rdd = stratified.flatMap(sample_stratum)
            return sampled_rdd
        
        # 2. Reservoir sampling for streaming data
        def reservoir_sample(iterator, k=1000):
            """Reservoir sampling algorithm"""
            import random
            
            reservoir = []
            for i, item in enumerate(iterator):
                if i < k:
                    reservoir.append(item)
                else:
                    # Replace element with probability k/i
                    j = random.randint(0, i)
                    if j < k:
                        reservoir[j] = item
            
            return reservoir
        
        # Apply reservoir sampling to each partition
        reservoir_sampled = rdd.mapPartitions(lambda it: reservoir_sample(it, 1000))
        return reservoir_sampled
    
    def advanced_caching_strategies(self, rdd):
        """Advanced caching and persistence strategies"""
        
        from pyspark import StorageLevel
        
        # 1. Conditional caching based on RDD characteristics
        def smart_cache(rdd):
            # Estimate RDD size and access pattern
            num_partitions = rdd.getNumPartitions()
            
            if num_partitions < 10:
                # Small RDD - use memory only
                return rdd.persist(StorageLevel.MEMORY_ONLY)
            elif num_partitions < 100:
                # Medium RDD - use memory and disk
                return rdd.persist(StorageLevel.MEMORY_AND_DISK)
            else:
                # Large RDD - use serialized storage
                return rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)
        
        # 2. Tiered caching strategy
        def tiered_cache(rdd, tier="hot"):
            if tier == "hot":
                return rdd.persist(StorageLevel.MEMORY_ONLY)
            elif tier == "warm":
                return rdd.persist(StorageLevel.MEMORY_AND_DISK)
            elif tier == "cold":
                return rdd.persist(StorageLevel.DISK_ONLY)
            else:
                return rdd
        
        return smart_cache(rdd)

# Usage examples
sc = spark.sparkContext

# Create sample RDDs
rdd1 = sc.parallelize([(i, f"value_{i}") for i in range(1000)])
rdd2 = sc.parallelize([(i, f"data_{i}") for i in range(500, 1500)])

advanced_ops = AdvancedRDDOperations(sc)

# Advanced join
joined_rdd = advanced_ops.advanced_join_strategies(rdd1, rdd2)

# Advanced aggregations
numeric_rdd = sc.parallelize([("A", 1), ("B", 2), ("A", 3), ("B", 4), ("A", 5)])
stats_rdd = advanced_ops.advanced_aggregations(numeric_rdd)

# Sliding window
window_rdd = advanced_ops.sliding_window_operations(numeric_rdd, window_size=2)

print("Advanced RDD operations completed")
````

---

## Catalyst Optimizer Deep Dive

### Understanding Catalyst Internals

```python
class CatalystOptimizerAnalysis:
    """Analyze and understand Catalyst optimizer behavior"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
    
    def analyze_query_plan(self, df, description=""):
        """Detailed analysis of query execution plan"""
        
        print(f"\n{'='*60}")
        print(f"QUERY PLAN ANALYSIS: {description}")
        print(f"{'='*60}")
        
        # 1. Logical Plan
        print("\n1. LOGICAL PLAN:")
        print("-" * 40)
        logical_plan = df._jdf.logicalPlan()
        print(logical_plan.toString())
        
        # 2. Optimized Logical Plan
        print("\n2. OPTIMIZED LOGICAL PLAN:")
        print("-" * 40)
        optimized_plan = df._jdf.queryExecution().optimizedPlan()
        print(optimized_plan.toString())
        
        # 3. Physical Plan
        print("\n3. PHYSICAL PLAN:")
        print("-" * 40)
        df.explain(extended=True)
        
        # 4. Code Generation
        print("\n4. GENERATED CODE:")
        print("-" * 40)
        try:
            code_gen = df._jdf.queryExecution().debug().codegen()
            print(code_gen[:500] + "..." if len(code_gen) > 500 else code_gen)
        except:
            print("Code generation not available for this query")
        
        return {
            "logical_plan": str(logical_plan),
            "optimized_plan": str(optimized_plan)
        }
    
    def demonstrate_optimizations(self):
        """Demonstrate various Catalyst optimizations"""
        
        # Create sample data
        large_df = spark.range(1000000).toDF("id") \
            .withColumn("category", (col("id") % 10).cast("string")) \
            .withColumn("value", (col("id") * 1.5).cast("double"))
        
        small_df = spark.range(10).toDF("category_id") \
            .withColumn("category", col("category_id").cast("string")) \
            .withColumn("category_name", concat(lit("Category_"), col("category")))
        
        # 1. Predicate Pushdown
        print("\n" + "="*60)
        print("PREDICATE PUSHDOWN OPTIMIZATION")
        print("="*60)
        
        filtered_df = large_df.filter(col("id") > 100000).filter(col("category") == "5")
        self.analyze_query_plan(filtered_df, "Predicate Pushdown")
        
        # 2. Column Pruning
        print("\n" + "="*60)
        print("COLUMN PRUNING OPTIMIZATION")
        print("="*60)
        
        pruned_df = large_df.select("id", "category")
        self.analyze_query_plan(pruned_df, "Column Pruning")
        
        # 3. Broadcast Join
        print("\n" + "="*60)
        print("BROADCAST JOIN OPTIMIZATION")
        print("="*60)
        
        from pyspark.sql.functions import broadcast
        broadcast_join_df = large_df.join(broadcast(small_df), "category")
        self.analyze_query_plan(broadcast_join_df, "Broadcast Join")
        
        # 4. Constant Folding
        print("\n" + "="*60)
        print("CONSTANT FOLDING OPTIMIZATION")
        print("="*60)
        
        constant_folded_df = large_df.withColumn("computed", lit(5) + lit(3) * lit(2))
        self.analyze_query_plan(constant_folded_df, "Constant Folding")
        
        # 5. Filter Pushdown through Join
        print("\n" + "="*60)
        print("FILTER PUSHDOWN THROUGH JOIN")
        print("="*60)
        
        join_filter_df = large_df.join(small_df, "category") \
                                 .filter(col("id") > 500000)
        self.analyze_query_plan(join_filter_df, "Filter Pushdown Through Join")
    
    def custom_optimization_rules(self):
        """Demonstrate custom optimization rules"""
        
        # Example: Custom rule to optimize specific patterns
        # Note: This is conceptual - actual custom rules require Scala implementation
        
        print("\n" + "="*60)
        print("CUSTOM OPTIMIZATION CONCEPTS")
        print("="*60)
        
        # 1. Rule-based optimization patterns
        optimization_patterns = {
            "redundant_projections": "Remove unnecessary projections",
            "subquery_elimination": "Eliminate correlated subqueries",
            "join_reordering": "Reorder joins based on selectivity",
            "aggregate_pushdown": "Push aggregations closer to source"
        }
        
        for pattern, description in optimization_patterns.items():
            print(f"- {pattern}: {description}")
        
        # 2. Cost-based optimization hints
        cbo_hints = [
            "/*+ BROADCAST(small_table) */",
            "/*+ MERGE(table1, table2) */", 
            "/*+ SHUFFLE_HASH(table1) */",
            "/*+ SORT_MERGE(table1, table2) */"
        ]
        
        print(f"\nCost-Based Optimization Hints:")
        for hint in cbo_hints:
            print(f"  {hint}")
    
    def performance_impact_analysis(self):
        """Analyze performance impact of different optimizations"""
        
        import time
        
        # Create test data
        df = spark.range(100000).toDF("id") \
            .withColumn("category", (col("id") % 100).cast("string")) \
            .withColumn("value", col("id") * 2.5)
        
        lookup_df = spark.range(100).toDF("cat_id") \
            .withColumn("category", col("cat_id").cast("string")) \
            .withColumn("description", concat(lit("Desc_"), col("cat_id")))
        
        # Test scenarios
        scenarios = {
            "unoptimized_join": lambda: df.join(lookup_df, df.category == lookup_df.category),
            "broadcast_join": lambda: df.join(broadcast(lookup_df), df.category == lookup_df.category),
            "filtered_first": lambda: df.filter(col("id") > 50000).join(lookup_df, "category"),
            "aggregated": lambda: df.groupBy("category").agg(avg("value").alias("avg_value"))
        }
        
        results = {}
        
        for scenario_name, scenario_func in scenarios.items():
            print(f"\nTesting scenario: {scenario_name}")
            
            start_time = time.time()
            result_df = scenario_func()
            count = result_df.count()  # Force execution
            end_time = time.time()
            
            execution_time = end_time - start_time
            results[scenario_name] = {
                "execution_time": execution_time,
                "row_count": count
            }
            
            print(f"  Execution time: {execution_time:.2f}s")
            print(f"  Row count: {count}")
        
        # Compare results
        print(f"\n{'Scenario':<20} {'Time (s)':<10} {'Rows':<10}")
        print("-" * 40)
        for scenario, metrics in results.items():
            print(f"{scenario:<20} {metrics['execution_time']:<10.2f} {metrics['row_count']:<10}")
        
        return results

# Usage example
catalyst_analyzer = CatalystOptimizerAnalysis(spark)

# Run comprehensive analysis
catalyst_analyzer.demonstrate_optimizations()
catalyst_analyzer.custom_optimization_rules()
performance_results = catalyst_analyzer.performance_impact_analysis()
```

---

## Delta Lake Advanced Features

### Delta Lake Time Travel and Version Control

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import *

class DeltaLakeAdvanced:
    """Advanced Delta Lake operations and patterns"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
    
    def create_versioned_dataset(self, data_path, initial_data):
        """Create a versioned dataset with Delta Lake"""
        
        # Initial write
        initial_data.write \
            .format("delta") \
            .mode("overwrite") \
            .save(data_path)
        
        print(f"Created initial Delta table at {data_path}")
        
        # Create Delta table reference
        delta_table = DeltaTable.forPath(self.spark, data_path)
        return delta_table
    
    def demonstrate_time_travel(self, delta_table_path):
        """Demonstrate time travel capabilities"""
        
        print("\n" + "="*60)
        print("DELTA LAKE TIME TRAVEL DEMO")
        print("="*60)
        
        # Read current version
        current_df = self.spark.read.format("delta").load(delta_table_path)
        print(f"Current version rows: {current_df.count()}")
        
        # Read specific version
        version_0_df = self.spark.read \
            .format("delta") \
            .option("versionAsOf", 0) \
            .load(delta_table_path)
        print(f"Version 0 rows: {version_0_df.count()}")
        
        # Read as of timestamp
        from datetime import datetime, timedelta
        yesterday = datetime.now() - timedelta(days=1)
        
        try:
            historical_df = self.spark.read \
                .format("delta") \
                .option("timestampAsOf", yesterday.strftime("%Y-%m-%d")) \
                .load(delta_table_path)
            print(f"Historical version rows: {historical_df.count()}")
        except:
            print("No historical data available for yesterday")
        
        # Show table history
        delta_table = DeltaTable.forPath(self.spark, delta_table_path)
        history_df = delta_table.history()
        
        print("\nTable History:")
        history_df.select("version", "timestamp", "operation", "operationMetrics").show(truncate=False)
        
        return history_df
    
    def advanced_merge_operations(self, delta_table_path, updates_df):
        """Advanced merge operations with complex conditions"""
        
        delta_table = DeltaTable.forPath(self.spark, delta_table_path)
        
        # Complex merge with multiple conditions
        merge_result = delta_table.alias("target") \
            .merge(
                updates_df.alias("source"),
                "target.id = source.id"
            ) \
            .whenMatchedUpdate(
                condition="source.last_updated > target.last_updated",
                set={
                    "value": "source.value",
                    "last_updated": "source.last_updated",
                    "update_count": "target.update_count + 1"
                }
            ) \
            .whenMatchedDelete(
                condition="source.is_deleted = true"
            ) \
            .whenNotMatchedInsert(
                condition="source.is_deleted = false",
                values={
                    "id": "source.id",
                    "value": "source.value",
                    "last_updated": "source.last_updated",
                    "update_count": "1"
                }
            ) \
            .execute()
        
        print("Advanced merge operation completed")
        
        # Get merge metrics
        last_operation = delta_table.history(1).collect()[0]
        metrics = last_operation["operationMetrics"]
        
        print(f"Merge metrics:")
        for metric, value in metrics.asDict().items():
            print(f"  {metric}: {value}")
        
        return merge_result
    
    def schema_evolution_demo(self, delta_table_path):
        """Demonstrate schema evolution capabilities"""
        
        print("\n" + "="*60)
        print("SCHEMA EVOLUTION DEMO")
        print("="*60)
        
        delta_table = DeltaTable.forPath(self.spark, delta_table_path)
        
        # Show current schema
        current_schema = self.spark.read.format("delta").load(delta_table_path).schema
        print("Current schema:")
        for field in current_schema.fields:
            print(f"  {field.name}: {field.dataType}")
        
        # Add new columns with schema evolution
        new_data = self.spark.createDataFrame([
            (1001, "new_value", "2024-01-01", 1, "new_column_value", 42.0),
            (1002, "another_value", "2024-01-02", 1, "another_new_value", 43.5)
        ], ["id", "value", "last_updated", "update_count", "new_string_col", "new_numeric_col"])
        
        # Write with schema evolution enabled
        new_data.write \
            .format("delta") \
            .mode("append") \
            .option("mergeSchema", "true") \
            .save(delta_table_path)
        
        print("\nSchema after evolution:")
        evolved_schema = self.spark.read.format("delta").load(delta_table_path).schema
        for field in evolved_schema.fields:
            print(f"  {field.name}: {field.dataType}")
        
        return evolved_schema
    
    def optimize_delta_table(self, delta_table_path):
        """Optimize Delta table with compaction and Z-ordering"""
        
        print("\n" + "="*60)
        print("DELTA TABLE OPTIMIZATION")
        print("="*60)
        
        # File compaction
        print("Running OPTIMIZE command...")
        self.spark.sql(f"OPTIMIZE delta.`{delta_table_path}`")
        
        # Z-ordering for better query performance
        print("Running Z-ORDER optimization...")
        self.spark.sql(f"OPTIMIZE delta.`{delta_table_path}` ZORDER BY (id)")
        
        # Vacuum old files (be careful in production)
        print("Running VACUUM command...")
        self.spark.sql(f"VACUUM delta.`{delta_table_path}` RETAIN 168 HOURS")  # 7 days
        
        # Show table details
        print("\nTable details after optimization:")
        table_details = self.spark.sql(f"DESCRIBE DETAIL delta.`{delta_table_path}`")
        table_details.show(truncate=False)
    
    def data_quality_constraints(self, delta_table_path):
        """Implement data quality constraints"""
        
        delta_table = DeltaTable.forPath(self.spark, delta_table_path)
        
        # Add table constraints
        constraints = [
            ("id_not_null", "id IS NOT NULL"),
            ("id_positive", "id > 0"),
            ("value_length", "LENGTH(value) > 0"),
            ("update_count_valid", "update_count >= 1")
        ]
        
        for constraint_name, constraint_expr in constraints:
            try:
                self.spark.sql(f"""
                    ALTER TABLE delta.`{delta_table_path}` 
                    ADD CONSTRAINT {constraint_name} 
                    CHECK ({constraint_expr})
                """)
                print(f"Added constraint: {constraint_name}")
            except Exception as e:
                print(f"Failed to add constraint {constraint_name}: {e}")
        
        # Test constraint violation
        try:
            invalid_data = self.spark.createDataFrame([
                (-1, "invalid", "2024-01-01", 0)  # Violates id_positive and update_count_valid
            ], ["id", "value", "last_updated", "update_count"])
            
            invalid_data.write \
                .format("delta") \
                .mode("append") \
                .save(delta_table_path)
                
        except Exception as e:
            print(f"Constraint violation caught: {e}")
    
    def streaming_with_delta(self, input_stream, output_path):
        """Integrate Delta Lake with streaming"""
        
        # Streaming write to Delta
        streaming_query = input_stream \
            .writeStream \
            .format("delta") \
            .outputMode("append") \
            .option("checkpointLocation", f"{output_path}_checkpoint") \
            .option("path", output_path) \
            .trigger(processingTime="30 seconds") \
            .start()
        
        print(f"Started streaming write to Delta table: {output_path}")
        
        # Streaming read from Delta (for downstream processing)
        delta_stream = self.spark \
            .readStream \
            .format("delta") \
            .load(output_path)
        
        # Process streaming data
        processed_stream = delta_stream \
            .groupBy(window(col("timestamp"), "5 minutes")) \
            .agg(count("*").alias("event_count"))
        
        processed_query = processed_stream \
            .writeStream \
            .format("console") \
            .outputMode("update") \
            .trigger(processingTime="1 minute") \
            .start()
        
        return streaming_query, processed_query

# Usage example
delta_advanced = DeltaLakeAdvanced(spark)

# Create sample data
initial_data = spark.createDataFrame([
    (1, "value1", "2024-01-01", 1),
    (2, "value2", "2024-01-01", 1),
    (3, "value3", "2024-01-01", 1)
], ["id", "value", "last_updated", "update_count"])

# Create versioned dataset
# delta_table_path = "/tmp/delta_advanced_demo"
# delta_table = delta_advanced.create_versioned_dataset(delta_table_path, initial_data)

# Demonstrate various features
# history = delta_advanced.demonstrate_time_travel(delta_table_path)
# schema = delta_advanced.schema_evolution_demo(delta_table_path)
# delta_advanced.optimize_delta_table(delta_table_path)
# delta_advanced.data_quality_constraints(delta_table_path)

print("Delta Lake advanced features demonstration completed")
```

---

## Advanced Analytics and Aggregations

### Complex Analytical Functions

````python
class AdvancedAnalytics:
    """Advanced analytical functions and patterns"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
    
    def cohort_analysis(self, user_events_df):
        """Perform cohort analysis on user events"""
        
        # Define first purchase month as cohort
        first_purchase = user_events_df \
            .filter(col("event_type") == "purchase") \
            .groupBy("user_id") \
            .agg(min("timestamp").alias("first_purchase_date"))
        
        # Add cohort information to events
        cohort_data = user_events_df \
            .join(first_purchase, "user_id") \
            .withColumn("cohort_month", 
                       date_trunc("month", col("first_purchase_date"))) \
            .withColumn("event_month", 
                       date_trunc("month", col("timestamp"))) \
            .withColumn("period_number",
                       months_between(col("event_month"), col("cohort_month")))
        
        # Calculate cohort metrics
        cohort_table = cohort_data \
            .groupBy("cohort_month", "period_number") \
            .agg(
                countDistinct("user_id").alias("active_users"),
                sum(when(col("event_type") == "purchase", col("value")).otherwise(0)).alias("revenue")
            )
        
        # Calculate cohort sizes
        cohort_sizes = cohort_data \
            .filter(col("period_number") == 0) \
            .groupBy("cohort_month") \
            .agg(countDistinct("user_id").alias("cohort_size"))
        
        # Calculate retention rates
        retention_rates = cohort_table \
            .join(cohort_sizes, "cohort_month") \
            .withColumn("retention_rate", 
                       col("active_users") / col("cohort_size"))
        
        return retention_rates
    
    def customer_segmentation_rfm(self, transaction_data):
        """RFM (Recency, Frequency, Monetary) analysis for customer segmentation"""
        
        from pyspark.sql.window import Window
        
        # Calculate RFM metrics
        current_date = lit("2024-01-01")  # Use appropriate current date
        
        rfm_metrics = transaction_data \
            .groupBy("customer_id") \
            .agg(
                datediff(current_date, max("transaction_date")).alias("recency"),
                count("transaction_id").alias("frequency"),
                sum("amount").alias("monetary")
            )
        
        # Calculate quintiles for each metric
        window_spec = Window.orderBy("recency")
        rfm_with_scores = rfm_metrics \
            .withColumn("recency_score", 
                       ntile(5).over(Window.orderBy(col("recency").desc()))) \
            .withColumn("frequency_score", 
                       ntile(5).over(Window.orderBy("frequency"))) \
            .withColumn("monetary_score", 
                       ntile(5).over(Window.orderBy("monetary")))
        
        # Create R# PySpark Advanced Features

#pyspark #advanced #ml #graph #streaming #delta-lake #catalyst #rdd #optimization #enterprise

## Overview

This note covers advanced PySpark features that go beyond basic data processing, including machine learning integration, graph processing, advanced RDD operations, Catalyst optimizer internals, Delta Lake integration, and enterprise-grade features. These capabilities enable sophisticated data science workflows and production-scale analytics.

---

## Machine Learning with MLlib

### MLlib Pipeline Architecture

```python
from pyspark.ml import Pipeline, PipelineModel
from pyspark.ml.feature import *
from pyspark.ml.classification import *
from pyspark.ml.regression import *
from pyspark.ml.clustering import *
from pyspark.ml.evaluation import *
from pyspark.ml.tuning import *

# Create a comprehensive ML pipeline
def create_ml_pipeline():
    """Create a complete ML pipeline with feature engineering and model training"""
    
    # Feature engineering stages
    string_indexer = StringIndexer(inputCol="category", outputCol="category_index")
    one_hot_encoder = OneHotEncoder(inputCol="category_index", outputCol="category_vector")
    
    # Numerical feature processing
    assembler_numerical = VectorAssembler(
        inputCols=["age", "income", "score"],
        outputCol="numerical_features"
    )
    
    scaler = StandardScaler(
        inputCol="numerical_features",
        outputCol="scaled_numerical_features",
        withStd=True,
        withMean=True
    )
    
    # Text processing pipeline
    tokenizer = Tokenizer(inputCol="description", outputCol="words")
    remover = StopWordsRemover(inputCol="words", outputCol="filtered_words")
    hashing_tf = HashingTF(inputCol="filtered_words", outputCol="text_features", numFeatures=10000)
    idf = IDF(inputCol="text_features", outputCol="tfidf_features")
    
    # Final feature assembly
    final_assembler = VectorAssembler(
        inputCols=["category_vector", "scaled_numerical_features", "tfidf_features"],
        outputCol="features"
    )
    
    # Model
    rf = RandomForestClassifier(
        featuresCol="features",
        labelCol="label",
        numTrees=100,
        maxDepth=10,
        seed=42
    )
    
    # Create pipeline
    pipeline = Pipeline(stages=[
        string_indexer,
        one_hot_encoder,
        assembler_numerical,
        scaler,
        tokenizer,
        remover,
        hashing_tf,
        idf,
        final_assembler,
        rf
    ])
    
    return pipeline

# Advanced pipeline with feature selection
def create_advanced_pipeline_with_feature_selection():
    """Pipeline with automatic feature selection and hyperparameter tuning"""
    
    # Basic preprocessing
    string_indexer = StringIndexer(inputCol="category", outputCol="category_index")
    encoder = OneHotEncoder(inputCol="category_index", outputCol="category_vector")
    
    # Assemble all features
    assembler = VectorAssembler(
        inputCols=["category_vector", "feature1", "feature2", "feature3", "feature4", "feature5"],
        outputCol="raw_features"
    )
    
    # Feature selection
    selector = ChiSqSelector(
        featuresCol="raw_features",
        outputCol="selected_features",
        labelCol="label",
        numTopFeatures=10
    )
    
    # Model with hyperparameter tuning
    lr = LogisticRegression(
        featuresCol="selected_features",
        labelCol="label",
        maxIter=100
    )
    
    # Create pipeline
    pipeline = Pipeline(stages=[string_indexer, encoder, assembler, selector, lr])
    
    # Hyperparameter tuning
    param_grid = ParamGridBuilder() \
        .addGrid(selector.numTopFeatures, [5, 10, 15]) \
        .addGrid(lr.regParam, [0.01, 0.1, 1.0]) \
        .addGrid(lr.elasticNetParam, [0.0, 0.5, 1.0]) \
        .build()
    
    # Cross validation
    evaluator = BinaryClassificationEvaluator(
        labelCol="label",
        rawPredictionCol="rawPrediction",
        metricName="areaUnderROC"
    )
    
    cv = CrossValidator(
        estimator=pipeline,
        estimatorParamMaps=param_grid,
        evaluator=evaluator,
        numFolds=5,
        seed=42
    )
    
    return cv

# Usage example
# Create sample data
sample_data = spark.createDataFrame([
    (0, "electronics", 25, 50000, 0.8, "high quality product"),
    (1, "clothing", 30, 60000, 0.9, "comfortable and stylish"),
    (0, "books", 45, 40000, 0.7, "educational content"),
    (1, "electronics", 35, 70000, 0.95, "latest technology")
], ["label", "category", "age", "income", "score", "description"])

# Train pipeline
pipeline = create_ml_pipeline()
model = pipeline.fit(sample_data)

# Make predictions
predictions = model.transform(sample_data)
predictions.select("label", "prediction", "probability").show()
````

### Advanced Model Evaluation and Selection

```python
class MLModelEvaluator:
    """Comprehensive model evaluation framework"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.models = {}
        self.evaluation_results = {}
    
    def evaluate_classification_models(self, models_dict, test_data, label_col="label"):
        """Evaluate multiple classification models"""
        
        evaluators = {
            "auc": BinaryClassificationEvaluator(
                labelCol=label_col, 
                rawPredictionCol="rawPrediction",
                metricName="areaUnderROC"
            ),
            "accuracy": MulticlassClassificationEvaluator(
                labelCol=label_col,
                predictionCol="prediction",
                metricName="accuracy"
            ),
            "f1": MulticlassClassificationEvaluator(
                labelCol=label_col,
                predictionCol="prediction",
                metricName="f1"
            ),
            "precision": MulticlassClassificationEvaluator(
                labelCol=label_col,
                predictionCol="prediction",
                metricName="weightedPrecision"
            ),
            "recall": MulticlassClassificationEvaluator(
                labelCol=label_col,
                predictionCol="prediction",
                metricName="weightedRecall"
            )
        }
        
        results = {}
        
        for model_name, model in models_dict.items():
            print(f"Evaluating {model_name}...")
            
            # Make predictions
            predictions = model.transform(test_data)
            
            # Calculate metrics
            model_metrics = {}
            for metric_name, evaluator in evaluators.items():
                try:
                    score = evaluator.evaluate(predictions)
                    model_metrics[metric_name] = score
                except Exception as e:
                    print(f"Could not calculate {metric_name} for {model_name}: {e}")
                    model_metrics[metric_name] = None
            
            # Additional metrics
            model_metrics.update(self._calculate_additional_metrics(predictions, label_col))
            
            results[model_name] = model_metrics
            
        self.evaluation_results = results
        return results
    
    def _calculate_additional_metrics(self, predictions, label_col):
        """Calculate additional custom metrics"""
        
        # Confusion matrix elements
        tp = predictions.filter((col(label_col) == 1) & (col("prediction") == 1)).count()
        tn = predictions.filter((col(label_col) == 0) & (col("prediction") == 0)).count()
        fp = predictions.filter((col(label_col) == 0) & (col("prediction") == 1)).count()
        fn = predictions.filter((col(label_col) == 1) & (col("prediction") == 0)).count()
        
        # Calculate metrics
        precision = tp / (tp + fp) if (tp + fp) > 0 else 0
        recall = tp / (tp + fn) if (tp + fn) > 0 else 0
        specificity = tn / (tn + fp) if (tn + fp) > 0 else 0
        
        return {
            "true_positives": tp,
            "true_negatives": tn,
            "false_positives": fp,
            "false_negatives": fn,
            "precision_manual": precision,
            "recall_manual": recall,
            "specificity": specificity
        }
    
    def compare_models(self, primary_metric="auc"):
        """Compare models and rank by primary metric"""
        
        if not self.evaluation_results:
            print("No evaluation results available. Run evaluate_classification_models first.")
            return None
        
        # Sort models by primary metric
        sorted_models = sorted(
            self.evaluation_results.items(),
            key=lambda x: x[1].get(primary_metric, 0),
            reverse=True
        )
        
        print(f"\nModel Comparison (ranked by {primary_metric}):")
        print("-" * 80)
        print(f"{'Model':<20} {primary_metric.upper():<10} {'Accuracy':<10} {'F1':<10} {'Precision':<12} {'Recall':<10}")
        print("-" * 80)
        
        for model_name, metrics in sorted_models:
            print(f"{model_name:<20} "
                  f"{metrics.get(primary_metric, 0):<10.4f} "
                  f"{metrics.get('accuracy', 0):<10.4f} "
                  f"{metrics.get('f1', 0):<10.4f} "
                  f"{metrics.get('precision', 0):<12.4f} "
                  f"{metrics.get('recall', 0):<10.4f}")
        
        return sorted_models
    
    def feature_importance_analysis(self, model, feature_names):
        """Analyze feature importance for tree-based models"""
        
        try:
            if hasattr(model, 'featureImportances'):
                importances = model.featureImportances.toArray()
                
                feature_importance_df = self.spark.createDataFrame([
                    (feature_names[i], float(importances[i]))
                    for i in range(len(feature_names))
                ], ["feature", "importance"])
                
                return feature_importance_df.orderBy(col("importance").desc())
            else:
                print("Model does not support feature importance analysis")
                return None
                
        except Exception as e:
            print(f"Error analyzing feature importance: {e}")
            return None

# Usage example
evaluator = MLModelEvaluator(spark)

# Train multiple models
models = {
    "RandomForest": RandomForestClassifier(numTrees=100).fit(train_data),
    "LogisticRegression": LogisticRegression().fit(train_data),
    "GBT": GBTClassifier(maxIter=50).fit(train_data)
}

# Evaluate all models
results = evaluator.evaluate_classification_models(models, test_data)

# Compare models
best_models = evaluator.compare_models("auc")

# Analyze feature importance
feature_names = ["feature1", "feature2", "feature3", "feature4"]
importance_df = evaluator.feature_importance_analysis(models["RandomForest"], feature_names)
if importance_df:
    importance_df.show()
```

### Online Learning and Model Updates

```python
class OnlineLearningFramework:
    """Framework for online learning with streaming data"""
    
    def __init__(self, spark_session, model_path, checkpoint_path):
        self.spark = spark_session
        self.model_path = model_path
        self.checkpoint_path = checkpoint_path
        self.current_model = None
        self.model_version = 0
    
    def initialize_model(self, initial_training_data):
        """Initialize model with historical data"""
        
        # Create initial pipeline
        pipeline = Pipeline(stages=[
            VectorAssembler(inputCols=["feature1", "feature2", "feature3"], outputCol="features"),
            LogisticRegression(featuresCol="features", labelCol="label")
        ])
        
        # Train initial model
        self.current_model = pipeline.fit(initial_training_data)
        self.save_model()
        
        print(f"Initialized model version {self.model_version}")
        return self.current_model
    
    def update_model_streaming(self, streaming_data):
        """Update model with streaming data using micro-batches"""
        
        def update_model_batch(batch_df, batch_id):
            """Update model for each micro-batch"""
            
            if batch_df.count() > 0:
                print(f"Updating model with batch {batch_id}, records: {batch_df.count()}")
                
                try:
                    # Combine with historical data (simplified approach)
                    # In production, you might use more sophisticated incremental learning
                    
                    # For demonstration, retrain on recent batches
                    if batch_df.count() >= 100:  # Minimum batch size for retraining
                        
                        # Create new pipeline (same structure)
                        pipeline = Pipeline(stages=[
                            VectorAssembler(inputCols=["feature1", "feature2", "feature3"], outputCol="features"),
                            LogisticRegression(featuresCol="features", labelCol="label")
                        ])
                        
                        # Retrain model
                        updated_model = pipeline.fit(batch_df)
                        
                        # Evaluate model performance
                        if self._validate_model_performance(updated_model, batch_df):
                            self.current_model = updated_model
                            self.model_version += 1
                            self.save_model()
                            print(f"Model updated to version {self.model_version}")
                        else:
                            print("Model update rejected due to performance degradation")
                    
                except Exception as e:
                    print(f"Error updating model: {e}")
        
        # Start streaming query for model updates
        query = streaming_data.writeStream \
            .foreachBatch(update_model_batch) \
            .option("checkpointLocation", f"{self.checkpoint_path}/model_updates") \
            .trigger(processingTime="5 minutes") \
            .start()
        
        return query
    
    def _validate_model_performance(self, new_model, validation_data):
        """Validate new model performance against current model"""
        
        if self.current_model is None:
            return True  # First model
        
        try:
            # Evaluate current model
            current_predictions = self.current_model.transform(validation_data)
            current_evaluator = BinaryClassificationEvaluator(
                labelCol="label", 
                rawPredictionCol="rawPrediction"
            )
            current_auc = current_evaluator.evaluate(current_predictions)
            
            # Evaluate new model
            new_predictions = new_model.transform(validation_data)
            new_auc = current_evaluator.evaluate(new_predictions)
            
            # Accept new model if it's better or within acceptable degradation
            improvement_threshold = -0.05  # Allow 5% degradation
            performance_change = new_auc - current_auc
            
            print(f"Current AUC: {current_auc:.4f}, New AUC: {new_auc:.4f}, Change: {performance_change:.4f}")
            
            return performance_change >= improvement_threshold
            
        except Exception as e:
            print(f"Error validating model: {e}")
            return False
    
    def save_model(self):
        """Save current model to persistent storage"""
        
        if self.current_model:
            model_version_path = f"{self.model_path}/version_{self.model_version}"
            self.current_model.write().overwrite().save(model_version_path)
            
            # Save metadata
            metadata = {
                "version": self.model_version,
                "timestamp": datetime.now().isoformat(),
                "model_path": model_version_path
            }
            
            metadata_df = self.spark.createDataFrame([metadata])
            metadata_df.write.mode("append").json(f"{self.model_path}/metadata")
    
    def load_model(self, version=None):
        """Load model from persistent storage"""
        
        if version is None:
            version = self.model_version
        
        try:
            model_path = f"{self.model_path}/version_{version}"
            self.current_model = PipelineModel.load(model_path)
            self.model_version = version
            print(f"Loaded model version {version}")
            return self.current_model
        except Exception as e:
            print(f"Error loading model: {e}")
            return None
    
    def predict_streaming(self, streaming_data):
        """Make predictions on streaming data"""
        
        def make_predictions(batch_df, batch_id):
            """Make predictions for each batch"""
            
            if batch_df.count() > 0 and self.current_model:
                predictions = self.current_model.transform(batch_df)
                
                # Save predictions
                predictions.select("id", "features", "prediction", "probability") \
                    .write \
                    .mode("append") \
                    .parquet(f"{self.model_path}/predictions")
                
                print(f"Made predictions for batch {batch_id}: {batch_df.count()} records")
        
        query = streaming_data.writeStream \
            .foreachBatch(make_predictions) \
            .option("checkpointLocation", f"{self.checkpoint_path}/predictions") \
            .trigger(processingTime="1 minute") \
            .start()
        
        return query

# Usage example
online_learner = OnlineLearningFramework(
    spark, 
    "/models/online_learning", 
    "/checkpoints/online_learning"
)

# Initialize with historical data
# initial_model = online_learner.initialize_model(historical_data)

# Start online learning with streaming data
# update_query = online_learner.update_model_streaming(streaming_training_data)

# Start making predictions
# prediction_query = online_learner.predict_streaming(streaming_inference_data)
```

---

## Graph Processing with GraphFrames

### GraphFrames Fundamentals

```python
# Install GraphFrames if needed
# spark.sparkContext.addPyFile("graphframes-0.8.2-spark3.0-s_2.12.jar")

from graphframes import GraphFrame
from pyspark.sql.functions import *

def create_sample_graph():
    """Create a sample social network graph"""
    
    # Vertices (users)
    vertices = spark.createDataFrame([
        ("1", "Alice", 25, "Engineer"),
        ("2", "Bob", 30, "Manager"),
        ("3", "Charlie", 28, "Designer"),
        ("4", "Diana", 32, "Analyst"),
        ("5", "Eve", 27, "Developer"),
        ("6", "Frank", 35, "Director")
    ], ["id", "name", "age", "role"])
    
    # Edges (relationships)
    edges = spark.createDataFrame([
        ("1", "2", "friend", 0.8),
        ("1", "3", "colleague", 0.6),
        ("2", "3", "friend", 0.9),
        ("2", "4", "manager", 1.0),
        ("3", "5", "friend", 0.7),
        ("4", "5", "colleague", 0.5),
        ("4", "6", "reports_to", 1.0),
        ("5", "6", "colleague", 0.4),
        ("1", "5", "friend", 0.8)
    ], ["src", "dst", "relationship", "weight"])
    
    # Create GraphFrame
    graph = GraphFrame(vertices, edges)
    return graph

# Advanced graph analytics
class GraphAnalytics:
    """Advanced graph analytics using GraphFrames"""
    
    def __init__(self, graph):
        self.graph = graph
    
    def basic_graph_statistics(self):
        """Calculate basic graph statistics"""
        
        stats = {
            "num_vertices": self.graph.vertices.count(),
            "num_edges": self.graph.edges.count(),
            "avg_degree": self.graph.degrees.agg(avg("degree")).collect()[0][0]
        }
        
        print(f"Graph Statistics:")
        print(f"  Vertices: {stats['num_vertices']}")
        print(f"  Edges: {stats['num_edges']}")
        print(f"  Average Degree: {stats['avg_degree']:.2f}")
        
        return stats
    
    def find_influential_users(self, algorithm="pagerank"):
        """Find influential users using centrality measures"""
        
        if algorithm == "pagerank":
            # PageRank algorithm
            pagerank_results = self.graph.pageRank(resetProbability=0.15, maxIter=10)
            
            influential_users = pagerank_results.vertices \
                .select("id", "name", "pagerank") \
                .orderBy(col("pagerank").desc()) \
                .limit(10)
            
            print("Top 10 Influential Users (PageRank):")
            influential_users.show()
            
            return influential_users
        
        elif algorithm == "degree_centrality":
            # Degree centrality
            degree_centrality = self.graph.degrees \
                .join(self.graph.vertices, "id") \
                .select("id", "name", "degree") \
                .orderBy(col("degree").desc())
            
            print("Users by Degree Centrality:")
            degree_centrality.show()
            
            return degree_centrality
    
    def community_detection(self):
        """Detect communities using Label Propagation Algorithm"""
        
        # Label Propagation Algorithm
        communities = self.graph.labelPropagation(maxIter=5)
        
        # Analyze communities
        community_stats = communities.groupBy("label") \
            .agg(
                count("*").alias("community_size"),
                collect_list("name").alias("members")
            ) \
            .orderBy(col("community_size").desc())
        
        print("Detected Communities:")
        community_stats.show(truncate=False)
        
        return communities, community_stats
    
    def shortest_paths_analysis(self, landmarks):
        """Calculate shortest paths from landmarks"""
        
        # Shortest paths to landmarks
        shortest_paths = self.graph.shortestPaths(landmarks=landmarks)
        
        print(f"Shortest Paths from landmarks {landmarks}:")
        shortest_paths.select("id", "name", "distances").show(truncate=False)
        
        return shortest_paths
    
    def motif_analysis(self):
        """Find common motifs/patterns in the graph"""
        
        # Find triangles (3-cycles)
        triangles = self.graph.find("(a)-[e1]->(b); (b)-[e2]->(c); (c)-[e3]->(a)")
        
        print("Triangles in the graph:")
        triangles.select(
            col("a.name").alias("person1"),
            col("b.name").alias("person2"),
            col("c.name").alias("person3")
        ).show()
        
        # Find mutual friends pattern
        mutual_friends = self.graph.find("(a)-[e1]->(b); (b)-[e2]->(a)")
        
        print("Mutual connections:")
        mutual_friends.select(
            col("a.name").alias("person1"),
            col("b.name").alias("person2"),
            col("e1.relationship").alias("relationship1"),
            col("e2.relationship").alias("relationship2")
        ).show()
        
        return triangles, mutual_friends
    
    def recommendation_engine(self, user_id, max_recommendations=5):
        """Simple friend recommendation based on mutual connections"""
        
        # Find friends of friends
        friends_of_friends = self.graph.find(f"""
            (user)-[e1]->(friend1);
            (friend1)-[e2]->(friend2);
            (friend2)-[e3]->(potential_friend)
        """).filter(f"user.id = '{user_id}' AND potential_friend.id != '{user_id}'")
        
        # Count mutual connections and recommend
        recommendations = friends_of_friends \
            .groupBy("potential_friend.id", "potential_friend.name") \
            .agg(count("*").alias("mutual_connections")) \
            .orderBy(col("mutual_connections").desc()) \
            .limit(max_recommendations)
        
        print(f"Friend recommendations for user {user_id}:")
        recommendations.show()
        
        return recommendations

# Usage example
graph = create_sample_graph()
analytics = GraphAnalytics(graph)

# Perform various analyses
stats = analytics.basic_graph_statistics()
influential = analytics.find_influential_users("pagerank")
communities, community_stats = analytics.community_detection()
recommendations = analytics.recommendation_engine("1")
```

### Advanced Graph Algorithms

```python
class AdvancedGraphAlgorithms:
    """Advanced graph algorithms for complex analytics"""
    
    def __init__(self, graph):
        self.graph = graph
    
    def collaborative_filtering_graph(self, user_item_interactions):
        """Build collaborative filtering using graph-based approach"""
        
        # Create bipartite graph of users and items
        user_vertices = user_item_interactions.select("user_id").distinct() \
            .withColumnRenamed("user_id", "id") \
            .withColumn("type", lit("user"))
        
        item_vertices = user_item_interactions.select("item_id").distinct() \
            .withColumnRenamed("item_id", "id") \
            .withColumn("type", lit("item"))
        
        bipartite_vertices = user_vertices.union(item_vertices)
        
        # Create edges from interactions
        bipartite_edges = user_item_interactions.select(
            col("user_id").alias("src"),
            col("item_id").alias("dst"),
            col("rating").alias("weight")
        ).withColumn("relationship", lit("rates"))
        
        # Create bipartite graph
        bipartite_graph = GraphFrame(bipartite_vertices, bipartite_edges)
        
        # Find item-item similarity through user connections
        item_similarity = bipartite_graph.find("""
            (item1)-[e1]-(user)-[e2]-(item2)
        """).filter("item1.type = 'item' AND item2.type = 'item' AND item1.id != item2.id")
        
        # Calculate similarity scores
        similarity_scores = item_similarity \
            .groupBy("item1.id", "item2.id") \
            .agg(
                count("*").alias("common_users"),
                avg((col("e1.weight") + col("e2.weight")) / 2).alias("avg_rating")
            ) \
            .withColumn("similarity_score", 
                       col("common_users") * col("avg_rating"))
        
        return bipartite_graph, similarity_scores
    
    def fraud_detection_graph(self, transaction_data):
        """Fraud detection using graph analysis"""
        
        # Create transaction graph
        account_vertices = transaction_data.select("from_account").distinct() \
            .withColumnRenamed("from_account", "id") \
            .withColumn("type", lit("account"))
        
        transaction_edges = transaction_data.select(
            col("from_account").alias("src"),
            col("to_account").alias("dst"),
            col("amount").alias("weight"),
            col("timestamp")
        ).withColumn("relationship", lit("transfer"))
        
        transaction_graph = GraphFrame(account_vertices, transaction_edges)
        
        # Detect suspicious patterns
        
        # 1. Circular transfers (money laundering pattern)
        circular_transfers = transaction_graph.find("""
            (a)-[e1]->(b); (b)-[e2]->(c); (c)-[e3]->(a)
        """).filter("""
            e1.timestamp < e2.timestamp AND 
            e2.timestamp < e3.timestamp AND
            abs(e1.weight - e3.weight) / e1.weight < 0.1
        """)  # Similar amounts
        
        # 2. Hub accounts (accounts with many connections)
        hub_accounts = transaction_graph.degrees \
            .filter(col("degree") > 10) \
            .orderBy(col("degree").desc())
        
        # 3. Burst activity (many transactions in short time)
        burst_activity = transaction_edges \
            .groupBy("src", window(col("timestamp"), "1 hour")) \
            .agg(
                count("*").alias("transactions_per_hour"),
                sum("weight").alias("total_amount")
            ) \
            .filter(col("transactions_per_hour") > 20)
        
        return {
            "graph": transaction_graph,
            "circular_transfers": circular_transfers,
            "hub_accounts": hub_accounts,
            "burst_activity": burst_activity
        }
    
    def supply_chain_analysis(self, supply_chain_data):
        """Analyze supply chain networks for optimization"""
        
        # Create supply chain graph
        supplier_vertices = supply_chain_data.select("supplier_id", "supplier_name", "location") \
            .distinct() \
            .withColumnRenamed("supplier_id", "id") \
            .withColumn("type", lit("supplier"))
        
        supply_edges = supply_chain_data.select(
            col("supplier_id").alias("src"),
            col("customer_id").alias("dst"),
            col("delivery_time").alias("weight"),
            col("cost"),
            col("quantity")
        ).withColumn("relationship", lit("supplies"))
        
        supply_graph = GraphFrame(supplier_vertices, supply_edges)
        
        # Analysis
        
        # 1. Critical suppliers (high degree centrality)
        critical_suppliers = supply_graph.degrees \
            .join(supplier_vertices.select("id", "supplier_name"), "id") \
            .orderBy(col("degree").desc()) \
            .limit(10)
        
        # 2. Supply chain paths and bottlenecks
        paths_analysis = supply_graph.shortestPaths(landmarks=["SUPPLIER_001", "SUPPLIER_002"])
        
        # 3. Risk analysis - single points of failure
        risk_analysis = supply_edges \
            .groupBy("dst") \
            .agg(
                countDistinct("src").alias("supplier_count"),
                avg("weight").alias("avg_delivery_time"),
                sum("quantity").alias("total_quantity")
            ) \
            .filter(col("supplier_count") == 1)  # Single supplier dependency
        
        return {
            "graph": supply_graph,
            "critical_suppliers": critical_suppliers,
            "paths_analysis": paths_analysis,
            "risk_analysis": risk_analysis
        }

# Usage examples
advanced_algorithms = AdvancedGraphAlgorithms(graph)

# Example: Collaborative filtering
# user_item_data = spark.createDataFrame([
#     ("user1", "item
```
```