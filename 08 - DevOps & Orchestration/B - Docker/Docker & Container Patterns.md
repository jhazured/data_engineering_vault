# Docker & Container Patterns

Container patterns for building scalable, reliable distributed services — drawn from "Designing Distributed Systems" (Brendan Burns).

## Single-Node Patterns

### Sidecar Pattern

A helper container that extends the main container's functionality:

```
┌─────────────────────────────┐
│           Pod               │
│  ┌──────────┐ ┌──────────┐ │
│  │   App    │ │  Sidecar  │ │
│  │ (main)   │ │ (helper)  │ │
│  └──────────┘ └──────────┘ │
│       shared filesystem     │
└─────────────────────────────┘
```

**Use cases:**
- **Log shipping** — sidecar collects logs and ships to Elasticsearch/S3
- **Config sync** — sidecar watches for config changes and reloads
- **TLS termination** — sidecar handles certificates, app speaks plain HTTP
- **Monitoring** — sidecar exports metrics to Prometheus

**Benefit:** Main app stays simple; cross-cutting concerns are modularized.

### Ambassador Pattern

A proxy container that simplifies external connections:

```
App Container → Ambassador Container → External Service(s)
                (handles routing,       (multiple shards,
                 retries, failover)      different environments)
```

**Use cases:**
- **Service mesh proxy** — Envoy/Istio sidecar handling service discovery
- **Database sharding proxy** — ambassador routes queries to correct shard
- **Environment abstraction** — app connects to `localhost:5432`, ambassador routes to dev/staging/prod database

### Adapter Pattern

Normalizes output from the main container:

```
App Container → Adapter Container → Monitoring System
(custom logs)   (transforms to      (expects standard
                 standard format)    format)
```

**Use case:** Legacy app produces custom log format; adapter converts to JSON/structured logging without modifying the app.

## Multi-Node Patterns

### Replicated Load-Balanced Services

Multiple identical containers behind a load balancer:

```
              ┌─── Container 1 (App)
Load Balancer ├─── Container 2 (App)
              └─── Container 3 (App)
```

- **Stateless services** — any instance can handle any request
- Scale horizontally by adding replicas
- Health checks remove unhealthy instances

### Sharded Services

Requests are routed to a specific instance based on a key:

```
               ┌─── Shard 1 (keys A-M)
Shard Router ──├─── Shard 2 (keys N-Z)
               └─── Shard 3 (overflow)
```

- **Stateful services** — each shard owns a subset of data
- Consistent hashing minimizes redistribution when shards are added/removed
- Use case: caches, databases, user session stores

### Scatter/Gather Pattern

Send request to all nodes, merge results:

```
              ┌─── Node 1 → partial result ──┐
Request ──────├─── Node 2 → partial result ───┼──→ Merge → Response
              └─── Node 3 → partial result ──┘
```

- **Use case:** Distributed search (query all shards, merge top-K results)
- **Risk:** Latency = slowest node (tail latency problem)
- **Mitigation:** Timeouts, redundant requests to slow nodes

## Work Queue Systems

```
Task Queue ──→ Worker 1 ──→ Results
             → Worker 2 ──→ Results
             → Worker 3 ──→ Results
```

- Decouple producers from consumers
- Scale workers independently based on queue depth
- Failed tasks can be retried or sent to dead-letter queue
- **Examples:** Celery (Python), Sidekiq (Ruby), SQS + Lambda

## Docker Fundamentals

### Dockerfile Patterns

```dockerfile
# Multi-stage build — small final image
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY . .
CMD ["python", "main.py"]
```

**Multi-stage builds:** Build dependencies in a large image, copy only runtime files to a slim image.

### docker-compose for Development

```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      - DATABASE_URL=postgresql://db:5432/mydb
    depends_on: [db]

  db:
    image: postgres:15
    volumes: ["pgdata:/var/lib/postgresql/data"]
    environment:
      POSTGRES_PASSWORD: dev_password

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

### Container Best Practices

| Practice | Why |
|----------|-----|
| One process per container | Easier to scale, monitor, and debug |
| Use `.dockerignore` | Prevent copying `.venv`, `.git`, `node_modules` into image |
| Pin image versions | `python:3.12-slim` not `python:latest` |
| Non-root user | Security — `USER 1000` in Dockerfile |
| Health checks | Orchestrator can restart unhealthy containers |
| Minimize layers | Combine RUN commands with `&&` |
| Use slim/alpine images | Smaller attack surface, faster pulls |

## Kubernetes Concepts (Brief)

| Concept | What It Is |
|---------|-----------|
| **Pod** | Smallest deployable unit — one or more containers sharing network/storage |
| **Deployment** | Manages replicated pods with rolling updates |
| **Service** | Stable network endpoint for a set of pods |
| **ConfigMap/Secret** | Externalized configuration and credentials |
| **StatefulSet** | For stateful workloads (databases) — stable pod names and storage |
| **Job/CronJob** | One-off or scheduled batch work |
| **Ingress** | HTTP routing to services |

## When to Containerize Data Pipelines

**Good fit:**
- Reproducible environments (local = CI = prod)
- Multi-language pipelines (Python + Java + Go)
- Microservices or event-driven architectures
- Teams with varied local setups

**Often unnecessary:**
- Pure SQL transformations (dbt runs fine without containers)
- Managed services (Snowflake, BigQuery — no infrastructure to manage)
- Simple scripts with virtual environments

---

## Multi-Stage Builds for Data Pipelines

Multi-stage builds separate heavy build toolchains from lean runtime images. The builder stage installs compilers, downloads SDKs, and builds wheels; the final stage copies only the artifacts needed at runtime.

**Spark + GCP pipeline image (from Dockerfile.ubuntu):**

```dockerfile
FROM ubuntu:22.04 AS builder

