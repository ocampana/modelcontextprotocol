# SEP-{NUMBER}: Pull-Based Event Delivery for MCP Streamable HTTP (`subscriptions/poll`)

> **Note**: This document follows the standard SEP structure for Extensions Track proposals in the Model Context Protocol.

- **Status**: Draft
- **Type**: Extensions Track
- **Created**: 2026-08-21
- **Author(s)**: Ottavio Campana <ottavio@campana.vi.it> (@ocampana), Willy Sagefalk <willy.sagefalk@axis.com> (@willysagefalk)
- **Sponsor**: None
- **PR**: https://github.com/modelcontextprotocol/modelcontextprotocol/pull/{NUMBER}

## Abstract

This SEP proposes a standardized bounded long-polling mechanism (`subscriptions/create` and `subscriptions/poll`) for consuming MCP subscriptions over Streamable HTTP, as a complementary consumption mode alongside the long-lived SSE stream obtained via `subscriptions/listen`. Inspired by the widely adopted ONVIF Realtime PullPoints specification, this pattern lets edge devices and servers operating behind firewalls, NATs, or in networks with strict intermediary-imposed connection lifetimes receive near-real-time notifications using only bounded, client-initiated HTTP exchanges — without requiring an indefinitely open streaming response or an inbound webhook endpoint.

## Motivation

In many enterprise and industrial domains — such as physical security, edge video analytics, and smart infrastructure — MCP servers run directly on embedded hardware or within isolated local networks. These environments present connectivity constraints that are not fully addressed by streaming-based event delivery alone:

1. **Bounded connection lifetimes.** MCP 2026-07-28 provides `subscriptions/listen` for long-lived server-to-client notifications delivered over a request-scoped SSE response. However, many industrial, surveillance, and IoT deployments prohibit inbound connections to clients or central systems, and the intermediaries typically present in these networks — industrial firewalls, VPN concentrators, managed proxies — frequently impose maximum connection lifetimes that are incompatible with an indefinitely open HTTP response.
2. **Constrained embedded clients.** Holding open a streaming HTTP response and incrementally parsing an SSE event stream carries non-trivial state-management cost on resource-constrained microcontrollers and embedded Linux devices. Some of these implementations are already built around bounded, sequential HTTP request/response cycles, and a polling model fits that shape more naturally than a persistent stream.
3. **ONVIF compatibility.** For over 15 years, the ONVIF standard has relied on Realtime PullPoints as the foundational method for event delivery across millions of camera streams. As ONVIF extends its scope to include AI and device management over MCP, a standardized pull-based event-consumption mechanism in MCP would let ONVIF PullPoint semantics map directly onto MCP, easing protocol-translation and adoption.

This SEP does not claim that MCP lacks a mechanism for asynchronous event delivery, and it does not attempt to replace server-initiated callback/webhook delivery, which is a separate pattern being investigated by the Triggers & Events WG for deployments where the server can reach the client (or a central callback endpoint) directly. Instead, it standardizes a second, client-initiated consumption mode for the existing MCP subscription model — bounded long polling — for deployments where an indefinitely open streaming response is undesirable or impossible:

```
                    Event delivery

           Client initiated          Server initiated
                  |                         |
        +---------+---------+            callback /
        |                   |             webhook
       SSE               long poll
  subscriptions/       subscriptions/
      listen                poll
```

## Specification

This proposal defines three new RPC methods under the `Extensions Track` for MCP Streamable HTTP transport: `subscriptions/create`, `subscriptions/poll`, and `subscriptions/delete`.

### Relationship to `subscriptions/listen` and `tasks/get`

MCP 2026-07-28 removes protocol-level session state, but explicitly supports applications minting explicit handles that are carried across subsequent requests to represent server-side state — the pattern `tasks/get` uses for task handles. This SEP applies the same principle to event delivery: `subscriptions/create` mints an explicit subscription handle (`subscriptionId`), and `subscriptions/poll` is a bounded, repeatable request that carries that handle to retrieve queued events — mirroring how `tasks/get` repeatedly polls task state using a task handle rather than requiring a persistent stream.

| | Handle | Consumption | Connection shape |
|---|---|---|---|
| Tasks | task handle | `tasks/get` | bounded request/response |
| Subscriptions (stream) | subscription handle | `subscriptions/listen` | long-lived SSE response |
| Subscriptions (poll) | subscription handle | `subscriptions/poll` | bounded long poll |

The subscription itself — its topic filters and its event queue — is server-side application state represented by an explicit handle, not implicit protocol/session state. `subscriptions/listen` and `subscriptions/poll` are two different ways of consuming the same underlying subscription.

