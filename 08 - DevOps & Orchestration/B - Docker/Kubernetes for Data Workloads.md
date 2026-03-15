# Kubernetes for Data Workloads

Kubernetes (K8s) orchestrates containerized workloads across clusters — scheduling pods, managing networking, handling storage, and providing declarative configuration. This note covers K8s concepts through the lens of data engineering: running Spark, Airflow, and Jupyter on K8s, managing secrets for data platform credentials, and right-sizing resources for ETL/ML workloads.

See also: [[Docker & Container Patterns]], [[Terraform for Data Infrastructure]]

---

## Core Concepts

| Resource | Purpose |
|----------|---------|
| **Pod** | Smallest deployable unit — one or more containers sharing network and storage. Ephemeral by default. |
| **Deployment** | Declares desired pod state (image, replicas, update strategy). The controller ensures reality matches the spec. |
| **Service** | Stable network endpoint that routes traffic to a set of pods via label selectors. |
| **Namespace** | Logical cluster partition for isolating environments, teams, or projects. |
| **ConfigMap** | Externalised non-sensitive configuration (key-value pairs or files). |
| **Secret** | Base64-encoded sensitive data (tokens, passwords, TLS certs). Encrypted at rest in etcd when configured. |
| **PersistentVolume (PV)** | Cluster-level storage resource (NFS, EBS, GCE PD, Azure Disk). |
| **PersistentVolumeClaim (PVC)** | A pod's request for storage — binds to a PV. |
| **Ingress** | HTTP/HTTPS routing rules that map external URLs to internal Services. |
| **StatefulSet** | Like a Deployment but with stable pod names and persistent storage — used for databases and stateful services. |
| **Job / CronJob** | One-off or scheduled batch work — ideal for ETL tasks. |

---

## Namespace Isolation

Namespaces provide environment separation. From the Delta Lake project:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: delta-lake
  labels:
    name: delta-lake
    environment: production
    project: databricks-delta-lake
```

Labels like `environment: production` and `project: databricks-delta-lake` enable filtering with `kubectl get pods -l environment=production` and integration with policy engines like OPA/Gatekeeper.

### Environment Separation Strategy

```
Namespace: data-platform-dev       # developers iterate freely
Namespace: data-platform-staging   # mirrors prod, runs integration tests
Namespace: data-platform-prod      # locked down, CI/CD deploys only
```

### Resource Quotas

Prevent a single namespace from consuming the entire cluster:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: data-platform-quota
  namespace: data-platform-prod
spec:
  hard:
    requests.cpu: "32"
    requests.memory: 64Gi
    limits.cpu: "64"
    limits.memory: 128Gi
    pods: "100"
    persistentvolumeclaims: "20"
```

### Network Policies

Restrict cross-namespace traffic — production should not accept connections from dev:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: data-platform-prod
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}   # only pods within this namespace
```

---

## ConfigMaps & Secrets

### ConfigMaps — Externalising Configuration

Non-sensitive settings live in a ConfigMap, decoupled from the container image:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: delta-lake-config
  namespace: delta-lake
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  DATABRICKS_CATALOG: "main"
  DATABRICKS_SCHEMA: "default"
```

### Secrets — Credential Management

Secrets store sensitive values as base64-encoded data:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: delta-lake-secrets
  namespace: delta-lake
type: Opaque
data:
  DATABRICKS_HOST: ""    # echo -n 'value' | base64
  DATABRICKS_TOKEN: ""
  DATABASE_PASSWORD: ""
```

**Secret types:**
- `Opaque` — generic key-value pairs (most common)
- `kubernetes.io/dockerconfigjson` — registry credentials for pulling private images
- `kubernetes.io/tls` — TLS certificate and private key pairs
- `kubernetes.io/service-account-token` — auto-generated SA tokens

### Mounting as Env Vars vs Files

The Delta Lake deployment injects ConfigMap and Secret values as environment variables:

```yaml
env:
- name: ENVIRONMENT
  valueFrom:
    configMapKeyRef:
      name: delta-lake-config
      key: ENVIRONMENT
- name: DATABRICKS_TOKEN
  valueFrom:
    secretKeyRef:
      name: delta-lake-secrets
      key: DATABRICKS_TOKEN
```

Alternatively, mount the entire ConfigMap/Secret as a volume (useful for config files like `spark-defaults.conf`):

```yaml
volumes:
- name: spark-config
  configMap:
    name: spark-cluster-config
