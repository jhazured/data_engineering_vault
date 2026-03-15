
#pyspark #troubleshooting #debugging #diagnosis #workflow #systematic #production #incident-response

---

## 🎯 Purpose

Systematic troubleshooting workflows for diagnosing complex PySpark issues. This guide provides step-by-step procedures for identifying root causes, isolating problems, and implementing solutions. Perfect for production incident response and complex debugging scenarios.

---

## 📋 Troubleshooting Workflow Index

- [[#🚨 Emergency Incident Response]]
- [[#🔍 Systematic Diagnosis Framework]]
- [[#⚡ Performance Troubleshooting]]
- [[#💾 Memory Issues Diagnosis]]
- [[#📊 Data Quality Problems]]
- [[#🌊 Streaming Issues]]
- [[#🔗 Join Problems]]
- [[#🎛️ Configuration Issues]]
- [[#🔌 Connectivity Problems]]
- [[#📈 Monitoring & Observability]]

---

## 🚨 Emergency Incident Response

### **Incident Severity Assessment**

#### **P0 - Critical (Production Down)**

```python
def p0_incident_response(spark, affected_application=None):
    """Immediate response for critical production incidents"""
    
    print("🚨 P0 INCIDENT RESPONSE INITIATED")
    print("=" * 50)
    
    # Step 1: Quick health check
    health_status = {
        "spark_context_alive": False,
        "cluster_responsive": False,
        "memory_available": False,
        "executors_healthy": False
    }
    
    try:
        # Check SparkContext
        if not spark.sparkContext._jsc.sc().isStopped():
            health_status["spark_context_alive"] = True
            print("✅ SparkContext is alive")
        else:
            print("❌ SparkContext is stopped")
            return {"action": "RESTART_APPLICATION", "severity": "CRITICAL"}
        
        # Check cluster
        status = spark.sparkContext.statusTracker()
        executors = status.getExecutorInfos()
        if len(executors) > 0:
            health_status["cluster_responsive"] = True
            health_status["executors_healthy"] = len(executors)
            print(f"✅ Cluster responsive: {len(executors)} executors")
        else:
            print("❌ No executors available")
            return {"action": "CHECK_CLUSTER_RESOURCES", "severity": "CRITICAL"}
        
        # Check memory
        try:
            import psutil
            memory = psutil.virtual_memory()
            if memory.percent < 90:
                health_status["memory_available"] = True
                print(f"✅ Memory usage acceptable: {memory.percent}%")
            else:
                print(f"⚠️ High memory usage: {memory.percent}%")
                return {"action": "EMERGENCY_MEMORY_CLEANUP", "severity": "HIGH"}
        except ImportError:
            print("⚠️ Cannot check memory - psutil not available")
        
        print("\n✅ System appears stable - proceeding with detailed diagnosis")
        return {"action": "DETAILED_DIAGNOSIS", "severity": "MEDIUM"}
        
    except Exception as e:
        print(f"❌ Health check failed: {e}")
        return {"action": "ESCALATE_TO_PLATFORM_TEAM", "severity": "CRITICAL"}

def emergency_stabilization(spark):
    """Emergency stabilization procedures"""
    
    print("🔧 APPLYING EMERGENCY STABILIZATION")
    
    stabilization_actions = []
    
    # 1. Clear all caches
    try:
        spark.catalog.clearCache()
        stabilization_actions.append("✅ Cleared all caches")
    except Exception as e:
        stabilization_actions.append(f"❌ Cache clear failed: {e}")
    
    # 2. Trigger garbage collection
    try:
        spark.sparkContext._jvm.System.gc()
        stabilization_actions.append("✅ Triggered GC")
    except Exception as e:
        stabilization_actions.append(f"❌ GC trigger failed: {e}")
    
    # 3. Enable conservative settings
    conservative_configs = {
        "spark.sql.adaptive.enabled": "true",
        "spark.sql.adaptive.coalescePartitions.enabled": "true",
        "spark.sql.shuffle.partitions": "200"
    }
    
    for key, value in conservative_configs.items():
        try:
            spark.conf.set(key, value)
            stabilization_actions.append(f"✅ Set {key}={value}")
        except Exception as e:
            stabilization_actions.append(f"❌ Config failed {key}: {e}")
    
    for action in stabilization_actions:
        print(action)
    
    return stabilization_actions
```

#### **P1 - High (Performance Degraded)**

```python
def p1_incident_response(spark):
    """Response for high priority performance issues"""
    
    print("⚠️ P1 INCIDENT RESPONSE")
    print("=" * 40)
    
    performance_metrics = {}
    
    # Check active jobs
    status = spark.sparkContext.statusTracker()
    active_jobs = status.getActiveJobIds()
    active_stages = status.getActiveStageIds()
    
    performance_metrics.update({
        "active_jobs": len(active_jobs),
        "active_stages": len(active_stages),
        "timestamp": time.time()
    })
    
    print(f"Active jobs: {len(active_jobs)}")
    print(f"Active stages: {len(active_stages)}")
    
    # Check for long-running stages
    stage_infos = status.getActiveStageInfos()
    long_running_stages = []
    
    current_time = time.time() * 1000  # Convert to milliseconds
    
    for stage in stage_infos:
        if hasattr(stage, 'submissionTime') and stage.submissionTime:
            runtime = current_time - stage.submissionTime
            if runtime > 300000:  # 5 minutes
                long_running_stages.append({
                    "stage_id": stage.stageId,
                    "runtime_minutes": runtime / 60000,
                    "num_tasks": stage.numTasks,
                    "failed_tasks": stage.numFailedTasks
                })
    
    if long_running_stages:
        print("⚠️ Long-running stages detected:")
        for stage in long_running_stages:
            print(f"  Stage {stage['stage_id']}: {stage['runtime_minutes']:.1f}min, "
                  f"Tasks: {stage['num_tasks']}, Failed: {stage['failed_tasks']}")
    
    return performance_metrics

def incident_communication_template(severity, description, impact, actions_taken):
    """Generate incident communication template"""
    
    template = f"""
INCIDENT ALERT - {severity}
=========================
Time: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

DESCRIPTION:
{description}

IMPACT:
{impact}

ACTIONS TAKEN:
{chr(10).join(f"- {action}" for action in actions_taken)}

NEXT STEPS:
- Monitoring recovery
- Root cause analysis in progress
- Updates every 15 minutes

Status Dashboard: [Link to monitoring]
Incident Commander: [Name]
"""
    return template
```

---

## 🔍 Systematic Diagnosis Framework

### **The PSPACE Method**

**P**erformance - **S**chema - **P**artitions - **A**pplication - **C**luster - **E**nvironment

```python
class PSPACEDiagnostic:
    """Systematic diagnostic framework using PSPACE methodology"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.results = {}
    
    def diagnose_performance(self, df=None):
        """P - Performance Analysis"""
        
        print("🔍 PERFORMANCE ANALYSIS")
        print("-" * 30)
        
        perf_results = {}
        
        # Check Spark UI accessibility
        try:
            app_id = self.spark.sparkContext.applicationId
            ui_port = self.spark.sparkContext.uiWebUrl
            perf_results["spark_ui"] = f"✅ Accessible at {ui_port}"
        except Exception as e:
            perf_results["spark_ui"] = f"❌ Not accessible: {e}"
        
        # Check recent job performance
        status = self.spark.sparkContext.statusTracker()
        recent_jobs = status.getActiveJobIds()
        
        if recent_jobs:
            perf_results["active_jobs"] = f"⚠️ {len(recent_jobs)} active jobs"
        else:
            perf_results["active_jobs"] = "✅ No active jobs"
        
        # DataFrame-specific performance
        if df is not None:
            try:
                start_time = time.time()
                sample_count = df.limit(1000).count()
                sample_time = time.time() - start_time
                
                perf_results["sample_performance"] = f"✅ 1000 rows in {sample_time:.2f}s"
                
                # Estimate full performance
                if sample_time > 5:
                    perf_results["performance_warning"] = "⚠️ Slow sampling detected"
                
            except Exception as e:
                perf_results["sample_performance"] = f"❌ Sampling failed: {e}"
        
        self.results["performance"] = perf_results
        
        for key, value in perf_results.items():
            print(f"  {key}: {value}")
        
        return perf_results
    
    def diagnose_schema(self, df):
        """S - Schema Analysis"""
        
        print("\n🔍 SCHEMA ANALYSIS")
        print("-" * 30)
        
        schema_results = {}
        
        try:
            # Basic schema info
            schema_results["column_count"] = len(df.columns)
            schema_results["schema_accessible"] = "✅ Schema readable"
            
            # Check for common schema issues
            column_names = df.columns
            
            # Duplicate column names
            duplicates = [col for col in set(column_names) if column_names.count(col) > 1]
            if duplicates:
                schema_results["duplicate_columns"] = f"⚠️ Duplicates: {duplicates}"
            else:
                schema_results["duplicate_columns"] = "✅ No duplicates"
            
            # Special characters in column names
            special_chars = [col for col in column_names if not col.replace('_', '').isalnum()]
            if special_chars:
                schema_results["special_chars"] = f"⚠️ Special chars: {special_chars[:3]}"
            else:
                schema_results["special_chars"] = "✅ Clean column names"
            
            # Data types analysis
            type_distribution = {}
            for name, dtype in df.dtypes:
                type_distribution[dtype] = type_distribution.get(dtype, 0) + 1
            
            schema_results["type_distribution"] = type_distribution
            
            # Check for complex types
            complex_types = [name for name, dtype in df.dtypes 
                           if dtype in ['array', 'map', 'struct']]
            if complex_types:
                schema_results["complex_types"] = f"ℹ️ Found: {len(complex_types)} complex columns"
            
        except Exception as e:
            schema_results["schema_error"] = f"❌ Schema analysis failed: {e}"
        
        self.results["schema"] = schema_results
        
        for key, value in schema_results.items():
            print(f"  {key}: {value}")
        
        return schema_results
    
    def diagnose_partitions(self, df):
        """P - Partition Analysis"""
        
        print("\n🔍 PARTITION ANALYSIS")
        print("-" * 30)
        
        partition_results = {}
        
        try:
            # Basic partition info
            num_partitions = df.rdd.getNumPartitions()
            partition_results["partition_count"] = num_partitions
            
            # Partition size analysis
            partition_sizes = df.rdd.mapPartitions(lambda x: [sum(1 for _ in x)]).collect()
            
            if partition_sizes:
                avg_size = sum(partition_sizes) / len(partition_sizes)
                max_size = max(partition_sizes)
                min_size = min(partition_sizes)
                
                partition_results.update({
                    "avg_partition_size": f"{avg_size:.0f} rows",
                    "max_partition_size": f"{max_size:,} rows",
                    "min_partition_size": f"{min_size:,} rows"
                })
                
                # Skew analysis
                if avg_size > 0:
                    skew_ratio = max_size / avg_size
                    if skew_ratio > 10:
                        partition_results["skew_warning"] = f"⚠️ High skew: {skew_ratio:.1f}x"
                    elif skew_ratio > 5:
                        partition_results["skew_warning"] = f"⚠️ Moderate skew: {skew_ratio:.1f}x"
                    else:
                        partition_results["skew_status"] = f"✅ Low skew: {skew_ratio:.1f}x"
                
                # Partition health assessment
                if num_partitions > 1000:
                    partition_results["partition_health"] = "⚠️ Too many partitions"
                elif num_partitions < 10 and avg_size > 10000:
                    partition_results["partition_health"] = "⚠️ Too few partitions"
                else:
                    partition_results["partition_health"] = "✅ Partition count looks good"
            
        except Exception as e:
            partition_results["partition_error"] = f"❌ Partition analysis failed: {e}"
        
        self.results["partitions"] = partition_results
        
        for key, value in partition_results.items():
            print(f"  {key}: {value}")
        
        return partition_results
    
    def diagnose_application(self):
        """A - Application Analysis"""
        
        print("\n🔍 APPLICATION ANALYSIS")
        print("-" * 30)
        
        app_results = {}
        
        try:
            # Application info
            app_results["app_name"] = self.spark.sparkContext.appName
            app_results["app_id"] = self.spark.sparkContext.applicationId
            app_results["spark_version"] = self.spark.version
            
            # Resource allocation
            status = self.spark.sparkContext.statusTracker()
            executors = status.getExecutorInfos()
            
            if executors:
                total_cores = sum(exec_info.totalCores for exec_info in executors)
                total_memory = sum(exec_info.maxMemory for exec_info in executors)
                
                app_results.update({
                    "executor_count": len(executors),
                    "total_cores": total_cores,
                    "total_memory_gb": f"{total_memory / (1024**3):.1f} GB"
                })
            
            # Check for failed executors
            failed_jobs = len([job for job in status.getActiveJobIds()])  # Simplified
            app_results["failed_jobs"] = failed_jobs
            
        except Exception as e:
            app_results["app_error"] = f"❌ Application analysis failed: {e}"
        
        self.results["application"] = app_results
        
        for key, value in app_results.items():
            print(f"  {key}: {value}")
        
        return app_results
    
    def diagnose_cluster(self):
        """C - Cluster Analysis"""
        
        print("\n🔍 CLUSTER ANALYSIS")
        print("-" * 30)
        
        cluster_results = {}
        
        try:
            # Master info
            master = self.spark.sparkContext.master
            cluster_results["master"] = master
            
            # Deployment mode analysis
            if "local" in master.lower():
                cluster_results["deployment"] = "ℹ️ Local mode"
            elif "yarn" in master.lower():
                cluster_results["deployment"] = "ℹ️ YARN cluster"
            elif "k8s" in master.lower():
                cluster_results["deployment"] = "ℹ️ Kubernetes cluster"
            else:
                cluster_results["deployment"] = f"ℹ️ Standalone: {master}"
            
            # Executor health
            status = self.spark.sparkContext.statusTracker()
            executors = status.getExecutorInfos()
            
            active_executors = len([e for e in executors if e.isActive])
            total_executors = len(executors)
            
            cluster_results["executor_health"] = f"{active_executors}/{total_executors} active"
            
            if active_executors < total_executors:
                cluster_results["executor_warning"] = "⚠️ Some executors inactive"
            
        except Exception as e:
            cluster_results["cluster_error"] = f"❌ Cluster analysis failed: {e}"
        
        self.results["cluster"] = cluster_results
        
        for key, value in cluster_results.items():
            print(f"  {key}: {value}")
        
        return cluster_results
    
    def diagnose_environment(self):
        """E - Environment Analysis"""
        
        print("\n🔍 ENVIRONMENT ANALYSIS")
        print("-" * 30)
        
        env_results = {}
        
        try:
            # Python environment
            import sys
            env_results["python_version"] = sys.version.split()[0]
            
            # Memory info
            try:
                import psutil
                memory = psutil.virtual_memory()
                disk = psutil.disk_usage('/')
                
                env_results.update({
                    "system_memory": f"{memory.percent}% used",
                    "system_disk": f"{disk.percent}% used"
                })
                
                if memory.percent > 90:
                    env_results["memory_warning"] = "⚠️ High system memory usage"
                if disk.percent > 90:
                    env_results["disk_warning"] = "⚠️ High disk usage"
                    
            except ImportError:
                env_results["system_monitoring"] = "ℹ️ psutil not available"
            
            # Spark configuration
            critical_configs = [
                "spark.executor.memory",
                "spark.driver.memory", 
                "spark.sql.shuffle.partitions",
                "spark.serializer"
            ]
            
            config_status = {}
            for config in critical_configs:
                try:
                    value = self.spark.conf.get(config)
                    config_status[config] = value
                except:
                    config_status[config] = "NOT_SET"
            
            env_results["configurations"] = config_status
            
        except Exception as e:
            env_results["env_error"] = f"❌ Environment analysis failed: {e}"
        
        self.results["environment"] = env_results
        
        for key, value in env_results.items():
            if isinstance(value, dict):
                print(f"  {key}:")
                for k, v in value.items():
                    print(f"    {k}: {v}")
            else:
                print(f"  {key}: {value}")
        
        return env_results
    
    def run_full_diagnosis(self, df=None):
        """Run complete PSPACE diagnosis"""
        
        print("🔍 RUNNING FULL PSPACE DIAGNOSIS")
        print("=" * 50)
        
        # Run all diagnostic phases
        self.diagnose_performance(df)
        
        if df is not None:
            self.diagnose_schema(df)
            self.diagnose_partitions(df)
        
        self.diagnose_application()
        self.diagnose_cluster()
        self.diagnose_environment()
        
        # Generate summary
        self.generate_summary()
        
        return self.results
    
    def generate_summary(self):
        """Generate diagnostic summary with recommendations"""
        
        print("\n📋 DIAGNOSTIC SUMMARY")
        print("=" * 50)
        
        issues_found = []
        recommendations = []
        
        # Analyze results for issues
        for category, results in self.results.items():
            for key, value in results.items():
                if isinstance(value, str):
                    if "⚠️" in value or "❌" in value:
                        issues_found.append(f"{category}.{key}: {value}")
        
        if issues_found:
            print("🚨 ISSUES FOUND:")
            for issue in issues_found:
                print(f"  - {issue}")
        else:
            print("✅ No critical issues detected")
        
        # Generate recommendations based on findings
        if "partition" in str(self.results):
            partition_info = self.results.get("partitions", {})
            if "skew_warning" in str(partition_info):
                recommendations.append("Consider repartitioning to reduce data skew")
            
            if "Too many partitions" in str(partition_info):
                recommendations.append("Use coalesce() to reduce partition count")
        
        if recommendations:
            print("\n💡 RECOMMENDATIONS:")
            for rec in recommendations:
                print(f"  - {rec}")
        
        print(f"\n🕒 Diagnosis completed at {datetime.now().strftime('%H:%M:%S')}")
```

---

## ⚡ Performance Troubleshooting

### **Performance Issue Classification**

```python
class PerformanceTroubleshooter:
    """Systematic performance troubleshooting"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.baseline_metrics = {}
        self.current_metrics = {}
    
    def establish_baseline(self, test_df):
        """Establish performance baseline"""
        
        print("📊 ESTABLISHING PERFORMANCE BASELINE")
        print("-" * 40)
        
        # Simple operations baseline
        operations = {
            "count": lambda df: df.count(),
            "sample_1k": lambda df: df.limit(1000).count(),
            "simple_filter": lambda df: df.filter(col("id") > 100).count(),
            "simple_select": lambda df: df.select("id").count()
        }
        
        for op_name, operation in operations.items():
            try:
                start_time = time.time()
                result = operation(test_df)
                duration = time.time() - start_time
                
                self.baseline_metrics[op_name] = {
                    "duration": duration,
                    "result": result,
                    "timestamp": start_time
                }
                
                print(f"✅ {op_name}: {duration:.2f}s ({result:,} rows)")
                
            except Exception as e:
                print(f"❌ {op_name} failed: {e}")
                self.baseline_metrics[op_name] = {"error": str(e)}
        
        return self.baseline_metrics
    
    def compare_performance(self, test_df):
        """Compare current performance against baseline"""
        
        print("\n📊 PERFORMANCE COMPARISON")
        print("-" * 40)
        
        if not self.baseline_metrics:
            print("⚠️ No baseline established - run establish_baseline() first")
            return
        
        performance_degradation = []
        
        for op_name in self.baseline_metrics:
            if "error" in self.baseline_metrics[op_name]:
                continue
                
            try:
                # Re-run the operation
                if op_name == "count":
                    start_time = time.time()
                    result = test_df.count()
                elif op_name == "sample_1k":
                    start_time = time.time()
                    result = test_df.limit(1000).count()
                elif op_name == "simple_filter":
                    start_time = time.time()
                    result = test_df.filter(col("id") > 100).count()
                elif op_name == "simple_select":
                    start_time = time.time()
                    result = test_df.select("id").count()
                
                current_duration = time.time() - start_time
                baseline_duration = self.baseline_metrics[op_name]["duration"]
                
                ratio = current_duration / baseline_duration
                
                self.current_metrics[op_name] = {
                    "duration": current_duration,
                    "result": result,
                    "ratio": ratio
                }
                
                if ratio > 2.0:
                    status = f"🔴 DEGRADED {ratio:.1f}x slower"
                    performance_degradation.append((op_name, ratio))
                elif ratio > 1.5:
                    status = f"🟡 SLOWER {ratio:.1f}x"
                else:
                    status = f"🟢 OK {ratio:.1f}x"
                
                print(f"{status} {op_name}: {current_duration:.2f}s (was {baseline_duration:.2f}s)")
                
            except Exception as e:
                print(f"❌ {op_name} failed: {e}")
        
        if performance_degradation:
            print(f"\n⚠️ Performance degradation detected in {len(performance_degradation)} operations")
            return self._diagnose_performance_issues(performance_degradation)
        else:
            print("\n✅ Performance within acceptable range")
            return None
    
    def _diagnose_performance_issues(self, degraded_operations):
        """Diagnose specific performance issues"""
        
        print("\n🔍 DIAGNOSING PERFORMANCE ISSUES")
        print("-" * 40)
        
        diagnosis = {
            "likely_causes": [],
            "recommended_actions": [],
            "investigation_steps": []
        }
        
        # Analyze patterns in degraded operations
        all_degraded = len(degraded_operations) == len(self.baseline_metrics)
        
        if all_degraded:
            diagnosis["likely_causes"].extend([
                "Cluster resource contention",
                "Memory pressure",
                "Network issues",
                "Storage I/O bottlenecks"
            ])
            
            diagnosis["recommended_actions"].extend([
                "Check cluster resource utilization",
                "Review concurrent workloads",
                "Examine GC logs for memory pressure",
                "Monitor storage I/O metrics"
            ])
        
        else:
            # Specific operation issues
            degraded_ops = [op[0] for op in degraded_operations]
            
            if "count" in degraded_ops:
                diagnosis["likely_causes"].append("Data scanning inefficiency")
                diagnosis["recommended_actions"].append("Check for data skew and partition issues")
            
            if "simple_filter" in degraded_ops:
                diagnosis["likely_causes"].append("Predicate pushdown not working")
                diagnosis["recommended_actions"].append("Verify file format supports pushdown")
        
        # Add general investigation steps
        diagnosis["investigation_steps"].extend([
            "Check Spark UI for stage timing details",
            "Review executor logs for errors",
            "Monitor JVM GC activity",
            "Analyze query execution plans"
        ])
        
        for category, items in diagnosis.items():
            print(f"\n{category.replace('_', ' ').title()}:")
            for item in items:
                print(f"  - {item}")
        
        return diagnosis
    
    def deep_performance_analysis(self, problematic_df):
        """Deep dive performance analysis"""
        
        print("\n🔬 DEEP PERFORMANCE ANALYSIS")
        print("-" * 40)
        
        analysis_results = {}
        
        # 1. Query plan analysis
        print("1. Query Plan Analysis:")
        try:
            plan_str = str(problematic_df._jdf.queryExecution().executedPlan())
            
            # Count expensive operations
            expensive_ops = {
                "Exchange": plan_str.count("Exchange"),
                "Sort": plan_str.count("Sort"),
                "HashAggregate": plan_str.count("HashAggregate"),
                "BroadcastHashJoin": plan_str.count("BroadcastHashJoin"),
                "SortMergeJoin": plan_str.count("SortMergeJoin")
            }
            
            analysis_results["expensive_operations"] = expensive_ops
            
            for op, count in expensive_ops.items():
                if count > 0:
                    if op == "Exchange" and count > 3:
                        print(f"  ⚠️ {op}: {count} (high shuffle cost)")
                    elif op == "Sort" and count > 2:
                        print(f"  ⚠️ {op}: {count} (expensive sorting)")
                    else:
                        print(f"  ℹ️ {op}: {count}")
            
        except Exception as e:
            print(f"  ❌ Query plan analysis failed: {e}")
        
        # 2. Partition analysis
        print("\n2. Partition Analysis:")
        try:
            num_partitions = problematic_df.rdd.getNumPartitions()
            partition_sizes = problematic_df.rdd.mapPartitions(lambda x: [sum(1 for _ in x)]).collect()
            
            if partition_sizes:
                avg_size = sum(partition_sizes) / len(partition_sizes)
                max_size = max(partition_sizes)
                min_size = min(partition_sizes)
                
                analysis_results["partition_stats"] = {
                    "count": num_partitions,
                    "avg_size": avg_size,
                    "max_size": max_size,
                    "min_size": min_size,
                    "skew_ratio": max_size / avg_size if avg_size > 0 else 0
                }
                
                print(f"  Partitions: {num_partitions}")
                print(f"  Avg size: {avg_size:.0f} rows")
                print(f"  Size range: {min_size:,} - {max_size:,}")
                
                skew_ratio = max_size / avg_size if avg_size > 0 else 0
                if skew_ratio > 10:
                    print(f"  🔴 HIGH SKEW: {skew_ratio:.1f}x")
                elif skew_ratio > 5:
                    print(f"  🟡 MODERATE SKEW: {skew_ratio:.1f}x")
                else:
                    print(f"  🟢 Low skew: {skew_ratio:.1f}x")
        
        except Exception as e:
            print(f"  ❌ Partition analysis failed: {e}")
        
        # 3. Memory estimation
        print("\n3. Memory Analysis:")
        try:
            # Rough memory estimation
            row_count = problematic_df.limit(10000).count()
            column_count = len(problematic_df.columns)
            
            # Estimate memory per row (very rough)
            estimated_row_size = column_count * 20  # 20 bytes per column average
            estimated_total_mb = (row_count * estimated_row_size) / (1024 * 1024)
           
           analysis_results["memory_estimation"] = {
               "estimated_rows": row_count,
               "estimated_size_mb": estimated_total_mb,
               "columns": column_count
           }
           
           print(f"  Estimated rows: {row_count:,}")
           print(f"  Estimated size: {estimated_total_mb:.1f} MB")
           print(f"  Columns: {column_count}")
           
           # Memory recommendations
           executor_memory = self.spark.conf.get("spark.executor.memory", "1g")
           if "g" in executor_memory.lower():
               executor_gb = float(executor_memory.lower().replace("g", ""))
               memory_ratio = estimated_total_mb / (executor_gb * 1024)
               
               if memory_ratio > 0.8:
                   print(f"  🔴 HIGH MEMORY USAGE: {memory_ratio:.1f}x executor memory")
               elif memory_ratio > 0.5:
                   print(f"  🟡 MODERATE MEMORY USAGE: {memory_ratio:.1f}x executor memory")
               else:
                   print(f"  🟢 ACCEPTABLE MEMORY USAGE: {memory_ratio:.1f}x executor memory")
       
       except Exception as e:
           print(f"  ❌ Memory analysis failed: {e}")
       
       return analysis_results
   
   def generate_optimization_recommendations(self, analysis_results):
       """Generate specific optimization recommendations"""
       
       print("\n💡 OPTIMIZATION RECOMMENDATIONS")
       print("-" * 40)
       
       recommendations = []
       
       # Partition-based recommendations
       if "partition_stats" in analysis_results:
           stats = analysis_results["partition_stats"]
           
           if stats["skew_ratio"] > 5:
               recommendations.append({
                   "priority": "HIGH",
                   "category": "Partitioning",
                   "issue": f"Data skew detected ({stats['skew_ratio']:.1f}x)",
                   "solution": "Use repartition() with appropriate key or salting technique",
                   "code": "df.repartition(col('partition_key'))"
               })
           
           if stats["count"] > 1000:
               recommendations.append({
                   "priority": "MEDIUM", 
                   "category": "Partitioning",
                   "issue": f"Too many partitions ({stats['count']})",
                   "solution": "Use coalesce() to reduce partition count",
                   "code": f"df.coalesce({min(200, stats['count'] // 5)})"
               })
           
           if stats["avg_size"] < 1000:
               recommendations.append({
                   "priority": "MEDIUM",
                   "category": "Partitioning", 
                   "issue": f"Small partitions (avg {stats['avg_size']:.0f} rows)",
                   "solution": "Increase partition size to reduce overhead",
                   "code": "df.repartition(num_partitions // 2)"
               })
       
       # Operation-based recommendations
       if "expensive_operations" in analysis_results:
           ops = analysis_results["expensive_operations"]
           
           if ops.get("Exchange", 0) > 3:
               recommendations.append({
                   "priority": "HIGH",
                   "category": "Shuffling",
                   "issue": f"Excessive shuffling ({ops['Exchange']} exchanges)",
                   "solution": "Reduce shuffles with broadcast joins or better partitioning",
                   "code": "spark.conf.set('spark.sql.adaptive.enabled', 'true')"
               })
           
           if ops.get("Sort", 0) > 2:
               recommendations.append({
                   "priority": "MEDIUM",
                   "category": "Sorting",
                   "issue": f"Multiple sorts detected ({ops['Sort']})",
                   "solution": "Combine operations to reduce sorting overhead",
                   "code": "# Use sortWithinPartitions() instead of orderBy() when possible"
               })
       
       # Memory-based recommendations
       if "memory_estimation" in analysis_results:
           mem = analysis_results["memory_estimation"]
           
           if mem["estimated_size_mb"] > 5000:  # > 5GB
               recommendations.append({
                   "priority": "HIGH",
                   "category": "Memory",
                   "issue": f"Large dataset ({mem['estimated_size_mb']:.0f} MB)",
                   "solution": "Enable memory optimization features",
                   "code": """
spark.conf.set('spark.sql.adaptive.enabled', 'true')
spark.conf.set('spark.sql.adaptive.coalescePartitions.enabled', 'true')
spark.conf.set('spark.serializer', 'org.apache.spark.serializer.KryoSerializer')
"""
               })
       
       # Display recommendations
       for i, rec in enumerate(recommendations, 1):
           print(f"\n{i}. [{rec['priority']}] {rec['category']}: {rec['issue']}")
           print(f"   Solution: {rec['solution']}")
           print(f"   Code: {rec['code']}")
       
       return recommendations
```

## 💾 Memory Issues Diagnosis

### **Memory Troubleshooting Toolkit**

```python
class MemoryTroubleshooter:
    """Comprehensive memory issue diagnosis and resolution"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.memory_profile = {}
    
    def diagnose_memory_issues(self):
        """Comprehensive memory diagnosis"""
        
        print("💾 MEMORY DIAGNOSIS")
        print("=" * 40)
        
        memory_analysis = {}
        
        # 1. JVM Memory Analysis
        print("1. JVM Memory Analysis:")
        try:
            runtime = self.spark.sparkContext._jvm.java.lang.Runtime.getRuntime()
            max_memory = runtime.maxMemory()
            total_memory = runtime.totalMemory() 
            free_memory = runtime.freeMemory()
            used_memory = total_memory - free_memory
            
            memory_analysis["jvm"] = {
                "max_mb": max_memory / (1024 * 1024),
                "total_mb": total_memory / (1024 * 1024),
                "used_mb": used_memory / (1024 * 1024),
                "free_mb": free_memory / (1024 * 1024),
                "usage_percent": (used_memory / max_memory) * 100
            }
            
            usage_pct = memory_analysis["jvm"]["usage_percent"]
            
            if usage_pct > 90:
                status = "🔴 CRITICAL"
            elif usage_pct > 75:
                status = "🟡 HIGH"
            else:
                status = "🟢 OK"
            
            print(f"   {status} JVM Memory: {usage_pct:.1f}% used")
            print(f"   Used: {memory_analysis['jvm']['used_mb']:.0f} MB")
            print(f"   Max: {memory_analysis['jvm']['max_mb']:.0f} MB")
            
        except Exception as e:
            print(f"   ❌ JVM analysis failed: {e}")
        
        # 2. Spark Memory Configuration
        print("\n2. Spark Memory Configuration:")
        memory_configs = {
            "spark.executor.memory": "Executor heap size",
            "spark.executor.memoryOffHeap.enabled": "Off-heap enabled",
            "spark.executor.memoryOffHeap.size": "Off-heap size",
            "spark.sql.execution.arrow.maxRecordsPerBatch": "Arrow batch size",
            "spark.executor.memoryFraction": "Memory fraction for execution"
        }
        
        config_analysis = {}
        for config, description in memory_configs.items():
            try:
                value = self.spark.conf.get(config)
                config_analysis[config] = value
                print(f"   {config}: {value}")
            except:
                print(f"   {config}: NOT SET")
                config_analysis[config] = "NOT_SET"
        
        memory_analysis["configuration"] = config_analysis
        
        # 3. Storage Memory Analysis
        print("\n3. Storage Memory Analysis:")
        try:
            status = self.spark.sparkContext.statusTracker()
            executor_infos = status.getExecutorInfos()
            
            total_storage_memory = 0
            used_storage_memory = 0
            
            for executor in executor_infos:
                if hasattr(executor, 'memoryUsed') and hasattr(executor, 'maxMemory'):
                    used_storage_memory += executor.memoryUsed
                    total_storage_memory += executor.maxMemory
            
            if total_storage_memory > 0:
                storage_usage_pct = (used_storage_memory / total_storage_memory) * 100
                
                memory_analysis["storage"] = {
                    "used_mb": used_storage_memory / (1024 * 1024),
                    "total_mb": total_storage_memory / (1024 * 1024), 
                    "usage_percent": storage_usage_pct
                }
                
                if storage_usage_pct > 80:
                    status = "🔴 HIGH"
                elif storage_usage_pct > 60:
                    status = "🟡 MODERATE"
                else:
                    status = "🟢 LOW"
                
                print(f"   {status} Storage memory: {storage_usage_pct:.1f}% used")
                print(f"   Executors: {len(executor_infos)}")
            else:
                print("   ℹ️ No storage memory information available")
        
        except Exception as e:
            print(f"   ❌ Storage analysis failed: {e}")
        
        # 4. GC Analysis (if available)
        print("\n4. Garbage Collection Analysis:")
        try:
            # Try to get GC info from JVM
            gc_beans = self.spark.sparkContext._jvm.java.lang.management.ManagementFactory.getGarbageCollectorMXBeans()
            
            gc_info = {}
            for gc_bean in gc_beans:
                name = gc_bean.getName()
                collection_count = gc_bean.getCollectionCount()
                collection_time = gc_bean.getCollectionTime()
                
                gc_info[name] = {
                    "collections": collection_count,
                    "time_ms": collection_time
                }
            
            memory_analysis["gc"] = gc_info
            
            total_gc_time = sum(info["time_ms"] for info in gc_info.values())
            total_collections = sum(info["collections"] for info in gc_info.values())
            
            if total_gc_time > 30000:  # 30 seconds
                print(f"   🔴 HIGH GC overhead: {total_gc_time/1000:.1f}s total")
            elif total_gc_time > 10000:  # 10 seconds  
                print(f"   🟡 MODERATE GC overhead: {total_gc_time/1000:.1f}s total")
            else:
                print(f"   🟢 LOW GC overhead: {total_gc_time/1000:.1f}s total")
            
            print(f"   Total collections: {total_collections}")
            
        except Exception as e:
            print(f"   ℹ️ GC analysis not available: {e}")
        
        self.memory_profile = memory_analysis
        return memory_analysis
    
    def detect_memory_patterns(self, problematic_operations=None):
        """Detect memory usage patterns and issues"""
        
        print("\n🔍 MEMORY PATTERN ANALYSIS")
        print("-" * 40)
        
        patterns = {
            "memory_leaks": [],
            "excessive_caching": [],
            "large_objects": [],
            "gc_pressure": []
        }
        
        # Analyze JVM memory trends
        if "jvm" in self.memory_profile:
            jvm_usage = self.memory_profile["jvm"]["usage_percent"]
            
            if jvm_usage > 90:
                patterns["memory_leaks"].append("JVM memory usage critically high")
            
            if jvm_usage > 80:
                patterns["gc_pressure"].append("High memory pressure detected")
        
        # Analyze storage memory
        if "storage" in self.memory_profile:
            storage_usage = self.memory_profile["storage"]["usage_percent"]
            
            if storage_usage > 70:
                patterns["excessive_caching"].append("High storage memory usage - check cached DataFrames")
        
        # Analyze GC patterns
        if "gc" in self.memory_profile:
            gc_info = self.memory_profile["gc"]
            total_gc_time = sum(info["time_ms"] for info in gc_info.values())
            
            if total_gc_time > 30000:
                patterns["gc_pressure"].append("Excessive GC time indicates memory pressure")
            
            # Look for specific GC collectors
            for collector_name, info in gc_info.items():
                if "old" in collector_name.lower() or "tenured" in collector_name.lower():
                    if info["time_ms"] > 10000:
                        patterns["gc_pressure"].append(f"Old generation GC pressure: {collector_name}")
        
        # Analyze configuration issues
        if "configuration" in self.memory_profile:
            config = self.memory_profile["configuration"]
            
            if config.get("spark.executor.memoryOffHeap.enabled") == "false":
                patterns["large_objects"].append("Off-heap memory disabled - consider enabling for large datasets")
            
            executor_memory = config.get("spark.executor.memory", "1g")
            if "m" in executor_memory.lower():
                mem_mb = int(executor_memory.lower().replace("m", ""))
                if mem_mb < 2048:
                    patterns["memory_leaks"].append("Executor memory very low - may cause frequent spills")
        
        # Display patterns
        for pattern_type, issues in patterns.items():
            if issues:
                print(f"\n{pattern_type.replace('_', ' ').title()}:")
                for issue in issues:
                    print(f"  🔍 {issue}")
        
        return patterns
    
    def memory_optimization_strategy(self, memory_patterns):
        """Generate memory optimization strategy"""
        
        print("\n💡 MEMORY OPTIMIZATION STRATEGY")
        print("-" * 40)
        
        optimizations = []
        
        # Address memory leaks
        if memory_patterns["memory_leaks"]:
            optimizations.append({
                "priority": "CRITICAL",
                "category": "Memory Leaks",
                "actions": [
                    "Clear unused cached DataFrames: spark.catalog.clearCache()",
                    "Increase executor memory allocation",
                    "Review DataFrame lifecycle management",
                    "Use unpersist() after cache operations"
                ],
                "code": """
# Clear all caches
spark.catalog.clearCache()

# Increase executor memory
spark.conf.set('spark.executor.memory', '8g')
spark.conf.set('spark.executor.memoryFraction', '0.8')
"""
            })
        
        # Address excessive caching
        if memory_patterns["excessive_caching"]:
            optimizations.append({
                "priority": "HIGH",
                "category": "Caching Strategy",
                "actions": [
                    "Review which DataFrames actually need caching",
                    "Use disk-based storage levels for large DataFrames", 
                    "Implement selective caching strategy",
                    "Monitor cache hit ratios"
                ],
                "code": """
# Use memory and disk storage
df.persist(StorageLevel.MEMORY_AND_DISK_SER)

# Check what's currently cached
spark.catalog.listTables()
for table in spark.catalog.listTables():
    if table.isTemporary:
        print(f"Cached: {table.name}")
"""
            })
        
        # Address large objects
        if memory_patterns["large_objects"]:
            optimizations.append({
                "priority": "HIGH", 
                "category": "Large Objects",
                "actions": [
                    "Enable off-heap memory",
                    "Use columnar storage formats (Parquet)",
                    "Implement data compression",
                    "Break large operations into smaller chunks"
                ],
                "code": """
# Enable off-heap memory
spark.conf.set('spark.executor.memoryOffHeap.enabled', 'true')
spark.conf.set('spark.executor.memoryOffHeap.size', '4g')

# Use Kryo serialization for better memory efficiency
spark.conf.set('spark.serializer', 'org.apache.spark.serializer.KryoSerializer')
"""
            })
        
        # Address GC pressure
        if memory_patterns["gc_pressure"]:
            optimizations.append({
                "priority": "HIGH",
                "category": "GC Optimization", 
                "actions": [
                    "Tune JVM GC parameters",
                    "Increase executor heap size",
                    "Reduce object creation in UDFs",
                    "Use primitive types where possible"
                ],
                "code": """
# GC tuning options
spark.conf.set('spark.executor.extraJavaOptions', 
    '-XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:G1HeapRegionSize=16m')

# Increase memory allocation
spark.conf.set('spark.executor.memory', '8g')
spark.conf.set('spark.driver.memory', '4g')
"""
            })
        
        # Display optimization strategy
        for i, opt in enumerate(optimizations, 1):
            print(f"\n{i}. [{opt['priority']}] {opt['category']}")
            print("   Actions:")
            for action in opt["actions"]:
                print(f"     • {action}")
            print(f"   Implementation:")
            print(f"{opt['code']}")
        
        return optimizations
    
    def emergency_memory_cleanup(self):
        """Emergency procedures for critical memory situations"""
        
        print("🚨 EMERGENCY MEMORY CLEANUP")
        print("=" * 40)
        
        cleanup_actions = []
        
        try:
            # 1. Clear all caches immediately
            self.spark.catalog.clearCache()
            cleanup_actions.append("✅ Cleared all DataFrame caches")
        except Exception as e:
            cleanup_actions.append(f"❌ Cache clear failed: {e}")
        
        try:
            # 2. Force garbage collection
            self.spark.sparkContext._jvm.System.gc()
            cleanup_actions.append("✅ Triggered JVM garbage collection")
        except Exception as e:
            cleanup_actions.append(f"❌ GC trigger failed: {e}")
        
        try:
            # 3. Check for large broadcast variables
            broadcast_vars = []
            # Note: This is a simplified check - in practice you'd track broadcasts
            cleanup_actions.append("ℹ️ Broadcast variable cleanup not implemented")
        except Exception as e:
            cleanup_actions.append(f"❌ Broadcast check failed: {e}")
        
        try:
            # 4. Apply conservative memory settings
            conservative_settings = {
                "spark.sql.execution.arrow.maxRecordsPerBatch": "1000",
                "spark.sql.adaptive.coalescePartitions.enabled": "true", 
                "spark.sql.adaptive.advisoryPartitionSizeInBytes": "64MB"
            }
            
            for key, value in conservative_settings.items():
                self.spark.conf.set(key, value)
            
            cleanup_actions.append("✅ Applied conservative memory settings")
            
        except Exception as e:
            cleanup_actions.append(f"❌ Configuration update failed: {e}")
        
        # Display results
        for action in cleanup_actions:
            print(f"  {action}")
        
        # Re-check memory status
        print("\n📊 Post-cleanup memory status:")
        updated_analysis = self.diagnose_memory_issues()
        
        return cleanup_actions
```

## 📊 Data Quality Problems

### **Data Quality Diagnostic Framework**

python

```python
class DataQualityTroubleshooter:
    """Comprehensive data quality issue diagnosis"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.quality_rules = {}
        self.failed_checks = []
    
    def comprehensive_data_quality_check(self, df, column_profiles=None):
        """Run comprehensive data quality analysis"""
        
        print("📊 COMPREHENSIVE DATA QUALITY CHECK")
        print("=" * 50)
        
        quality_report = {
            "basic_stats": {},
            "null_analysis": {},
            "uniqueness_check": {},
            "data_type_issues": {},
            "pattern_violations": {},
            "outlier_detection": {},
            "referential_integrity": {}
        }
        
        # 1. Basic Statistics
        print("1. Basic Statistics Analysis:")
        try:
            row_count = df.count()
            col_count = len(df.columns)
            
            quality_report["basic_stats"] = {
                "total_rows": row_count,
                "total_columns": col_count,
                "empty_dataset": row_count == 0
            }
            
            print(f"   Rows: {row_count:,}")
            print(f"   Columns: {col_count}")
            
            if row_count == 0:
                print("   🔴 CRITICAL: Dataset is empty")
                return quality_report
            
        except Exception as e:
            print(f"   ❌ Basic stats failed: {e}")
            quality_report["basic_stats"]["error"] = str(e)
        
        # 2. Null Value Analysis
        print("\n2. Null Value Analysis:")
        try:
            null_counts = {}
            total_rows = quality_report["basic_stats"]["total_rows"]
            
            for column in df.columns:
                null_count = df.filter(col(column).isNull()).count()
                null_percentage = (null_count / total_rows) * 100
                
                null_counts[column] = {
                    "null_count": null_count,
                    "null_percentage": null_percentage
                }
                
                if null_percentage > 50:
                    print(f"   🔴 {column}: {null_percentage:.1f}% null (CRITICAL)")
                elif null_percentage > 20:
                    print(f"   🟡 {column}: {null_percentage:.1f}% null (HIGH)")
                elif null_percentage > 0:
                    print(f"   ℹ️ {column}: {null_percentage:.1f}% null")
            
            quality_report["null_analysis"] = null_counts
            
        except Exception as e:
            print(f"   ❌ Null analysis failed: {e}")
        
        # 3. Uniqueness Check
        print("\n3. Uniqueness Analysis:")
        try:
            uniqueness_stats = {}
            
            for column in df.columns[:10]:  # Limit to first 10 columns
                try:
                    distinct_count = df.select(column).distinct().count()
                    total_count = df.select(column).count()
                    uniqueness_ratio = distinct_count / total_count if total_count > 0 else 0
                    
                    uniqueness_stats[column] = {
                        "distinct_count": distinct_count,
                        "total_count": total_count,
                        "uniqueness_ratio": uniqueness_ratio
                    }
                    
                    if uniqueness_ratio == 1.0:
                        print(f"   ✅ {column}: 100% unique (potential key)")
                    elif uniqueness_ratio < 0.1:
                        print(f"   🟡 {column}: {uniqueness_ratio:.1%} unique (low cardinality)")
                    else:
                        print(f"   ℹ️ {column}: {uniqueness_ratio:.1%} unique")
                        
                except Exception as col_error:
                    print(f"   ❌ {column}: uniqueness check failed")
            
            quality_report["uniqueness_check"] = uniqueness_stats
            
        except Exception as e:
            print(f"   ❌ Uniqueness analysis failed: {e}")
        
        # 4. Data Type Issues
        print("\n4. Data Type Validation:")
        try:
            type_issues = {}
            
            for column_name, data_type in df.dtypes:
                issues = []
                
                if data_type == "string":
                    # Check for numeric strings
                    try:
                        numeric_count = df.filter(
                            col(column_name).rlike("^[0-9]+\.?[0-9]*$")
                        ).count()
                        
                        if numeric_count > total_rows * 0.8:
                            issues.append("Mostly numeric - consider converting to numeric type")
                    except:
                        pass
                    
                    # Check for date strings
                    try:
                        date_pattern_count = df.filter(
                            col(column_name).rlike("^[0-9]{4}-[0-9]{2}-[0-9]{2}")
                        ).count()
                        
                        if date_pattern_count > total_rows * 0.5:
                            issues.append("Date patterns detected - consider date type")
                    except:
                        pass
                
                elif data_type in ["int", "bigint", "double", "float"]:
                    # Check for negative values in ID-like columns
                    if "id" in column_name.lower():
                        try:
                            negative_count = df.filter(col(column_name) < 0).count()
                            if negative_count > 0:
                                issues.append(f"Negative values in ID field: {negative_count}")
                        except:
                            pass
                
                if issues:
                    type_issues[column_name] = issues
                    for issue in issues:
                        print(f"   🟡 {column_name}: {issue}")
            
            quality_report["data_type_issues"] = type_issues
            
        except Exception as e:
            print(f"   ❌ Data type validation failed: {e}")
        
        # 5. Pattern Violations (if column profiles provided)
        if column_profiles:
            print("\n5. Pattern Validation:")
            try:
                pattern_violations = {}
                
                for column, profile in column_profiles.items():
                    if column not in df.columns:
                        continue
                    
                    violations = []
                    
                    # Check expected patterns
                    if "pattern" in profile:
                        try:
                            pattern_match_count = df.filter(
                                col(column).rlike(profile["pattern"])
                            ).count()
                            
                            match_ratio = pattern_match_count / total_rows
                            
                            if match_ratio < 0.9:
                                violations.append(f"Pattern match only {match_ratio:.1%}")
                        except:
                            violations.append("Pattern validation failed")
                    
                    # Check value ranges
                    if "min_value" in profile or "max_value" in profile:
                        try:
                            if "min_value" in profile:
                                below_min = df.filter(col(column) < profile["min_value"]).count()
                                if below_min > 0:
                                    violations.append(f"{below_min} values below minimum")
                            
                            if "max_value" in profile:
                                above_max = df.filter(col(column) > profile["max_value"]).count()
                                if above_max > 0:
                                    violations.append(f"{above_max} values above maximum")
                        except:
                            violations.append("Range validation failed")
                    
                    if violations:
                        pattern_violations[column] = violations
                        for violation in violations:
                            print(f"   🔴 {column}: {violation}")
                
                quality_report["pattern_violations"] = pattern_violations
                
            except Exception as e:
                print(f"   ❌ Pattern validation failed: {e}")
        
        return quality_report
    
    def detect_data_corruption(self, df):
        """Detect various forms of data corruption"""
        
        print("\n🔍 DATA CORRUPTION DETECTION")
        print("-" * 40)
        
        corruption_findings = {
            "encoding_issues": [],
            "truncation_issues": [],
            "parsing_errors": [],
            "duplicate_records": []
        }
        
        # 1. Encoding Issues
        try:
            for column in df.columns:
                if dict(df.dtypes)[column] == "string":
                    # Check for encoding artifacts
                    encoding_issues = df.filter(
                        col(column).contains("�") |  # Replacement character
                        col(column).contains("\\x") |  # Hex sequences
                        col(column).rlike(".*[\\x00-\\x08\\x0B\\x0C\\x0E-\\x1F].*")  # Control chars
                    ).count()
                    
                    if encoding_issues > 0:
                        corruption_findings["encoding_issues"].append({
                            "column": column,
                            "affected_rows": encoding_issues
                        })
                        print(f"   🔴 {column}: {encoding_issues} rows with encoding issues")
        
        except Exception as e:
            print(f"   ❌ Encoding check failed: {e}")
        
        # 2. Truncation Issues
        try:
            for column in df.columns:
                if dict(df.dtypes)[column] == "string":
                    # Look for suspiciously uniform string lengths
                    length_dist = df.select(length(col(column)).alias("len")).groupBy("len").count()
                    length_counts = length_dist.collect()
                    
                    if length_counts:
                        max_count_row = max(length_counts, key=lambda x: x["count"])
                        total_rows = sum(row["count"] for row in length_counts)
                        
                        if max_count_row["count"] / total_rows > 0.5 and max_count_row["len"] > 10:
                            corruption_findings["truncation_issues"].append({
                                "column": column,
                                "suspicious_length": max_count_row["len"],
                                "affected_percentage": (max_count_row["count"] / total_rows) * 100
                            })
                            print(f"   🟡 {column}: {(max_count_row['count'] / total_rows) * 100:.1f}% "
                                  f"at length {max_count_row['len']} (possible truncation)")
        
        except Exception as e:
            print(f"   ❌ Truncation check failed: {e}")
        
        # 3. Parsing Errors (look for patterns indicating failed parsing)
        try:
            for column in df.columns:
                data_type = dict(df.dtypes)[column]
                
                if data_type == "string":
                    # Look for failed numeric parsing (strings that look like error messages)
                    error_patterns = [
                        "error", "invalid", "failed", "null", "nan", "#n/a", "#div/0!"
                    ]
                    
                    error_count = 0
                    for pattern in error_patterns:
                        error_count += df.filter(
                            lower(col(column)).contains(pattern)
                        ).count()
                    
                    if error_count > 0:
corruption_findings["parsing_errors"].append({
                           "column": column,
                           "error_indicators": error_count
                       })
                       print(f"   🔴 {column}: {error_count} potential parsing errors")
       
       except Exception as e:
           print(f"   ❌ Parsing error check failed: {e}")
       
       # 4. Duplicate Records Detection
       try:
           total_rows = df.count()
           unique_rows = df.distinct().count()
           duplicate_count = total_rows - unique_rows
           
           if duplicate_count > 0:
               duplicate_percentage = (duplicate_count / total_rows) * 100
               corruption_findings["duplicate_records"].append({
                   "total_duplicates": duplicate_count,
                   "percentage": duplicate_percentage
               })
               
               if duplicate_percentage > 10:
                   print(f"   🔴 {duplicate_count:,} duplicate records ({duplicate_percentage:.1f}%)")
               else:
                   print(f"   🟡 {duplicate_count:,} duplicate records ({duplicate_percentage:.1f}%)")
           else:
               print("   ✅ No duplicate records found")
       
       except Exception as e:
           print(f"   ❌ Duplicate check failed: {e}")
       
       return corruption_findings
   
   def data_consistency_validation(self, df, business_rules=None):
       """Validate data consistency against business rules"""
       
       print("\n📋 DATA CONSISTENCY VALIDATION")
       print("-" * 40)
       
       consistency_results = {
           "cross_column_validation": {},
           "business_rule_violations": {},
           "temporal_consistency": {},
           "referential_consistency": {}
       }
       
       # 1. Cross-Column Validation
       try:
           print("1. Cross-Column Relationships:")
           
           # Example validations (customize based on your data)
           cross_validations = []
           
           # Date consistency (start_date <= end_date)
           date_columns = [col for col, dtype in df.dtypes if 'date' in col.lower() or 'timestamp' in col.lower()]
           
           if len(date_columns) >= 2:
               start_cols = [col for col in date_columns if 'start' in col.lower() or 'begin' in col.lower()]
               end_cols = [col for col in date_columns if 'end' in col.lower() or 'finish' in col.lower()]
               
               for start_col in start_cols:
                   for end_col in end_cols:
                       try:
                           invalid_dates = df.filter(col(start_col) > col(end_col)).count()
                           if invalid_dates > 0:
                               cross_validations.append(f"🔴 {start_col} > {end_col}: {invalid_dates} violations")
                           else:
                               cross_validations.append(f"✅ {start_col} <= {end_col}: consistent")
                       except:
                           cross_validations.append(f"❌ Cannot compare {start_col} and {end_col}")
           
           # Numeric relationships (e.g., total = sum of parts)
           numeric_columns = [col for col, dtype in df.dtypes if dtype in ['int', 'bigint', 'double', 'float']]
           
           if 'total' in [col.lower() for col in numeric_columns]:
               total_col = next(col for col in numeric_columns if col.lower() == 'total')
               part_cols = [col for col in numeric_columns if 'part' in col.lower() or 'component' in col.lower()]
               
               if part_cols:
                   try:
                       # Simple sum validation for demonstration
                       calculated_total = df.select(sum([col(c) for c in part_cols]).alias("calc_total"))
                       # This is a simplified example - real implementation would be more complex
                       cross_validations.append(f"ℹ️ Total vs sum validation setup for {total_col}")
                   except:
                       cross_validations.append(f"❌ Cannot validate total calculation")
           
           for validation in cross_validations:
               print(f"   {validation}")
           
           consistency_results["cross_column_validation"] = cross_validations
           
       except Exception as e:
           print(f"   ❌ Cross-column validation failed: {e}")
       
       # 2. Business Rule Validation
       if business_rules:
           print("\n2. Business Rule Validation:")
           try:
               rule_violations = {}
               
               for rule_name, rule_config in business_rules.items():
                   try:
                       if rule_config["type"] == "range":
                           column = rule_config["column"]
                           min_val = rule_config.get("min")
                           max_val = rule_config.get("max")
                           
                           violations = 0
                           if min_val is not None:
                               violations += df.filter(col(column) < min_val).count()
                           if max_val is not None:
                               violations += df.filter(col(column) > max_val).count()
                           
                           rule_violations[rule_name] = violations
                           
                           if violations > 0:
                               print(f"   🔴 {rule_name}: {violations} violations")
                           else:
                               print(f"   ✅ {rule_name}: passed")
                       
                       elif rule_config["type"] == "enum":
                           column = rule_config["column"]
                           valid_values = rule_config["values"]
                           
                           violations = df.filter(~col(column).isin(valid_values)).count()
                           rule_violations[rule_name] = violations
                           
                           if violations > 0:
                               print(f"   🔴 {rule_name}: {violations} invalid values")
                           else:
                               print(f"   ✅ {rule_name}: all values valid")
                       
                       elif rule_config["type"] == "custom":
                           # Custom SQL expression
                           condition = rule_config["condition"]
                           violations = df.filter(~expr(condition)).count()
                           rule_violations[rule_name] = violations
                           
                           if violations > 0:
                               print(f"   🔴 {rule_name}: {violations} violations")
                           else:
                               print(f"   ✅ {rule_name}: passed")
                   
                   except Exception as rule_error:
                       print(f"   ❌ {rule_name}: validation failed - {rule_error}")
                       rule_violations[rule_name] = f"ERROR: {rule_error}"
               
               consistency_results["business_rule_violations"] = rule_violations
               
           except Exception as e:
               print(f"   ❌ Business rule validation failed: {e}")
       
       return consistency_results
   
   def generate_data_quality_report(self, quality_results, corruption_findings, consistency_results):
       """Generate comprehensive data quality report"""
       
       print("\n📊 DATA QUALITY SUMMARY REPORT")
       print("=" * 50)
       
       # Overall quality score calculation
       total_checks = 0
       passed_checks = 0
       critical_issues = 0
       
       # Analyze results
       if "basic_stats" in quality_results and not quality_results["basic_stats"].get("empty_dataset", False):
           total_checks += 1
           passed_checks += 1
       
       # Null analysis scoring
       if "null_analysis" in quality_results:
           for column, null_info in quality_results["null_analysis"].items():
               total_checks += 1
               if null_info["null_percentage"] < 20:
                   passed_checks += 1
               elif null_info["null_percentage"] > 50:
                   critical_issues += 1
       
       # Corruption findings
       for corruption_type, findings in corruption_findings.items():
           if findings:
               if corruption_type in ["encoding_issues", "parsing_errors"]:
                   critical_issues += len(findings)
               total_checks += len(findings)
       
       # Calculate quality score
       quality_score = (passed_checks / total_checks * 100) if total_checks > 0 else 0
       
       # Generate report
       print(f"📈 Overall Data Quality Score: {quality_score:.1f}%")
       
       if quality_score >= 90:
           quality_status = "🟢 EXCELLENT"
       elif quality_score >= 75:
           quality_status = "🟡 GOOD"
       elif quality_score >= 50:
           quality_status = "🟠 POOR"
       else:
           quality_status = "🔴 CRITICAL"
       
       print(f"📊 Quality Status: {quality_status}")
       print(f"🔍 Total Checks: {total_checks}")
       print(f"✅ Passed Checks: {passed_checks}")
       print(f"🚨 Critical Issues: {critical_issues}")
       
       # Priority issues
       print(f"\n🚨 HIGH PRIORITY ISSUES:")
       priority_issues = []
       
       # Empty dataset
       if quality_results.get("basic_stats", {}).get("empty_dataset", False):
           priority_issues.append("Dataset is completely empty")
       
       # High null percentages
       if "null_analysis" in quality_results:
           for column, null_info in quality_results["null_analysis"].items():
               if null_info["null_percentage"] > 50:
                   priority_issues.append(f"{column}: {null_info['null_percentage']:.1f}% null values")
       
       # Corruption issues
       for corruption_type, findings in corruption_findings.items():
           if corruption_type == "encoding_issues":
               for finding in findings:
                   priority_issues.append(f"Encoding issues in {finding['column']}: {finding['affected_rows']} rows")
           elif corruption_type == "parsing_errors":
               for finding in findings:
                   priority_issues.append(f"Parsing errors in {finding['column']}: {finding['error_indicators']} indicators")
       
       if priority_issues:
           for i, issue in enumerate(priority_issues, 1):
               print(f"   {i}. {issue}")
       else:
           print("   ✅ No high priority issues detected")
       
       # Recommendations
       print(f"\n💡 RECOMMENDATIONS:")
       recommendations = []
       
       if critical_issues > 0:
           recommendations.append("Address critical data corruption issues immediately")
       
       if quality_score < 75:
           recommendations.append("Implement data validation at ingestion point")
           recommendations.append("Set up automated data quality monitoring")
       
       # Null value recommendations
       high_null_columns = []
       if "null_analysis" in quality_results:
           for column, null_info in quality_results["null_analysis"].items():
               if null_info["null_percentage"] > 20:
                   high_null_columns.append(column)
       
       if high_null_columns:
           recommendations.append(f"Review data collection for columns: {', '.join(high_null_columns[:3])}")
       
       # Duplication recommendations
       if any("duplicate_records" in finding for finding in corruption_findings.values() if finding):
           recommendations.append("Implement deduplication strategy")
       
       if not recommendations:
           recommendations.append("Continue monitoring data quality trends")
       
       for i, rec in enumerate(recommendations, 1):
           print(f"   {i}. {rec}")
       
       return {
           "quality_score": quality_score,
           "status": quality_status,
           "total_checks": total_checks,
           "passed_checks": passed_checks,
           "critical_issues": critical_issues,
           "priority_issues": priority_issues,
           "recommendations": recommendations
       }

```

## 🌊 Streaming Issues

### **Streaming Troubleshooting Framework**

python

```python
class StreamingTroubleshooter:
    """Specialized troubleshooting for Spark Streaming applications"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.streaming_metrics = {}
    
    def diagnose_streaming_health(self, streaming_query=None):
        """Comprehensive streaming application health check"""
        
        print("🌊 STREAMING HEALTH DIAGNOSIS")
        print("=" * 40)
        
        streaming_health = {
            "query_status": {},
            "performance_metrics": {},
            "resource_utilization": {},
            "error_analysis": {}
        }
        
        # 1. Query Status Analysis
        print("1. Streaming Query Status:")
        try:
            active_streams = self.spark.streams.active
            
            streaming_health["query_status"]["active_queries"] = len(active_streams)
            
            if not active_streams:
                print("   ⚠️ No active streaming queries found")
                return streaming_health
            
            for i, query in enumerate(active_streams):
                query_info = {
                    "id": query.id,
                    "name": query.name,
                    "status": "unknown"
                }
                
                try:
                    if query.isActive:
                        query_info["status"] = "active"
                        print(f"   ✅ Query {i+1} ({query.name}): Active")
                    else:
                        query_info["status"] = "inactive"
                        print(f"   🔴 Query {i+1} ({query.name}): Inactive")
                    
                    # Get recent progress
                    recent_progress = query.recentProgress
                    if recent_progress:
                        latest = recent_progress[-1]
                        query_info.update({
                            "batch_id": latest.get("batchId", "unknown"),
                            "input_rows_per_sec": latest.get("inputRowsPerSecond", 0),
                            "processing_time": latest.get("durationMs", {}).get("triggerExecution", 0)
                        })
                        
                        print(f"      Input rate: {query_info['input_rows_per_sec']:.1f} rows/sec")
                        print(f"      Processing time: {query_info['processing_time']}ms")
                
                except Exception as query_error:
                    print(f"   ❌ Query {i+1}: Status check failed - {query_error}")
                    query_info["error"] = str(query_error)
                
                streaming_health["query_status"][f"query_{i+1}"] = query_info
        
        except Exception as e:
            print(f"   ❌ Query status analysis failed: {e}")
        
        # 2. Performance Metrics Analysis
        print("\n2. Performance Metrics:")
        try:
            if active_streams:
                for i, query in enumerate(active_streams):
                    try:
                        recent_progress = query.recentProgress
                        
                        if recent_progress and len(recent_progress) > 1:
                            # Analyze last few batches
                            last_5_batches = recent_progress[-5:]
                            
                            processing_times = [
                                batch.get("durationMs", {}).get("triggerExecution", 0) 
                                for batch in last_5_batches
                            ]
                            
                            input_rates = [
                                batch.get("inputRowsPerSecond", 0)
                                for batch in last_5_batches
                            ]
                            
                            avg_processing_time = sum(processing_times) / len(processing_times)
                            avg_input_rate = sum(input_rates) / len(input_rates)
                            
                            performance_metrics = {
                                "avg_processing_time_ms": avg_processing_time,
                                "avg_input_rate": avg_input_rate,
                                "processing_times": processing_times,
                                "input_rates": input_rates
                            }
                            
                            # Performance assessment
                            if avg_processing_time > 30000:  # 30 seconds
                                status = "🔴 SLOW"
                            elif avg_processing_time > 10000:  # 10 seconds
                                status = "🟡 MODERATE"
                            else:
                                status = "🟢 FAST"
                            
                            print(f"   Query {i+1} - {status}")
                            print(f"      Avg processing: {avg_processing_time:.0f}ms")
                            print(f"      Avg input rate: {avg_input_rate:.1f} rows/sec")
                            
                            # Check for performance degradation
                            if len(processing_times) > 2:
                                recent_avg = sum(processing_times[-2:]) / 2
                                earlier_avg = sum(processing_times[:-2]) / (len(processing_times) - 2)
                                
                                if recent_avg > earlier_avg * 1.5:
                                    print(f"      ⚠️ Performance degrading: {recent_avg/earlier_avg:.1f}x slower")
                            
                            streaming_health["performance_metrics"][f"query_{i+1}"] = performance_metrics
                    
                    except Exception as perf_error:
                        print(f"   ❌ Query {i+1} performance analysis failed: {perf_error}")
        
        except Exception as e:
            print(f"   ❌ Performance analysis failed: {e}")
        
        # 3. Resource Utilization
        print("\n3. Resource Utilization:")
        try:
            # Check executor status
            status = self.spark.sparkContext.statusTracker()
            executors = status.getExecutorInfos()
            
            active_executors = len([e for e in executors if e.isActive])
            total_cores = sum(e.totalCores for e in executors if e.isActive)
            
            resource_metrics = {
                "active_executors": active_executors,
                "total_cores": total_cores,
                "executor_details": []
            }
            
            print(f"   Active executors: {active_executors}")
            print(f"   Total cores: {total_cores}")
            
            # Check for resource issues
            if active_executors == 0:
                print("   🔴 CRITICAL: No active executors")
            elif active_executors < 2:
                print("   🟡 WARNING: Very few executors available")
            else:
                print("   ✅ Adequate executor resources")
            
            streaming_health["resource_utilization"] = resource_metrics
        
        except Exception as e:
            print(f"   ❌ Resource analysis failed: {e}")
        
        return streaming_health
    
    def analyze_streaming_lag(self, streaming_query):
        """Analyze streaming lag and backpressure issues"""
        
        print("\n📊 STREAMING LAG ANALYSIS")
        print("-" * 40)
        
        lag_analysis = {
            "current_lag": {},
            "lag_trend": {},
            "backpressure_indicators": [],
            "recommendations": []
        }
        
        try:
            recent_progress = streaming_query.recentProgress
            
            if not recent_progress:
                print("   ⚠️ No progress information available")
                return lag_analysis
            
            # Analyze recent batches
            batch_count = min(10, len(recent_progress))
            recent_batches = recent_progress[-batch_count:]
            
            print(f"Analyzing last {len(recent_batches)} batches:")
            
            # Extract key metrics
            batch_metrics = []
            for batch in recent_batches:
                metrics = {
                    "batch_id": batch.get("batchId", 0),
                    "input_rows": batch.get("numInputRows", 0),
                    "input_rate": batch.get("inputRowsPerSecond", 0),
                    "processing_time": batch.get("durationMs", {}).get("triggerExecution", 0),
                    "scheduling_delay": batch.get("durationMs", {}).get("schedulingDelay", 0),
                    "timestamp": batch.get("timestamp", "")
                }
                batch_metrics.append(metrics)
            
            # Calculate lag indicators
            if batch_metrics:
                avg_processing_time = sum(b["processing_time"] for b in batch_metrics) / len(batch_metrics)
                max_processing_time = max(b["processing_time"] for b in batch_metrics)
                avg_input_rate = sum(b["input_rate"] for b in batch_metrics) / len(batch_metrics)
                
                lag_analysis["current_lag"] = {
                    "avg_processing_time_ms": avg_processing_time,
                    "max_processing_time_ms": max_processing_time,
                    "avg_input_rate": avg_input_rate
                }
                
                print(f"   Average processing time: {avg_processing_time:.0f}ms")
                print(f"   Maximum processing time: {max_processing_time:.0f}ms")
                print(f"   Average input rate: {avg_input_rate:.1f} rows/sec")
                
                # Identify backpressure indicators
                backpressure_indicators = []
                
                # Check for increasing processing times
                if len(batch_metrics) >= 5:
                    early_batches = batch_metrics[:len(batch_metrics)//2]
                    late_batches = batch_metrics[len(batch_metrics)//2:]
                    
                    early_avg = sum(b["processing_time"] for b in early_batches) / len(early_batches)
                    late_avg = sum(b["processing_time"] for b in late_batches) / len(late_batches)
                    
                    if late_avg > early_avg * 1.3:
                        backpressure_indicators.append("Processing time increasing trend")
                        print("   🔴 Processing time is increasing")
                
                # Check for high scheduling delays
                high_scheduling_delays = [b for b in batch_metrics if b["scheduling_delay"] > 5000]
                if high_scheduling_delays:
                    backpressure_indicators.append(f"High scheduling delays in {len(high_scheduling_delays)} batches")
                    print(f"   🟡 High scheduling delays detected")
                
                # Check for processing time > trigger interval
                # Assuming default trigger interval of 1 second (1000ms)
                trigger_interval = 1000  # This should be configurable based on your trigger
                slow_batches = [b for b in batch_metrics if b["processing_time"] > trigger_interval]
                
                if slow_batches:
                    backpressure_indicators.append(f"Processing slower than trigger in {len(slow_batches)} batches")
                    print(f"   🔴 Processing slower than trigger interval")
                
                lag_analysis["backpressure_indicators"] = backpressure_indicators
                
                # Generate recommendations
                recommendations = []
                
                if avg_processing_time > 10000:  # 10 seconds
                    recommendations.append("Consider increasing parallelism or optimizing query")
                
                if len(backpressure_indicators) > 0:
                    recommendations.append("Enable adaptive query execution")
                    recommendations.append("Review resource allocation")
                
                if max_processing_time > avg_processing_time * 3:
                    recommendations.append("Investigate batch size variation - consider micro-batching")
                
                lag_analysis["recommendations"] = recommendations
                
                print("\n💡 Recommendations:")
                for i, rec in enumerate(recommendations, 1):
                    print(f"   {i}. {rec}")
        
        except Exception as e:
            print(f"   ❌ Lag analysis failed: {e}")
        
        return lag_analysis
    
    def streaming_error_diagnosis(self, streaming_query):
        """Diagnose streaming-specific errors and failures"""
        
        print("\n🔍 STREAMING ERROR DIAGNOSIS")
        print("-" * 40)
        
        error_analysis = {
            "query_exceptions": [],
            "recent_errors": [],
            "error_patterns": {},
            "recovery_status": {}
        }
        
        try:
            # Check query exception
            if hasattr(streaming_query, 'exception') and streaming_query.exception():
                exception = streaming_query.exception()
                error_analysis["query_exceptions"].append({
                    "message": str(exception),
                    "type": type(exception).__name__
                })
                print(f"   🔴 Query Exception: {exception}")
            
            # Analyze recent progress for errors
            recent_progress = streaming_query.recentProgress
            
            if recent_progress:
                for batch in recent_progress[-5:]:  # Last 5 batches
                    batch_id = batch.get("batchId", "unknown")
                    
                    # Check for error indicators in batch
                    sources = batch.get("sources", [])
                    for source in sources:
                        if "numInputRows" in source and source["numInputRows"] == 0:
                            # Could indicate source issues
                            if batch_id not in [e.get("batch_id") for e in error_analysis["recent_errors"]]:
                                error_analysis["recent_errors"].append({
                                    "batch_id": batch_id,
                                    "issue": "No input rows",
                                    "source": source.get("description", "unknown")
                                })
                    
                    # Check for unusually long processing times
                    processing_time = batch.get("durationMs", {}).get("triggerExecution", 0)
                    if processing_time > 60000:  # 1 minute
                        error_analysis["recent_errors"].append({
                            "batch_id": batch_id,
                            "issue": f"Very slow processing: {processing_time}ms",
                            "type": "performance"
                        })
                
                # Analyze error patterns
                error_types = {}
                for error in error_analysis["recent_errors"]:
                    error_type = error.get("type", error.get("issue", "unknown"))
                    error_types[error_type] = error_types.get(error_type, 0) + 1
                
                error_analysis["error_patterns"] = error_types
                
                if error_analysis["recent_errors"]:
                    print("   Recent Issues:")
                    for error in error_analysis["recent_errors"][-3:]:  # Show last 3
                        print(f"     Batch {error['batch_id']}: {error['issue']}")
                else:
                    print("   ✅ No recent errors detected")
                
                if error_types:
                    print("   Error Patterns:")
                    for error_type, count in error_types.items():
                        print(f"     {error_type}: {count} occurrences")
        
        except Exception as e:
            print(f"   ❌ Error diagnosis failed: {e}")
        
        return error_analysis
    
    def streaming_optimization_recommendations(self, health_analysis, lag_analysis, error_analysis):
        """Generate optimization recommendations for streaming applications"""
        
        print("\n🚀 STREAMING OPTIMIZATION RECOMMENDATIONS")
        print("=" * 50)
        
        recommendations = {
            "immediate_actions": [],
            "configuration_changes": [],
            "architectural_improvements": [],
            "monitoring_enhancements": []
        }
        
        # Immediate actions based on health
        if health_analysis.get("query_status", {}).get("active_queries", 0) == 0:
            recommendations["immediate_actions"].append("Restart streaming queries immediately")
        
        # Performance-based recommendations
        performance_metrics = health_analysis.get("performance_metrics", {})
        for query_id, metrics in performance_metrics.items():
            avg_time = metrics.get("avg_processing_time_ms", 0)
            
            if avg_time > 30000:  # 30 seconds
                recommendations["immediate_actions"].append(f"Investigate {query_id} performance - very slow processing")
            elif avg_time > 10000:  # 10 seconds
                recommendations["configuration_changes"].append(f"Optimize {query_id} - consider increasing resources")
        
        # Lag-based recommendations
        if lag_analysis.get("backpressure_indicators"):
            recommendations["configuration_changes"].extend([
                "Enable backpressure handling: spark.streaming.backpressure.enabled=true",
                "Tune batch interval based on processing capacity",
                "Consider using structured streaming with trigger intervals"
            ])
        
        # Error-based recommendations
        if error_analysis.get("query_exceptions"):
            recommendations["immediate_actions"].append("Address query exceptions before proceeding")
        
        error_patterns = error_analysis.get("error_patterns", {})
        if "No input rows" in error_patterns:
            recommendations["immediate_actions"].append("Check data source connectivity and availability")
        
        if "performance" in error_patterns:
            recommendations["architectural_improvements"].extend([
                "Consider partitioning strategy optimization",
                "Evaluate data source throughput capacity",
                "Review sink performance and parallelism"
            ])
        
        # Resource-based recommendations
        resource_util = health_analysis.get("resource_utilization", {})
        active_executors = resource_util.get("active_executors", 0)
        
        if active_executors < 2:
            recommendations["configuration_changes"].append("Increase executor count for better parallelism")
        
        # General streaming optimizations
        recommendations["configuration_changes"].extend([
            "Enable adaptive query execution for dynamic optimization",
            "Configure appropriate checkpoint location for fault tolerance",
            "Set up proper serialization (Kryo) for better performance"
        ])
        
        recommendations["monitoring_enhancements"].extend([
            "Set up Spark UI monitoring dashboard",
            "Configure alerts for processing lag thresholds", 
            "Implement custom metrics collection for business KPIs",
            "Set up log aggregation for error tracking"
        ])
        
        # Display recommendations by category
        for category, items in recommendations.items():
            if items:
                print(f"\n{category.replace('_', ' ').title()}:")
                for i, item in enumerate(items, 1):
                    print(f"   {i}. {item}")
        
        return recommendations
```

## 🔗 Join Problems

### **Join Optimization and Troubleshooting**

python

```python
class JoinTroubleshooter:
    """Specialized troubleshooting for join operations"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.join_analysis = {}
    
    def analyze_join_performance(self, df1, df2, join_keys, join_type="inner"):
        """Comprehensive join performance analysis"""
        
        print("🔗 JOIN PERFORMANCE ANALYSIS")
        print("=" * 40)
        
        join_metrics = {
            "data_characteristics": {},
            "join_strategy_analysis": {},
            "skew_detection": {},
            "optimization_opportunities": []
        }
        
        # 1. Data Characteristics Analysis
        print("1. Data Characteristics:")
        try:
            # Get basic stats for both DataFrames
            df1_count = df1.count()
            df2_count = df2.count()
            
            df1_partitions = df1.rdd.getNumPartitions()
            df2_partitions = df2.rdd.getNumPartitions()
            
            join_metrics["data_characteristics"] = {
                "df1_rows": df1_count,
                "df2_rows": df2_count,
                "df1_partitions": df1_partitions,
                "df2_partitions": df2_partitions,
                "size_ratio": df1_count / df2_count if df2_count > 0 else float('inf')
            }
            
            print(f"   Left DF: {df1_count:,} rows, {df1_partitions} partitions")
print(f"   Right DF: {df2_count:,} rows, {df2_partitions} partitions")
           print(f"   Size ratio: {join_metrics['data_characteristics']['size_ratio']:.2f}")
           
           # Analyze join key characteristics
           if isinstance(join_keys, str):
               join_keys = [join_keys]
           
           for key in join_keys:
               if key in df1.columns and key in df2.columns:
                   # Check key cardinality
                   df1_distinct = df1.select(key).distinct().count()
                   df2_distinct = df2.select(key).distinct().count()
                   
                   print(f"   Join key '{key}': {df1_distinct:,} unique in left, {df2_distinct:,} in right")
                   
                   # Check for null values in join keys
                   df1_nulls = df1.filter(col(key).isNull()).count()
                   df2_nulls = df2.filter(col(key).isNull()).count()
                   
                   if df1_nulls > 0 or df2_nulls > 0:
                       print(f"   ⚠️ Null values in join key: {df1_nulls} (left), {df2_nulls} (right)")
                       join_metrics["optimization_opportunities"].append(
                           f"Handle null values in join key '{key}'"
                       )
       
       except Exception as e:
           print(f"   ❌ Data characteristics analysis failed: {e}")
       
       # 2. Join Strategy Analysis
       print("\n2. Join Strategy Analysis:")
       try:
           # Create the join to analyze execution plan
           if isinstance(join_keys, list) and len(join_keys) == 1:
               join_condition = df1[join_keys[0]] == df2[join_keys[0]]
           else:
               # Multiple keys
               conditions = [df1[key] == df2[key] for key in join_keys if key in df1.columns and key in df2.columns]
               join_condition = conditions[0]
               for condition in conditions[1:]:
                   join_condition = join_condition & condition
           
           test_join = df1.join(df2, join_condition, join_type)
           
           # Analyze execution plan
           plan_str = str(test_join._jdf.queryExecution().executedPlan())
           
           join_strategy_analysis = {
               "broadcast_join": "BroadcastHashJoin" in plan_str,
               "sort_merge_join": "SortMergeJoin" in plan_str,
               "shuffle_hash_join": "ShuffleHashJoin" in plan_str,
               "cartesian_join": "CartesianProduct" in plan_str,
               "exchange_count": plan_str.count("Exchange")
           }
           
           join_metrics["join_strategy_analysis"] = join_strategy_analysis
           
           # Determine current strategy
           if join_strategy_analysis["broadcast_join"]:
               current_strategy = "🟢 Broadcast Hash Join"
               print(f"   Current strategy: {current_strategy} (optimal for small right table)")
           elif join_strategy_analysis["sort_merge_join"]:
               current_strategy = "🟡 Sort Merge Join"
               print(f"   Current strategy: {current_strategy} (good for large tables)")
           elif join_strategy_analysis["shuffle_hash_join"]:
               current_strategy = "🟡 Shuffle Hash Join"
               print(f"   Current strategy: {current_strategy}")
           elif join_strategy_analysis["cartesian_join"]:
               current_strategy = "🔴 Cartesian Product"
               print(f"   Current strategy: {current_strategy} (AVOID - very expensive)")
               join_metrics["optimization_opportunities"].append("CRITICAL: Avoid cartesian product joins")
           else:
               current_strategy = "❓ Unknown strategy"
               print(f"   Current strategy: {current_strategy}")
           
           # Analyze shuffle operations
           if join_strategy_analysis["exchange_count"] > 2:
               print(f"   ⚠️ High shuffle cost: {join_strategy_analysis['exchange_count']} exchanges")
               join_metrics["optimization_opportunities"].append("Reduce shuffle operations")
           elif join_strategy_analysis["exchange_count"] <= 1:
               print(f"   ✅ Low shuffle cost: {join_strategy_analysis['exchange_count']} exchanges")
           
       except Exception as e:
           print(f"   ❌ Join strategy analysis failed: {e}")
       
       # 3. Skew Detection
       print("\n3. Data Skew Detection:")
       try:
           skew_results = {}
           
           for key in join_keys:
               if key in df1.columns:
                   # Analyze key distribution in left DataFrame
                   key_distribution = df1.groupBy(key).count().collect()
                   
                   if key_distribution:
                       counts = [row['count'] for row in key_distribution]
                       avg_count = sum(counts) / len(counts)
                       max_count = max(counts)
                       
                       skew_ratio = max_count / avg_count if avg_count > 0 else 0
                       
                       skew_results[f"{key}_left"] = {
                           "avg_count": avg_count,
                           "max_count": max_count,
                           "skew_ratio": skew_ratio,
                           "unique_keys": len(counts)
                       }
                       
                       if skew_ratio > 10:
                           print(f"   🔴 HIGH SKEW in left '{key}': {skew_ratio:.1f}x")
                           join_metrics["optimization_opportunities"].append(
                               f"Address data skew in left table key '{key}'"
                           )
                       elif skew_ratio > 3:
                           print(f"   🟡 MODERATE SKEW in left '{key}': {skew_ratio:.1f}x")
                       else:
                           print(f"   ✅ LOW SKEW in left '{key}': {skew_ratio:.1f}x")
               
               if key in df2.columns:
                   # Similar analysis for right DataFrame
                   key_distribution = df2.groupBy(key).count().collect()
                   
                   if key_distribution:
                       counts = [row['count'] for row in key_distribution]
                       avg_count = sum(counts) / len(counts)
                       max_count = max(counts)
                       
                       skew_ratio = max_count / avg_count if avg_count > 0 else 0
                       
                       skew_results[f"{key}_right"] = {
                           "avg_count": avg_count,
                           "max_count": max_count,
                           "skew_ratio": skew_ratio,
                           "unique_keys": len(counts)
                       }
                       
                       if skew_ratio > 10:
                           print(f"   🔴 HIGH SKEW in right '{key}': {skew_ratio:.1f}x")
                       elif skew_ratio > 3:
                           print(f"   🟡 MODERATE SKEW in right '{key}': {skew_ratio:.1f}x")
                       else:
                           print(f"   ✅ LOW SKEW in right '{key}': {skew_ratio:.1f}x")
           
           join_metrics["skew_detection"] = skew_results
       
       except Exception as e:
           print(f"   ❌ Skew detection failed: {e}")
       
       self.join_analysis = join_metrics
       return join_metrics
   
   def join_optimization_recommendations(self, join_analysis):
       """Generate specific join optimization recommendations"""
       
       print("\n💡 JOIN OPTIMIZATION RECOMMENDATIONS")
       print("-" * 50)
       
       recommendations = {
           "immediate_optimizations": [],
           "configuration_tuning": [],
           "data_preparation": [],
           "alternative_strategies": []
       }
       
       data_chars = join_analysis.get("data_characteristics", {})
       strategy_analysis = join_analysis.get("join_strategy_analysis", {})
       skew_detection = join_analysis.get("skew_detection", {})
       
       # 1. Immediate Optimizations
       
       # Broadcast join opportunities
       df1_rows = data_chars.get("df1_rows", 0)
       df2_rows = data_chars.get("df2_rows", 0)
       
       if df2_rows < 10000000 and not strategy_analysis.get("broadcast_join", False):  # 10M rows
           recommendations["immediate_optimizations"].append({
               "priority": "HIGH",
               "optimization": "Enable broadcast join for smaller table",
               "code": """
# Force broadcast join for smaller table
from pyspark.sql.functions import broadcast
result = df1.join(broadcast(df2), join_condition, "inner")
""",
               "benefit": "Eliminates shuffle operations, significant performance gain"
           })
       
       elif df1_rows < 10000000 and not strategy_analysis.get("broadcast_join", False):
           recommendations["immediate_optimizations"].append({
               "priority": "HIGH", 
               "optimization": "Enable broadcast join for smaller table",
               "code": """
# Force broadcast join for smaller table
from pyspark.sql.functions import broadcast
result = broadcast(df1).join(df2, join_condition, "inner")
""",
               "benefit": "Eliminates shuffle operations, significant performance gain"
           })
       
       # Cartesian product warning
       if strategy_analysis.get("cartesian_join", False):
           recommendations["immediate_optimizations"].append({
               "priority": "CRITICAL",
               "optimization": "Fix cartesian product join",
               "code": """
# Add proper join condition
# AVOID: df1.join(df2)  # This creates cartesian product
# USE: df1.join(df2, join_condition)
""",
               "benefit": "Prevents exponential data explosion"
           })
       
       # 2. Configuration Tuning
       
       # Broadcast threshold tuning
       current_broadcast_threshold = self.spark.conf.get("spark.sql.autoBroadcastJoinThreshold", "10MB")
       
       if df2_rows > 0 and df2_rows < 50000000:  # Up to 50M rows
           recommendations["configuration_tuning"].append({
               "optimization": "Increase broadcast threshold",
               "code": """
# Increase broadcast threshold for larger tables
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "200MB")
""",
               "current_value": current_broadcast_threshold,
               "benefit": "Enables broadcast for larger tables that fit in memory"
           })
       
       # Adaptive query execution
       recommendations["configuration_tuning"].append({
           "optimization": "Enable adaptive query execution",
           "code": """
# Enable AQE for dynamic join optimization
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
""",
           "benefit": "Automatic join strategy optimization at runtime"
       })
       
       # 3. Data Preparation
       
       # Handle data skew
       high_skew_keys = []
       for key, skew_info in skew_detection.items():
           if skew_info.get("skew_ratio", 0) > 5:
               high_skew_keys.append(key)
       
       if high_skew_keys:
           recommendations["data_preparation"].append({
               "optimization": "Address data skew with salting technique",
               "code": """
# Salt heavily skewed keys
from pyspark.sql.functions import rand, concat, lit

# Add salt to skewed DataFrame
salted_df = skewed_df.withColumn("salted_key", 
   concat(col("skewed_column"), lit("_"), (rand() * 100).cast("int")))

# Replicate smaller DataFrame
replicated_df = small_df.select([col(c) for c in small_df.columns] + 
   [((monotonically_increasing_id() % 100)).alias("salt")])
replicated_df = replicated_df.withColumn("salted_key",
   concat(col("join_column"), lit("_"), col("salt")))

# Join on salted keys
result = salted_df.join(replicated_df, "salted_key")
""",
               "affected_keys": high_skew_keys,
               "benefit": "Distributes skewed data across multiple partitions"
           })
       
       # Null value handling
       optimization_ops = join_analysis.get("optimization_opportunities", [])
       null_issues = [op for op in optimization_ops if "null" in op.lower()]
       
       if null_issues:
           recommendations["data_preparation"].append({
               "optimization": "Handle null values in join keys",
               "code": """
# Option 1: Filter out nulls before join
df1_clean = df1.filter(col("join_key").isNotNull())
df2_clean = df2.filter(col("join_key").isNotNull())

# Option 2: Replace nulls with placeholder
df1_filled = df1.fillna({"join_key": "__NULL__"})
df2_filled = df2.fillna({"join_key": "__NULL__"})
""",
               "benefit": "Prevents unexpected results and improves performance"
           })
       
       # 4. Alternative Strategies
       
       # Bucketing for frequent joins
       if df1_rows > 50000000 or df2_rows > 50000000:  # Large tables
           recommendations["alternative_strategies"].append({
               "optimization": "Implement bucketing for frequent joins",
               "code": """
# Create bucketed tables
df1.write.mode("overwrite").bucketBy(16, "join_key").saveAsTable("bucketed_table1")
df2.write.mode("overwrite").bucketBy(16, "join_key").saveAsTable("bucketed_table2")

# Join bucketed tables (no shuffle needed)
bucketed1 = spark.table("bucketed_table1")
bucketed2 = spark.table("bucketed_table2")
result = bucketed1.join(bucketed2, "join_key")
""",
               "benefit": "Eliminates shuffle for frequently joined tables"
           })
       
       # Dimension table broadcasting
       size_ratio = data_chars.get("size_ratio", 1)
       if size_ratio > 100 or size_ratio < 0.01:  # Very different sizes
           recommendations["alternative_strategies"].append({
               "optimization": "Consider dimension table pattern",
               "code": """
# For dimension table joins, always broadcast the smaller table
# Ensure dimension table is cached
dimension_df.cache()
fact_df.join(broadcast(dimension_df), join_condition)
""",
               "benefit": "Optimal pattern for fact-dimension joins"
           })
       
       # Display recommendations
       for category, recs in recommendations.items():
           if recs:
               print(f"\n{category.replace('_', ' ').title()}:")
               for i, rec in enumerate(recs, 1):
                   print(f"\n   {i}. {rec['optimization']}")
                   if 'priority' in rec:
                       print(f"      Priority: {rec['priority']}")
                   print(f"      Benefit: {rec['benefit']}")
                   if 'current_value' in rec:
                       print(f"      Current: {rec['current_value']}")
                   print(f"      Implementation:")
                   print(f"{rec['code']}")
       
       return recommendations
   
   def join_performance_test(self, df1, df2, join_keys, join_type="inner"):
       """Performance test different join strategies"""
       
       print("\n⚡ JOIN PERFORMANCE TESTING")
       print("-" * 40)
       
       test_results = {}
       
       # Prepare join condition
       if isinstance(join_keys, str):
           join_keys = [join_keys]
       
       if len(join_keys) == 1:
           join_condition = df1[join_keys[0]] == df2[join_keys[0]]
       else:
           conditions = [df1[key] == df2[key] for key in join_keys]
           join_condition = conditions[0]
           for condition in conditions[1:]:
               join_condition = join_condition & condition
       
       # Test 1: Default join strategy
       print("1. Testing default join strategy...")
       try:
           start_time = time.time()
           default_result = df1.join(df2, join_condition, join_type)
           row_count = default_result.count()
           default_time = time.time() - start_time
           
           test_results["default"] = {
               "duration": default_time,
               "row_count": row_count,
               "strategy": "default"
           }
           
           print(f"   ✅ Default: {default_time:.2f}s ({row_count:,} rows)")
           
       except Exception as e:
           print(f"   ❌ Default join failed: {e}")
           test_results["default"] = {"error": str(e)}
       
       # Test 2: Broadcast join (if applicable)
       df2_count = df2.count()
       if df2_count < 10000000:  # Only test if reasonable size
           print("2. Testing broadcast join...")
           try:
               start_time = time.time()
               broadcast_result = df1.join(broadcast(df2), join_condition, join_type)
               row_count = broadcast_result.count()
               broadcast_time = time.time() - start_time
               
               test_results["broadcast"] = {
                   "duration": broadcast_time,
                   "row_count": row_count,
                   "strategy": "broadcast"
               }
               
               print(f"   ✅ Broadcast: {broadcast_time:.2f}s ({row_count:,} rows)")
               
               # Compare with default
               if "default" in test_results and "error" not in test_results["default"]:
                   improvement = test_results["default"]["duration"] / broadcast_time
                   if improvement > 1.2:
                       print(f"   🚀 {improvement:.1f}x faster than default")
                   elif improvement < 0.8:
                       print(f"   🐌 {1/improvement:.1f}x slower than default")
               
           except Exception as e:
               print(f"   ❌ Broadcast join failed: {e}")
               test_results["broadcast"] = {"error": str(e)}
       else:
           print("2. Skipping broadcast join test (table too large)")
       
       # Test 3: With adaptive query execution
       print("3. Testing with adaptive query execution...")
       try:
           # Save current AQE setting
           current_aqe = self.spark.conf.get("spark.sql.adaptive.enabled", "false")
           
           # Enable AQE
           self.spark.conf.set("spark.sql.adaptive.enabled", "true")
           self.spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
           
           start_time = time.time()
           aqe_result = df1.join(df2, join_condition, join_type)
           row_count = aqe_result.count()
           aqe_time = time.time() - start_time
           
           test_results["adaptive"] = {
               "duration": aqe_time,
               "row_count": row_count,
               "strategy": "adaptive"
           }
           
           print(f"   ✅ Adaptive: {aqe_time:.2f}s ({row_count:,} rows)")
           
           # Restore AQE setting
           self.spark.conf.set("spark.sql.adaptive.enabled", current_aqe)
           
       except Exception as e:
           print(f"   ❌ Adaptive join failed: {e}")
           test_results["adaptive"] = {"error": str(e)}
       
       # Performance summary
       print("\n📊 Performance Summary:")
       valid_results = {k: v for k, v in test_results.items() if "error" not in v}
       
       if valid_results:
           best_strategy = min(valid_results.keys(), key=lambda k: valid_results[k]["duration"])
           best_time = valid_results[best_strategy]["duration"]
           
           print(f"   🏆 Best strategy: {best_strategy} ({best_time:.2f}s)")
           
           for strategy, result in valid_results.items():
               duration = result["duration"]
               relative_perf = duration / best_time
               
               if strategy == best_strategy:
                   status = "🏆 BEST"
               elif relative_perf < 1.2:
                   status = "🟢 GOOD"
               elif relative_perf < 2.0:
                   status = "🟡 OK"
               else:
                   status = "🔴 SLOW"
               
               print(f"   {status} {strategy}: {duration:.2f}s ({relative_perf:.1f}x)")
       
       return test_results 
```

## 🎛️ Configuration Issues

### **Configuration Troubleshooting Toolkit**

python

```python
class ConfigurationTroubleshooter:
    """Comprehensive Spark configuration analysis and optimization"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.config_analysis = {}
        
        # Define configuration categories and their importance
        self.config_categories = {
            "memory": {
                "configs": [
                    "spark.executor.memory",
                    "spark.driver.memory", 
                    "spark.executor.memoryFraction",
                    "spark.executor.memoryOffHeap.enabled",
                    "spark.executor.memoryOffHeap.size"
                ],
                "importance": "CRITICAL"
            },
            "parallelism": {
                "configs": [
                    "spark.executor.instances",
                    "spark.executor.cores",
                    "spark.sql.shuffle.partitions",
                    "spark.default.parallelism"
                ],
                "importance": "HIGH"
            },
            "serialization": {
                "configs": [
                    "spark.serializer",
                    "spark.sql.execution.arrow.maxRecordsPerBatch"
                ],
                "importance": "MEDIUM"
            },
            "adaptive": {
                "configs": [
                    "spark.sql.adaptive.enabled",
                    "spark.sql.adaptive.coalescePartitions.enabled",
                    "spark.sql.adaptive.skewJoin.enabled"
                ],
                "importance": "HIGH"
            },
            "broadcast": {
                "configs": [
                    "spark.sql.autoBroadcastJoinThreshold",
                    "spark.sql.broadcastTimeout"
                ],
                "importance": "MEDIUM"
            },
            "compression": {
                "configs": [
                    "spark.sql.parquet.compression.codec",
                    "spark.io.compression.codec",
                    "spark.rdd.compress"
                ],
                "importance": "LOW"
            }
        }
    
    def comprehensive_config_audit(self):
        """Perform comprehensive configuration audit"""
        
        print("🎛️ COMPREHENSIVE CONFIGURATION AUDIT")
        print("=" * 50)
        
        audit_results = {
            "current_config": {},
            "missing_config": {},
            "suboptimal_config": {},
            "deprecated_config": {},
            "recommendations": []
        }
        
        # 1. Collect current configuration
        print("1. Current Configuration Analysis:")
        
        all_configs = {}
        for category, info in self.config_categories.items():
            category_configs = {}
            
            for config_key in info["configs"]:
                try:
                    value = self.spark.conf.get(config_key)
                    category_configs[config_key] = value
                    all_configs[config_key] = value
                except:
                    category_configs[config_key] = "NOT_SET"
                    all_configs[config_key] = "NOT_SET"
            
            audit_results["current_config"][category] = category_configs
            
            # Display category results
            print(f"\n   {category.title()} ({info['importance']} priority):")
            for config_key, value in category_configs.items():
                if value == "NOT_SET":
                    print(f"     ❌ {config_key}: {value}")
                else:
                    print(f"     ✅ {config_key}: {value}")
        
        # 2. Identify missing critical configurations
        print(f"\n2. Missing Configuration Analysis:")
        
        critical_missing = []
        high_missing = []
        
        for category, info in self.config_categories.items():
            missing_in_category = []
            
            for config_key in info["configs"]:
                if all_configs[config_key] == "NOT_SET":
                    missing_in_category.append(config_key)
                    
                    if info["importance"] == "CRITICAL":
                        critical_missing.append(config_key)
                    elif info["importance"] == "HIGH":
                        high_missing.append(config_key)
            
            if missing_in_category:
                audit_results["missing_config"][category] = missing_in_category
        
        if critical_missing:
            print(f"   🔴 CRITICAL missing configurations:")
            for config in critical_missing:
                print(f"     - {config}")
        
        if high_missing:
            print(f"   🟡 HIGH priority missing configurations:")
            for config in high_missing:
                print(f"     - {config}")
        
        if not critical_missing and not high_missing:
            print(f"   ✅ All high-priority configurations are set")
        
        # 3. Detect suboptimal configurations
        print(f"\n3. Suboptimal Configuration Detection:")
        
        suboptimal_issues = self._detect_suboptimal_configs(all_configs)
        audit_results["suboptimal_config"] = suboptimal_issues
        
        if suboptimal_issues:
            for issue_type, issues in suboptimal_issues.items():
                if issues:
                    print(f"   🟡 {issue_type.replace('_', ' ').title()}:")
                    for issue in issues:
                        print(f"     - {issue}")
        else:
            print(f"   ✅ No obvious suboptimal configurations detected")
        
        # 4. Check for deprecated configurations
        print(f"\n4. Deprecated Configuration Check:")
        
        deprecated_configs = self._check_deprecated_configs()
        audit_results["deprecated_config"] = deprecated_configs
        
        if deprecated_configs:
            print(f"   ⚠️ Deprecated configurations found:")
            for config, message in deprecated_configs.items():
                print(f"     - {config}: {message}")
        else:
            print(f"   ✅ No deprecated configurations detected")
        
        # 5. Generate recommendations
        audit_results["recommendations"] = self._generate_config_recommendations(
            all_configs, suboptimal_issues, critical_missing, high_missing
        )
        
        self.config_analysis = audit_results
        return audit_results
    
    def _detect_suboptimal_configs(self, configs):
        """Detect suboptimal configuration patterns"""
        
        issues = {
            "memory_issues": [],
            "parallelism_issues": [],
            "performance_issues": [],
            "resource_mismatch": []
        }
        
        # Memory configuration issues
        executor_memory = configs.get("spark.executor.memory", "NOT_SET")
        driver_memory = configs.get("spark.driver.memory", "NOT_SET")
        
        if executor_memory != "NOT_SET":
            # Extract memory value (assuming format like "4g", "2048m")
            try:
                if executor_memory.endswith("g"):
                    executor_gb = float(executor_memory[:-1])
                elif executor_memory.endswith("m"):
                    executor_gb = float(executor_memory[:-1]) / 1024
                else:
                    executor_gb = float(executor_memory) / (1024**3)  # Assume bytes
                
                if executor_gb < 2:
                    issues["memory_issues"].append(f"Executor memory very low: {executor_memory}")
                elif executor_gb > 32:
                    issues["memory_issues"].append(f"Executor memory very high: {executor_memory} (may cause GC issues)")
                
            except:
                issues["memory_issues"].append(f"Invalid executor memory format: {executor_memory}")
        
        # Parallelism issues
        shuffle_partitions = configs.get("spark.sql.shuffle.partitions", "NOT_SET")
        executor_instances = configs.get("spark.executor.instances", "NOT_SET")
        executor_cores = configs.get("spark.executor.cores", "NOT_SET")
        
        if shuffle_partitions != "NOT_SET":
            try:
                partitions = int(shuffle_partitions)
                if partitions == 200:  # Default value
                    issues["parallelism_issues"].append("Using default shuffle partitions (200) - consider tuning")
                elif partitions > 2000:
                    issues["parallelism_issues"].append(f"Very high shuffle partitions: {partitions}")
                elif partitions < 10:
                    issues["parallelism_issues"].append(f"Very low shuffle partitions: {partitions}")
            except:
                issues["parallelism_issues"].append(f"Invalid shuffle partitions: {shuffle_partitions}")
        
        # Performance configuration issues
        serializer = configs.get("spark.serializer", "NOT_SET")
        if serializer == "NOT_SET" or "Java" in serializer:
            issues["performance_issues"].append("Not using Kryo serializer (performance impact)")
        
        adaptive_enabled = configs.get("spark.sql.adaptive.enabled", "NOT_SET")
        if adaptive_enabled == "false" or adaptive_enabled == "NOT_SET":
            issues["performance_issues"].append("Adaptive query execution disabled")
        
        # Resource mismatch detection
        if (executor_instances != "NOT_SET" and executor_cores != "NOT_SET" and 
            shuffle_partitions != "NOT_SET"):
            try:
                instances = int(executor_instances)
                cores = int(executor_cores)
                partitions = int(shuffle_partitions)
                
                total_cores = instances * cores
                
                if partitions > total_cores * 4:
                    issues["resource_mismatch"].append(
                        f"Too many partitions ({partitions}) for available cores ({total_cores})"
                    )
                elif partitions < total_cores / 2:
                    issues["resource_mismatch"].append(
                        f"Too few partitions ({partitions}) for available cores ({total_cores})"
                    )
            except:
                pass
        
        return issues
    
    def _check_deprecated_configs(self):
        """Check for deprecated configuration parameters"""
        
        deprecated_configs = {}
        
        # Define known deprecated configurations
        deprecated_list = {
            "spark.sql.tungsten.enabled": "Tungsten is always enabled in Spark 2.0+",
            "spark.sql.codegen": "Code generation is always enabled in Spark 2.0+",
            "spark.shuffle.consolidateFiles": "Deprecated in Spark 2.0",
            "spark.akka.timeout": "Replaced with spark.network.timeout"
        }
        
        for deprecated_config, message in deprecated_list.items():
            try:
                value = self.spark.conf.get(deprecated_config)
deprecated_configs[deprecated_config] = f"{message} (current value: {value})"
           except:
               # Config not set, which is good for deprecated configs
               pass
       
       return deprecated_configs
   
   def _generate_config_recommendations(self, configs, suboptimal_issues, critical_missing, high_missing):
       """Generate specific configuration recommendations"""
       
       recommendations = []
       
       # Critical missing configurations
       if critical_missing:
           recommendations.append({
               "priority": "CRITICAL",
               "category": "Missing Critical Configs",
               "issue": f"{len(critical_missing)} critical configurations missing",
               "action": "Set essential memory and resource configurations",
               "code": """
# Essential memory configurations
spark.conf.set("spark.executor.memory", "4g")
spark.conf.set("spark.driver.memory", "2g")
spark.conf.set("spark.executor.memoryFraction", "0.8")
""",
               "configs": critical_missing
           })
       
       # Memory optimizations
       memory_issues = suboptimal_issues.get("memory_issues", [])
       if memory_issues:
           recommendations.append({
               "priority": "HIGH",
               "category": "Memory Optimization",
               "issue": f"{len(memory_issues)} memory configuration issues",
               "action": "Optimize memory allocation and enable off-heap memory",
               "code": """
# Optimized memory configuration
spark.conf.set("spark.executor.memory", "6g")
spark.conf.set("spark.executor.memoryOffHeap.enabled", "true")
spark.conf.set("spark.executor.memoryOffHeap.size", "2g")
spark.conf.set("spark.executor.memoryFraction", "0.8")
""",
               "issues": memory_issues
           })
       
       # Performance optimizations
       performance_issues = suboptimal_issues.get("performance_issues", [])
       if performance_issues:
           recommendations.append({
               "priority": "HIGH",
               "category": "Performance Optimization", 
               "issue": f"{len(performance_issues)} performance configuration issues",
               "action": "Enable key performance features",
               "code": """
# Performance optimizations
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
""",
               "issues": performance_issues
           })
       
       # Parallelism tuning
       parallelism_issues = suboptimal_issues.get("parallelism_issues", [])
       if parallelism_issues:
           # Try to estimate optimal partition count
           try:
               executor_instances = int(configs.get("spark.executor.instances", "4"))
               executor_cores = int(configs.get("spark.executor.cores", "2"))
               total_cores = executor_instances * executor_cores
               optimal_partitions = total_cores * 2  # 2x cores is a good starting point
           except:
               optimal_partitions = 100  # Fallback
           
           recommendations.append({
               "priority": "MEDIUM",
               "category": "Parallelism Tuning",
               "issue": f"{len(parallelism_issues)} parallelism configuration issues",
               "action": "Tune partition count based on cluster resources",
               "code": f"""
# Parallelism optimization
spark.conf.set("spark.sql.shuffle.partitions", "{optimal_partitions}")
spark.conf.set("spark.default.parallelism", "{optimal_partitions}")
""",
               "issues": parallelism_issues
           })
       
       # Resource mismatch fixes
       resource_issues = suboptimal_issues.get("resource_mismatch", [])
       if resource_issues:
           recommendations.append({
               "priority": "MEDIUM",
               "category": "Resource Alignment",
               "issue": f"{len(resource_issues)} resource mismatch issues",
               "action": "Align partition count with available cores",
               "code": """
# Calculate optimal partitions based on cores
total_cores = executor_instances * executor_cores
optimal_partitions = total_cores * 2

spark.conf.set("spark.sql.shuffle.partitions", str(optimal_partitions))
""",
               "issues": resource_issues
           })
       
       return recommendations
   
   def environment_specific_tuning(self, environment_type="production", workload_type="mixed"):
       """Generate environment-specific configuration recommendations"""
       
       print(f"🎯 ENVIRONMENT-SPECIFIC TUNING: {environment_type.upper()}")
       print("=" * 50)
       
       tuning_recommendations = {
           "development": {
               "focus": "Fast feedback and resource efficiency",
               "configs": {
                   "spark.executor.memory": "2g",
                   "spark.driver.memory": "1g", 
                   "spark.sql.shuffle.partitions": "20",
                   "spark.sql.adaptive.enabled": "true",
                   "spark.serializer": "org.apache.spark.serializer.KryoSerializer"
               }
           },
           "testing": {
               "focus": "Stability and consistent performance",
               "configs": {
                   "spark.executor.memory": "4g",
                   "spark.driver.memory": "2g",
                   "spark.sql.shuffle.partitions": "100", 
                   "spark.sql.adaptive.enabled": "true",
                   "spark.sql.adaptive.coalescePartitions.enabled": "true",
                   "spark.serializer": "org.apache.spark.serializer.KryoSerializer"
               }
           },
           "production": {
               "focus": "Maximum performance and reliability",
               "configs": {
                   "spark.executor.memory": "8g",
                   "spark.driver.memory": "4g",
                   "spark.executor.memoryOffHeap.enabled": "true",
                   "spark.executor.memoryOffHeap.size": "4g", 
                   "spark.sql.shuffle.partitions": "200",
                   "spark.sql.adaptive.enabled": "true",
                   "spark.sql.adaptive.coalescePartitions.enabled": "true",
                   "spark.sql.adaptive.skewJoin.enabled": "true",
                   "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
                   "spark.sql.execution.arrow.maxRecordsPerBatch": "10000"
               }
           }
       }
       
       workload_adjustments = {
           "etl": {
               "description": "ETL/Batch processing workloads",
               "adjustments": {
                   "spark.sql.shuffle.partitions": "+50%",  # More partitions for large data
                   "spark.executor.memoryOffHeap.size": "+2g",  # More off-heap for caching
                   "spark.sql.parquet.compression.codec": "snappy"  # Fast compression
               }
           },
           "analytics": {
               "description": "Interactive analytics workloads", 
               "adjustments": {
                   "spark.sql.autoBroadcastJoinThreshold": "100MB",  # Larger broadcast threshold
                   "spark.sql.adaptive.advisoryPartitionSizeInBytes": "128MB",  # Larger partitions
                   "spark.sql.execution.arrow.maxRecordsPerBatch": "20000"  # Larger batches
               }
           },
           "streaming": {
               "description": "Real-time streaming workloads",
               "adjustments": {
                   "spark.sql.streaming.checkpointLocation": "/path/to/checkpoints",
                   "spark.sql.streaming.stateStore.providerClass": "org.apache.spark.sql.execution.streaming.state.HDFSBackedStateStoreProvider",
                   "spark.sql.adaptive.enabled": "false"  # Disable AQE for streaming
               }
           }
       }
       
       # Get base configuration for environment
       base_config = tuning_recommendations.get(environment_type, tuning_recommendations["production"])
       
       print(f"Environment: {environment_type}")
       print(f"Focus: {base_config['focus']}")
       print(f"Workload: {workload_type}")
       
       if workload_type in workload_adjustments:
           print(f"Workload description: {workload_adjustments[workload_type]['description']}")
       
       print(f"\nRecommended Configuration:")
       
       # Apply base configuration
       final_config = base_config["configs"].copy()
       
       # Apply workload-specific adjustments
       if workload_type in workload_adjustments:
           adjustments = workload_adjustments[workload_type]["adjustments"]
           
           for config_key, adjustment in adjustments.items():
               if config_key in final_config:
                   # Handle percentage adjustments
                   if isinstance(adjustment, str) and "%" in adjustment:
                       try:
                           base_value = final_config[config_key]
                           if base_value.endswith("g"):
                               base_num = float(base_value[:-1])
                               percent = float(adjustment.replace("%", "").replace("+", ""))
                               new_value = base_num * (1 + percent/100)
                               final_config[config_key] = f"{new_value:.0f}g"
                           elif base_value.isdigit():
                               base_num = int(base_value)
                               percent = float(adjustment.replace("%", "").replace("+", ""))
                               new_value = int(base_num * (1 + percent/100))
                               final_config[config_key] = str(new_value)
                       except:
                           # If adjustment fails, use original value
                           pass
                   else:
                       final_config[config_key] = adjustment
               else:
                   final_config[config_key] = adjustment
       
       # Display final configuration
       for config_key, value in final_config.items():
           print(f"   {config_key} = {value}")
       
       # Generate implementation code
       print(f"\nImplementation Code:")
       print("```python")
       for config_key, value in final_config.items():
           print(f'spark.conf.set("{config_key}", "{value}")')
       print("```")
       
       return final_config
   
   def configuration_validation(self, target_configs=None):
       """Validate current configuration against targets or best practices"""
       
       print("✅ CONFIGURATION VALIDATION")
       print("=" * 40)
       
       validation_results = {
           "compliant": [],
           "non_compliant": [],
           "missing": [],
           "validation_score": 0
       }
       
       # Define validation rules if no targets provided
       if target_configs is None:
           target_configs = {
               "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
               "spark.sql.adaptive.enabled": "true",
               "spark.sql.adaptive.coalescePartitions.enabled": "true"
           }
       
       total_checks = len(target_configs)
       passed_checks = 0
       
       print("Validating against target configuration:")
       
       for config_key, expected_value in target_configs.items():
           try:
               current_value = self.spark.conf.get(config_key)
               
               if current_value == expected_value:
                   validation_results["compliant"].append({
                       "config": config_key,
                       "expected": expected_value,
                       "actual": current_value
                   })
                   print(f"   ✅ {config_key}: {current_value}")
                   passed_checks += 1
               else:
                   validation_results["non_compliant"].append({
                       "config": config_key,
                       "expected": expected_value,
                       "actual": current_value
                   })
                   print(f"   ❌ {config_key}: {current_value} (expected: {expected_value})")
                   
           except:
               validation_results["missing"].append({
                   "config": config_key,
                   "expected": expected_value
               })
               print(f"   ❌ {config_key}: NOT SET (expected: {expected_value})")
       
       # Calculate validation score
       validation_score = (passed_checks / total_checks) * 100 if total_checks > 0 else 0
       validation_results["validation_score"] = validation_score
       
       print(f"\nValidation Score: {validation_score:.1f}%")
       
       if validation_score >= 90:
           print("🟢 EXCELLENT - Configuration meets best practices")
       elif validation_score >= 75:
           print("🟡 GOOD - Minor configuration improvements needed")
       elif validation_score >= 50:
           print("🟠 POOR - Significant configuration issues")
       else:
           print("🔴 CRITICAL - Major configuration problems")
       
       # Generate fix commands for non-compliant configs
       if validation_results["non_compliant"] or validation_results["missing"]:
           print(f"\nConfiguration Fixes:")
           
           all_issues = validation_results["non_compliant"] + validation_results["missing"]
           
           for issue in all_issues:
               config_key = issue["config"]
               expected_value = issue["expected"]
               print(f'   spark.conf.set("{config_key}", "{expected_value}")')
       
       return validation_results
   
   def generate_configuration_template(self, cluster_size="medium", use_case="general"):
       """Generate complete configuration template"""
       
       print(f"📋 CONFIGURATION TEMPLATE GENERATOR")
       print("=" * 50)
       
       templates = {
           "small": {
               "description": "Small cluster (2-4 executors)",
               "configs": {
                   "spark.executor.instances": "4",
                   "spark.executor.cores": "2", 
                   "spark.executor.memory": "4g",
                   "spark.driver.memory": "2g",
                   "spark.sql.shuffle.partitions": "16"
               }
           },
           "medium": {
               "description": "Medium cluster (5-20 executors)",
               "configs": {
                   "spark.executor.instances": "10",
                   "spark.executor.cores": "4",
                   "spark.executor.memory": "8g", 
                   "spark.driver.memory": "4g",
                   "spark.sql.shuffle.partitions": "80"
               }
           },
           "large": {
               "description": "Large cluster (20+ executors)",
               "configs": {
                   "spark.executor.instances": "50",
                   "spark.executor.cores": "4",
                   "spark.executor.memory": "12g",
                   "spark.driver.memory": "8g",
                   "spark.sql.shuffle.partitions": "400"
               }
           }
       }
       
       use_case_configs = {
           "general": {
               "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
               "spark.sql.adaptive.enabled": "true",
               "spark.sql.adaptive.coalescePartitions.enabled": "true"
           },
           "etl": {
               "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
               "spark.sql.adaptive.enabled": "true",
               "spark.sql.adaptive.coalescePartitions.enabled": "true",
               "spark.sql.parquet.compression.codec": "snappy",
               "spark.executor.memoryOffHeap.enabled": "true",
               "spark.executor.memoryOffHeap.size": "4g"
           },
           "analytics": {
               "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
               "spark.sql.adaptive.enabled": "true",
               "spark.sql.adaptive.coalescePartitions.enabled": "true",
               "spark.sql.autoBroadcastJoinThreshold": "100MB",
               "spark.sql.execution.arrow.maxRecordsPerBatch": "20000"
           }
       }
       
       # Build complete configuration
       base_config = templates.get(cluster_size, templates["medium"])
       use_case_config = use_case_configs.get(use_case, use_case_configs["general"])
       
       complete_config = {**base_config["configs"], **use_case_config}
       
       print(f"Cluster Size: {cluster_size}")
       print(f"Description: {base_config['description']}")
       print(f"Use Case: {use_case}")
       
       print(f"\nComplete Configuration Template:")
       print("```python")
       print("# Spark Configuration Template")
       print(f"# Cluster: {cluster_size}, Use Case: {use_case}")
       print()
       
       # Group configs by category for better readability
       config_groups = {
           "Resource Allocation": ["spark.executor.instances", "spark.executor.cores", "spark.executor.memory", "spark.driver.memory"],
           "Performance": ["spark.serializer", "spark.sql.adaptive.enabled", "spark.sql.adaptive.coalescePartitions.enabled"],
           "Parallelism": ["spark.sql.shuffle.partitions", "spark.default.parallelism"],
           "Memory Management": ["spark.executor.memoryOffHeap.enabled", "spark.executor.memoryOffHeap.size"],
           "Optimization": ["spark.sql.autoBroadcastJoinThreshold", "spark.sql.execution.arrow.maxRecordsPerBatch", "spark.sql.parquet.compression.codec"]
       }
       
       for group_name, group_configs in config_groups.items():
           group_has_configs = any(config in complete_config for config in group_configs)
           
           if group_has_configs:
               print(f"# {group_name}")
               for config_key in group_configs:
                   if config_key in complete_config:
                       print(f'spark.conf.set("{config_key}", "{complete_config[config_key]}")')
               print()
       
       # Add any remaining configs not in groups
       remaining_configs = {k: v for k, v in complete_config.items() 
                          if not any(k in group for group in config_groups.values())}
       
       if remaining_configs:
           print("# Additional Configurations")
           for config_key, value in remaining_configs.items():
               print(f'spark.conf.set("{config_key}", "{value}")')
       
       print("```")
       
       return complete_config
```

## 🔌 Connectivity Problems

### **Network and Connectivity Troubleshooting**

```python
class ConnectivityTroubleshooter:
    """Network connectivity and external system troubleshooting"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.connectivity_results = {}
    
    def comprehensive_connectivity_check(self):
        """Perform comprehensive connectivity diagnostics"""
        
        print("🔌 COMPREHENSIVE CONNECTIVITY CHECK")
        print("=" * 50)
        
        connectivity_status = {
            "cluster_connectivity": {},
            "storage_connectivity": {},
            "external_services": {},
            "network_performance": {}
        }
        
        # 1. Cluster Internal Connectivity
        print("1. Cluster Internal Connectivity:")
        try:
            # Check Spark context and cluster manager
            master = self.spark.sparkContext.master
            app_id = self.spark.sparkContext.applicationId
            
            connectivity_status["cluster_connectivity"]["master"] = master
            connectivity_status["cluster_connectivity"]["app_id"] = app_id
            
            print(f"   Master: {master}")
            print(f"   Application ID: {app_id}")
            
            # Check executor connectivity
            status = self.spark.sparkContext.statusTracker()
            executors = status.getExecutorInfos()
            
            active_executors = [e for e in executors if e.isActive]
            failed_executors = [e for e in executors if not e.isActive]
            
            connectivity_status["cluster_connectivity"]["executors"] = {
                "total": len(executors),
                "active": len(active_executors),
                "failed": len(failed_executors)
            }
            
            if len(active_executors) > 0:
                print(f"   ✅ Executors: {len(active_executors)} active, {len(failed_executors)} failed")
            else:
                print(f"   🔴 No active executors available")
            
            # Test simple distributed operation
            try:
                test_rdd = self.spark.sparkContext.parallelize(range(100), 4)
                result = test_rdd.sum()
                print(f"   ✅ Distributed computation test: {result}")
                connectivity_status["cluster_connectivity"]["distributed_test"] = "PASSED"
            except Exception as e:
                print(f"   ❌ Distributed computation failed: {e}")
                connectivity_status["cluster_connectivity"]["distributed_test"] = f"FAILED: {e}"
        
        except Exception as e:
            print(f"   ❌ Cluster connectivity check failed: {e}")
        
        return connectivity_status
    
    def test_storage_connectivity(self, storage_paths=None):
        """Test connectivity to various storage systems"""
        
        print("\n2. Storage System Connectivity:")
        
        storage_results = {}
        
        # Default storage paths to test if none provided
        if storage_paths is None:
            storage_paths = {
                "local": "/tmp",
                "hdfs": "hdfs://localhost:9000/",
                "s3": "s3a://test-bucket/",
                "gcs": "gs://test-bucket/"
            }
        
        for storage_type, path in storage_paths.items():
            try:
                print(f"   Testing {storage_type} ({path}):")
                
                # Test read access
                try:
                    test_df = self.spark.range(10).coalesce(1)
                    test_path = f"{path.rstrip('/')}/spark_connectivity_test"
                    
                    # Write test
                    test_df.write.mode("overwrite").parquet(test_path)
                    print(f"     ✅ Write access confirmed")
                    
                    # Read test
                    read_df = self.spark.read.parquet(test_path)
                    count = read_df.count()
                    print(f"     ✅ Read access confirmed ({count} rows)")
                    
                    # Cleanup
                    try:
                        # This is a simplified cleanup - in practice you'd use appropriate APIs
                        pass
                    except:
                        pass
                    
                    storage_results[storage_type] = "ACCESSIBLE"
                    
                except Exception as storage_error:
                    if "not found" in str(storage_error).lower() or "no such file" in str(storage_error).lower():
                        print(f"     🟡 Path not accessible: {storage_error}")
                        storage_results[storage_type] = "PATH_NOT_FOUND"
                    elif "permission" in str(storage_error).lower() or "access denied" in str(storage_error).lower():
                        print(f"     🔴 Permission denied: {storage_error}")
                        storage_results[storage_type] = "PERMISSION_DENIED"
                    else:
                        print(f"     ❌ Connection failed: {storage_error}")
                        storage_results[storage_type] = f"FAILED: {storage_error}"
            
            except Exception as e:
                print(f"   ❌ {storage_type} test failed: {e}")
                storage_results[storage_type] = f"ERROR: {e}"
        
        return storage_results
    
    def test_external_services(self, service_configs=None):
        """Test connectivity to external services and databases"""
        
        print("\n3. External Services Connectivity:")
        
        external_results = {}
        
        # Default service configurations
        if service_configs is None:
            service_configs = {
                "jdbc_postgres": {
                    "url": "jdbc:postgresql://localhost:5432/testdb",
                    "properties": {"user": "test", "password": "test"}
                },
                "jdbc_mysql": {
                    "url": "jdbc:mysql://localhost:3306/testdb", 
                    "properties": {"user": "test", "password": "test"}
                }
            }
        
        for service_name, config in service_configs.items():
            try:
                print(f"   Testing {service_name}:")
                
                if service_name.startswith("jdbc"):
                    # Test JDBC connection
                    try:
                        # Simple connection test
                        test_query = "(SELECT 1 as test_column) as test_table"
                        
                        df = self.spark.read.format("jdbc") \
                            .option("url", config["url"]) \
                            .option("dbtable", test_query) \
                            .options(**config["properties"]) \
                            .load()
                        
                        result = df.collect()
                        print(f"     ✅ Connection successful")
                        external_results[service_name] = "CONNECTED"
                        
                    except Exception as jdbc_error:
                        error_msg = str(jdbc_error).lower()
                        if "connection refused" in error_msg:
                            print(f"     🔴 Connection refused - service may be down")
                            external_results[service_name] = "CONNECTION_REFUSED"
                        elif "authentication" in error_msg or "password" in error_msg:
                            print(f"     🔴 Authentication failed")
                            external_results[service_name] = "AUTH_FAILED"
                        elif "timeout" in error_msg:
                            print(f"     🟡 Connection timeout")
                            external_results[service_name] = "TIMEOUT"
                        else:
                            print(f"     ❌ Connection failed: {jdbc_error}")
                            external_results[service_name] = f"FAILED: {jdbc_error}"
                
                elif service_name.startswith("kafka"):
                    # Test Kafka connectivity (placeholder)
                    print(f"     ℹ️ Kafka connectivity test not implemented")
                    external_results[service_name] = "NOT_IMPLEMENTED"
                
                elif service_name.startswith("elasticsearch"):
                    # Test Elasticsearch connectivity (placeholder)
                    print(f"     ℹ️ Elasticsearch connectivity test not implemented")
                    external_results[service_name] = "NOT_IMPLEMENTED"
                
                else:
                    print(f"     ❌ Unknown service type")
                    external_results[service_name] = "UNKNOWN_TYPE"
            
            except Exception as e:
                print(f"   ❌ {service_name} test failed: {e}")
                external_results[service_name] = f"ERROR: {e}"
        
        return external_results
    
    def network_performance_analysis(self):
        """Analyze network performance and identify bottlenecks"""
        
        print("\n4. Network Performance Analysis:")
        
        performance_metrics = {}
        
        try:
            # Test data transfer performance
            print("   Testing data transfer performance:")
            
            # Create test data of different sizes
            test_sizes = [1000, 10000, 100000]  # rows
            
            for size in test_sizes:
                try:
                    # Create test DataFrame
                    start_time = time.time()
                    test_df = self.spark.range(size).selectExpr("id", "id * 2 as doubled", "rand() as random_val")
                    
                    # Force computation and data movement
                    result_count = test_df.repartition(4).count()
                    transfer_time = time.time() - start_time
                    
                    throughput = size / transfer_time  # rows per second
                    
                    performance_metrics[f"transfer_{size}"] = {
                        "size": size,
                        "time_seconds": transfer_time,
                        "throughput_rows_sec": throughput
                    }
                    
                    print(f"     {size:,} rows: {transfer_time:.2f}s ({throughput:.0f} rows/sec)")
                    
                except Exception as test_error:
                    print(f"     ❌ {size} rows test failed: {test_error}")
        
        except Exception as e:
            print(f"   ❌ Network performance test failed: {e}")
        
        # Analyze shuffle performance
        try:
            print("   Testing shuffle performance:")
            
            start_time = time.time()
            test_df = self.spark.range(50000).selectExpr("id", "id % 100 as group_key", "rand() as value")
            
            # Force shuffle operation
            shuffled_result = test_df.groupBy("group_key").count().collect()
            shuffle_time = time.time() - start_time
            
            performance_metrics["shuffle_test"] = {
                "time_seconds": shuffle_time,
                "groups": len(shuffled_result)
            }
            
            print(f"     Shuffle operation: {shuffle_time:.2f}s ({len(shuffled_result)} groups)")
            
            if shuffle_time > 10:
                print(f"     ⚠️ Slow shuffle performance detected")
            
        except Exception as e:
            print(f"   ❌ Shuffle performance test failed: {e}")
        
        return performance_metrics
    
    def diagnose_connectivity_issues(self, connectivity_results):
        """Diagnose connectivity issues and provide solutions"""
        
        print("\n🔍 CONNECTIVITY ISSUE DIAGNOSIS")
        print("-" * 40)
        
        issues_found = []
        recommendations = []
        
        # Analyze cluster connectivity
        cluster_conn = connectivity_results.get("cluster_connectivity", {})
        
        if cluster_conn.get("executors", {}).get("active", 0) == 0:
            issues_found.append("No active executors - cluster communication failure")
            recommendations.extend([
                "Check cluster manager status (YARN/Kubernetes/Standalone)",
                "Verify network connectivity between driver and cluster manager",
                "Check firewall rules and security groups"
            ])
        
        failed_executors = cluster_conn.get("executors", {}).get("failed", 0)
        if failed_executors > 0:
            issues_found.append(f"{failed_executors} failed executors")
            recommendations.extend([
                "Check executor logs for failure reasons",
                "Verify resource availability on worker nodes",
                "Check for memory or disk space issues"
            ])
        
        if cluster_conn.get("distributed_test") and "FAILED" in cluster_conn["distributed_test"]:
            issues_found.append("Distributed computation test failed")
            recommendations.extend([
                "Check network connectivity between executors",
                "Verify Spark configuration consistency across nodes",
                "Check for serialization issues"
            ])
        
        # Analyze storage connectivity
        storage_conn = connectivity_results.get("storage_connectivity", {})
        
        for storage_type, status in storage_conn.items():
            if "FAILED" in status or "PERMISSION_DENIED" in status:
                issues_found.append(f"{storage_type} storage connectivity issues")
                
                if "PERMISSION_DENIED" in status:
                    recommendations.extend([
                        f"Check {storage_type} credentials and permissions",
                        f"Verify IAM roles/policies for {storage_type} access",
                        f"Test {storage_type} access outside of Spark"
                    ])
                else:
                    recommendations.extend([
                        f"Verify {storage_type} service availability",
                        f"Check {storage_type} connection parameters",
f"Test network connectivity to {storage_type} endpoints"
                   ])
       
       # Analyze external service connectivity
       external_conn = connectivity_results.get("external_services", {})
       
       for service_name, status in external_conn.items():
           if "FAILED" in status or status in ["CONNECTION_REFUSED", "AUTH_FAILED", "TIMEOUT"]:
               issues_found.append(f"{service_name} connectivity issues")
               
               if status == "CONNECTION_REFUSED":
                   recommendations.extend([
                       f"Verify {service_name} service is running",
                       f"Check {service_name} port accessibility",
                       f"Verify firewall rules for {service_name}"
                   ])
               elif status == "AUTH_FAILED":
                   recommendations.extend([
                       f"Verify {service_name} credentials",
                       f"Check {service_name} user permissions",
                       f"Validate connection string format"
                   ])
               elif status == "TIMEOUT":
                   recommendations.extend([
                       f"Check network latency to {service_name}",
                       f"Increase connection timeout settings",
                       f"Verify {service_name} is not overloaded"
                   ])
       
       # Display results
       if issues_found:
           print("🚨 Issues Found:")
           for i, issue in enumerate(issues_found, 1):
               print(f"   {i}. {issue}")
           
           print("\n💡 Recommendations:")
           for i, rec in enumerate(set(recommendations), 1):  # Remove duplicates
               print(f"   {i}. {rec}")
       else:
           print("✅ No connectivity issues detected")
       
       return {
           "issues": issues_found,
           "recommendations": list(set(recommendations))
       }
   
   def network_troubleshooting_toolkit(self):
       """Comprehensive network troubleshooting toolkit"""
       
       print("\n🛠️ NETWORK TROUBLESHOOTING TOOLKIT")
       print("=" * 50)
       
       toolkit_commands = {
           "Basic Network Tests": [
               "# Test basic connectivity",
               "ping <hostname>",
               "telnet <hostname> <port>",
               "nslookup <hostname>"
           ],
           "Spark-Specific Tests": [
               "# Check Spark UI accessibility",
               "curl http://<driver-host>:4040",
               "",
               "# Test executor connectivity", 
               "spark.sparkContext.parallelize(range(10)).collect()",
               "",
               "# Check cluster manager",
               "spark.sparkContext.master",
               "spark.sparkContext.statusTracker().getExecutorInfos()"
           ],
           "Storage Connectivity": [
               "# Test HDFS connectivity",
               "hdfs dfs -ls /",
               "",
               "# Test S3 connectivity",
               "aws s3 ls s3://bucket-name/",
               "",
               "# Test file system access",
               "spark.read.text('file:///path/to/test')"
           ],
           "Database Connectivity": [
               "# Test JDBC connection",
               """spark.read.format("jdbc") \\
   .option("url", "jdbc:postgresql://host:5432/db") \\
   .option("dbtable", "(SELECT 1) as test") \\
   .option("user", "username") \\
   .option("password", "password") \\
   .load().show()"""
           ],
           "Performance Diagnostics": [
               "# Monitor network usage",
               "iftop -i <interface>",
               "nethogs",
               "",
               "# Check bandwidth",
               "iperf3 -c <server-host>",
               "",
               "# Monitor Spark network metrics",
               "# Check Spark UI -> Executors -> Storage Memory"
           ],
           "Security Diagnostics": [
               "# Check firewall rules",
               "iptables -L",
               "ufw status",
               "",
               "# Test SSL/TLS connectivity", 
               "openssl s_client -connect <host>:<port>",
               "",
               "# Check Kerberos authentication",
               "klist",
               "kinit <principal>"
           ]
       }
       
       for category, commands in toolkit_commands.items():
           print(f"\n{category}:")
           for command in commands:
               if command.strip():
                   print(f"   {command}")
               else:
                   print()
       
       return toolkit_commands
   
   def automated_connectivity_fix(self, connectivity_issues):
       """Attempt automated fixes for common connectivity issues"""
       
       print("\n🔧 AUTOMATED CONNECTIVITY FIXES")
       print("-" * 40)
       
       fix_results = []
       
       # Fix 1: Restart SparkContext if executors are dead
       if any("No active executors" in issue for issue in connectivity_issues.get("issues", [])):
           try:
               print("Attempting SparkContext restart...")
               
               # This is a placeholder - actual implementation would depend on deployment
               # In practice, you might need to restart the application
               fix_results.append("⚠️ SparkContext restart required - manual intervention needed")
               
           except Exception as e:
               fix_results.append(f"❌ SparkContext restart failed: {e}")
       
       # Fix 2: Clear caches and retry connections
       try:
           print("Clearing caches and resetting connections...")
           self.spark.catalog.clearCache()
           
           # Trigger garbage collection
           self.spark.sparkContext._jvm.System.gc()
           
           fix_results.append("✅ Caches cleared and GC triggered")
           
       except Exception as e:
           fix_results.append(f"❌ Cache clearing failed: {e}")
       
       # Fix 3: Apply conservative network settings
       try:
           print("Applying conservative network settings...")
           
           conservative_settings = {
               "spark.network.timeout": "800s",
               "spark.executor.heartbeatInterval": "60s",
               "spark.sql.broadcastTimeout": "1200s"
           }
           
           for key, value in conservative_settings.items():
               self.spark.conf.set(key, value)
           
           fix_results.append("✅ Conservative network settings applied")
           
       except Exception as e:
           fix_results.append(f"❌ Network settings update failed: {e}")
       
       # Display fix results
       print("\nFix Results:")
       for result in fix_results:
           print(f"   {result}")
       
       return fix_results
```

## 📈 Monitoring & Observability

### **Comprehensive Monitoring Framework**

python

```python
class MonitoringFramework:
    """Advanced monitoring and observability for Spark applications"""
    
    def __init__(self, spark_session):
        self.spark = spark_session
        self.monitoring_data = {}
        self.alert_thresholds = {
            "memory_usage": 85,  # percent
            "gc_time": 30000,    # milliseconds
            "task_failure_rate": 5,  # percent
            "processing_lag": 60000  # milliseconds
        }
    
    def comprehensive_health_dashboard(self):
        """Generate comprehensive health dashboard"""
        
        print("📈 SPARK APPLICATION HEALTH DASHBOARD")
        print("=" * 60)
        
        dashboard_data = {
            "application_info": {},
            "resource_utilization": {},
            "performance_metrics": {},
            "error_analysis": {},
            "health_score": 0
        }
        
        # 1. Application Information
        print("1. Application Overview:")
        try:
            app_info = {
                "app_name": self.spark.sparkContext.appName,
                "app_id": self.spark.sparkContext.applicationId,
                "spark_version": self.spark.version,
                "start_time": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                "master": self.spark.sparkContext.master
            }
            
            dashboard_data["application_info"] = app_info
            
            print(f"   Application: {app_info['app_name']}")
            print(f"   ID: {app_info['app_id']}")
            print(f"   Spark Version: {app_info['spark_version']}")
            print(f"   Master: {app_info['master']}")
            
        except Exception as e:
            print(f"   ❌ Application info collection failed: {e}")
        
        # 2. Resource Utilization
        print(f"\n2. Resource Utilization:")
        try:
            status = self.spark.sparkContext.statusTracker()
            executors = status.getExecutorInfos()
            
            if executors:
                total_cores = sum(e.totalCores for e in executors if e.isActive)
                total_memory = sum(e.maxMemory for e in executors if e.isActive)
                used_memory = sum(e.memoryUsed for e in executors if e.isActive)
                
                active_executors = len([e for e in executors if e.isActive])
                failed_executors = len([e for e in executors if not e.isActive])
                
                memory_utilization = (used_memory / total_memory * 100) if total_memory > 0 else 0
                
                resource_metrics = {
                    "active_executors": active_executors,
                    "failed_executors": failed_executors,
                    "total_cores": total_cores,
                    "total_memory_gb": total_memory / (1024**3),
                    "used_memory_gb": used_memory / (1024**3),
                    "memory_utilization_pct": memory_utilization
                }
                
                dashboard_data["resource_utilization"] = resource_metrics
                
                # Resource status indicators
                if failed_executors > 0:
                    executor_status = f"🔴 {active_executors} active, {failed_executors} failed"
                elif active_executors < 2:
                    executor_status = f"🟡 {active_executors} active (low)"
                else:
                    executor_status = f"🟢 {active_executors} active"
                
                if memory_utilization > self.alert_thresholds["memory_usage"]:
                    memory_status = f"🔴 {memory_utilization:.1f}% (HIGH)"
                elif memory_utilization > 70:
                    memory_status = f"🟡 {memory_utilization:.1f}% (MODERATE)"
                else:
                    memory_status = f"🟢 {memory_utilization:.1f}% (OK)"
                
                print(f"   Executors: {executor_status}")
                print(f"   Total Cores: {total_cores}")
                print(f"   Memory Usage: {memory_status}")
                print(f"   Memory: {used_memory/(1024**3):.1f}GB / {total_memory/(1024**3):.1f}GB")
            else:
                print(f"   ❌ No executor information available")
        
        except Exception as e:
            print(f"   ❌ Resource utilization check failed: {e}")
        
        # 3. Performance Metrics
        print(f"\n3. Performance Metrics:")
        try:
            # Job and stage information
            active_jobs = status.getActiveJobIds()
            active_stages = status.getActiveStageIds()
            
            perf_metrics = {
                "active_jobs": len(active_jobs),
                "active_stages": len(active_stages)
            }
            
            print(f"   Active Jobs: {len(active_jobs)}")
            print(f"   Active Stages: {len(active_stages)}")
            
            # Get GC information if available
            try:
                gc_beans = self.spark.sparkContext._jvm.java.lang.management.ManagementFactory.getGarbageCollectorMXBeans()
                
                total_gc_time = 0
                total_gc_count = 0
                
                for gc_bean in gc_beans:
                    total_gc_time += gc_bean.getCollectionTime()
                    total_gc_count += gc_bean.getCollectionCount()
                
                perf_metrics.update({
                    "gc_time_ms": total_gc_time,
                    "gc_count": total_gc_count
                })
                
                if total_gc_time > self.alert_thresholds["gc_time"]:
                    gc_status = f"🔴 {total_gc_time/1000:.1f}s (HIGH)"
                elif total_gc_time > 10000:
                    gc_status = f"🟡 {total_gc_time/1000:.1f}s (MODERATE)"
                else:
                    gc_status = f"🟢 {total_gc_time/1000:.1f}s (OK)"
                
                print(f"   GC Time: {gc_status}")
                print(f"   GC Collections: {total_gc_count}")
                
            except Exception as gc_error:
                print(f"   GC Info: ❌ Not available")
            
            dashboard_data["performance_metrics"] = perf_metrics
            
        except Exception as e:
            print(f"   ❌ Performance metrics collection failed: {e}")
        
        # 4. Calculate Health Score
        health_score = self._calculate_health_score(dashboard_data)
        dashboard_data["health_score"] = health_score
        
        print(f"\n4. Overall Health Score:")
        if health_score >= 90:
            health_status = "🟢 EXCELLENT"
        elif health_score >= 75:
            health_status = "🟡 GOOD"
        elif health_score >= 50:
            health_status = "🟠 POOR"
        else:
            health_status = "🔴 CRITICAL"
        
        print(f"   {health_status} ({health_score:.0f}/100)")
        
        self.monitoring_data = dashboard_data
        return dashboard_data
    
    def _calculate_health_score(self, dashboard_data):
        """Calculate overall application health score"""
        
        score_components = {
            "resource_score": 0,
            "performance_score": 0,
            "stability_score": 0
        }
        
        # Resource utilization score
        resource_util = dashboard_data.get("resource_utilization", {})
        
        if resource_util:
            resource_score = 100
            
            # Deduct for failed executors
            failed_executors = resource_util.get("failed_executors", 0)
            active_executors = resource_util.get("active_executors", 1)
            
            if failed_executors > 0:
                failure_rate = failed_executors / (active_executors + failed_executors)
                resource_score -= failure_rate * 50
            
            # Deduct for high memory usage
            memory_util = resource_util.get("memory_utilization_pct", 0)
            if memory_util > 90:
                resource_score -= 30
            elif memory_util > 80:
                resource_score -= 15
            
            score_components["resource_score"] = max(0, resource_score)
        
        # Performance score
        perf_metrics = dashboard_data.get("performance_metrics", {})
        
        if perf_metrics:
            performance_score = 100
            
            # Deduct for high GC time
            gc_time = perf_metrics.get("gc_time_ms", 0)
            if gc_time > 60000:  # 60 seconds
                performance_score -= 40
            elif gc_time > 30000:  # 30 seconds
                performance_score -= 20
            
            score_components["performance_score"] = max(0, performance_score)
        
        # Stability score (simplified)
        score_components["stability_score"] = 85  # Default good stability
        
        # Calculate weighted average
        weights = {"resource_score": 0.4, "performance_score": 0.4, "stability_score": 0.2}
        
        total_score = sum(score_components[component] * weight 
                         for component, weight in weights.items())
        
        return total_score
    
    def real_time_monitoring(self, duration_minutes=5, interval_seconds=30):
        """Real-time monitoring with periodic updates"""
        
        print(f"📊 REAL-TIME MONITORING ({duration_minutes} minutes)")
        print("=" * 60)
        
        monitoring_history = []
        start_time = time.time()
        end_time = start_time + (duration_minutes * 60)
        
        try:
            while time.time() < end_time:
                current_time = datetime.now().strftime("%H:%M:%S")
                
                # Collect current metrics
                try:
                    status = self.spark.sparkContext.statusTracker()
                    executors = status.getExecutorInfos()
                    
                    if executors:
                        active_executors = len([e for e in executors if e.isActive])
                        total_memory = sum(e.maxMemory for e in executors if e.isActive)
                        used_memory = sum(e.memoryUsed for e in executors if e.isActive)
                        memory_util = (used_memory / total_memory * 100) if total_memory > 0 else 0
                        
                        metrics = {
                            "timestamp": current_time,
                            "active_executors": active_executors,
                            "memory_utilization": memory_util,
                            "active_jobs": len(status.getActiveJobIds()),
                            "active_stages": len(status.getActiveStageIds())
                        }
                        
                        monitoring_history.append(metrics)
                        
                        # Display current status
                        print(f"{current_time} | Executors: {active_executors:2d} | "
                              f"Memory: {memory_util:5.1f}% | Jobs: {metrics['active_jobs']:2d} | "
                              f"Stages: {metrics['active_stages']:2d}")
                        
                        # Check for alerts
                        alerts = self._check_alerts(metrics)
                        for alert in alerts:
                            print(f"         🚨 ALERT: {alert}")
                    
                    else:
                        print(f"{current_time} | ❌ No executor information available")
                
                except Exception as e:
                    print(f"{current_time} | ❌ Monitoring error: {e}")
                
                # Wait for next interval
                time.sleep(interval_seconds)
        
        except KeyboardInterrupt:
            print(f"\n⏹️ Monitoring stopped by user")
        
        # Generate monitoring summary
        if monitoring_history:
            self._generate_monitoring_summary(monitoring_history)
        
        return monitoring_history
    
    def _check_alerts(self, current_metrics):
        """Check current metrics against alert thresholds"""
        
        alerts = []
        
        # Memory utilization alert
        memory_util = current_metrics.get("memory_utilization", 0)
        if memory_util > self.alert_thresholds["memory_usage"]:
            alerts.append(f"High memory usage: {memory_util:.1f}%")
        
        # No active executors alert
        active_executors = current_metrics.get("active_executors", 0)
        if active_executors == 0:
            alerts.append("No active executors")
        
        # Too many active jobs alert
        active_jobs = current_metrics.get("active_jobs", 0)
        if active_jobs > 10:
            alerts.append(f"High job count: {active_jobs}")
        
        return alerts
    
    def _generate_monitoring_summary(self, monitoring_history):
        """Generate summary from monitoring history"""
        
        print(f"\n📋 MONITORING SUMMARY")
        print("-" * 40)
        
        if not monitoring_history:
            print("No monitoring data collected")
            return
        
        # Calculate statistics
        memory_utils = [m["memory_utilization"] for m in monitoring_history]
        executor_counts = [m["active_executors"] for m in monitoring_history]
        job_counts = [m["active_jobs"] for m in monitoring_history]
        
        summary_stats = {
            "duration_minutes": len(monitoring_history) * 0.5,  # Assuming 30s intervals
            "avg_memory_util": sum(memory_utils) / len(memory_utils),
            "max_memory_util": max(memory_utils),
            "min_memory_util": min(memory_utils),
            "avg_executors": sum(executor_counts) / len(executor_counts),
            "max_jobs": max(job_counts),
            "total_samples": len(monitoring_history)
        }
        
        print(f"Duration: {summary_stats['duration_minutes']:.1f} minutes")
        print(f"Total Samples: {summary_stats['total_samples']}")
        print(f"Memory Utilization: {summary_stats['avg_memory_util']:.1f}% avg, "
              f"{summary_stats['max_memory_util']:.1f}% max")
        print(f"Average Executors: {summary_stats['avg_executors']:.1f}")
        print(f"Max Concurrent Jobs: {summary_stats['max_jobs']}")
        
        # Identify patterns
        print(f"\nPatterns Detected:")
        
        if summary_stats['max_memory_util'] > 90:
            print("   🔴 High memory pressure detected")
        
        if len(set(executor_counts)) > 3:
            print("   🟡 Executor count fluctuation detected")
        
        if summary_stats['max_jobs'] > 5:
            print("   🟡 High concurrent job activity")
        
        return summary_stats
    
    def generate_monitoring_report(self):
        """Generate comprehensive monitoring report"""
        
        print(f"\n📊 COMPREHENSIVE MONITORING REPORT")
        print("=" * 60)
        print(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        
        # Collect all monitoring data
        report_data = {
            "executive_summary": {},
            "detailed_metrics": {},
            "recommendations": [],
            "trend_analysis": {}
        }
        
        # Executive Summary
        print(f"\n📋 EXECUTIVE SUMMARY")
        print("-" * 30)
        
        dashboard_data = self.comprehensive_health_dashboard()
        health_score = dashboard_data.get("health_score", 0)
        
        executive_summary = {
            "overall_health": "GOOD" if health_score > 75 else "NEEDS_ATTENTION",
            "health_score": health_score,
            "key_metrics": {
                "active_executors": dashboard_data.get("resource_utilization", {}).get("active_executors", 0),
                "memory_utilization": dashboard_data.get("resource_utilization", {}).get("memory_utilization_pct", 0),
                "active_jobs": dashboard_data.get("performance_metrics", {}).get("active_jobs", 0)
            }
        }
        
        report_data["executive_summary"] = executive_summary
        
        print(f"Overall Health: {executive_summary['overall_health']} ({health_score:.0f}/100)")
        print(f"Active Executors: {executive_summary['key_metrics']['active_executors']}")
        print(f"Memory Utilization: {executive_summary['key_metrics']['memory_utilization']:.1f}%")
        
        # Recommendations
        print(f"\n💡 RECOMMENDATIONS")
        print("-" * 30)
        
        recommendations = []
        
        if health_score < 75:
            recommendations.append("Investigate performance bottlenecks")
        
        if executive_summary['key_metrics']['memory_utilization'] > 80:
            recommendations.append("Consider increasing executor memory allocation")
        
        if executive_summary['key_metrics']['active_executors'] < 2:
            recommendations.append("Increase executor count for better parallelism")
        
        if not recommendations:
            recommendations.append("Continue monitoring - system appears healthy")
        
        for i, rec in enumerate(recommendations, 1):
            print(f"   {i}. {rec}")
        
        report_data["recommendations"] = recommendations
        
        print(f"\n📈 MONITORING SETUP RECOMMENDATIONS")
        print("-" * 30)
        
        monitoring_setup = [
            "Set up automated alerting for memory usage > 85%",
            "Configure log aggregation for error tracking", 
            "Implement custom metrics collection for business KPIs",
            "Set up Spark UI monitoring dashboard",
            "Configure performance baseline tracking"
        ]
        
        for i, setup in enumerate(monitoring_setup, 1):
            print(f"   {i}. {setup}")
        
        return report_data
```

---

## 🎯 Quick Reference & Summary

### **Emergency Response Checklist**

```python
def emergency_response_checklist():
    """Quick emergency response checklist for production incidents"""
    
    print("🚨 EMERGENCY RESPONSE CHECKLIST")
    print("=" * 50)
    
    checklist = {
        "immediate_assessment": [
            "Check if SparkContext is alive: spark.sparkContext._jsc.sc().isStopped()",
            "Verify executor availability: spark.sparkContext.statusTracker().getExecutorInfos()",
            "Check system memory: psutil.virtual_memory().percent",
            "Test basic operations: spark.range(10).count()"
        ],
        "stabilization_actions": [
            "Clear all caches: spark.catalog.clearCache()",
            "Trigger garbage collection: spark.sparkContext._jvm.System.gc()",
            "Apply conservative settings: enable AQE, reduce partitions",
            "Monitor resource utilization trends"
        ],
        "escalation_criteria": [
            "SparkContext is stopped or unresponsive",
            "Zero active executors for > 5 minutes",
            "Memory usage > 95% for > 2 minutes",
            "No successful operations for > 10 minutes"
        ]
    }
    
    for category, items in checklist.items():
        print(f"\n{category.replace('_', ' ').title()}:")
        for item in items:
            print(f"   ☐ {item}")
    
    return checklist

def troubleshooting_decision_tree():
    """Decision tree for systematic troubleshooting approach"""
    
    decision_tree = """
🌳 TROUBLESHOOTING DECISION TREE

START: Is the application running?
├─ NO → Check cluster resources, restart application
└─ YES → Are executors active?
   ├─ NO → Check cluster connectivity, resource allocation
   └─ YES → Is performance acceptable?
      ├─ NO → Run PSPACE diagnosis
      │       ├─ High memory usage → Memory troubleshooting
      │       ├─ Data skew → Partition analysis 
      │       ├─ Slow joins → Join optimization
      │       └─ General slowness → Performance analysis
      └─ YES → Are there errors?
         ├─ YES → Error pattern analysis
         └─ NO → Implement monitoring, done ✅
"""
    
    print(decision_tree)
    return decision_tree

# Quick usage examples
def quick_usage_examples():
    """Quick usage examples for common troubleshooting scenarios"""
    
    examples = {
        "Basic Health Check": """
# Quick health check
diagnostic = PSPACEDiagnostic(spark)
results = diagnostic.run_full_diagnosis(df)
""",
        
        "Memory Issues": """
# Memory troubleshooting
memory_troubleshooter = MemoryTroubleshooter(spark)
memory_analysis = memory_troubleshooter.diagnose_memory_issues()
memory_patterns = memory_troubleshooter.detect_memory_patterns()
""",
        
        "Join Performance": """
# Join performance analysis
join_troubleshooter = JoinTroubleshooter(spark)
join_metrics = join_troubleshooter.analyze_join_performance(df1, df2, "join_key")
recommendations = join_troubleshooter.join_optimization_recommendations(join_metrics)
""",
        
        "Configuration Audit": """
# Configuration troubleshooting
config_troubleshooter = ConfigurationTroubleshooter(spark)
audit_results = config_troubleshooter.comprehensive_config_audit()
""",
        
        "Emergency Response": """
# Emergency incident response
response = p0_incident_response(spark)
if response["action"] == "EMERGENCY_MEMORY_CLEANUP":
    cleanup_actions = emergency_memory_cleanup()
"""
    }
    
    print("🔧 QUICK USAGE EXAMPLES")
    print("=" * 50)
    
    for scenario, code in examples.items():
        print(f"\n{scenario}:")
        print(code)
    
    return examples

# Print quick reference
if __name__ == "__main__":
    emergency_response_checklist()
    print("\n")
    troubleshooting_decision_tree()
    print("\n")
    quick_usage_examples()
```