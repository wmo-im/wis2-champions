# GRep Test Client

## Running and Refreshing

- **Running:** `docker compose up --build`
- **Refresh:** `docker rm mqtt-subscriber-web`
- **Web client:** http://localhost:2280

---

## Description

### MQTT Subscriber Web

GitHub repo: [6a6d74/mqtt-subscriber-web](https://github.com/6a6d74/mqtt-subscriber-web/tree/main)

A web application for testing WIS2 (World Meteorological Organization Information System) Global Broker coverage by comparing messages received live against those available via the GRep (Global Replay) service.

For more information see the project [README](https://github.com/6a6d74/mqtt-subscriber-web/blob/main/README.md).

### What it does

1. **Live subscription** — connects to one or more WIS2 Global Brokers via MQTT and collects messages for a user-defined duration
2. **Pubtime windowing** — filters messages by `properties.pubtime` (not wall-clock time) to handle broker relay lag; only messages within `[START_PUBTIME, END_PUBTIME]` are counted
3. **Deduplication** — unique messages are stored in Redis keyed by message ID
4. **GRep HTTP fetch** — after a configurable delay, queries the GRep service HTTP API for the same time window and topic to get a reference message count
5. **GRep MQTT replay** — triggers the GRep service to replay messages via MQTT, reconciles each replayed message against Redis, and reports any messages not received live (orphans)
6. **Summary** — prints a full reconciliation report to the event log and shows key metrics (received, GRep matched, coverage %) in the Results panel

---

## Architecture

| Layer | Technology |
|-------|-----------|
| Backend | Python / Flask 3.1, served by Gunicorn (1 worker, 8 threads) |
| Frontend | Vue 3 SPA (CDN, no build step), communicates via SSE + REST |
| Broker client | paho-mqtt, one thread per broker (up to 5 simultaneous) |
| State / dedup | Redis |
| Deployment | Docker Compose, linux/arm64, on jztnet bridge network |

Single Gunicorn worker is required — broker threads, SSE listener queues, and job state are all shared in-memory. SSE streams live events (messages, logs, tick countdowns) to the browser in real time.

---

## What is SSE?

Server-Sent Events — a browser standard for one-way push from server to client over a persistent HTTP connection.

The browser opens a single long-lived `GET /api/stream` request and the server keeps it open, writing event frames as data becomes available:

```
event: log
data: {"message": "Connected to broker"}

event: tick
data: {"remaining": 42.0}
```

The browser's `EventSource` API handles reconnection automatically if the connection drops.

**Why SSE here instead of WebSockets:**
- The browser only needs to receive events (broker messages, log lines, countdown ticks) — it never needs to push data back over the same channel
- SSE is simpler: plain HTTP, no upgrade handshake, works through proxies
- WebSockets would be overkill for this use case