containers:
- name: spark-driver
  volumeMounts:
  - name: spark-config
    mountPath: /opt/spark/conf
```

### Secret Rotation Patterns

- **External Secrets Operator** — syncs secrets from AWS Secrets Manager, HashiCorp Vault, or Azure Key Vault into K8s Secrets automatically
- **Sealed Secrets** — encrypt secrets in Git; only the cluster can decrypt them
- **CSI Secrets Store Driver** — mounts secrets directly from a vault as a filesystem volume, bypassing etcd entirely

---

## Deployment Patterns

The Delta Lake API deployment demonstrates production-grade patterns:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delta-lake-api
  namespace: delta-lake
  labels:
    app: delta-lake-api
    component: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: delta-lake-api
  template:
    spec:
      containers:
      - name: api
        image: delta-lake-api:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Key patterns:**

- **Requests vs limits** — requests guarantee minimum resources for scheduling; limits cap maximum usage. Setting `requests.memory: 256Mi` and `limits.memory: 512Mi` gives the pod a guaranteed baseline with burst room.
- **Liveness probe** — K8s restarts the container if `/health` fails after 30s initial delay. Catches deadlocks and hung processes.
- **Readiness probe** — K8s removes the pod from Service endpoints if `/ready` fails. Prevents traffic to pods still loading models or warming caches.
- **Rolling updates** — default strategy replaces pods incrementally. Configure `maxSurge` and `maxUnavailable` to control rollout speed.

### Pod Disruption Budgets

Ensure minimum availability during voluntary disruptions (node drains, cluster upgrades):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: delta-lake-api-pdb
  namespace: delta-lake
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: delta-lake-api
```

---

## Services & Networking

The Delta Lake project uses a ClusterIP service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: delta-lake-api-service
  namespace: delta-lake
spec:
  selector:
    app: delta-lake-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: ClusterIP
```

### Service Types

| Type | Scope | Use Case |
|------|-------|----------|
| **ClusterIP** | Internal only | Default. Microservice-to-microservice communication. |
| **NodePort** | External via node IP:port | Development/testing. Exposes on port 30000-32767. |
| **LoadBalancer** | External via cloud LB | Production public-facing APIs. Provisions an AWS ALB/NLB, GCP LB, etc. |
| **Headless** (`clusterIP: None`) | No virtual IP; DNS returns pod IPs | StatefulSets (Kafka, ZooKeeper) where clients need direct pod access. |

### DNS Resolution

Every Service gets a DNS entry: `<service-name>.<namespace>.svc.cluster.local`. The Delta Lake API is reachable within the cluster at `delta-lake-api-service.delta-lake.svc.cluster.local`.

Pods within the same namespace can use the short name: `delta-lake-api-service`.

---

## Data Workload Patterns

### Spark on Kubernetes

Submit Spark jobs directly to the K8s cluster as the resource manager:

```bash
spark-submit \
  --master k8s://https://<k8s-api-server>:6443 \
  --deploy-mode cluster \
  --name delta-etl-job \
  --conf spark.kubernetes.namespace=data-platform-prod \
  --conf spark.kubernetes.container.image=spark-etl:3.5.1 \
  --conf spark.kubernetes.driver.request.cores=2 \
  --conf spark.kubernetes.executor.request.cores=4 \
  --conf spark.kubernetes.executor.instances=10 \
  --conf spark.kubernetes.driver.secrets.delta-lake-secrets=/mnt/secrets \
  --conf spark.kubernetes.node.selector.workload-type=spark \
  local:///opt/spark/app/etl_pipeline.py
```

K8s creates a driver pod that spawns executor pods. Executors are cleaned up automatically when the job completes. Node selectors route Spark pods to high-memory nodes.

See also: [[Apache Spark Architecture]]

### Airflow KubernetesExecutor

Each Airflow task runs as an isolated pod — no shared worker state, clean dependency isolation:

```yaml
executor: KubernetesExecutor
kubernetes_executor_config:
  worker_container_repository: airflow-tasks
  worker_container_tag: "2.8.1"
  namespace: airflow-prod
  delete_worker_pods: true
  delete_worker_pods_on_failure: false   # keep failed pods for debugging