### 1. Allocation of Event Queue (`subscriptions/create`)

To receive asynchronous events, the client sends a `subscriptions/create` request to allocate a server-side event buffer.

#### Request Parameters
- `topics` (`array` of `string`, required): List of topic filters to subscribe to.
- `bufferSize` (`integer`, optional): Maximum number of unacknowledged events the server should retain (default: `100`).
- `ttlMs` (`integer`, optional): Queue idle timeout in milliseconds — the queue is discarded if no `subscriptions/poll` request arrives within this window (default: `60000`).

#### Request Example
```http
POST /mcp HTTP/1.1
Host: camera.local
Content-Type: application/json
MCP-Protocol-Version: 2026-07-28
Mcp-Method: subscriptions/create
Accept: application/json, text/event-stream

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "subscriptions/create",
  "params": {
    "topics": ["ai/analytics/motion", "ai/analytics/object_detection"],
    "bufferSize": 50,
    "ttlMs": 60000
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
    "resultType": "complete",
    "subscriptionId": "sub_01HXYZ89",
    "ttlMs": 60000
  }
}
```

---

### 2. Bounded Long-Poll Event Fetching (`subscriptions/poll`)

The client issues a bounded HTTP request to pull events from the allocated queue.

#### Request Parameters
- `subscriptionId` (`string`, required): Handle returned by `subscriptions/create`.
- `timeoutMs` (`integer`, required): Maximum duration the server should hold the request open if no events are pending. Servers MUST cap accepted values (e.g. at 60,000 ms) and MUST reject larger values with `-32602`.
- `maxEvents` (`integer`, optional): Maximum number of events to return in a single batch (default: `10`).
- `afterSequence` (`integer`, optional): Return only events with `sequence` strictly greater than this value. If omitted, defaults to the server-tracked cursor for this subscription (see **Delivery Semantics** below), or `0` on the first poll.

#### Request Example
```http
POST /mcp HTTP/1.1
Host: camera.local
Content-Type: application/json
MCP-Protocol-Version: 2026-07-28
Mcp-Method: subscriptions/poll
Accept: application/json, text/event-stream

{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "subscriptions/poll",
  "params": {
    "subscriptionId": "sub_01HXYZ89",
    "timeoutMs": 30000,
    "maxEvents": 5,
    "afterSequence": 471
  }
}
```

#### Server Behavioral Requirements
1. **Immediate delivery.** If events with `sequence > afterSequence` are already present in the queue when the request arrives, the server MUST return HTTP 200 OK immediately with the array of available events.
2. **Bounded hold.** If no such events are present, the server MUST hold the HTTP connection open until either an event matching the subscription topics occurs, or `timeoutMs` elapses, whichever is first — responding with HTTP 200 OK in both cases (with an empty `events` array on timeout).
3. **Single outstanding poll.** A server MUST NOT process more than one outstanding `subscriptions/poll` request per `subscriptionId` at a time. If a second poll for the same subscription arrives while one is still pending, the server MUST fail the newer request immediately with error `-32003` (`Poll Already Pending`).
4. **Continuous polling loop.** Upon receiving a response (whether containing events or empty due to timeout), the client immediately issues the next `subscriptions/poll` request with `afterSequence` set to the highest `sequence` it received, forming a continuous polling loop.

#### Response Example (Event Triggered)
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "resultType": "complete",
    "subscriptionId": "sub_01HXYZ89",
    "events": [
      {
        "sequence": 472,
        "type": "ai/analytics/motion",
        "timestamp": "2026-08-21T06:10:00Z",
        "data": { "source": "cam_01", "state": "motion_detected" }
      }
    ],
    "nextCursor": 472
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
    "resultType": "complete",
    "subscriptionId": "sub_01HXYZ89",
    "events": [],
    "nextCursor": 471
  }
}
```

#### Response Example (Buffer Overflow, Reported In-Band)
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "resultType": "complete",
    "subscriptionId": "sub_01HXYZ89",
    "events": [
      { "sequence": 151, "type": "ai/analytics/motion", "timestamp": "2026-08-21T06:12:03Z", "data": {} }
    ],
    "nextCursor": 151,
    "overflow": {
      "dropped": true,
      "firstAvailableSequence": 151
    }
  }
}
```

---

### 3. Explicit Subscription Teardown (`subscriptions/delete`)

Clients that no longer need a subscription SHOULD release it explicitly rather than relying solely on `ttlMs` expiry, so the server can free the buffer promptly.

#### Request Parameters
- `subscriptionId` (`string`, required): Handle returned by `subscriptions/create`.

