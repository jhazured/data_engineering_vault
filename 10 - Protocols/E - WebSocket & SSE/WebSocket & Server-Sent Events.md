**Tags:** #websocket #sse #real-time #protocols #streaming #data-engineering

# WebSocket & Server-Sent Events

Real-time communication protocols that go beyond the traditional HTTP request-response cycle. Both are essential for data engineering use cases where low-latency, push-based delivery is required.

## WebSocket Protocol

WebSocket (RFC 6455) provides full-duplex, bidirectional communication over a single, long-lived TCP connection. Once established, either side can send messages at any time without the overhead of repeated HTTP handshakes.

### Handshake

The connection begins as a standard HTTP request, then upgrades to the WebSocket protocol.

```
Client → Server:
  GET /ws/pipeline-status HTTP/1.1
  Host: api.example.com
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13

Server → Client:
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the `101 Switching Protocols` response, both sides communicate via WebSocket frames — no further HTTP overhead.

### Frame Structure

WebSocket data is transmitted in lightweight frames:

| Field | Size | Purpose |
|-------|------|---------|
| **FIN bit** | 1 bit | Indicates final fragment of a message |
| **Opcode** | 4 bits | Frame type: text (0x1), binary (0x2), close (0x8), ping (0x9), pong (0xA) |
| **Mask bit** | 1 bit | Client-to-server frames must be masked |
| **Payload length** | 7-64 bits | Variable-length encoding (up to ~16 EB theoretically) |
| **Payload** | Variable | The actual message data |

Text frames carry UTF-8 data; binary frames carry arbitrary bytes. Control frames (ping, pong, close) manage the connection lifecycle.

### Heartbeat (Ping/Pong)

Long-lived connections can silently drop due to intermediary proxies, NAT timeouts, or network issues. Ping/pong frames detect dead connections.

```
Server → Client:  PING (opcode 0x9)
Client → Server:  PONG (opcode 0xA)   ← automatic response

If no PONG received within timeout → close connection and reconnect
```

Best practice: send a ping every 30-60 seconds. Most WebSocket libraries handle pong responses automatically.

### Python WebSocket Server Example

```python
import asyncio
import websockets
import json

connected_clients: set = set()

async def handler(websocket):
    connected_clients.add(websocket)
    try:
        async for message in websocket:
            event = json.loads(message)
            # Broadcast to all connected clients
            for client in connected_clients:
                if client != websocket:
                    await client.send(json.dumps(event))
    finally:
        connected_clients.discard(websocket)

async def main():
    async with websockets.serve(handler, "0.0.0.0", 8765):
        await asyncio.Future()  # Run forever

asyncio.run(main())
```

### Python WebSocket Client Example

```python
import asyncio
import websockets
import json

async def listen():
    uri = "ws://localhost:8765/ws/pipeline-status"
    async with websockets.connect(uri) as ws:
        async for message in ws:
            event = json.loads(message)
            print(f"Pipeline {event['pipeline_id']}: {event['status']}")

asyncio.run(listen())
```

## Server-Sent Events (SSE)

SSE is a simpler, unidirectional protocol where the server pushes events to the client over a standard HTTP connection. The client uses the `EventSource` API (built into all modern browsers).

### How SSE Works

```
Client → Server:
  GET /events/pipeline-status HTTP/1.1
  Accept: text/event-stream

Server → Client (keeps connection open):
  data: {"pipeline_id": "etl-daily", "status": "running"}

  event: error
  data: {"pipeline_id": "etl-daily", "message": "Source timeout"}

  id: 42
  data: {"pipeline_id": "etl-daily", "status": "completed"}
  retry: 5000
```

### SSE Message Format

| Field | Purpose |
|-------|---------|
| `data:` | The event payload (can span multiple lines) |
| `event:` | Custom event type (default is `message`) |
| `id:` | Event ID for resumption after reconnection |
| `retry:` | Reconnection interval in milliseconds |

### Automatic Reconnection

SSE has built-in reconnection. If the connection drops, the browser automatically reconnects and sends the `Last-Event-ID` header so the server can resume from where the client left off.

```
Client reconnects:
  GET /events/pipeline-status HTTP/1.1
  Last-Event-ID: 42

