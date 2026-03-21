# Pipeline Observability & Monitoring

Observability is the ability to understand a system's internal state from its external outputs. For data pipelines, this means answering: did the data arrive, is it correct, and how long did it take?

See also: [[Snowflake Cost Monitoring]], [[dbt Testing & Data Quality]]

---

## Three Pillars of Observability

| Pillar | What It Captures | Pipeline Example |
|--------|-----------------|------------------|
| **Logs** | Discrete events with context | `extraction_complete: 14,320 rows from source_orders` |
| **Metrics** | Numeric measurements over time | `job_duration_seconds{job="orders_etl"} 342.7` |
| **Traces** | Request flow across services/stages | Extract -> Transform -> Load spans with timing |

**Logs** tell you *what happened*. **Metrics** tell you *how the system is performing*. **Traces** tell you *where time was spent* across distributed stages. A mature pipeline needs all three.

---

## Structured Logging

### Standard Logging vs structlog

Standard library logging produces flat strings that are difficult to parse and query at scale. Structured logging emits machine-readable JSON with consistent fields, making log aggregation and alerting far more effective.

```python
# Standard logging -- flat string, hard to filter
logging.info("Job orders_etl completed in 342.7s, processed 14320 rows")

# Structured logging -- JSON, every field is queryable
import structlog
logger = structlog.get_logger()
logger.info("job_complete",
    job_name="orders_etl",
    duration_seconds=342.7,
    records_processed=14320,
    run_id="run-20260315-001"
)
# Output: {"event": "job_complete", "job_name": "orders_etl", "duration_seconds": 342.7, ...}
```

### Context Binding

Bind context once at job start so every subsequent log line carries it automatically:

```python
logger = structlog.get_logger()
log = logger.bind(job_name="orders_etl", run_id="run-20260315-001", environment="prod")

log.info("stage_start", stage="extract")
# All fields (job_name, run_id, environment) are included automatically
log.info("stage_complete", stage="extract", records=14320)
```

Key context fields for pipeline logs: `job_name`, `run_id`, `stage`, `environment`, `table_name`, `timestamp`.

### Log Levels for Data Pipelines

| Level | When to Use | Pipeline Example |
|-------|-------------|------------------|
| `DEBUG` | Detailed diagnostic info, disabled in prod | Row-level transformations, SQL queries issued |
| `INFO` | Normal operational events | Stage start/complete, record counts, job duration |
| `WARNING` | Unexpected but non-fatal conditions | Row count dropped 40% from yesterday, slow query |
| `ERROR` | Operation failed but job can continue or retry | Single table load failed, API timeout on retry |
| `CRITICAL` | Entire pipeline must stop | Database unreachable, credentials expired, data corruption |

### File and Stream Handlers

From `framework/utils.py` in the project -- the `Logger` class configures both console output and optional file logging:

```python
class Logger:
    @staticmethod
    def setup_logging(level: int = logging.INFO, log_file: Optional[str] = None):
        if log_file:
            log_path = Path(log_file)
            log_path.parent.mkdir(parents=True, exist_ok=True)

        formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s',
            datefmt='%Y-%m-%d %H:%M:%S'
        )

        handlers = [logging.StreamHandler()]
        if log_file:
            handlers.append(logging.FileHandler(log_file))

        logging.basicConfig(level=level,
            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
            handlers=handlers)
```

Best practice: always log to both `stderr` (for real-time visibility in CI/CD) and a rotating file (for post-mortem analysis). In containerised environments, log to `stdout`/`stderr` and let the orchestrator handle aggregation.

---

## Metrics Collection

### Custom MetricsCollector Pattern

The project's `MetricsCollector` in `framework/utils.py` emits structured JSON metrics at each ETL stage:

```python
class MetricsCollector:
    def __init__(self):
        self.logger = logging.getLogger(f"{__name__}.metrics")

    def record_job_start(self, job_name: str):
        self._log_metric({
            'event': 'job_start',
            'job_name': job_name,
            'timestamp': datetime.utcnow().isoformat()
        })

    def record_extraction_complete(self, records_count: int):
        self._log_metric({
            'event': 'extraction_complete',
            'records_extracted': records_count,
            'timestamp': datetime.utcnow().isoformat()
        })

    def record_transformation_complete(self, input_records: int, output_records: int):
        self._log_metric({
            'event': 'transformation_complete',
            'input_records': input_records,
            'output_records': output_records,
            'records_filtered': input_records - output_records,
            'timestamp': datetime.utcnow().isoformat()
        })

    def record_job_complete(self, job_name: str, duration: float, records_processed: int):
        self._log_metric({
            'event': 'job_complete',
            'job_name': job_name,
            'duration_seconds': duration,
            'records_processed': records_processed,
            'timestamp': datetime.utcnow().isoformat()
        })

    def _log_metric(self, metric_data: Dict[str, Any]):
        self.logger.info(json.dumps(metric_data))
```

This pattern -- logging JSON metric events at stage boundaries -- works well for batch jobs. Each event is independently parseable and can be ingested by any log aggregation tool.

### Prometheus Client for Python Pipelines

Prometheus provides four metric types suited to different pipeline measurements:

| Type | Purpose | Pipeline Use Case |
|------|---------|-------------------|
| `Counter` | Monotonically increasing count | Total records processed, total errors |
| `Gauge` | Value that goes up and down | Current queue depth, active connections |
| `Histogram` | Distribution of values in buckets | Job duration distribution, batch size distribution |
| `Summary` | Similar to histogram with quantiles | p50/p95/p99 query latency |

```python
from prometheus_client import Counter, Histogram, Gauge, push_to_gateway, CollectorRegistry

registry = CollectorRegistry()

records_processed = Counter('etl_records_processed_total',
    'Total records processed', ['job_name', 'stage'], registry=registry)
job_duration = Histogram('etl_job_duration_seconds',
    'Job duration in seconds', ['job_name'],
    buckets=[30, 60, 120, 300, 600, 1800, 3600], registry=registry)
last_success = Gauge('etl_last_success_timestamp',
    'Timestamp of last successful run', ['job_name'], registry=registry)

# During pipeline execution
records_processed.labels(job_name='orders_etl', stage='extract').inc(14320)
with job_duration.labels(job_name='orders_etl').time():
    run_pipeline()
last_success.labels(job_name='orders_etl').set_to_current_time()
```

