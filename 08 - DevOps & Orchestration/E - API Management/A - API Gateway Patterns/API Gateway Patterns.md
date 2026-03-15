Tags: #api-gateway #devops #microservices #security #cloud-architecture

---

# API Gateway Patterns

## What Is an API Gateway?

An API gateway is a single entry point that sits between clients and backend services. It handles cross-cutting concerns — routing, authentication, rate limiting, request transformation, and observability — so that individual services do not need to implement these independently. In data engineering, API gateways front data APIs, model serving endpoints, metadata services, and webhook receivers.

```
Clients (dashboards, apps, partners)
        │
        ▼
   ┌─────────────┐
   │ API Gateway  │  ← routing, auth, rate limiting, transformation
   └─────────────┘
        │
   ┌────┼────┐
   ▼    ▼    ▼
 Svc A Svc B Svc C   ← backend services (data API, ML model, metadata)
```

### Core Responsibilities

| Responsibility | Description |
|----------------|-------------|
| **Request routing** | Route inbound requests to the correct backend based on path, headers, or method |
| **Authentication and authorisation** | Validate API keys, OAuth tokens, or JWTs before requests reach backends |
| **Rate limiting** | Protect backends from overload by throttling requests per client or globally |
| **Request/response transformation** | Modify headers, rewrite paths, reshape payloads between client and backend formats |
| **SSL/TLS termination** | Handle HTTPS at the gateway so backends can communicate over plain HTTP internally |
| **Caching** | Cache responses for idempotent reads to reduce backend load |
| **Load balancing** | Distribute traffic across multiple backend instances |
| **Observability** | Centralised logging, metrics, and distributed tracing for all API traffic |

---

## AWS API Gateway

AWS offers three API Gateway types, each suited to different use cases:

### REST API vs HTTP API vs WebSocket API

| Aspect | REST API | HTTP API | WebSocket API |
|--------|----------|----------|---------------|
| **Protocol** | HTTP/1.1 | HTTP/1.1, HTTP/2 | WebSocket |
| **Latency** | Higher (more features) | Lower (~60% less) | Persistent connection |
| **Cost** | Higher | ~70% cheaper | Per-message pricing |
| **Features** | Full (caching, WAF, usage plans, API keys, request validation) | Minimal (JWT auth, CORS, Lambda/HTTP proxy) | Bidirectional messaging |
| **Authorisation** | IAM, Cognito, Lambda authoriser, API keys | JWT authoriser, IAM, Lambda | Lambda authoriser |
| **Use case** | Public APIs with usage plans, partner APIs | Internal microservices, Lambda backends | Real-time dashboards, streaming |
| **OpenAPI support** | Full import/export | Import only | N/A |

**Guidance:** Use HTTP API unless you specifically need REST API features (caching, request validation, usage plans, WAF integration). Use WebSocket API for real-time bidirectional communication.

### Key Patterns with AWS API Gateway

```
Client → API Gateway → Lambda → DynamoDB / S3 / Snowflake
                     → ECS / Fargate service
                     → Step Functions (orchestration)
                     → SQS (async ingestion)
```

- **Lambda proxy integration** — the gateway passes the entire request to Lambda and returns the Lambda response directly. Minimal gateway configuration.
- **VPC Link** — securely connect to private resources (ECS tasks, RDS instances) in a VPC without exposing them to the internet.
- **Stage variables** — parameterise deployments across dev/staging/prod without duplicating API definitions.
- **Usage plans and API keys** — throttle and meter partner access with per-key quotas.

---

## Kong Gateway

Kong is an open-source, plugin-extensible API gateway built on NGINX and OpenResty. It is available as Kong OSS, Kong Enterprise, and Kong Konnect (managed SaaS).

### Architecture

```
Client → Kong Gateway → Upstream Services
              │
         Kong Database (PostgreSQL) or DB-less mode (declarative YAML)
```

### Key Features