```

Tasks can override the pod spec per-task using `executor_config` in the DAG, specifying custom images, resource requests, or node selectors.

See also: [[Apache Airflow Architecture]]

### JupyterHub on K8s

JupyterHub with KubeSpawner provisions a dedicated pod per user session:

```yaml
singleuser:
  image:
    name: jupyter/datascience-notebook
    tag: latest
  memory:
    limit: 8G
    guarantee: 4G
  cpu:
    limit: 4
    guarantee: 2
  storage:
    capacity: 20Gi
    dynamic:
      storageClass: gp3
```

### Batch Jobs and CronJobs

One-off ETL runs and scheduled pipelines:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-data-sync
  namespace: data-platform-prod
spec:
  schedule: "0 2 * * *"          # 2 AM daily
  concurrencyPolicy: Forbid       # skip if previous run still active
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 7200  # timeout after 2 hours
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: sync
            image: data-sync:1.4.2
            resources:
              requests:
                memory: "2Gi"
                cpu: "1"
```

---

## Storage

### PersistentVolumes and Claims

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spark-checkpoint-pvc
  namespace: data-platform-prod
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 100Gi
```

### StorageClasses and Dynamic Provisioning

StorageClasses define how volumes are provisioned. Cloud providers offer pre-configured classes:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "5000"
  throughput: "250"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

`WaitForFirstConsumer` delays volume creation until a pod is scheduled, ensuring the volume is in the same availability zone as the pod.

### CSI Drivers for Cloud Storage

- **AWS EBS CSI** — block storage for single-pod workloads (checkpoints, local caches)
- **AWS EFS CSI** — shared NFS for multi-pod access (shared datasets, model artifacts)
- **GCS FUSE CSI** — mount GCS buckets as local filesystems in pods
- **Azure Blob CSI** — mount Azure Blob containers

---

## Helm Charts

Helm packages K8s manifests into reusable, versioned charts with templated values.

### Chart Structure

```
my-chart/
  Chart.yaml          # name, version, dependencies
  values.yaml         # default configuration
  templates/
    deployment.yaml   # {{ .Values.image.repository }}
    service.yaml
    configmap.yaml
    _helpers.tpl      # reusable template snippets
```

### Common Operations

```bash
# Add a repo and install a chart
helm repo add apache-airflow https://airflow.apache.org
helm install airflow apache-airflow/airflow \
  --namespace airflow --create-namespace \
  -f custom-values.yaml

# Upgrade with new values
helm upgrade airflow apache-airflow/airflow -f custom-values.yaml

# Rollback
helm rollback airflow 1
```

### Key DE Helm Charts

| Chart | Repo | Notes |
|-------|------|-------|
| **Apache Airflow** | `apache-airflow/airflow` | KubernetesExecutor, Git-sync for DAGs, PgBouncer |
| **Spark Operator** | `spark-operator/spark-operator` | CRD-based Spark job management |
| **JupyterHub** | `jupyterhub/jupyterhub` | KubeSpawner, per-user pods and storage |
| **Prometheus Stack** | `prometheus-community/kube-prometheus-stack` | Prometheus, Grafana, AlertManager bundled |
| **Strimzi (Kafka)** | `strimzi/strimzi-kafka-operator` | Kafka clusters as K8s custom resources |

### Chart Dependencies

Declare sub-charts in `Chart.yaml`:

```yaml
dependencies:
- name: postgresql
  version: "12.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
```

---

## Monitoring

### Prometheus + Grafana Stack

The standard K8s monitoring stack:

```
Pods (metrics endpoint) --> Prometheus (scrape & store) --> Grafana (visualize)
                        --> AlertManager (alert on thresholds)
```

Install via Helm:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f monitoring-values.yaml
```

### Container Resource Monitoring

`metrics-server` provides real-time CPU/memory usage for `kubectl top` and the HorizontalPodAutoscaler:

```bash
kubectl top pods -n data-platform-prod
kubectl top nodes
```

### Log Aggregation

```
Pods (stdout/stderr) --> Fluent Bit (DaemonSet, lightweight) --> Elasticsearch --> Kibana
                     or  Fluentd (more plugins, heavier)     --> S3 / CloudWatch
```

Fluent Bit runs as a DaemonSet (one pod per node) and tails container logs from `/var/log/containers/`. For Spark jobs, configure log4j to write structured JSON so Fluent Bit can parse fields automatically.

---

## Best Practices for Data Engineering

### Resource Right-Sizing