### Push Gateway for Batch Jobs

Batch jobs are short-lived -- they cannot serve a `/metrics` endpoint for Prometheus to scrape. Use the Prometheus Push Gateway instead:

```python
from prometheus_client import push_to_gateway

# At job completion, push all collected metrics
push_to_gateway('pushgateway:9091', job='orders_etl', registry=registry)
```

---

## Distributed Tracing

### OpenTelemetry for Pipeline Spans

OpenCensus has been superseded by OpenTelemetry. Use it to create spans for each pipeline stage:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(CloudTraceSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("etl.pipeline")

with tracer.start_as_current_span("orders_etl") as root_span:
    root_span.set_attribute("job.run_id", "run-20260315-001")

    with tracer.start_as_current_span("extract"):
        data = extract_from_source()

    with tracer.start_as_current_span("transform") as span:
        result = transform_data(data)
        span.set_attribute("records.input", len(data))
        span.set_attribute("records.output", len(result))

    with tracer.start_as_current_span("load"):
        load_to_bigquery(result)
```

### Trace Context Propagation

When pipelines span multiple services (e.g., an API trigger kicks off a Dataflow job that writes to BigQuery), propagate trace context via headers or metadata:

```python
from opentelemetry.propagate import inject, extract

# Producer: inject context into message headers
headers = {}
inject(headers)
publish_message(data, headers=headers)

# Consumer: extract context from incoming message
ctx = extract(carrier=incoming_headers)
with tracer.start_as_current_span("downstream_process", context=ctx):
    process(data)
```

### GCP Cloud Trace Integration

Cloud Trace is native to GCP. The `CloudTraceSpanExporter` sends spans directly to Stackdriver/Cloud Trace, providing end-to-end visibility for pipelines running on Dataflow, Cloud Functions, or GKE.

---

## Pipeline Health Indicators

### Data Freshness

Track when each table was last updated and alert when it exceeds the expected SLA. See [[Snowflake Cost Monitoring]] for a SQL-based freshness query pattern.

```python
freshness_age = Gauge('data_freshness_seconds',
    'Seconds since table was last updated', ['dataset', 'table'])

last_modified = get_table_last_modified('analytics', 'orders')
freshness_age.labels(dataset='analytics', table='orders').set(
    (datetime.utcnow() - last_modified).total_seconds()
)
```

### Volume Anomaly Detection

Compare today's record count against a rolling average. Flag deviations beyond two standard deviations:

```python
import numpy as np

historical_counts = get_last_n_days_counts(table='orders', days=30)
mean, std = np.mean(historical_counts), np.std(historical_counts)
today_count = get_today_count('orders')

if abs(today_count - mean) > 2 * std:
    logger.warning("volume_anomaly",
        table="orders", today=today_count,
        expected_mean=round(mean), std_devs=round((today_count - mean) / std, 2))
```

### SLA Tracking and Stage Duration Trends

Record stage durations over time and set SLA thresholds per job. If `orders_etl` normally takes 5 minutes and suddenly takes 20, that is an early warning even if the job succeeds.

### Error Rate Monitoring

Track error rates as a ratio of failed runs to total runs over a rolling window. An error rate above 5% in a 24-hour window should trigger investigation.

---

## Log4j Configuration for Spark

Spark is notoriously verbose. The project's `config/log4j.properties` controls this:

```properties
# Root logger - show only INFO and above
log4j.rootCategory=INFO, console

# Console appender
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.target=System.err
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1} - %m%n

# Silence noisy components
log4j.logger.org.apache.spark=INFO
log4j.logger.org.spark-project=ERROR
log4j.logger.org.apache.hadoop=ERROR
log4j.logger.org.apache.kafka=ERROR
log4j.logger.org.apache.zookeeper=ERROR
log4j.logger.org.eclipse.jetty=OFF
log4j.logger.io.netty=ERROR
```

Key tuning strategies:
- Set `org.apache.hadoop` and `org.apache.kafka` to `ERROR` -- their INFO output is extremely noisy and rarely actionable
- Set `org.eclipse.jetty` to `OFF` -- Spark UI server logs are irrelevant for pipeline debugging
- Keep `org.apache.spark` at `INFO` for stage/task completion visibility, or bump to `WARN` in production if logs are too large
- Add per-package overrides for your own code: `log4j.logger.com.mycompany.etl=DEBUG`

---

## Alerting Patterns

### Threshold-Based Alerts

Define static thresholds for known failure modes:

| Metric | Warning Threshold | Critical Threshold |
|--------|-------------------|---------------------|
| Job duration | > 2x historical average | > 5x or timeout |
| Record count | < 50% of expected | 0 records |
| Error rate (24h) | > 5% | > 15% |
| Data freshness | > 2x expected interval | > 4x expected interval |

### Anomaly Detection Alerts

Complement static thresholds with statistical anomaly detection. Moving average + standard deviation works well for pipelines with regular patterns. More sophisticated approaches use Prophet or simple z-score alerting.

### Escalation Tiers

| Tier | Channel | When | Example |
|------|---------|------|---------|
| P3 - Low | Slack `#data-alerts` | Informational, self-healing | Warning threshold, auto-retry succeeded |
| P2 - Medium | Slack + Email | Requires attention within hours | Job failed after retries, SLA at risk |
| P1 - High | PagerDuty + Slack | Immediate response needed | Production pipeline down, data corruption |

### Alert Fatigue Prevention

- **Group related alerts**: one notification for "3 tables in the orders pipeline failed" not three separate alerts
- **Add silence windows** for maintenance periods and known deployment windows
- **Require runbooks**: every alert should link to a runbook with investigation steps
- **Review alert frequency monthly**: if an alert fires > 5 times without action, tune or remove it

---

## Monitoring Dashboards

### Key Metrics to Display

A pipeline monitoring dashboard should surface these at a glance:

| Panel | Metric | Visualisation |
|-------|--------|---------------|
| Job Status | Last N runs: pass/fail | Status history (green/red dots) |
| Job Duration | Duration over time per job | Time series line chart |
| Records Processed | Count per run per stage | Stacked bar chart |
| Error Rate | Failures / total runs (24h rolling) | Single stat with threshold colouring |
| Data Freshness | Time since last update per table | Table with conditional formatting |
| Resource Usage | CPU, memory, disk per job | Gauges or time series |