- **Plugin ecosystem** — over 100 plugins for authentication (OAuth2, OIDC, key-auth, LDAP), traffic control (rate limiting, request size limiting), logging (Datadog, Prometheus, Splunk), and transformation (request-transformer, response-transformer).
- **DB-less mode** — configure entirely via declarative YAML/JSON, suitable for GitOps workflows and [[Kubernetes for Data Workloads|Kubernetes]] deployments.
- **Kubernetes Ingress Controller** — deploy Kong as a K8s Ingress, managing routes via `Ingress` resources and plugins via CRDs.
- **Service mesh** — Kong Mesh (built on Envoy/Kuma) extends gateway patterns to east-west (service-to-service) traffic.

### Declarative Configuration Example

```yaml
_format_version: "3.0"
services:
  - name: data-api
    url: http://data-api.internal:8000
    routes:
      - name: data-api-route
        paths:
          - /api/v1/data
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
          redis_host: redis.internal
      - name: key-auth
        config:
          key_names: ["X-API-Key"]
```

---

## Azure API Management (APIM)

Azure API Management is a fully managed gateway with a developer portal, policy engine, and built-in analytics.

### Components

| Component | Purpose |
|-----------|---------|
| **Gateway** | Processes API calls, enforces policies, routes to backends |
| **Developer portal** | Auto-generated, customisable portal for API consumers to discover, test, and subscribe |
| **Management plane** | Azure portal / ARM templates / Bicep for configuration |
| **Policy engine** | XML-based policies applied at inbound, backend, outbound, and on-error stages |

### Policy Example

```xml
<policies>
    <inbound>
        <validate-jwt header-name="Authorization"
                      failed-validation-httpcode="401">
            <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration" />
            <required-claims>
                <claim name="aud" match="all">
                    <value>{api-client-id}</value>
                </claim>
            </required-claims>
        </validate-jwt>
        <rate-limit calls="100" renewal-period="60" />
        <set-header name="X-Request-Id" exists-action="skip">
            <value>@(context.RequestId.ToString())</value>
        </set-header>
    </inbound>
    <backend>
        <forward-request />
    </backend>
    <outbound>
        <set-header name="X-Powered-By" exists-action="delete" />
    </outbound>
</policies>
```

### Tiers

- **Consumption** — serverless, pay-per-call, no infrastructure. Cold start latency.
- **Developer** — non-production, no SLA. Good for development and testing.
- **Standard/Premium** — production workloads, VNet integration (Premium), multi-region (Premium).

---

## Authentication Patterns

### API Key Authentication

The simplest pattern — a static key passed in a header or query parameter:

```
GET /api/v1/data HTTP/1.1
X-API-Key: sk-abc123def456
```

- **Pros:** simple to implement, easy to rotate per client
- **Cons:** no identity context, key leakage risk, no expiry unless enforced externally
- **Best for:** internal services, low-sensitivity data endpoints, partner integrations with IP whitelisting

### OAuth 2.0

Delegated authorisation using access tokens issued by an authorisation server:

```
Client → Auth Server (request token with client_credentials)
       ← Access Token (JWT, 1-hour expiry)
Client → API Gateway (Authorization: Bearer <token>)
       → Gateway validates token → Backend
```

- **Client credentials grant** — service-to-service (no user context). Used for pipeline-to-API calls.
- **Authorisation code grant** — user-facing applications (dashboards, portals). Redirect-based flow.
- **Token introspection** — gateway calls the auth server to validate opaque tokens.

### JWT Validation

The gateway validates JSON Web Tokens without contacting the auth server on every request:

1. Extract the JWT from the `Authorization: Bearer` header
2. Verify the signature against the JWKS (JSON Web Key Set) endpoint
3. Check `exp` (expiry), `iss` (issuer), and `aud` (audience) claims
4. Optionally extract custom claims (roles, scopes) for fine-grained authorisation

```
Gateway checks:
  ✓ Signature valid (RS256 against JWKS)
  ✓ Token not expired (exp > now)
  ✓ Issuer matches (iss = auth.company.com)
  ✓ Audience matches (aud = data-api)
  ✓ Scopes include required scope (read:data)
```

---

## Rate Limiting Strategies

