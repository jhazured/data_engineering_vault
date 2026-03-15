---
tags:
  - api
  - testing
  - devops
  - data-engineering
  - tooling
created: 2026-03-15
---

# Postman & API Testing

API testing is a critical practice in data engineering, where pipelines frequently depend on [[REST APIs]], webhooks, and third-party service endpoints. Postman provides a comprehensive platform for building, testing, and automating API interactions, making it an essential tool for validating data sources before committing them to production ingestion workflows.

---

## Postman Overview

Postman organises API work around three core concepts: **collections**, **environments**, and **variables**.

### Collections

A collection is a group of related API requests, stored together for organisation and reuse. Collections can be nested into folders, shared across teams, and versioned.

- Group requests by service (e.g., `Snowflake API`, `Fivetran API`, `dbt Cloud API`)
- Add collection-level authentication to apply credentials across all requests
- Export collections as JSON for version control alongside pipeline code

### Environments

Environments are sets of key-value pairs that allow switching context between deployment targets without modifying individual requests.

- Typical environments: `Development`, `Staging`, `Production`
- Each environment stores endpoint URLs, credentials, and configuration values
- Switch environments via the dropdown in the top-right corner of the Postman UI

### Variables

Variables follow a hierarchy of scope and precedence:

| Scope       | Lifetime            | Use Case                                      |
| ----------- | ------------------- | --------------------------------------------- |
| Global      | Across all requests | Shared constants, utility values               |
| Collection  | Within a collection | API version, base paths                        |
| Environment | Per environment     | Host URLs, credentials, tokens                 |
| Data        | Per iteration       | CSV/JSON row values in collection runner       |
| Local       | Single request      | Computed values in pre-request scripts         |

Precedence flows from narrowest to broadest: **Local > Data > Environment > Collection > Global**. Reference variables with double-brace syntax: `{{variable_name}}`.

---

## Request Building

### HTTP Methods

| Method   | Purpose                          | Typical Body |
| -------- | -------------------------------- | ------------ |
| `GET`    | Retrieve resources               | None         |
| `POST`   | Create resources or submit data  | JSON / Form  |
| `PUT`    | Replace a resource entirely      | JSON         |
| `PATCH`  | Partial update                   | JSON         |
| `DELETE` | Remove a resource                | None / JSON  |

### Headers

Common headers for data engineering API work:

```
Content-Type: application/json
Accept: application/json
X-Request-ID: {{$guid}}
Authorization: Bearer {{access_token}}
```

### Request Body

**Raw JSON** is the most common format for modern APIs:

```json
{
  "warehouse": "COMPUTE_WH",
  "database": "ANALYTICS",
  "schema": "PUBLIC",
  "statement": "SELECT * FROM raw.events LIMIT 10"
}
```

**Form-data** is used for file uploads and legacy APIs. **x-www-form-urlencoded** is typical for OAuth token exchanges.

### Authentication

Postman supports multiple authentication strategies, configurable at the request or collection level:

- **Bearer Token** -- paste or reference a `{{token}}` variable; Postman adds the `Authorization: Bearer <token>` header automatically
- **API Key** -- specify key name, value, and whether it is sent as a header or query parameter
- **OAuth 2.0** -- built-in flow for obtaining and refreshing tokens; supports authorisation code, client credentials, and PKCE grants
- **Basic Auth** -- username and password encoded as Base64; suitable for internal services and legacy endpoints

---

## Environment Management

### Dev/Staging/Prod Switching

Structure environments to mirror deployment tiers:

```
# Development
base_url: http://localhost:8080/api/v1
api_key: dev-key-xxxxx

# Staging
base_url: https://staging.example.com/api/v1
api_key: stg-key-xxxxx

# Production
base_url: https://api.example.com/api/v1
api_key: prod-key-xxxxx
```

Requests reference `{{base_url}}` and `{{api_key}}`, making environment switching seamless.

### Variable Inheritance

When the same variable name exists at multiple scopes, Postman resolves it using the precedence chain. This allows overriding a collection-level default with an environment-specific value without duplicating requests.

---

## Pre-Request Scripts

Pre-request scripts execute JavaScript before a request is sent. They are essential for dynamic authentication and parameterisation.

### Dynamic Variables

```javascript
// Generate a unique correlation ID
pm.variables.set("correlation_id", pm.variables.replaceIn("{{$guid}}"));

// Set current timestamp in ISO 8601
pm.variables.set("request_timestamp", new Date().toISOString());

// Generate a Unix epoch timestamp
pm.variables.set("epoch_ts", Math.floor(Date.now() / 1000));
```

### Token Refresh

Automatically refresh an expired OAuth token before each request:

```javascript
const tokenExpiry = pm.environment.get("token_expiry");
const now = Math.floor(Date.now() / 1000);

if (!tokenExpiry || now >= parseInt(tokenExpiry)) {
    pm.sendRequest({
        url: pm.environment.get("auth_url") + "/oauth/token",
        method: "POST",
        header: { "Content-Type": "application/x-www-form-urlencoded" },
        body: {
            mode: "urlencoded",
            urlencoded: [
                { key: "grant_type", value: "client_credentials" },
                { key: "client_id", value: pm.environment.get("client_id") },
                { key: "client_secret", value: pm.environment.get("client_secret") }
            ]
        }
    }, function (err, res) {
        const json = res.json();
        pm.environment.set("access_token", json.access_token);
        pm.environment.set("token_expiry", now + json.expires_in);
    });
}
```

---

## Test Scripts

Test scripts run after a response is received. They use the `pm.test()` assertion framework.

### Status Code Checks

```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Status code is in 2xx range", function () {
    pm.expect(pm.response.code).to.be.within(200, 299);
});
```

### Response Body Assertions

```javascript
pm.test("Response contains expected fields", function () {
    const json = pm.response.json();
    pm.expect(json).to.have.property("data");
    pm.expect(json.data).to.be.an("array").that.is.not.empty;
    pm.expect(json.data[0]).to.have.property("id");
});
```

### JSON Schema Validation

```javascript
const schema = {
    type: "object",
    required: ["status", "data"],
    properties: {
        status: { type: "string", enum: ["success"] },
        data: {
            type: "array",
            items: {
                type: "object",
                required: ["id", "created_at"],
                properties: {
                    id: { type: "integer" },
                    created_at: { type: "string", format: "date-time" }
                }
            }
        }
    }
};

pm.test("Response matches JSON schema", function () {
    pm.response.to.have.jsonSchema(schema);
});
```

### Response Time Assertions

```javascript
pm.test("Response time is below 500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});
```

---

## Collection Runner

The collection runner executes all requests in a collection sequentially, with support for iterations and data-driven testing.

### Batch Execution

- Run all requests in order, or filter by folder
- Set iteration count to repeat the full collection multiple times
- Add delays between requests to avoid rate limiting

### Data-Driven Testing With CSV/JSON

Provide an external data file to parameterise requests across iterations. Each row becomes one iteration.

**Example CSV** (`test_endpoints.csv`):

```csv
endpoint,expected_status,expected_field
/api/v1/sources,200,sources
/api/v1/connectors,200,connectors
/api/v1/destinations,200,destinations
```

Reference columns as `{{endpoint}}`, `{{expected_status}}`, and `{{expected_field}}` in requests and test scripts.

---

## Newman CLI

Newman is Postman's command-line collection runner. It enables integration of API tests into [[GitHub Actions for Python Projects]] and other CI/CD pipelines.

### Installation

```bash
npm install -g newman
npm install -g newman-reporter-htmlextra
```

### Basic Execution

```bash
newman run collection.json \
    --environment staging.json \
    --iteration-data test_data.csv \
    --reporters cli,htmlextra,junit \
    --reporter-htmlextra-export reports/api_test_report.html \
    --reporter-junit-export reports/junit_results.xml
```

### GitHub Actions Integration

```yaml
name: API Tests
on:
  schedule:
    - cron: "0 6 * * *"
  push:
    branches: [main]

jobs:
  api-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm install -g newman newman-reporter-htmlextra
      - run: |
          newman run postman/collection.json \
            --environment postman/staging.json \
            --reporters cli,htmlextra,junit \
            --reporter-htmlextra-export reports/report.html \
            --reporter-junit-export reports/results.xml
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: api-test-reports
          path: reports/
```

### GitLab CI Integration

```yaml
api_tests:
  image: node:20
  stage: test
  script:
    - npm install -g newman newman-reporter-htmlextra
    - newman run postman/collection.json
        --environment postman/staging.json
        --reporters cli,junit
        --reporter-junit-export results.xml
  artifacts:
    reports:
      junit: results.xml
```

---

## Mock Servers

Postman mock servers simulate API responses based on examples saved in a collection. This is valuable when:

- A data source API is not yet available but the schema is known
- Testing pipeline ingestion logic without hitting production endpoints
- Developing against rate-limited APIs

### Setup

1. Add example responses to each request in a collection
2. Create a mock server linked to the collection
3. Use the generated mock URL as `{{base_url}}` in a dedicated `Mock` environment
4. Postman matches incoming requests to saved examples by method, path, and headers

Mock servers return responses with realistic latency and support status code simulation for error-handling tests.

---

## Monitors

Monitors are scheduled collection runs hosted by Postman's cloud infrastructure.

- Schedule at intervals from every five minutes to weekly
- Receive alerts via email, Slack, or PagerDuty on test failures
- Track historical response times and error rates
- Useful for ongoing health checks of critical API data sources

### Configuration

- Select a collection and environment
- Set the schedule and region (to test from different geographies)
- Configure notification channels for failures
- Review run results in the Postman dashboard

---

## API Documentation

Postman auto-generates interactive documentation from collections:

- Each request becomes a documented endpoint with method, URL, headers, body, and example responses
- Markdown descriptions on collections, folders, and requests render as formatted text
- Published documentation is hosted at a shareable URL
- Supports custom domains for team-facing API documentation