### Grafana Basics

Grafana connects to Prometheus, Cloud Monitoring, BigQuery, and PostgreSQL as data sources. For pipeline monitoring:

1. Create a dashboard per pipeline domain (orders, inventory, customers)
2. Use template variables for `job_name` and `environment` so one dashboard serves all
3. Set up annotations for deployments and incidents to correlate changes with metric shifts
4. Configure alert rules directly in Grafana for simpler setups

### Cloud Monitoring Custom Dashboards

For GCP-native pipelines, Cloud Monitoring provides custom dashboards with metrics from Dataflow, BigQuery, and Cloud Functions. Use custom metrics (via OpenTelemetry or the Cloud Monitoring API) to push pipeline-specific measurements alongside infrastructure metrics.

---

## CI/CD Pipeline Monitoring

### Jenkins Build Health

The project's `Jenkinsfile` archives logs and test results on every run:

```groovy
post {
    always {
        archiveArtifacts artifacts: 'logs/**/*', allowEmptyArchive: true
        publishTestResults testResultsPattern: 'tests/reports/*.xml'
    }
    success {
        echo "GPMS Deployment Pipeline completed successfully!"
    }
    failure {
        echo "GPMS Deployment Pipeline failed. Check logs for details."
    }
}
```

### GCP Quota Monitoring in CI

The project's `gcp_utils.groovy` includes proactive quota checking before jobs run:

```groovy
def checkQuotas(String service = 'compute') {
    sh """
        gcloud ${service} project-info describe \
            --format='table(quotas[].metric,quotas[].usage,quotas[].limit)' | \
        awk 'NR>1 {
            usage=\$2; limit=\$3;
            if (usage && limit && usage > limit * 0.8) {
                print "Warning: " \$1 " quota usage is high: " usage "/" limit
            }
        }'
    """
}
```

This catches quota exhaustion *before* a Dataflow job fails 30 minutes into execution.

### Deployment Success Rates

Track these CI/CD health metrics over time:

- **Build success rate**: target > 95% on main branch
- **Mean time to recovery (MTTR)**: how quickly a broken build gets fixed
- **Test coverage trends**: the project uses `pytest --cov=framework --cov-report=xml` in Docker to generate coverage reports
- **Deployment frequency**: how often code reaches each environment (dev/test/uat/prod)
- **Test result archival**: JUnit XML output (`--junitxml=/app/data/test-results.xml`) enables trend analysis across builds

### Docker Resource Limits in CI

The project's `docker-compose.yml` sets resource constraints for test containers, which doubles as monitoring input:

```yaml
deploy:
  resources:
    limits:
      cpus: '2.0'
      memory: 2G
    reservations:
      cpus: '1.0'
      memory: 1G
```

If tests consistently hit the memory ceiling, it signals either a memory leak or that the test suite needs splitting.

---

## Data Freshness Monitoring With SLA Thresholds

The existing freshness gauge (above) captures *current* staleness, but production systems need tiered SLA thresholds per table so that alerts fire at the right urgency. The pattern below -- drawn from [[Fivetran]] connector monitoring -- classifies each table into `FRESH`, `WARNING`, or `STALE` based on connector-specific expectations.

### SQL-Based Freshness Classification

```sql
-- Per-table freshness view with tiered SLA thresholds
CREATE OR REPLACE VIEW fivetran_data_freshness AS
SELECT
    connector_name,
    table_name,
    last_sync_time,
    DATEDIFF(minute, last_sync_time, CURRENT_TIMESTAMP()) AS minutes_since_last_sync,
    CASE
        WHEN DATEDIFF(minute, last_sync_time, CURRENT_TIMESTAMP()) > 120 THEN 'STALE'
        WHEN DATEDIFF(minute, last_sync_time, CURRENT_TIMESTAMP()) > 60  THEN 'WARNING'
        ELSE 'FRESH'
    END AS freshness_status
FROM (
    SELECT 'azure_sql_connector' AS connector_name,
           'shipments'           AS table_name,
           MAX(updated_at)       AS last_sync_time
    FROM raw.azure_shipments
    UNION ALL
    SELECT 'telematics_webhook_connector', 'vehicle_telemetry', MAX(timestamp)
    FROM raw.telematics_data
    -- ... additional tables
);
```

Key design choices:

- **Threshold values are per-connector, not global.** A webhook connector delivering telematics every 5 minutes has a tighter SLA than a batch connector syncing hourly. Encode these expectations explicitly rather than applying a single staleness window across the board.
- **Three-tier classification** (`FRESH` / `WARNING` / `STALE`) maps directly to the escalation tiers defined in the Alerting Patterns section above. `WARNING` fires a Slack notification; `STALE` escalates to PagerDuty.
- **Alert view** sits on top of the freshness view and selects only rows where `freshness_status = 'STALE'`, producing actionable alert messages with the connector name, table name, and minutes since last sync.

### Freshness SLA Thresholds by Source Type

| Source Type | Expected Frequency | Warning Threshold | Critical (Stale) Threshold |
|-------------|-------------------|-------------------|----------------------------|
| Webhook / streaming | 5 minutes | > 10 minutes | > 30 minutes |
| REST API (traffic, weather) | 30 minutes | > 60 minutes | > 120 minutes |
| Database connector (batch) | 60 minutes | > 120 minutes | > 240 minutes |
| Daily file drop | 24 hours | > 26 hours | > 30 hours |

---

## Data Volume Anomaly Detection

Beyond the Python-based z-score approach (above), SQL views can continuously monitor volume at the connector level. The critical case to catch is `NO_DATA` -- zero records -- which signals a broken connector or upstream outage rather than normal variance.

### Volume Classification View

```sql
CREATE OR REPLACE VIEW fivetran_data_volume_monitoring AS
SELECT
    connector_name,
    table_name,
    record_count,
    data_size_mb,
    CASE
        WHEN record_count = 0    THEN 'NO_DATA'
        WHEN record_count < 100  THEN 'LOW_VOLUME'
        WHEN record_count < 1000 THEN 'MEDIUM_VOLUME'
        ELSE 'HIGH_VOLUME'
    END AS volume_status
FROM (
    SELECT
        'azure_sql_connector' AS connector_name,
        'customers'           AS table_name,
        COUNT(*)              AS record_count,
        ROUND(SUM(LENGTH(TO_JSON(OBJECT_CONSTRUCT(*)))) / 1024 / 1024, 2) AS data_size_mb
    FROM raw.azure_customers
    -- ... UNION ALL for each monitored table
);
```

