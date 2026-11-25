# TP Monitoring primer — observability and the stack

Background for [`../exercise.md`](../exercise.md): the three pillars of
observability, the six services of the stack, and what not to do. Read it once
before Step 1.

## What is observability?

**Observability is the ability to understand the internal state of a system
from its external outputs.** For a service, that means answering, at any
moment, three questions using three kinds of data:

| Question | Pillar | Data type | Backend (this TP) |
|---|---|---|---|
| *Is it fast? How much traffic? How many errors?* | **Metrics** | numbers over time | **Prometheus** |
| *Why did THIS request take long / fail?* | **Traces** | a chain of timed steps (spans) | **Tempo** |
| *What did the code actually log for that moment?* | **Logs** | text lines with labels | **Loki** |

Three objects you will meet again and again:

- **Metric** — a *number with labels*, sampled over time (gauge / counter /
  histogram). Example: `http_server_requests_seconds_count{uri="/listings", status="200"}`.
- **Span** — one timed segment of work; a **trace** is a tree of spans
  sharing the same `trace_id`.
- **Log line** — a text line with structured **labels** (`service`, `level`, …).

> **Mental model for the day:** the three pillars are *independent but
> correlated*. The key to correlation is that the **`trace_id`** that appears
> in your logs is the same one you can open a **trace** for in Tempo — and the
> **time** a metric spikes lines up with the logs and the slow trace. Your job
> is to make all three of those work for the listings API.

## The stack

The stack is provided in this folder as `docker-compose.yml`. It brings six
services, each with one job:

| Service | Image (version) | Host port | Job |
|---|---|---|---|
| Prometheus | `prom/prometheus:v2.53.1` | 9090 | store + query **metrics** |
| Loki | `grafana/loki:2.9.9` | 3100 | store + query **logs** |
| Tempo | `grafana/tempo:2.6.1` | 3200 | store + query **traces** |
| OTel Collector | `otel/opentelemetry-collector-contrib:0.108.0` | 4317 (gRPC), 4318 (HTTP) | receive **OTLP** spans from the app (HTTP, 4318), export to Tempo |
| Grafana Alloy | `grafana/alloy:v1.21.3` | 12345 (UI) | tail the app log file, ship to Loki |
| Grafana | `grafana/grafana:11.1.0` | 3000 | one UI over all three (Prometheus data source pre-provisioned; Tempo and Loki added in Steps 4-5) |

**Data flow** (this is exactly what you will make work in Steps 3-6) — three
independent pipelines from the same TP1 app, all read by Grafana:

| Pillar | Path | Delivery |
|---|---|---|
| **Metrics** | app `/actuator/prometheus` → **Prometheus** | **pulled** (scraped) |
| **Traces** | app → OTel Collector → **Tempo** | **pushed** (OTLP over HTTP) |
| **Logs** | app log file → **Alloy** → **Loki** | **pushed** (tailed) |

## What NOT to do

- Don't put a **business secret or PII** into a metric label or a log line
  (labels are high-cardinality; keep them stable and few).
- Don't log the whole request body at `DEBUG` on the hot path "to be safe" —
  it is a log-volume bomb and a privacy risk.
- Traces are data too: an HTTP client span records the full URL, query
  string included (an address, a search term). Know what your spans carry.
- Don't scrape the app for *logs* or *traces* — those are pushed/tailled,
  not scraped.
- Don't change the API's **status codes or JSON** to make a panel pretty.