This is particularly useful for documenting internal data platform APIs that serve analytics consumers.

---

## Data Engineering Use Cases

### Testing REST API Sources Before Building Ingestion Pipelines

Before writing a single line of pipeline code, use Postman to:

1. Explore the API's pagination strategy (offset, cursor, keyset)
2. Verify authentication and token lifecycle
3. Inspect response schemas and identify nested structures
4. Measure response times and rate limit headers
5. Save representative responses as examples for mock servers

This de-risks ingestion development and provides documentation for the team. See [[REST APIs]] for broader patterns.

### Validating Webhook Endpoints

Test webhook receivers by sending simulated payloads:

```json
{
  "event": "pipeline.completed",
  "pipeline_id": "pipe_abc123",
  "status": "success",
  "rows_processed": 150000,
  "completed_at": "2026-03-15T08:30:00Z"
}
```

Verify that the endpoint returns the correct status code, processes the payload, and triggers downstream actions.

### Snowflake REST API

Snowflake's SQL API enables statement execution over HTTP. Create a collection for common operations:

```
POST {{snowflake_url}}/api/v2/statements
Authorization: Bearer {{snowflake_jwt}}

{
  "statement": "SELECT COUNT(*) FROM raw.events WHERE event_date = CURRENT_DATE",
  "warehouse": "{{warehouse}}",
  "database": "{{database}}",
  "schema": "{{schema}}",
  "timeout": 60
}
```

Follow up with a `GET` to `/api/v2/statements/{{statement_handle}}` to poll for query completion and retrieve results.

### Fivetran/dbt Cloud API Testing

**Fivetran API** -- test connector status, trigger syncs, and verify sync history:

```
GET {{fivetran_url}}/v1/connectors/{{connector_id}}
Authorization: Basic {{fivetran_auth}}
```

**dbt Cloud API** -- trigger jobs, check run status, and retrieve artefacts:

```
POST {{dbt_cloud_url}}/api/v2/accounts/{{account_id}}/jobs/{{job_id}}/run/
Authorization: Token {{dbt_cloud_token}}

{ "cause": "Triggered via Postman" }
```

See [[API Gateway Patterns]] for managing access to these services centrally.

### Health Checks For Pipeline Endpoints

Create a dedicated health check collection that validates all critical endpoints:

```javascript
// Generic health check test script
pm.test("Endpoint is healthy", function () {
    pm.response.to.have.status(200);
});

pm.test("Response time is acceptable", function () {
    pm.expect(pm.response.responseTime).to.be.below(2000);
});

pm.test("Response body indicates healthy state", function () {
    const json = pm.response.json();
    pm.expect(json.status).to.be.oneOf(["healthy", "ok", "UP"]);
});
```

Schedule this collection as a monitor for continuous pipeline infrastructure validation.

---

## Postman Vs Alternatives

| Feature                  | Postman              | Insomnia             | HTTPie               | curl                 | Python requests      |
| ------------------------ | -------------------- | -------------------- | -------------------- | -------------------- | -------------------- |
| GUI                      | Yes                  | Yes                  | No (CLI)             | No (CLI)             | No (code)            |
| Collections              | Yes                  | Yes (workspaces)     | No                   | No                   | Manual               |
| Environments             | Yes                  | Yes                  | Sessions             | No                   | Manual               |
| Test Scripts             | Built-in (JS)       | Plugin-based         | No                   | No                   | pytest/unittest      |
| CI/CD Integration        | Newman               | Inso CLI             | Direct               | Direct               | Direct               |
| Mock Servers             | Built-in             | No                   | No                   | No                   | External libraries   |
| Monitors                 | Built-in             | No                   | No                   | cron + scripting     | cron + scripting     |
| API Documentation        | Auto-generated       | Limited              | No                   | No                   | Sphinx/MkDocs        |
| [[gRPC & GraphQL]]       | Yes                  | Yes                  | No                   | Limited              | Separate libraries   |
| Collaboration            | Team workspaces      | Git sync             | No                   | No                   | Version control      |
| Pricing (free tier)      | Generous             | Generous             | Free (OSS)           | Free (OSS)           | Free (OSS)           |
| Learning Curve           | Low                  | Low                  | Low                  | Medium               | Medium               |
| Scriptability            | JavaScript           | JavaScript (plugins) | Shell                | Shell                | Python               |
| Best For                 | Teams, full workflow | Lightweight GUI      | Quick CLI testing    | Scripting, automation| Pipeline integration |

**Recommendation for data engineers**: Use Postman for exploration and team collaboration during development. Use Python `requests` (or `httpx`) for production pipeline code. Use Newman in CI/CD to validate API contracts before deployment. Use `curl` or HTTPie for ad-hoc debugging in terminal sessions.

---

## See Also

- [[REST APIs]]
- [[gRPC & GraphQL]]
- [[API Gateway Patterns]]
- [[GitHub Actions for Python Projects]]