### Volume Alert Escalation

The alert view unions freshness, volume, and quality alerts into a single `fivetran_alert_conditions` view. Volume alerts use severity escalation:

| Volume Status | Severity | Action |
|---------------|----------|--------|
| `NO_DATA` | CRITICAL | Immediate investigation -- connector may be broken or source system down |
| `LOW_VOLUME` | MEDIUM | Review within hours -- possible partial sync or upstream filtering change |
| `MEDIUM_VOLUME` / `HIGH_VOLUME` | INFO | No action required |

The `data_size_mb` column (calculated via `OBJECT_CONSTRUCT(*)` serialisation in [[Snowflake]]) provides a secondary signal: if record count is normal but data size has halved, rows may be arriving with null or truncated columns.

---

## Connector Health Monitoring

### Health Scoring Model

A composite health score (0-100) per connector captures freshness, error rate, and throughput in a single number. This avoids the problem of separate alerts for related symptoms -- a connector that is both slow and stale is one issue, not two.

```sql
CREATE OR REPLACE VIEW fivetran_connector_health AS
SELECT
    connector_name,
    connector_type,
    last_sync_time,
    sync_frequency_minutes,
    records_synced_24h,
    data_freshness_status,
    error_count_24h,
    overall_health_score,
    CASE
        WHEN overall_health_score >= 90 THEN 'EXCELLENT'
        WHEN overall_health_score >= 80 THEN 'GOOD'
        WHEN overall_health_score >= 70 THEN 'FAIR'
        WHEN overall_health_score >= 60 THEN 'POOR'
        ELSE 'CRITICAL'
    END AS health_status
FROM (
    SELECT
        'telematics_webhook_connector' AS connector_name,
        'webhook'                      AS connector_type,
        MAX(timestamp)                 AS last_sync_time,
        5                              AS sync_frequency_minutes,
        COUNT(*)                       AS records_synced_24h,
        CASE
            WHEN DATEDIFF(minute, MAX(timestamp), CURRENT_TIMESTAMP()) <= 10 THEN 'FRESH'
            WHEN DATEDIFF(minute, MAX(timestamp), CURRENT_TIMESTAMP()) <= 30 THEN 'STALE'
            ELSE 'CRITICAL'
        END AS data_freshness_status,
        0 AS error_count_24h,
        CASE
            WHEN DATEDIFF(minute, MAX(timestamp), CURRENT_TIMESTAMP()) <= 10 THEN 100
            WHEN DATEDIFF(minute, MAX(timestamp), CURRENT_TIMESTAMP()) <= 30 THEN 80
            ELSE 40
        END AS overall_health_score
    FROM raw.telematics_data
    WHERE timestamp >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
    -- ... UNION ALL for each connector
);
```

### Sync Frequency Analysis With Window Functions

Use `LAG()` to measure the actual interval between consecutive syncs and compare it against the expected frequency. This catches connectors that are technically "active" but syncing far less often than configured.

```sql
CREATE OR REPLACE VIEW fivetran_sync_frequency AS
SELECT
    connector_name,
    table_name,
    expected_sync_frequency_minutes,
    actual_avg_sync_frequency_minutes,
    CASE
        WHEN actual_avg_sync_frequency_minutes > expected_sync_frequency_minutes * 1.5 THEN 'SLOW'
        WHEN actual_avg_sync_frequency_minutes < expected_sync_frequency_minutes * 0.5 THEN 'FAST'
        ELSE 'NORMAL'
    END AS sync_frequency_status
FROM (
    SELECT
        'telematics_webhook_connector' AS connector_name,
        'vehicle_telemetry'            AS table_name,
        5                              AS expected_sync_frequency_minutes,
        AVG(DATEDIFF(minute,
            LAG(timestamp) OVER (ORDER BY timestamp),
            timestamp
        )) AS actual_avg_sync_frequency_minutes
    FROM raw.telematics_data
    WHERE timestamp >= DATEADD(day, -7, CURRENT_DATE())
    -- ... UNION ALL for each connector/table
);
```

The `SLOW` status (actual frequency exceeding 1.5x expected) fires a `FREQUENCY_ALERT`. A `FAST` status is unusual and may indicate duplicate syncs or a misconfigured schedule.

### Connector Performance Trending

Track daily aggregates of sync events, durations, and throughput using `LAG()` and `DATE_TRUNC` to build performance trend views:

```sql
SELECT
    connector_name,
    DATE_TRUNC('day', sync_time) AS sync_date,
    COUNT(*)                     AS sync_events,
    AVG(sync_duration_minutes)   AS avg_sync_duration,
    MAX(sync_duration_minutes)   AS max_sync_duration,
    SUM(records_synced)          AS total_records_synced,
    AVG(records_per_minute)      AS avg_records_per_minute
FROM connector_sync_detail
GROUP BY connector_name, DATE_TRUNC('day', sync_time)
ORDER BY connector_name, sync_date;
```

### Fleet Health Dashboard View

A single summary view counts connectors by health tier for an at-a-glance fleet status:

```sql
SELECT
    COUNT(*) AS total_connectors,
    COUNT(CASE WHEN health_status = 'EXCELLENT' THEN 1 END) AS excellent,
    COUNT(CASE WHEN health_status = 'CRITICAL'  THEN 1 END) AS critical,
    AVG(overall_health_score) AS avg_health_score,
    CASE
        WHEN AVG(overall_health_score) >= 90 THEN 'EXCELLENT'
        WHEN AVG(overall_health_score) >= 80 THEN 'GOOD'
        WHEN AVG(overall_health_score) >= 70 THEN 'FAIR'
        ELSE 'POOR'
    END AS overall_system_health
FROM fivetran_connector_health;
```

---

## GCP Cloud Monitoring Patterns (Terraform)

See also: [[Terraform]]

The GCP monitoring module in `gcp_infra_terraform` defines dashboards, alert policies, notification channels, and uptime checks as Terraform resources. This infrastructure-as-code approach ensures monitoring configuration is version-controlled, peer-reviewed, and reproducible across environments.