- Start with generous requests, then tune using Prometheus metrics (`container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`)
- **Spark executors:** set requests equal to limits to get Guaranteed QoS (prevents OOM kills during shuffles)
- **API services:** set requests lower than limits to allow bursting (the Delta Lake API uses 250m/500m CPU)

### Node Affinity for Specialized Workloads

Route GPU/ML workloads to GPU nodes, memory-heavy Spark jobs to high-memory nodes:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["r5.4xlarge", "r5.8xlarge"]
```

### Pod Anti-Affinity for High Availability

Spread replicas across nodes/zones to survive failures:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: delta-lake-api
        topologyKey: kubernetes.io/hostname
```

### Autoscaling — HPA and VPA

The Delta Lake project includes a HorizontalPodAutoscaler:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: delta-lake-api-hpa
  namespace: delta-lake
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: delta-lake-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

- **HPA** — scales pod count based on CPU, memory, or custom metrics. Set `minReplicas: 2` for HA.
- **VPA** — adjusts resource requests/limits per pod. Do not run HPA and VPA on the same metric.
- **KEDA** — event-driven autoscaling based on queue depth (Kafka lag, SQS messages), ideal for data consumers.

### Cost Management

- Use **Spot/Preemptible nodes** for Spark executors and batch jobs (tolerations + node affinity)
- Set **LimitRanges** to enforce default resource requests per namespace
- Run **kubecost** or cloud-native cost tools to attribute spend per namespace/team
- **Cluster Autoscaler** scales nodes down during off-hours; pair with CronJobs that run during business hours
- Tag namespaces with cost-center labels for chargeback reporting

---

## Spark on Kubernetes

Running Apache Spark natively on Kubernetes replaces YARN as the cluster manager, using K8s pods as Spark drivers and executors. This unifies compute infrastructure — the same cluster runs Spark, Airflow, APIs, and ML workloads.

See also: [[Apache Spark Architecture]], [[PySpark Core Concepts]]

### spark-submit with Kubernetes

Submit Spark applications directly using `--master k8s://`:

```bash
spark-submit \
  --master k8s://https://k8s-api-server:6443 \
  --deploy-mode cluster \
  --name etl-daily-transform \
  --conf spark.kubernetes.namespace=spark-jobs \
  --conf spark.kubernetes.container.image=my-registry/spark:3.5.1 \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark-sa \
  --conf spark.driver.cores=2 \
  --conf spark.driver.memory=4g \
  --conf spark.executor.cores=4 \
  --conf spark.executor.memory=8g \
  --conf spark.executor.instances=10 \
  --conf spark.kubernetes.file.upload.path=s3a://spark-staging/uploads \
  local:///opt/spark/app/daily_transform.py
```

The `local://` path references a file inside the container image. For external files, use `s3a://` or `hdfs://` paths.

### Spark Operator (SparkApplication CRD)

The Spark Operator (maintained by Kubeflow) provides a Kubernetes-native way to manage Spark applications using Custom Resource Definitions:

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: daily-etl
  namespace: spark-jobs
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: my-registry/spark:3.5.1
  mainApplicationFile: local:///opt/spark/app/daily_transform.py
  sparkVersion: "3.5.1"

  driver:
    cores: 2
    memory: "4g"
    serviceAccount: spark-sa
    labels:
      app: daily-etl
      component: driver
    nodeSelector:
      workload-type: spark-driver

  executor:
    cores: 4
    memory: "8g"
    instances: 10
    labels:
      app: daily-etl
      component: executor
    nodeSelector:
      workload-type: spark-executor

  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
    onFailureRetryInterval: 30
    onSubmissionFailureRetries: 2

  monitoring:
    exposeDriverMetrics: true
    exposeExecutorMetrics: true
    prometheus:
      jmxExporterJar: /prometheus/jmx_prometheus_javaagent.jar
      port: 8090
```

Install the operator via Helm: `helm install spark-operator spark-operator/spark-operator --namespace spark-operator --create-namespace`

The operator also supports `ScheduledSparkApplication` for cron-based scheduling, eliminating the need for an external scheduler for simple recurring jobs.

### Driver and Executor Pod Lifecycle

```
spark-submit / SparkApplication
        │
        ▼
   Driver Pod (created by K8s)
        │
        ├── Requests executor pods from K8s API
        ▼
   Executor Pods (created dynamically)
        │
        ├── Execute tasks, report results to driver
        ▼
   Job completes → Executor pods terminated
                 → Driver pod enters Completed state
