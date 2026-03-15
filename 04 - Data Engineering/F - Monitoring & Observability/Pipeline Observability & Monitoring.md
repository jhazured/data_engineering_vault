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