### Monitoring Dashboard as Code

Cloud Monitoring dashboards are defined as JSON within `google_monitoring_dashboard`. The mosaic layout positions tiles on a grid:

```hcl
resource "google_monitoring_dashboard" "dashboard" {
  count = var.create_dashboard ? 1 : 0

  dashboard_json = jsonencode({
    displayName = var.dashboard_name
    mosaicLayout = {
      tiles = [
        {
          width  = 6
          height = 4
          widget = {
            title = "CPU Utilisation"
            xyChart = {
              dataSets = [{
                timeSeriesQuery = {
                  timeSeriesFilter = {
                    filter = "metric.type=\"compute.googleapis.com/instance/cpu/utilization\""
                    aggregation = {
                      alignmentPeriod    = "60s"
                      perSeriesAligner   = "ALIGN_MEAN"
                      crossSeriesReducer = "REDUCE_MEAN"
                    }
                  }
                }
                plotType = "LINE"
              }]
            }
          }
        }
        // Additional tiles: Memory, Disk, Network
      ]
    }
  })

  project = var.project_id
}
```

Key patterns:
- **Conditional creation** via `count` and a `var.create_dashboard` flag -- environments that do not need dashboards simply set it to `false`
- **Aggregation settings**: `ALIGN_MEAN` with 60-second alignment smooths noisy metrics; `REDUCE_MEAN` aggregates across instances for fleet-level views
- **Grid positioning**: `xPos` and `yPos` control tile layout (6-wide grid); related metrics should be placed adjacently

### Alert Policies

Each alert policy uses `condition_threshold` with a duration window to avoid transient spikes:

```hcl
resource "google_monitoring_alert_policy" "high_cpu" {
  count        = var.enable_cpu_alerts ? 1 : 0
  display_name = "High CPU Usage Alert"
  combiner     = "OR"

  conditions {
    display_name = "CPU utilisation is high"
    condition_threshold {
      filter          = "metric.type=\"compute.googleapis.com/instance/cpu/utilization\""
      duration        = "300s"
      comparison      = "COMPARISON_GREATER_THAN"
      threshold_value = var.cpu_threshold
      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  notification_channels = var.notification_email != "" ? [
    google_monitoring_notification_channel.email[0].id
  ] : []

  alert_strategy {
    auto_close = "1800s"
  }

  project = var.project_id
}
```

Pattern notes:
- **Duration of 300s** means the condition must hold for 5 continuous minutes before firing -- this filters out momentary spikes
- **Auto-close of 1800s** (30 minutes) automatically resolves the incident if the condition clears, preventing stale open alerts
- **Notification channels** are conditionally attached -- if no email is configured, the alert still exists but does not notify (useful for non-production environments)
- The same pattern applies for memory, disk, and instance-down alerts with different metric filters and thresholds

### Notification Channels

```hcl
resource "google_monitoring_notification_channel" "email" {
  count        = var.notification_email != "" ? 1 : 0
  display_name = "Email Notification Channel"
  type         = "email"
  labels = {
    email_address = var.notification_email
  }
  project = var.project_id
}
```

Additional channel types: `slack`, `pagerduty`, `webhook`, `sms`. Each is a separate `google_monitoring_notification_channel` resource referenced by alert policies.

### Uptime Checks

```hcl
resource "google_monitoring_uptime_check_config" "uptime_check" {
  count        = var.enable_uptime_checks ? 1 : 0
  display_name = var.uptime_check_name
  timeout      = "10s"
  period       = "60s"

  http_check {
    path           = var.uptime_check_path
    port           = var.uptime_check_port
    use_ssl        = var.uptime_check_use_ssl
    request_method = "GET"
  }

  monitored_resource {
    type = "uptime_url"
    labels = {
      host       = var.uptime_check_host
      project_id = var.project_id
    }
  }
}
```

Use uptime checks for pipeline API endpoints (e.g., a health-check endpoint on a Cloud Run ingestion service). A 60-second check period with a 10-second timeout catches outages within 2 minutes.

---

## Fabric Pipeline Execution Logging Patterns

See also: [[Microsoft Fabric]]

Microsoft Fabric pipelines use a T0 control layer for centralised logging. The pattern captures pipeline start, completion, and failure as rows in a dedicated log table, with indexes on `pipeline_name` and `status` for efficient querying.

### Pipeline Log Table

```sql
CREATE TABLE t0.pipeline_log (
    log_id         INT IDENTITY(1,1) PRIMARY KEY,
    pipeline_name  VARCHAR(200) NOT NULL,
    execution_id   VARCHAR(100),
    start_time     DATETIME2 NOT NULL,
    end_time       DATETIME2,
    duration_seconds INT,
    status         VARCHAR(20) NOT NULL,  -- Running, Success, Failed, Cancelled
    rows_processed INT,
    data_size_mb   DECIMAL(10,2),
    error_message  VARCHAR(MAX),
    created_at     DATETIME2 DEFAULT GETDATE()
);

CREATE INDEX idx_pipeline_log_name_time ON t0.pipeline_log(pipeline_name, start_time);
CREATE INDEX idx_pipeline_log_status ON t0.pipeline_log(status, start_time);
```

### Logging Pipeline Events via Script Activities

**On start** (Script Activity at pipeline entry):

```sql
INSERT INTO t0.pipeline_log (pipeline_name, execution_id, start_time, status)
VALUES (
    '@{pipeline().Pipeline}',
    '@{pipeline().RunId}',
    '@{pipeline().TriggerTime}',
    'Running'
);
```

**On success** (Script Activity after main processing):

```sql
UPDATE t0.pipeline_log
SET end_time         = GETDATE(),
    duration_seconds = DATEDIFF(SECOND, start_time, GETDATE()),
    status           = 'Success',
    rows_processed   = @{activity('CopyActivity').output.rowsCopied},
    data_size_mb     = @{activity('CopyActivity').output.dataRead} / 1024.0 / 1024.0
WHERE pipeline_name = '@{pipeline().Pipeline}'
  AND execution_id  = '@{pipeline().RunId}'
  AND status        = 'Running';
```

**On failure** (On Failure Activity path):