Server resumes from event 43 onwards
```

### Python SSE Server Example (Flask)

```python
from flask import Flask, Response
import json
import time

app = Flask(__name__)

@app.route("/events/pipeline-status")
def stream():
    def generate():
        event_id = 0
        while True:
            status = get_pipeline_status()  # Your logic
            event_id += 1
            yield f"id: {event_id}\ndata: {json.dumps(status)}\n\n"
            time.sleep(2)

    return Response(generate(), mimetype="text/event-stream")
```

### JavaScript EventSource Client

```javascript
const source = new EventSource("/events/pipeline-status");

source.onmessage = (event) => {
    const data = JSON.parse(event.data);
    updateDashboard(data);
};

source.addEventListener("error", (event) => {
    const data = JSON.parse(event.data);
    showAlert(data.message);
});

source.onerror = () => {
    console.log("Connection lost — reconnecting automatically...");
};
```

## Protocol Comparison

| Aspect | WebSocket | SSE | HTTP Polling | gRPC Streaming |
|--------|-----------|-----|-------------|----------------|
| **Direction** | Bidirectional | Server-to-client only | Client-initiated | Bidirectional |
| **Transport** | TCP (upgraded from HTTP) | HTTP/1.1 or HTTP/2 | HTTP | HTTP/2 |
| **Data format** | Text or binary | Text (UTF-8) only | Any | Protocol Buffers (binary) |
| **Browser support** | All modern browsers | All modern browsers (no IE) | Universal | Requires grpc-web proxy |
| **Reconnection** | Manual (application logic) | Automatic (built-in) | N/A (each request is new) | Manual |
| **Overhead per message** | 2-14 bytes framing | ~10-50 bytes (SSE headers) | Full HTTP headers (~200+ bytes) | Low (HTTP/2 framing) |
| **Scaling** | Stateful connections (harder) | Stateful (but simpler) | Stateless (easiest) | Stateful connections |
| **Proxy/firewall** | May be blocked (non-HTTP) | Works through HTTP proxies | No issues | Requires HTTP/2 support |
| **Best for** | Chat, gaming, collaborative editing | Dashboards, notifications, logs | Simple integrations, batch checks | Microservice streaming, high-throughput |

## Data Engineering Use Cases

### Real-Time Dashboards

WebSocket or SSE connections push metric updates to monitoring dashboards without polling. This is the standard pattern for tools like [[Apache Kafka]] monitoring UIs, [[Airflow]] task status, and custom pipeline dashboards built with [[Streamlit]] or Grafana.

**Recommended protocol:** SSE — unidirectional server-to-client push is sufficient, and automatic reconnection simplifies client code.

### CDC Event Delivery

[[Change Data Capture]] events from databases (via Debezium, DMS, or custom triggers) can be forwarded to downstream consumers in near real-time.

```
Database → CDC tool → Kafka → WebSocket gateway → Dashboard / Alerting
```

**Recommended protocol:** WebSocket — bidirectional communication allows clients to subscribe to specific tables or filter criteria.

### Pipeline Status and Alerting

Orchestrators like [[Airflow]] or custom schedulers can push task-level status changes to a WebSocket/SSE endpoint, enabling:
- Live DAG progress visualisation
- Instant failure alerts without polling the Airflow API
- SLA breach notifications

### Event Sourcing Consumers

Applications following an [[Event-Driven Architecture]] pattern can expose an SSE endpoint that replays the event log from a given offset, enabling:
- Lightweight consumers that do not need a Kafka client
- Browser-based event stream inspectors for debugging

## When to Choose What

| Scenario | Recommended |
|----------|-------------|
| Server pushes updates, client only listens | **SSE** |
| Client and server both send messages | **WebSocket** |
| Simple integration, infrequent checks | **HTTP polling** |
| High-throughput microservice communication | **gRPC streaming** |
| Browser client with no build tooling | **SSE** (native EventSource API) |
| Binary data streaming | **WebSocket** or **gRPC** |

## Related Topics

- [[REST APIs]] — traditional request-response pattern, webhook patterns
- [[gRPC & GraphQL]] — binary streaming and schema-first APIs
- [[Apache Kafka]] — distributed event streaming at scale
- [[Event-Driven Architecture]] — broader architectural patterns
- [[Stream Processing Theory]] — windowing, watermarks, and processing guarantees
