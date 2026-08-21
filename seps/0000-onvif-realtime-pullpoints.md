# SEP-{NUMBER}: Realtime PullPoints Event Queue for Streamable HTTP

> **Note**: This document follows the standard SEP structure for Extensions Track proposals in the Model Context Protocol.

- **Status**: Draft
- **Type**: Extensions Track
- **Created**: 2026-08-21
- **Author(s)**: Ottavio Campana <ottavio@campana.vi.it> (@ocampana), Willy Sagefalk <willy.sagefalk@axis.com> (@willysagefalk)
- **Sponsor**: None
- **PR**: https://github.com/modelcontextprotocol/modelcontextprotocol/pull/{NUMBER}

## Abstract

This SEP proposes a standardized PullPoint-style long-polling event queue architecture (`subscriptions/create` and `notifications/poll`) for the Model Context Protocol (MCP) Streamable HTTP transport. Inspired by the widely adopted ONVIF Realtime PullPoints specification, this pattern enables edge devices and servers operating behind firewalls, NATs, or air-gapped networks to deliver real-time asynchronous notifications to clients without requiring incoming webhook endpoints or relying on Server-Sent Events (SSE), which were deprecated in the MCP specification on 2026-07-28.

## Motivation

In many enterprise and industrial domains—such as physical security, edge video analytics, and smart infrastructure—MCP servers run directly on embedded hardware or within isolated local networks. These environments present specific connectivity constraints:

1. **Inbound Unaccessibility (Firewalls & NATs)**: Webhook-based event delivery requires the receiver (client) to expose an accessible HTTP server endpoint. In surveillance and IoT environments, clients (or central VMS systems) are frequently situated behind firewalls or strict NAT boundaries where inbound webhooks are impossible without risky network reconfigurations.
2. **Streamable HTTP & SSE Deprecation**: Following the deprecation of persistent Server-Sent Events (SSE) in the MCP specification on 2026-07-28, Streamable HTTP is strictly request-oriented. Without SSE streams or incoming webhook endpoints, servers have no mechanism to push unprompted notifications to clients over standard HTTP.
3. **ONVIF Compatibility**: For over 15 years, the ONVIF standard has relied on Realtime PullPoints as the foundational method for real-time event delivery across millions of camera streams. As ONVIF extends its scope to include AI and device management over MCP, the absence of a standardized pull-based real-time event mechanism in MCP acts as a primary blocker for adoption.

Adding a standardized Realtime PullPoint long-polling queue resolves these issues by utilizing outbound HTTP POST requests while delivering real-time latency.

## Specification

This proposal defines two new RPC methods under the `Extensions Track` for MCP Streamable HTTP transport: `subscriptions/create` and `notifications/poll`.

### 1. Allocation of Event Queue (`subscriptions/create`)

To receive asynchronous events, the client sends a `subscriptions/create` request to allocate a server-side event buffer.

#### Request Parameters
- `topics` (`array` of `string`, required): List of topic filters to subscribe to.
- `buffer_size` (`integer`, optional): Maximum number of unread events the server should hold (default: `100`).
- `ttl_ms` (`integer`, optional): Queue time-to-live / idle timeout in milliseconds (default: `60000`).

#### Request Example
```http
POST /mcp HTTP/1.1
Host: camera.local
Content-Type: application/json
Mcp-Method: subscriptions/create

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "subscriptions/create",
  "params": {
    "topics": ["ai/analytics/motion", "ai/analytics/object_detection"],
    "buffer_size": 50,
    "ttl_ms": 60000
  }
}
```

#### Response Example
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "subscription_id": "sub_01HXYZ89",
    "ttl_ms": 60000
  }
}
```

---

### 2. Long-Poll Event Fetching (`notifications/poll`)

The client issues a blocking HTTP request to pull events from the allocated queue.

#### Request Parameters
- `subscription_id` (`string`, required): Identifier returned by `subscriptions/create`.
- `timeout_ms` (`integer`, required): Maximum duration (in milliseconds) the server should hold the request open if no events are pending.
- `max_messages` (`integer`, optional): Maximum number of queued notifications to return in a single batch (default: `10`).

#### Request Example
```http
POST /mcp HTTP/1.1
Host: camera.local
Content-Type: application/json
Mcp-Method: notifications/poll

{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "notifications/poll",
  "params": {
    "subscription_id": "sub_01HXYZ89",
    "timeout_ms": 30000,
    "max_messages": 5
  }
}
```

#### Server Behavioral Requirements
1. **Immediate Delivery**: If events are already present in the queue when the request arrives, the server MUST return HTTP 200 OK immediately with the array of available notifications.
2. **Blocking Hold**: If the queue is empty, the server MUST hold the HTTP connection open until either:
   - An event matching the subscription topics occurs (responds immediately with HTTP 200 OK).
   - `timeout_ms` elapses without events (responds with HTTP 200 OK and an empty `notifications` array `[]`).
3. **Continuous Polling Loop**: Upon receiving a response (whether containing events or empty due to timeout), the client immediately issues the next `notifications/poll` request, forming a continuous polling loop.

#### Response Example (Event Triggered)
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "subscription_id": "sub_01HXYZ89",
    "notifications": [
      {
        "method": "notifications/ai/event",
        "params": {
          "topic": "ai/analytics/motion",
          "timestamp": "2026-08-21T06:10:00Z",
          "data": { "source": "cam_01", "state": "motion_detected" }
        }
      }
    ]
  }
}
```