```sql
UPDATE t0.pipeline_log
SET end_time         = GETDATE(),
    duration_seconds = DATEDIFF(SECOND, start_time, GETDATE()),
    status           = 'Failed',
    error_message    = '@{activity('ErrorActivity').error.message}'
WHERE pipeline_name = '@{pipeline().Pipeline}'
  AND execution_id  = '@{pipeline().RunId}'
  AND status        = 'Running';
```

### Fabric Monitoring Across Architecture Layers

The T0-T5 architecture provides monitoring hooks at each layer:

| Layer | What to Monitor | Key Metrics |
|-------|----------------|-------------|
| **T0** (Control) | Pipeline executions, error log, alert configuration | Execution count, failure rate, alert volume |
| **T1** (Ingestion) | Data arrival, row counts, null checks | Records ingested, null percentage, freshness |
| **T2** (SCD2) | Merge operations, slowly changing dimension processing | Rows merged, SCD2 version counts, duration |
| **T3** (Transformation) | Business logic transforms, aggregations | Input/output row ratios, duration trends |
| **T5** (Presentation) | View performance, query patterns | Query duration, cache hit rate |

### Performance Logging Within Stored Procedures

Wrap MERGE and transformation logic with timing instrumentation:

```sql
CREATE PROCEDURE t2.usp_merge_dim_department_with_perf
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @StartTime DATETIME2 = GETDATE();
    DECLARE @RowsProcessed INT;

    -- Main MERGE logic here
    MERGE t2.dim_department AS target
    USING (SELECT * FROM t1_department) AS source
    ON target.dept_id = source.dept_id AND target.is_current = 1
    WHEN MATCHED THEN UPDATE SET ...
    WHEN NOT MATCHED THEN INSERT ...;

    SET @RowsProcessed = @@ROWCOUNT;

    INSERT INTO t0.performance_metrics (
        component_type, component_name, execution_date,
        duration_seconds, rows_processed
    )
    VALUES (
        'StoredProcedure', 't2.usp_merge_dim_department',
        @StartTime,
        DATEDIFF(MILLISECOND, @StartTime, GETDATE()) / 1000.0,
        @RowsProcessed
    );
END;
```

### Error Tracking With Severity-Based Escalation

The Fabric error logging pattern uses a stored procedure that both logs the error and conditionally triggers alerts for `Critical` and `High` severity:

```sql
CREATE PROCEDURE t0.usp_log_error
    @ComponentType  VARCHAR(50),
    @ComponentName  VARCHAR(200),
    @ErrorSeverity  VARCHAR(20),
    @ErrorMessage   VARCHAR(MAX),
    @ErrorDetails   VARCHAR(MAX) = NULL
AS
BEGIN
    INSERT INTO t0.error_log (
        component_type, component_name, error_severity,
        error_message, error_details
    )
    VALUES (@ComponentType, @ComponentName, @ErrorSeverity,
            @ErrorMessage, @ErrorDetails);

    IF @ErrorSeverity IN ('Critical', 'High')
    BEGIN
        EXEC t0.usp_send_alert
            @AlertType     = 'Error',
            @ComponentName = @ComponentName,
            @Message       = @ErrorMessage;
    END
END;
```

---

## Fabric Warehouse Monitoring Views (T0-T4)

The T0-T5 architecture supports a **view-per-concern** monitoring pattern — dedicated SQL views at each layer that can be queried directly, composed into dashboards, or used as alert sources. These views require no external tooling; they run against the same Fabric Warehouse that hosts the data.

### View Catalogue

| Layer | View | Purpose |
|-------|------|---------|
| **T0** | `audit.vw_latest_run_status` | One row per table — most recent pipeline outcome |
| **T0** | `audit.vw_recent_failures` | All FAILED records for investigation |
| **T0** | `audit.vw_layer_reconciliation` | Cross-layer row count diffs (T1 → T2 → T3 → T4) |
| **T0** | `control.vw_pipeline_control_state` | Current control table state (is_active, do_not_load, watermarks) |
| **T1** | `{schema}.vw_record_counts` | Row counts per T1 transient table |
| **T1** | `{schema}.vw_data_freshness` | Latest `etl_loaded_at` and minutes since load per table |
| **T2** | `{schema}.vw_record_counts` | Row counts per T2 persistent table |
| **T2** | `{schema}.vw_data_freshness` | Latest `etl_start_time` and minutes since load |
| **T2** | `{schema}.vw_scd2_health` | Orphaned historical rows, NULL PKs, deleted %, merge recency |
| **T3** | `{schema}.vw_data_quality` | NULL counts on key fields, bad date detection, missing FKs |
| **T4** | `presentation.vw_referential_integrity` | Fact rows with NULL surrogate keys per dimension |
| **T4** | `presentation.vw_dim_coverage` | Matched vs missing dimension keys with coverage percentage |

### Pattern 1: Latest Run Status (T0)

Shows the most recent outcome per table, filtering out `persistent_begin` markers (which are just "RUNNING" placeholders):

```sql
CREATE VIEW audit.vw_latest_run_status AS
WITH ranked AS (
    SELECT
        pipeline_name, table_name, load_layer,
        rows_loaded, run_status, error_message,
        start_time, loaded_at,
        ROW_NUMBER() OVER (
            PARTITION BY pipeline_name, table_name
            ORDER BY loaded_at DESC
        ) AS rn
    FROM audit.pipeline_run_log
    WHERE load_layer <> 'persistent_begin'
)
SELECT pipeline_name, table_name, load_layer,
       rows_loaded, run_status, error_message,
       start_time, loaded_at
FROM ranked
WHERE rn = 1;
```

**Use case:** Quick health check — "did everything succeed on the last run?" Filter on `run_status = 'FAILED'` for immediate investigation targets.

### Pattern 2: Cross-Layer Reconciliation (T0)

Compares row counts across all four layers to detect data loss or inflation between tiers:

