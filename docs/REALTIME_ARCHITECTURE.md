# Realtime Architecture

Fessor uses Django Channels and WebSockets to deliver streaming assistant responses and generation progress to the browser. This document describes how realtime delivery works, how the frontend consumes it, and how the system behaves under failure. For async job structure see [Async Workflows](ASYNC_WORKFLOWS.md). For overall request design see [System Design](SYSTEM_DESIGN.md).

---

## Overview

```text
Browser ←WebSocket→ Daphne (Channels) ←→ Redis channel layer
                              ↑
                    Django API / Celery workers publish events
```

REST remains the source of truth for durable data. WebSockets carry streaming tokens, progress notifications, and signals that prompt the client to refresh state.

---

## Django Channels

Channels extends Django to handle WebSocket connections alongside standard HTTP.

**Components**

- **Daphne** — ASGI server that serves HTTP and WebSocket traffic
- **Consumers** — Channel handlers that manage connect, disconnect, and message routing per logical channel (e.g., per book or per user session)
- **Channel layer** — Redis-backed pub/sub so events produced by any worker or API process reach the correct connected clients

This allows Celery workers running on separate machines to emit progress without holding open HTTP connections themselves.

---

## WebSockets

**Connection scope**

Clients subscribe to channels scoped to the resource they are viewing — typically a book generation job or an active reader session with the page assistant.

**Message types**

- Generation progress (outline ready, section complete, image attached, job finished)
- Assistant token stream (incremental response text)
- Assistant completion and error signals
- Heartbeat or keepalive where needed for long-lived connections

**Authentication**

Connections are established only for authenticated users. The consumer validates the session or token before joining a channel so users cannot subscribe to another account's generation stream.

---

## Assistant Streaming

The page assistant responds incrementally rather than waiting for the full completion of a long answer.

**Flow**

1. User sends a message in the assistant panel
2. API or consumer initiates a provider streaming call with section context
3. Tokens arrive from the provider over time
4. Each chunk is forwarded to the client over the WebSocket
5. On completion, the final message is persisted if the product stores conversation history

**Context**

The assistant is scoped to the active section. Context includes section content, learning objectives, and optionally attachment references the user is permitted to access.

**Edit workflows**

When the assistant performs an edit action, the stream may include tool or status events followed by a signal for the editor to reload or patch section content from the API.

---

## Progress Updates

During book generation, workers publish progress events as pipeline stages complete.

**Typical events**

- Planning finished — outline available
- Section N of M complete — reader table of contents updates
- Image attached to section
- Assessments generated
- Job complete or section-level failure

**Client behavior**

On progress events, the frontend updates local state or fetches the latest book record from the API. This hybrid approach keeps WebSocket payloads small and avoids duplicating full section bodies in every message.

---

## Frontend State Updates

The Next.js client maintains UI state driven by both REST and WebSocket input.

| Source | Updates |
|--------|---------|
| REST | Initial book load, section content, credit balance, editor saves |
| WebSocket | Streaming assistant text, generation progress badges, completion signals |

**Pattern**

1. Load durable state via REST on page mount
2. Open WebSocket subscription for live updates
3. Apply incremental UI changes from stream (assistant tokens, progress bars)
4. Re-fetch or patch from REST when a milestone indicates new persisted data

This keeps the UI responsive during long generations without requiring full page reloads.

---

## Failure and Reconnect Behavior

| Event | Behavior |
|-------|----------|
| WebSocket disconnect (network blip) | Client reconnects and resubscribes; assistant stream may truncate; user can resend message |
| Generation complete while disconnected | Client loads final state from REST on reconnect; book status in PostgreSQL is authoritative |
| Provider error mid-stream | Error event sent to client; partial assistant text may remain in UI; user can retry |
| Worker failure mid-generation | Progress stalls; async retry continues in background; reconnecting client receives later events when worker resumes |
| Auth expiry on socket | Connection closed; client redirects to login; no further events on old channel |

Clients should treat WebSocket data as ephemeral presentation state and always reconcile against PostgreSQL-backed API responses after reconnect.

---

## Production Scaling Considerations

**Multiple Daphne instances**

- WebSocket connections are sticky to a server process; load balancers must support WebSocket pass-through
- Redis channel layer broadcasts events to all Daphne instances so the correct consumer receives worker-published messages

**Redis channel layer capacity**

- Fan-out grows with connected clients and event frequency during active generation
- Monitor Redis memory and connection count as concurrent generation sessions increase

**Provider streaming limits**

- Assistant throughput is bounded by provider streaming rate and concurrent connection limits
- Rate limiting at the API protects against abuse and reduces risk of exhausting provider quotas

**Separation from workers**

- Workers publish to the channel layer rather than holding WebSockets open
- Worker fleet can scale on generation demand without matching WebSocket connection counts one-to-one

---

## Related Documents

- [System Design](SYSTEM_DESIGN.md)
- [Async Workflows](ASYNC_WORKFLOWS.md)
- [AI Pipeline](AI_PIPELINE.md)
- [Architecture](ARCHITECTURE.md)