ARG SPARK_VERSION=3.5.1
ARG JAVA_VERSION=11
ARG GCS_CONNECTOR_VERSION=hadoop3-2.2.21

# Install JDK, Python, build-essential, Google Cloud SDK
RUN apt-get update && apt-get install -y --no-install-recommends \
    openjdk-${JAVA_VERSION}-jdk python3.10 python3.10-pip \
    curl build-essential && \
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor ... && \
    apt-get install -y google-cloud-sdk && \
    rm -rf /var/lib/apt/lists/*

# Download Spark + connector JARs
RUN curl -sSL .../spark-${SPARK_VERSION}-bin-hadoop3.tgz | tar -xz -C /opt && \
    curl -sSL .../gcs-connector-${GCS_CONNECTOR_VERSION}.jar -o /opt/spark/jars/... && \
    rm -rf /opt/spark/examples /opt/spark/data /opt/spark/R

COPY requirements/prod.txt .
RUN pip3 install --no-cache-dir --user -r prod.txt

# --- Runtime stage: JRE instead of JDK, no compilers ---
FROM ubuntu:22.04

COPY --from=builder /usr/lib/jvm /usr/lib/jvm
COPY --from=builder /opt/spark /opt/spark
COPY --from=builder /usr/lib/google-cloud-sdk /usr/lib/google-cloud-sdk
COPY --from=builder /root/.local /usr/local
```

The runtime stage installs `openjdk-11-jre-headless` instead of the full JDK and omits `build-essential` entirely. This pattern also applies to Ansible images — install `build-essential`, `libffi-dev`, and `libyaml-dev` in the builder, then copy only `/usr/local/lib/python3.10/site-packages` and `/usr/local/bin` into the final stage.

See also: [[Infrastructure as Code with Ansible]]

## Dockerfile Best Practices for DE

### Non-Root Users

Always create a dedicated user for the workload:

```dockerfile
RUN groupadd -r etl_user && \
    useradd -r -g etl_user -d /home/etl_user -s /bin/bash etl_user && \
    mkdir -p /app /app/logs /app/checkpoints && \
    chown -R etl_user:etl_user /app
USER etl_user
```

### Health Checks

Tailor the health check to the actual runtime — a SparkSession instantiation for Spark containers, a version command for tooling containers:

```dockerfile
# Spark ETL container — validates the JVM + PySpark stack
HEALTHCHECK --interval=30s --timeout=30s --start-period=60s --retries=3 \
    CMD python3 -c "from pyspark.sql import SparkSession; \
    spark = SparkSession.builder.appName('healthcheck').getOrCreate(); \
    spark.stop()" || exit 1

# Ansible container — lightweight version check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD ansible --version || exit 1
```

### Signal Handling and Layer Caching

Use `STOPSIGNAL SIGTERM` so orchestrators can gracefully shut down long-running Spark jobs. Order Dockerfile instructions from least-changing to most-changing for optimal layer caching — system packages first, then pip dependencies, then application code last.

```dockerfile
STOPSIGNAL SIGTERM
```

### .dockerignore

Prevent bloating the build context:

```
.git
.venv
__pycache__
*.pyc
.pytest_cache
data/
logs/
*.egg-info
```

## Docker Compose for Local Development

### Profiles

Profiles let you define services that only start when explicitly requested, keeping `docker compose up` fast by default:

```yaml
services:
  etl_test:
    build:
      context: .
      dockerfile: docker/Dockerfile.ubuntu
    profiles: [test, ci]

  etl_dev:
    command: ["tail", "-f", "/dev/null"]  # keep container alive
    stdin_open: true
    tty: true
    profiles: [dev]

  jupyter:
    command: >
      jupyter lab --ip=0.0.0.0 --port=8888 --no-browser --allow-root
    ports: ["8888:8888"]
    profiles: [dev, jupyter]
```

Run with `docker compose --profile test up` or `docker compose --profile dev up`.

### Service Dependencies and Test Databases

A PostgreSQL container with init scripts provides a repeatable integration test target:

```yaml
  test_db:
    image: postgres:13-alpine
    environment:
      POSTGRES_DB: test_etl
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_password
    volumes:
      - test_db_data:/var/lib/postgresql/data
      - ./tests/fixtures/sql:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test_user -d test_etl"]
      interval: 10s
      timeout: 5s
      retries: 5
    profiles: [test, ci]
```

The `etl_test` service uses `depends_on: [test_db]` so Compose starts the database first.

### Networking and Resource Limits

Define a custom bridge network with explicit subnets to avoid collisions with corporate VPNs. Set resource limits to mirror CI runner constraints:

```yaml
networks:
  etl_test_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

services:
  etl_test:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
```

See also: [[Docker Networking]]

## Python Dependency Management in Containers

### Per-Environment Requirements Files

Maintain separate requirements files per environment — `dev.txt`, `test.txt`, `uat.txt`, `prod.txt`. Pass the target via a build argument:

```dockerfile
ARG ENV=prod.txt
COPY requirements/${ENV} ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
```

Build with `docker build --build-arg ENV=test.txt .` to get a test image with pytest and coverage tooling included.

### Wheel Caching and Offline Installs

For air-gapped or bandwidth-constrained environments, pre-download wheels and use `--find-links`:

```dockerfile
RUN pip install --no-cache-dir --user -r prod.txt
# Or with a local wheel cache:
# RUN pip install --no-cache-dir --find-links=/wheels -r prod.txt
```

Set `PIP_NO_CACHE_DIR=1` and `PIP_DISABLE_PIP_VERSION_CHECK=1` as environment variables to keep images deterministic and avoid unnecessary network calls.

### Virtualenv vs System Python

In containers, installing directly into system Python is acceptable — the container is already an isolated environment. Using `--user` installs into `/root/.local` in the builder, which is then copied to `/usr/local` in the runtime stage, keeping the separation clean.

## Spark in Docker

### Java + Python + Spark Layer Setup

The builder stage installs the full JDK for compilation, downloads the Spark distribution, and fetches connector JARs. The runtime stage needs only `jre-headless`:

```dockerfile
# Builder gets JDK + Spark + connectors
RUN curl -sSL .../spark-${SPARK_VERSION}-bin-hadoop3.tgz | tar -xz -C /opt

# Download GCS + BigQuery connectors into Spark's jars/ directory
RUN curl -sSL .../gcs-connector-hadoop3-2.2.21.jar -o ${SPARK_HOME}/jars/... && \
    curl -sSL .../spark-bigquery-with-dependencies_2.12-0.36.1.jar -o ${SPARK_HOME}/jars/...

# Runtime gets JRE only
COPY --from=builder /opt/spark /opt/spark
```

### Configuration and Port Exposure

Mount `spark-defaults.conf` into the container and set environment variables so PySpark finds the correct Python interpreter:

```dockerfile
ENV PYSPARK_PYTHON=python3 \
    PYSPARK_DRIVER_PYTHON=python3 \
    SPARK_CONF_DIR=/opt/spark/conf

COPY config/spark-defaults.conf ${SPARK_HOME}/conf/spark-defaults.conf
EXPOSE 4040 4041 4042 4043 8080 8081
```

Ports 4040-4043 are Spark UI (one per concurrent SparkContext), 8080-8081 are master/worker UIs. Create checkpoint and event log directories in the image so Spark structured streaming works out of the box:

```dockerfile
RUN mkdir -p /app/checkpoints /app/spark-events /app/spark-warehouse
```

See also: [[Apache Spark Architecture]], [[GCP BigQuery]]

## Makefile Integration

A Makefile wraps common docker compose operations into memorable targets:

```makefile
DC = docker compose

build:
	$(DC) build

test:
	$(DC) run --rm etl_test

up:
	$(DC) up

stop:
	$(DC) down

clean:
	$(DC) down -v --remove-orphans

clean-volumes:
	$(DC) down -v --remove-orphans
	docker volume prune -f
```

Key patterns:
- `run --rm` for test containers ensures they are removed after execution
- `down -v --remove-orphans` removes volumes and any containers from old compose files
- Separate `stop` (preserves volumes) from `clean` (destroys everything) to avoid accidental data loss
- The `DC` variable makes it easy to switch between `docker compose` and `docker-compose` (legacy)

Extend with environment-specific targets by passing build args:

```makefile
build-test:
	$(DC) build --build-arg ENV=test.txt

build-dev:
	$(DC) build --build-arg ENV=dev.txt
```

See also: [[CI CD Pipelines]], [[Makefile Patterns]]