```sql
CREATE VIEW audit.vw_layer_reconciliation AS
SELECT
    t1.table_name,
    t1.t1_rows,
    t2.t2_current_rows,       -- only etl_is_current=1 AND etl_is_deleted=0
    t3.t3_rows,
    t4.t4_rows,
    t2.t2_current_rows - t1.t1_rows AS t1_to_t2_diff,
    t3.t3_rows - t2.t2_current_rows AS t2_to_t3_diff,
    t4.t4_rows - t3.t3_rows AS t3_to_t4_diff
FROM (
    SELECT '{table}' AS table_name, COUNT(*) AS t1_rows
    FROM [T1_transient].[{schema}].[{table}]
    UNION ALL ...
) t1
LEFT JOIN (
    SELECT '{table}' AS table_name,
        SUM(CASE WHEN etl_is_current = 1 AND etl_is_deleted = 0
            THEN 1 ELSE 0 END) AS t2_current_rows
    FROM [T2_persistent].[{schema}].[{table}]
    UNION ALL ...
) t2 ON t1.table_name = t2.table_name
LEFT JOIN (...) t3 ON t1.table_name = t3.table_name
LEFT JOIN (...) t4 ON t1.table_name = t4.table_name;
```

**What to watch for:**
- `t1_to_t2_diff > 0` — T2 has more current rows than T1 delivered. Normal for incremental/windowed loads (T1 only contains the latest window, but T2 has accumulated history). Unexpected for snapshot tables.
- `t2_to_t3_diff <> 0` — T3 should mirror T2 current rows exactly (T3 sources `WHERE etl_is_current = 1`). Any difference indicates a stale T3 refresh.
- `t3_to_t4_diff <> 0` — T4 dims should match T3 row counts (TRUNCATE+INSERT). Facts may differ due to JOIN filtering.

### Pattern 3: Data Freshness Per Layer

Each layer has its own freshness view using the same structure — `MAX(etl_loaded_at)` with `DATEDIFF(MINUTE, ...)`:

```sql
CREATE VIEW [{schema}].[vw_data_freshness] AS
SELECT 'orders' AS table_name,
    MAX(etl_loaded_at) AS latest_load,
    DATEDIFF(MINUTE, MAX(etl_loaded_at), GETUTCDATE()) AS minutes_since_load,
    COUNT(*) AS row_count
FROM {schema}.orders
UNION ALL
-- ... repeat per table
```

**Freshness varies by layer:**
- **T1** — should be near-zero after a pipeline run (truncated and reloaded)
- **T2** — `etl_start_time` on current rows reflects the last merge timestamp
- **T3** — reflects the last integration SP execution
- **T4** — reflects the last dim/fact load

**Alert threshold:** If `minutes_since_load` exceeds 2x the expected schedule interval, investigate.

### Pattern 4: Data Quality Checks (T3)

T3 is the right layer for quality checks because type casting and business logic have been applied but the data hasn't been reshaped into star schema yet:

```sql
CREATE VIEW [{schema}].[vw_data_quality] AS
SELECT 'orders' AS table_name,
    COUNT(*) AS total_rows,
    SUM(CASE WHEN name IS NULL THEN 1 ELSE 0 END) AS null_name,
    NULL AS null_code,
    SUM(CASE WHEN order_date IS NULL THEN 1 ELSE 0 END) AS null_order_date,
    NULL AS bad_date_rows,
    SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS null_fk
FROM {schema}.orders
UNION ALL
SELECT 'order_lines',
    COUNT(*),
    SUM(CASE WHEN product_name IS NULL THEN 1 ELSE 0 END),
    NULL,
    SUM(CASE WHEN ship_date IS NULL THEN 1 ELSE 0 END),
    -- Detect source system default dates (1900-01-01) that indicate missing data
    SUM(CASE WHEN ship_date = '1900-01-01' THEN 1 ELSE 0 END),
    SUM(CASE WHEN order_id IS NULL OR product_id IS NULL THEN 1 ELSE 0 END)
FROM {schema}.order_lines
UNION ALL ...
```

**Design decisions:**
- Columns are **table-specific** — `null_order_date` is only relevant to date-bearing tables, so other tables return NULL for that column
- **Bad date detection** catches source system defaults (e.g. `1900-01-01`) that survived the T3 `TRY_CAST` transformation
- **Foreign key checks** (`null_fk`) flag rows where expected relationships are missing before they cause NULL surrogate keys in T4

### Pattern 5: Referential Integrity (T4)

After T4 loads dims and facts with surrogate key lookups, this view checks how many fact rows have NULL keys — meaning the dimension lookup failed:

```sql
CREATE VIEW presentation.vw_referential_integrity AS
SELECT 'fact_orders' AS table_name,
    COUNT(*) AS total_rows,
    SUM(CASE WHEN date_key IS NULL THEN 1 ELSE 0 END) AS null_date_key,
    SUM(CASE WHEN customer_key IS NULL THEN 1 ELSE 0 END) AS null_customer_key,
    SUM(CASE WHEN product_key IS NULL THEN 1 ELSE 0 END) AS null_product_key,
    NULL AS null_employee_key
FROM presentation.fact_orders
UNION ALL ...
```

**Companion view — dimension coverage with percentage:**

```sql
CREATE VIEW presentation.vw_dim_coverage AS
SELECT 'fact_orders' AS fact_table,
    'dim_customer' AS dimension,
    COUNT(*) AS total_rows,
    SUM(CASE WHEN customer_key IS NOT NULL THEN 1 ELSE 0 END) AS matched_rows,
    SUM(CASE WHEN customer_key IS NULL THEN 1 ELSE 0 END) AS missing_rows,
    CAST(SUM(CASE WHEN customer_key IS NULL THEN 1.0 ELSE 0 END)
        / NULLIF(COUNT(*), 0) * 100 AS DECIMAL(5,2)) AS missing_pct
FROM presentation.fact_orders
UNION ALL ...
```

**Alert thresholds:**
- `missing_pct > 0%` on `dim_date` — should never happen if `dim_date` covers the full date range
- `missing_pct > 5%` on business dimensions — indicates data gaps in the source or timing issues in the dim load order

### SCD2 Health Check (T2)