#### Response Example (Timeout Expired)
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "subscription_id": "sub_01HXYZ89",
    "notifications": []
  }
}
```

### Error Handling

The server MUST return standard JSON-RPC error codes when handling subscription/polling errors:

| Error Code | Error Message | Condition |
| :--- | :--- | :--- |
| `-32602` | `Invalid params` | Unrecognized `subscription_id` or invalid `timeout_ms`. |
| `-32001` | `Subscription Expired` | The queue was destroyed due to idle timeout (`ttl_ms` exceeded without a poll). |
| `-32002` | `Buffer Overflow` | The event queue reached capacity and unread messages were dropped. |

---

## Rationale

The Realtime PullPoint pattern strikes an optimal balance between standard stateless HTTP semantics and real-time push performance:

- **Firewall & NAT Friendly**: All connections are client-initiated outbound HTTP POST requests. No client-side open ports or incoming Webhook endpoints are required.
- **Sub-Millisecond Realtime Latency**: Because the server holds the pending HTTP connection open, events are flushed down the wire instantly when generated.
- **Robust Against Intermediaries**: Unlike persistent SSE streams (deprecated in MCP on 2026-07-28) that frequently get killed by aggressive HTTP proxy idle timeouts, `notifications/poll` requests have deterministic maximum lifespans (e.g., 30 seconds).
- **Direct Alignment with ONVIF**: The protocol flow maps 1:1 onto ONVIF Realtime PullPoints (`CreatePullPointSubscription` / `PullMessages`), enabling seamless protocol translation layers between ONVIF and MCP ecosystems.

## Backward Compatibility

This SEP introduces no breaking changes to the core MCP specification. It is designed as an additive extension (`Extensions Track`). Standard MCP clients and servers that do not negotiate PullPoint support continue operating using standard request-response JSON-RPC interactions.

## Security Implications

- **Resource Management & Denial of Service**: Long-polling holds server sockets open. Servers MUST enforce maximum allowed `timeout_ms` (e.g., capping requests at 60,000ms) and enforce maximum queue sizes per subscription.
- **Authentication & Authorization**: `notifications/poll` requests MUST pass the same authentication/authorization headers (e.g., standard HTTP Bearer tokens) as other MCP methods.
- **Subscription Lifecycle & Cleanup**: Servers MUST maintain an idle timer based on `ttl_ms`. If a client fails to re-poll within `ttl_ms`, the subscription and its memory buffer MUST be garbage-collected.

## Reference Implementation

### Conceptual Client Polling Loop (Python Pseudocode)

```python
import httpx
import time

class MCPPullPointClient:
    def __init__(self, server_url, token):
        self.client = httpx.Client(base_url=server_url, headers={"Authorization": f"Bearer {token}"})
        self.sub_id = None

    def subscribe(self, topics):
        resp = self.client.post("/mcp", json={
            "jsonrpc": "2.0", "id": 1,
            "method": "subscriptions/create",
            "params": {"topics": topics}
        })
        self.sub_id = resp.json()["result"]["subscription_id"]

    def poll_loop(self):
        while True:
            try:
                resp = self.client.post("/mcp", json={
                    "jsonrpc": "2.0", "id": 2,
                    "method": "notifications/poll",
                    "params": {
                        "subscription_id": self.sub_id,
                        "timeout_ms": 30000,
                        "max_messages": 10
                    }
                }, timeout=35.0)

                result = resp.json().get("result", {})
                notifications = result.get("notifications", [])
                
                for notification in notifications:
                    self.handle_event(notification)
                    
            except httpx.ReadTimeout:
                # HTTP library timeout safeguard; loop continues
                continue
            except Exception as e:
                time.sleep(2) # Backoff on connection errors

    def handle_event(self, event):
        print(f"Received event: {event}")
```

## Performance Implications

- **Connection Overhead**: Long-polling involves re-establishing HTTP headers on each poll/timeout cycle. When combined with HTTP/2 or HTTP/3 connection reuse, this overhead is negligible.
- **Server Memory Usage**: Event buffers consume small amounts of RAM per subscription. Enforcing capped buffer sizes (e.g., 50–100 messages) guarantees bounded memory usage even under event bursts.

## Alternatives Considered

1. **Webhooks**: Rejected because edge clients in surveillance/IoT networks are almost universally behind NATs/firewalls without inbound public URLs.
2. **Persistent SSE (Server-Sent Events)**: Rejected due to its official deprecation in the MCP specification on 2026-07-28, as well as its unpredictable behavior through corporate proxies and stateful socket management challenges on resource-constrained microcontrollers.