```

- The **driver pod** runs for the lifetime of the application, coordinating task scheduling and result aggregation
- **Executor pods** are created on demand and destroyed when the application completes
- Failed executor pods are automatically replaced (configurable via `spark.kubernetes.executor.deleteOnTermination`)
- Use `restartPolicy: OnFailure` in the SparkApplication CRD for automatic retries

### Dynamic Allocation on Kubernetes

Dynamic allocation scales executors up and down based on workload:

```properties
spark.dynamicAllocation.enabled=true
spark.dynamicAllocation.shuffleTracking.enabled=true
spark.dynamicAllocation.minExecutors=2
spark.dynamicAllocation.maxExecutors=50
spark.dynamicAllocation.executorIdleTimeout=60s
spark.dynamicAllocation.schedulerBacklogTimeout=10s
```

The `shuffleTracking.enabled` setting is essential on Kubernetes — it replaces the external shuffle service (which requires a DaemonSet) by tracking shuffle data within executors, preventing premature removal of executors holding shuffle data.

### Node Selectors and Tolerations

Route Spark pods to appropriate node pools:

```yaml
# Node selector — schedule on specific node types
nodeSelector:
  workload-type: spark-executor
  instance-family: r5

# Tolerations — allow scheduling on tainted (dedicated) nodes
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "spark"
    effect: "NoSchedule"
```

Common patterns:
- **Driver pods** on smaller, on-demand instances (reliable, low cost)
- **Executor pods** on larger, spot/preemptible instances (cost-effective, tolerant of interruption)
- Use **taints** on spot nodes (`dedicated=spark:NoSchedule`) so only Spark executors with matching tolerations are scheduled there

### Volume Mounts for Data

```yaml
volumes:
  - name: spark-data
    persistentVolumeClaim:
      claimName: spark-scratch-pvc
  - name: spark-config
    configMap:
      name: spark-defaults

volumeMounts:
  - name: spark-data
    mountPath: /data/scratch
  - name: spark-config
    mountPath: /opt/spark/conf
```

For large datasets, prefer object storage (S3, GCS) over PVCs. Use PVCs for checkpoint directories and local scratch space during shuffles.

### Monitoring with Prometheus

Expose Spark metrics via the JMX Prometheus exporter:

1. Include `jmx_prometheus_javaagent.jar` in the Spark container image
2. Configure the exporter in the SparkApplication (see CRD example above)
3. Add Prometheus `ServiceMonitor` to scrape Spark pods:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: spark-metrics
  namespace: spark-jobs
spec:
  selector:
    matchLabels:
      app: daily-etl
  endpoints:
    - port: metrics
      interval: 15s
```

Key metrics to monitor: `spark_driver_DAGScheduler_activeJobs`, `spark_executor_cpuTime`, `spark_executor_memoryUsed`, `spark_executor_shuffleBytesWritten`.

### Spark on Kubernetes vs YARN

| Aspect | Spark on Kubernetes | Spark on YARN |
|--------|---------------------|---------------|
| **Infrastructure** | Shared K8s cluster | Dedicated Hadoop cluster |
| **Resource isolation** | Container-level (cgroups, namespaces) | YARN queues and resource pools |
| **Scaling** | K8s cluster autoscaler + dynamic allocation | Fixed cluster or manual scaling |
| **Container images** | Custom Docker images per application | Shared Hadoop classpath |
| **Multi-tenancy** | Namespaces, RBAC, network policies | YARN queues, Kerberos |
| **Spot/preemptible** | Native support via node pools | Limited (depends on cloud provider) |
| **Dependency management** | Each job has its own container image | Shared classpath, dependency conflicts |
| **Operational overhead** | Managed K8s (EKS, GKE) reduces ops | Requires Hadoop admin expertise |
| **Ecosystem integration** | Runs alongside non-Spark workloads | Spark/Hadoop ecosystem only |
| **Maturity** | GA since Spark 3.1 (2021) | Mature, battle-tested |

**When to choose K8s:** existing Kubernetes infrastructure, desire for unified compute platform, need for custom container images, or cloud-native deployments. **When to choose YARN:** existing Hadoop investment, heavy HDFS usage, or on-premises deployments with established Hadoop operations.