See [[SCD Type 2 Patterns#SCD2 Health Check View]] for the `vw_scd2_health` view that monitors orphaned historical rows, NULL PKs, deleted percentage, and merge recency.

### Composing a Morning Health Check

Query all monitoring views in sequence for a daily operational check:

```sql
-- 1. Any failures since last check?
SELECT * FROM [T0_control].audit.vw_recent_failures
WHERE loaded_at >= DATEADD(HOUR, -12, GETUTCDATE());

-- 2. All tables green?
SELECT * FROM [T0_control].audit.vw_latest_run_status
WHERE run_status <> 'SUCCESS';

-- 3. Cross-layer alignment?
SELECT * FROM [T0_control].audit.vw_layer_reconciliation
WHERE t2_to_t3_diff <> 0 OR t3_to_t4_diff <> 0;

-- 4. Data quality issues?
SELECT * FROM [T3_integration].{schema}.vw_data_quality
WHERE null_fk > 0 OR bad_date_rows > 0;

-- 5. Referential integrity?
SELECT * FROM [T4_presentation].presentation.vw_dim_coverage
WHERE missing_pct > 0;
```

---

## Pipeline SLO Definition Patterns

Service Level Objectives (SLOs) formalise what "good" looks like for a data pipeline. While SLAs are contractual commitments to stakeholders, SLOs are internal engineering targets that the monitoring system enforces. Define SLOs across three dimensions: **latency**, **freshness**, and **completeness**.

### The Three SLO Dimensions

| Dimension | Definition | Example Target | How to Measure |
|-----------|-----------|----------------|----------------|
| **Latency** | Time from pipeline trigger to data availability | Orders pipeline completes within 10 minutes | `duration_seconds` in pipeline log or Prometheus histogram |
| **Freshness** | Maximum age of the newest record in a table | Shipments table updated within 2 hours of real time | `DATEDIFF(minute, MAX(updated_at), CURRENT_TIMESTAMP())` |
| **Completeness** | Proportion of expected records that arrived | >= 99.5% of source records land in the target | Compare source count vs target count per batch |

### Defining SLOs Per Pipeline

Encode SLO targets in a configuration table or YAML file so they are queryable and version-controlled:

```sql
-- Fabric T0 pattern: SLO configuration table
CREATE TABLE t0.pipeline_slo (
    slo_id             INT IDENTITY(1,1) PRIMARY KEY,
    pipeline_name      VARCHAR(200) NOT NULL,
    slo_dimension      VARCHAR(50)  NOT NULL,  -- Latency, Freshness, Completeness
    target_value       DECIMAL(10,2) NOT NULL,
    unit               VARCHAR(20)  NOT NULL,  -- seconds, minutes, percentage
    warning_threshold  DECIMAL(10,2),
    critical_threshold DECIMAL(10,2),
    enabled            BIT DEFAULT 1
);

-- Example entries
INSERT INTO t0.pipeline_slo VALUES
    ('orders_etl',     'Latency',      600,   'seconds',    480,   540,   1),
    ('orders_etl',     'Freshness',    120,   'minutes',    90,    110,   1),
    ('orders_etl',     'Completeness', 99.5,  'percentage', 99.0,  98.0,  1),
    ('telemetry_sync', 'Freshness',    10,    'minutes',    7,     9,     1);
```

### SLO Compliance Monitoring

Join the SLO configuration table against actual metrics to compute compliance:

```sql
-- Check latency SLO compliance over the last 7 days
SELECT
    s.pipeline_name,
    s.target_value AS slo_target_seconds,
    COUNT(*) AS total_runs,
    SUM(CASE WHEN p.duration_seconds <= s.target_value THEN 1 ELSE 0 END) AS runs_within_slo,
    ROUND(
        100.0 * SUM(CASE WHEN p.duration_seconds <= s.target_value THEN 1 ELSE 0 END) / COUNT(*),
        2
    ) AS slo_compliance_pct
FROM t0.pipeline_slo s
JOIN t0.pipeline_log p ON s.pipeline_name = p.pipeline_name
WHERE s.slo_dimension = 'Latency'
  AND s.enabled = 1
  AND p.status = 'Success'
  AND p.start_time >= DATEADD(day, -7, GETDATE())
GROUP BY s.pipeline_name, s.target_value;
```

### Error Budgets

An error budget is the inverse of the SLO: if the freshness SLO is 99.5% compliance over 30 days, the error budget allows 0.5% of measurements to breach the threshold. When the error budget is exhausted, freeze feature work and focus on reliability.

```
Error budget remaining = SLO target % - actual breach %
Example: 99.5% target - 99.2% actual = 0.3% remaining (of 0.5% budget)
```

Track error budget burn rate. If 50% of the monthly budget is consumed in the first week, the trajectory suggests the SLO will be breached -- alert early rather than waiting for the end of the window.

### SLO Review Cadence

- **Weekly**: review SLO compliance dashboards, investigate breaches
- **Monthly**: review error budget consumption, adjust thresholds if consistently too tight or too loose
- **Quarterly**: re-evaluate whether SLO dimensions and targets still reflect business requirements

---

## Summary Checklist

- [ ] Structured JSON logging with context binding (job_name, run_id, stage)
- [ ] MetricsCollector emitting stage-level events (start, complete, error, record counts)
- [ ] Prometheus metrics with Push Gateway for batch jobs
- [ ] OpenTelemetry tracing across pipeline stages
- [ ] Log4j tuned to suppress Spark/Hadoop noise
- [ ] Data freshness and volume anomaly monitoring
- [ ] Tiered alerting with escalation (Slack -> Email -> PagerDuty)
- [ ] Grafana or Cloud Monitoring dashboards with key pipeline metrics
- [ ] Jenkins post-build archival of logs, test results, and coverage
- [ ] GCP quota checks integrated into CI before job submission
- [ ] Data freshness SLA thresholds per connector/source type
- [ ] Volume anomaly detection with NO_DATA and LOW_VOLUME alerts
- [ ] Connector health scoring with sync frequency analysis (window functions)
- [ ] GCP Cloud Monitoring alert policies and dashboards defined in Terraform
- [ ] Uptime checks for pipeline API endpoints
- [ ] Fabric T0 pipeline execution logging (start, success, failure)
- [ ] Performance instrumentation within stored procedures
- [ ] Fabric monitoring views: latest run status, cross-layer reconciliation, data freshness per layer
- [ ] T3 data quality views: NULL counts on key fields, bad date detection, missing FK checks
- [ ] T4 referential integrity and dimension coverage views with missing_pct alerting
- [ ] SCD2 health check view: orphaned historical rows, NULL PKs, deleted percentage
- [ ] Composable morning health check query across all monitoring views
- [ ] Pipeline SLOs defined for latency, freshness, and completeness
- [ ] Error budget tracking with burn-rate alerting