#### Response Example
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "resultType": "complete",
    "subscriptionId": "sub_01HXYZ89"
  }
}
```

If a subscription is never explicitly deleted, the server MUST discard it once `ttlMs` has elapsed without a `subscriptions/poll` request.

---

### Delivery Semantics

This mechanism provides **at-least-once, in-order delivery per subscription**, using a monotonically increasing per-subscription `sequence` rather than destructive dequeue on send:

- Each event is assigned a `sequence` integer, strictly increasing within a given `subscriptionId`, starting at `1`.
- A `subscriptions/poll` response is **not** an acknowledgement by itself. The server retains delivered events in the buffer until the client's *next* poll carries an `afterSequence` at or beyond that event's `sequence`, at which point the server may prune it. This means:
  - If an HTTP response is generated but never received by the client (connection dropped, proxy timeout, client crash), the client's retry — using the same `afterSequence` it last knew about — simply receives the same events again. Consumers MUST treat event handling as idempotent, keyed by `sequence`.
  - If the client crashes after advancing its cursor but before finishing local processing, it is the client's responsibility to persist `nextCursor` if it needs durability across restarts; this SEP does not mandate server-side persistence beyond the buffer/TTL window.
- If the requested `afterSequence` is lower than the oldest event still retained (because older events were evicted to respect `bufferSize`), the server MUST resume from the oldest available event and MUST set `overflow.dropped: true` with `overflow.firstAvailableSequence` set accordingly, so the client can detect and account for the gap. This is reported in-band on an otherwise-successful response; it is not, by itself, an RPC error.
- Ordering is guaranteed only within a single subscription, not across subscriptions or across topics within a subscription.

### Error Handling

The server MUST return standard JSON-RPC error codes for the following conditions. Buffer overflow is reported in-band (see **Delivery Semantics**) rather than as an error, since a subscription that has dropped some events is still usable.

| Error Code | Error Message | Condition |
| :--- | :--- | :--- |
| `-32602` | `Invalid params` | Unrecognized `subscriptionId`, or `timeoutMs`/`afterSequence` outside accepted bounds. |
| `-32001` | `Subscription Expired` | The queue was discarded because `ttlMs` elapsed without a poll. |
| `-32003` | `Poll Already Pending` | A `subscriptions/poll` request was received for a `subscriptionId` that already has an outstanding poll in flight. |

### Extension Negotiation

Support for this mechanism MUST be negotiated using MCP's extension framework rather than assumed. Servers advertise support during initialization using a reverse-DNS extension identifier:

```json
{
  "capabilities": {
    "extensions": {
      "io.modelcontextprotocol/event-polling": {
        "maxPollTimeoutMs": 60000,
        "maxBufferSize": 1000
      }
    }
  }
}
```

Clients MUST NOT invoke `subscriptions/create`, `subscriptions/poll`, or `subscriptions/delete` unless the server has advertised the `io.modelcontextprotocol/event-polling` extension (identifier subject to confirmation with the Triggers & Events WG, and may be namespaced as experimental during incubation). `maxPollTimeoutMs` and `maxBufferSize` advertise the server's enforced caps for `timeoutMs` and `bufferSize`/`maxEvents` respectively.

## Rationale

The pull-based pattern is a deliberate complement to `subscriptions/listen`, not a replacement for it:

- **Firewall- and NAT-friendly.** All connections are client-initiated outbound HTTP POST requests. No client-side open ports or inbound webhook endpoints are required, which matters for many industrial, surveillance, and IoT deployments that prohibit inbound connections to clients or central systems.
- **Bounded connection lifetime.** Because `subscriptions/poll` requests have a deterministic maximum lifespan (e.g. capped at 60 seconds), they tolerate intermediaries that impose strict idle or maximum-duration limits on individual HTTP exchanges, which can be a poor fit for an indefinitely open SSE response in some managed networks.
- **Low added latency.** Events can be returned immediately over an already-pending poll request, avoiding the latency of periodic short-interval polling, without any normative claim about absolute wire latency across arbitrary network stacks.
- **Consistent with the Tasks pattern.** `subscriptions/poll` applies the same architectural principle MCP 2026-07-28 already uses for `tasks/get`: represent server-side state with an explicit handle, and let the client retrieve updates via bounded, repeatable requests instead of a persistent stream.
- **Direct alignment with ONVIF.** The protocol flow maps closely onto ONVIF Realtime PullPoints (`CreatePullPointSubscription` / `PullMessages`), enabling straightforward protocol-translation layers between ONVIF and MCP ecosystems. This is supporting evidence for the design, not the primary justification for it.

| Mechanism | Direction | Connection | Good fit |
|---|---|---|---|
| `subscriptions/listen` | server → client | long-lived SSE response | general MCP clients |
| `subscriptions/poll` | client → server | bounded long poll | edge/industrial/firewall-constrained systems |
| callback/webhook (Triggers & Events WG) | server → callback endpoint | new outbound request | cloud/event-driven systems |

## Backward Compatibility

This SEP introduces no breaking changes to the core MCP specification. It is an additive extension (`Extensions Track`), gated behind the `io.modelcontextprotocol/event-polling` capability described in **Extension Negotiation**. Standard MCP clients and servers that do not negotiate this extension continue operating using `subscriptions/listen` or ordinary request-response JSON-RPC interactions, unaffected.

## Security Implications

- **Resource management and denial of service.** Bounded long-polling still holds server sockets open for the duration of `timeoutMs`. Servers MUST enforce a maximum accepted `timeoutMs` (advertised via `maxPollTimeoutMs`) and a maximum buffer size per subscription (advertised via `maxBufferSize`).
- **Authentication and authorization.** `subscriptions/poll` and `subscriptions/delete` requests MUST pass the same authentication/authorization headers (e.g. standard HTTP Bearer tokens) as other MCP methods, re-checked on every request rather than only at `subscriptions/create` time.
- **Subscription lifecycle and cleanup.** Servers MUST maintain an idle timer based on `ttlMs`. If a client fails to re-poll within `ttlMs`, the subscription and its buffer MUST be garbage-collected.
- **Idempotent consumers.** Because delivery is at-least-once (see **Delivery Semantics**), downstream event handling MUST be idempotent with respect to `sequence`; a security-relevant action (e.g. triggering an alarm workflow) must not be safely triggerable twice by a duplicate delivery.

## Reference Implementation

### Conceptual Client Polling Loop (Python Pseudocode)

```python
import httpx
import time