### Fixed Window

Count requests in fixed time intervals (e.g., 100 requests per minute). Simple but allows bursts at window boundaries — a client could send 100 requests at 12:00:59 and another 100 at 12:01:00.

### Sliding Window

Tracks requests over a rolling time window, smoothing out boundary bursts. More memory-intensive but fairer.

### Token Bucket

Each client has a "bucket" of tokens that refills at a steady rate. A request consumes one token. Allows controlled bursts (up to bucket capacity) whilst maintaining a long-term average rate.

### Leaky Bucket

Requests queue and are processed at a fixed rate. Excess requests are dropped. Produces a smooth, predictable output rate — useful for protecting backends with strict throughput limits.

### Tiered Rate Limits

Apply different limits based on client tier, endpoint sensitivity, or HTTP method:

```
Free tier:     100 requests/hour,  1,000 requests/day
Standard tier: 1,000 requests/hour, 10,000 requests/day
Enterprise:    10,000 requests/hour, unlimited daily

Write endpoints (POST/PUT/DELETE): 10% of read limits
Health checks (/health): exempt from limits
```

---

## Request/Response Transformation

API gateways can modify requests and responses in-flight, decoupling client expectations from backend implementations:

| Transformation | Example |
|----------------|---------|
| **Path rewriting** | `/api/v1/customers` → `/internal/customer-service/list` |
| **Header injection** | Add `X-Correlation-Id`, `X-Forwarded-For`, internal auth headers |
| **Header removal** | Strip `Server`, `X-Powered-By` from responses |
| **Body transformation** | Convert XML response to JSON, rename fields, filter sensitive attributes |
| **Protocol translation** | Accept REST, forward as gRPC to backend |
| **Aggregation** | Combine responses from multiple backends into a single response |
| **Versioning** | Route `/v1/` to legacy backend, `/v2/` to new backend |

---

## Monitoring and Observability

### Key Metrics

| Metric | What It Tells You |
|--------|-------------------|
| **Request rate** (requests/sec) | Traffic volume and trends |
| **Error rate** (4xx, 5xx percentage) | Client errors vs server errors |
| **Latency** (p50, p95, p99) | Response time distribution — p99 catches tail latency |
| **Throttled requests** (429 count) | Rate limit effectiveness and client impact |
| **Backend latency** | Time spent in backend vs gateway overhead |
| **Cache hit ratio** | Effectiveness of response caching |

### Distributed Tracing

Inject trace context headers (`traceparent`, `X-Request-Id`) at the gateway so requests can be traced end-to-end through backend services. Integrates with [[Kubernetes for Data Workloads|Prometheus]], Jaeger, Datadog, or AWS X-Ray.

### Logging Best Practices

- Log request metadata (method, path, status, latency, client ID) for every request
- Avoid logging request/response bodies by default (PII risk) — enable selectively for debugging
- Structure logs as JSON for downstream parsing (Fluent Bit, Elasticsearch)
- Correlate gateway logs with backend logs using a shared request ID

---

## Gateway Pattern Selection Guide

| Scenario | Recommended Approach |
|----------|---------------------|
| **AWS-native, serverless** | AWS HTTP API + Lambda authoriser |
| **AWS with advanced features** | AWS REST API (caching, WAF, usage plans) |
| **Kubernetes-native** | Kong Ingress Controller or Envoy-based (Istio, Emissary) |
| **Azure ecosystem** | Azure API Management (Standard/Premium) |
| **Multi-cloud or vendor-neutral** | Kong OSS or Kong Enterprise |
| **Real-time streaming** | AWS WebSocket API or custom WebSocket gateway |
| **Service mesh (east-west)** | Istio / Linkerd / Kong Mesh |

---

## Related Notes

- [[Model Context Protocol (MCP)]] — AI tool integration protocol for querying external systems
- [[Kubernetes for Data Workloads]] — deploying gateways and services on Kubernetes
- [[Terraform for Data Infrastructure]] — provisioning API gateway resources as code
- [[GitHub Actions for Python Projects]] — CI/CD for API deployments