class MCPPullPointClient:
    def __init__(self, server_url, token):
        self.client = httpx.Client(base_url=server_url, headers={"Authorization": f"Bearer {token}"})
        self.sub_id = None
        self.cursor = 0

    def subscribe(self, topics):
        resp = self.client.post("/mcp", json={
            "jsonrpc": "2.0", "id": 1,
            "method": "subscriptions/create",
            "params": {"topics": topics}
        })
        self.sub_id = resp.json()["result"]["subscriptionId"]

    def poll_loop(self):
        while True:
            try:
                resp = self.client.post("/mcp", json={
                    "jsonrpc": "2.0", "id": 2,
                    "method": "subscriptions/poll",
                    "params": {
                        "subscriptionId": self.sub_id,
                        "timeoutMs": 30000,
                        "maxEvents": 10,
                        "afterSequence": self.cursor
                    }
                }, timeout=35.0)

                result = resp.json().get("result", {})
                events = result.get("events", [])

                for event in events:
                    self.handle_event(event)

                # Advance the cursor only after successful local processing,
                # so a lost response or crash simply re-delivers the same events.
                self.cursor = result.get("nextCursor", self.cursor)

            except httpx.ReadTimeout:
                # HTTP library timeout safeguard; loop continues with the same cursor
                continue
            except Exception:
                time.sleep(2)  # Backoff on connection errors; cursor is unchanged

    def handle_event(self, event):
        print(f"Received event: {event}")
```

## Performance Implications

- **Connection overhead.** Bounded long-polling involves re-establishing HTTP headers on each poll/timeout cycle. Combined with HTTP/2 or HTTP/3 connection reuse, this overhead is small.
- **Server memory usage.** Event buffers consume modest RAM per subscription. Enforcing capped buffer sizes (advertised via `maxBufferSize`) guarantees bounded memory usage even under event bursts, at the cost of the in-band overflow reporting described above.

## Alternatives Considered

1. **Webhooks.** Not pursued for this SEP because many industrial, surveillance, and IoT deployments prohibit inbound connections to clients or central systems; this deployment constraint, not its universality, is the relevant fact. Server-initiated callback delivery remains a valid complementary pattern and is in scope for the Triggers & Events WG's own work.
2. **`subscriptions/listen` as the only consumption mode.** `subscriptions/listen`'s long-lived SSE response is the right default for general MCP clients and is not being challenged by this SEP. It is, however, a poor fit for deployments where intermediaries impose strict maximum connection lifetimes, or where an embedded client is not well suited to maintaining an open streaming response. `subscriptions/poll` addresses that specific gap as a second consumption mode over the same subscription model, not a replacement.
3. **Unbounded short-interval polling of a plain resource.** Rejected as a baseline pattern because fixed-interval polling without a bounded hold either wastes requests when idle or adds up to one full interval of latency on events; the bounded-hold design in `subscriptions/poll` avoids both while keeping connection lifetimes deterministic.
