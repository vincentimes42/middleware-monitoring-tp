# TP Monitoring / Observability — Progressive Implementation (8h)

## Overview

This is a **follow-up** to the Spring Boot TP (TP1, `../springboot/`). We
reuse the **TP1 listings API** — unchanged domain — and wire it into a
full **observability stack** so every request can be *measured*, *traced*,
and *logged* across the three pillars of observability.

For each step, write tests first, then implement until they pass. The tests
you write *are part of your deliverable*.

The specification describes **behaviors, contracts, and observable signals**
(what a panel reads, what an alert fires on, what a query returns). It does
**not** prescribe internal implementation beyond the required dependencies
and configuration keys.

> **Prerequisite: TP1 is done and green.** The TP1 API (`../springboot`)
> passes its own Steps 1-6 checklists (CRUD, ownership, versioning) and runs
> on **port `8081`**. If TP1 is not at that state, finish it first — it is
> an independent 8h TP. This TP only adds *how we observe the API*, not new
> business behaviour.
>
> This is an **alternative** to [TP Keycloak](../keycloak/) for the second TP.
> Do not run both stacks at the same time.

---

## What is observability? (one minute)

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
  histogram). Example: `http_server_requests_seconds_count{uri="/create", status="200"}`.
- **Span** — one timed segment of work; a **trace** is a tree of spans
  sharing the same `trace_id`.
- **Log line** — a text line with structured **labels** (`service`, `level`, …).

> **Mental model for the day:** the three pillars are *independent but
> correlated*. The key to correlation is that the **`trace_id`** that appears
> in your logs is the same one you can open a **trace** for in Tempo — and the
> **time** a metric spikes lines up with the logs and the slow trace. Your job
> is to make all three of those work for the listings API.

---

## Before you start — the stack

The stack is provided in this folder as `docker-compose.yml`. It brings six
services, each with one job:

| Service | Image (version) | Host port | Job |
|---|---|---|---|
| Prometheus | `prom/prometheus:v2.53.1` | 9090 | store + query **metrics** |
| Loki | `grafana/loki:2.9.9` | 3100 | store + query **logs** |
| Tempo | `grafana/tempo:2.6.1` | 3200 | store + query **traces** |
| OTel Collector | `otel/opentelemetry-collector-contrib:0.108.0` | 4317, 4318 | receive **OTLP** spans from the app, export to Tempo |
| Promtail | `grafana/promtail:2.9.9` | (internal) | tail the app log file, ship to Loki |
| Grafana | `grafana/grafana:11.1.0` | 3000 | one UI over all three (datasources pre-provisioned) |

**Data flow** (this is exactly what you will make work in Steps 3-6):

```
   /actuator/prometheus (scraped)                                       ┌──────────────────────┐
   ┌────────────┐  ──────────────────────────────────────►  Prometheus  │  metrics
   │            │                                                       │
   │   TP1      │  ──OTLP gRPC :4317──► OTel Collector ──► Tempo        │ traces
   │listings API│  ──log file (logs/listings.log)──► Promtail ──► Loki  │ logs
   │(Spring Boot│ 
   │ port 8081) │  Grafana (3000) reads all three ──► panels + alerts
   │            │
   │            │
   │            │
   │            │
   └────────────┘
                  
```

- **Metrics** are *pulled* (Prometheus scrapes the app).
- **Traces** and **logs** are *pushed/tailled* (OTel → Collector → Tempo;
  logback file → Promtail → Loki).

The **TP1 app runs on the host** (not in Docker) on port `8081`, so:

- the OTLP endpoint the app pushes to is `http://localhost:4317` (host-published
  by the Collector);
- `Prometheus` reaches the app via `host.docker.internal:8081` (already wired in
  `docker-compose.yml` + `prometheus/prometheus.yml`);
- `Promtail` tails the shared folder `./logs` (the app writes
  `logs/listings.log` there).

---

## Domain & data model (this TP)

The domain does **not** change from TP1: the two actors (Visitor, Seller),
the two entities (`listing`, `user`), 1-N ownership, active lifecycle, and
V1/V2 visibility. What changes is **only** the observation layer.

| Concern | TP1 | This TP |
|---|---|---|
| HTTP endpoints, payloads, status codes | as TP1 | **unchanged** |
| How we *measure* the API | (none / basic actuator) | Micrometer → **Prometheus** |
| How we *trace* a request | (none) | Micrometer / OTel → **Collector → Tempo** |
| How we *collect logs* | console only | logback file → **Promtail → Loki** |
| Where we *look* in an incident | `tail -f` + guesswork | **Grafana** dashboards + **Prometheus alerts** |

> **Key idea**: the API keeps behaving exactly as TP1. You are adding a
> *lens* to see, in real time, what is happening inside it — and you do it
> with **standard, open-source middleware**, not vendor code.

---

## Scope

| Track | Steps |
|---|---|
| **Core** (required — must fit in the 8h) | 1 · 2 · 3 · 4 · 5 · 6 |
| **Bonus** (start only when the core checklists are green) | 7 (correlation & first SLO) · 8 (two services / a slow dependency) |
| **Delivery** | 9 (final integration) |

Each core step's "Acceptance checklist" is your "definition of done".
Bonus steps have their own acceptance checklists; doing them is optional
but counts for the grade if you deliver them.

---

## Conventions

- **Service name** used everywhere (labels, tags, panel queries): `listings-api`.
- **Ports** (host): Grafana `3000`, Prometheus `9090`, Loki `3100`, Tempo `3200`,
  OTLP `4317`/`4318`, app `8081`.
- **Metrics**: exposed by the app on `/actuator/prometheus` and scraped by
  Prometheus; all carry the `service="listings-api"` label.
- **Traces**: spans are exported over **OTLP gRPC** to `localhost:4317`;
  `service.name=listings-api`.
- **Logs**: the app writes a file `logs/listings.log`; Promtail ships it to
  Loki with labels `service="listings-api"` and `level` derived from the line.
- **Query languages**: PromQL (metrics), **TraceQL** (traces), **LogQL** (logs).
- Error JSON shape of the **API** is the same as TP1 — monitoring never changes
  the API's response format.

---

## Conventions — how to "write tests first" here

The possible ways of "testing" are:

| Level | What you assert | How |
|---|---|---|
| **Query** | a PromQL / TraceQL / LogQL expression returns the expected data | run the `curl` in `docs/verification.md` |
| **Panel** | a Grafana panel renders a sensible value after real traffic | screenshot / visual check + the panel query string |
| **Alert** | a Prometheus rule **fires** under the condition you built | `curl :9090/api/v1/rules` shows `state: firing` |
| **Unit (TP1)** | a custom metric (e.g. `listings_created_total`) increments per call | a JUnit test asserting the `MeterRegistry` counter value |

> **Rule:** you may not *pass* a step by only running the app. You must be able
> to show the *data* in the backend (metrics / trace / logs) for the scenario
> the step names.

---

### What NOT to do

- Don't put a **business secret or PII** into a metric label or a log line
  (labels are high-cardinality; keep them stable and few).
- Don't log the whole request body at `DEBUG` on the hot path "to be safe" —
  it is a log-volume bomb and a privacy risk.
- Don't scrape the app for *logs* or *traces* — those are pushed/tailled,
  not scraped.
- Don't change the API's **status codes or JSON** to make a panel pretty.

---

## Step 1 - Bring up the observable stack `CORE`

> **User story** — *As a **developer**, I want the full observability
> middleware to be **running and reachable**, so that I can instrument the
> API against **real backends** (Prometheus, Loki, Tempo, Grafana) in the
> following steps.*

### Behavior

| # | Requirement |
|---|---|
| 1.1 | `docker compose up -d` starts **six** services, all reachable via their health endpoint. |
| 1.2 | Grafana reports `health` green on its API. |
| 1.3 | Prometheus, Loki and Tempo each answer their readiness endpoint. |
| 1.4 | The OTel Collector's **OTLP gRPC** listener is reachable on host port `4317`. |

### Acceptance checklist

- [ ] `docker compose ps` shows `mon-prometheus`, `mon-loki`, `mon-tempo`,
      `mon-otel-collector`, `mon-promtail`, `mon-grafana` all `Up`
- [ ] `GET http://localhost:9090/-/healthy` → `Prometheus Server is Healthy.`
- [ ] `GET http://localhost:3100/ready` → `ready`
- [ ] `GET http://localhost:3200/ready` → `ready`
- [ ] Grafana at <http://localhost:3000> answers `GET /api/health` with `database: ok`
      (login `admin` / `admin`)
- [ ] You can open <http://localhost:3000> and Grafana is not "no data sources"

### Hints

- Start in `monitoring/`: `docker compose up -d`, then `docker compose ps`.
- `docker compose logs -f <service>` to watch a service start.
- All health endpoints are curlable without auth — no login needed for the
  backend readiness checks.

---

## Step 2 - Understand the three pillars (read the backends, don't trust) `CORE`

> **User story** — *As a **developer**, I want to **query each backend directly**
> and confirm it stores and returns data, so that I **know each pipeline is
> real** before I rely on the Grafana panels.*

This step is **read-only on the backends**: you confirm that each of the three
stores can hold and return a signal, using the *empty* stack, before wiring
the app in.

### Behavior

| # | Requirement |
|---|---|
| 2.1 | Prometheus can **store and query** a series: query it and see its own `up` / `prometheus_...` metrics. |
| 2.2 | Loki can **store and query** a stream: after injecting one line, `LogQL` returns it. |
| 2.3 | Tempo can **store and return** a trace search shape: a TraceQL/HTTP search returns a valid (possibly empty) structure. |

### Acceptance checklist

- [ ] `curl http://localhost:9090/api/v1/query --data-urlencode "query=up"` returns
      a JSON `result` with an `up` series (value `1`)
- [ ] You can push one log line through **Promtail** (see the app in Step 4, or the
      one-liner in `docs/verification.md`) and `LogQL` in the Loki query API returns it
- [ ] A Tempo search API call (last 10 min) returns a valid JSON object with a
      `traces` key (it may legitimately be empty)
- [ ] You can name (in your own words) the three query languages: **PromQL**,
      **LogQL**, **TraceQL**

### Hints

- Full, copy-pasteable `curl` recipes for each backend:
  [`docs/verification.md`](docs/verification.md) §2, §4, §3.
- Loki and Tempo "empty but valid" is the correct starting state — a stack with
  no data still returns well-formed JSON. This is the point: **the contract is
  real, even before the app is connected**.

---

## Step 3 - Expose metrics → Prometheus → Grafana `CORE`

> **User story** — *As a **platform developer**, I want the API to **expose
> HTTP + one business metric** as Prometheus metrics, **and I want Prometheus to
> scrape them and Grafana to show them**, so that I can build **capacity and
> error panels** and later **alerts** on top.*

This step is **split in two places** and they must both be done:

| Where | What to do |
|---|---|
| **TP1** (`../springboot`) | add the `prometheus` registry, expose `/actuator/prometheus`, record one **business** `Counter` |
| **This repo** (`monitoring/`) | write the **Prometheus scrape job** in `prometheus/prometheus.yml` |

### Changes to make in the TP1 (`../springboot`)

| What | From (TP1) | To (this step) |
|---|---|---|
| `pom.xml` | `spring-boot-starter-actuator` present | **add** `micrometer-registry-prometheus` |
| `application.yml` | `management.endpoints...` | ensure `/actuator/prometheus` is **exposed** + `service: listings-api` tag |
| Business metric | (none) | record a **Counter** (e.g. `listings_created_total`) in the create path |

### Changes to make in this repo (`monitoring/`)

| File | From (skeleton) | To (this step) |
|---|---|---|
| `prometheus/prometheus.yml` | self-scrape only | **add** the `listings-api` scrape job (see the hint in the file) |

### Behavior

| # | Requirement |
|---|---|
| 3.1 | `GET http://localhost:8081/actuator/prometheus` returns Prometheus **text exposition** format (`# HELP` / `# TYPE` lines). |
| 3.2 | The default HTTP metrics (`http_server_requests_seconds_*`) are present, tagged with `service="listings-api"`, `uri=`, `status=`. |
| 3.3 | A **business metric** `listings_created_total{service="listings-api"}` **increases by one per successful `POST /listings`**. |
| 3.4 | Prometheus **scrapes** the app: the series are visible via the Prometheus query API (not only on the app). |

### Acceptance checklist

- [ ] `curl -s http://localhost:8081/actuator/prometheus` shows `# HELP http_server_requests_seconds`
- [ ] After 1 `POST /listings`, `listings_created_total` value is **1 higher** than before the call
- [ ] `curl -s "http://localhost:9090/api/v1/query" --data-urlencode "query=listings_created_total{service="listings-api"}"` returns a value (i.e. **Prometheus now has the series** — the scrape job you added works)
- [ ] You created **at least one Grafana panel yourself** that plots request rate or total requests (not a pre-shipped panel)

### Hints

- Dependency: `io.micrometer:micrometer-registry-prometheus` (version-managed by Boot).
- In `application.yml` keep the existing `management.metrics.tags.service` and add
  `management.endpoints.web.exposure.include: ... prometheus`.
- A clean way to record the business counter: inject `MeterRegistry`, build
  `Counter.builder("listings_created_total").tag("service","listings-api").register(...)`.
- In `prometheus/prometheus.yml` you **add** the `listings-api` scrape job —
  the skeleton already shows the exact keys to set. To reload after editing:
  `curl -X POST http://localhost:9090/-/reload` (or `docker compose restart prometheus`).
 - Build your first panel in Grafana (Datasources → Prometheus, then
  "Add a panel") with a PromQL query like `sum(rate(http_server_requests_seconds_count{service="listings-api"}[1m]))`.

---

## Step 4 - Wire distributed tracing → Collector → Tempo `CORE`

> **User story** — *As a **platform developer**, I want each request to be
> **traced across the API**, with a span for the business operation, so that
> I can **open any slow request in Tempo** and see *exactly which operation
> took the time*.*

This step is **split in two places** and both must be done:

| Where | What to do |
|---|---|
| **TP1** (`../springboot`) | add tracing deps + a `Tracer` usage + an OTel SDK that exports OTLP to `localhost:4317` |
| **This repo** (`monitoring/`) | point the **Collector** at **Tempo** (`otel/collector-config.yaml`) |

### Changes to make in the TP1 (`../springboot`)

| What | From (TP1) | To (this step) |
|---|---|---|
| `pom.xml` | (no tracing) | **add** `micrometer-tracing-bridge-otel` + the OTel **Spring Boot starter** (auto-configures the OTLP exporter) |
| `application.yml` | (none) | `otel.sdk.*` → OTLP **gRPC** endpoint `http://localhost:4317`, `service.name: listings-api`, sampling 100% |
| Business code | (none) | record an explicit span (e.g. `listings.create`) on the create path via `Tracer` |

> The exact dependency set and the `otel.*` block are already used by a
> reference that boots: see the comment in `docs/verification.md` §"Tracing".

### Changes to make in this repo (`monitoring/`)

| File | From (skeleton) | To (this step) |
|---|---|---|
| `otel/collector-config.yaml` | receiver + `debug` sink | **add** the `otlp/tempo` **exporter** and put it on the traces pipeline |

### Behavior

| # | Requirement |
|---|---|
| 4.1 | A `GET /create` produces a **span named `listings.create`** (not only the auto-generated HTTP span). |
| 4.2 | That span is **stored in Tempo**: `curl .../api/traces/<traceId>` returns the span; and a Tempo **search** (last 10 min, `service.name=listings-api`) returns traces. |
| 4.3 | Each trace keeps a **`trace_id`** that is stable for the whole request. |

### Acceptance checklist

- [ ] `curl -s "http://localhost:3200/api/search" (last 10min, tags service.name=listings-api)` returns **≥ 1 trace** after you hit `/create`
- [ ] Opening one traceId in Tempo shows a span named **`listings.create`**
- [ ] In the TP1 code you created an explicit span (not only relying on the auto HTTP span)

### Hints

- The **OTel Spring Boot starter** (`opentelemetry-spring-boot-starter`) reads
  `otel.sdk.service.name` and `otel.sdk.exporter.otlp.endpoint` — set the
  protocol to `grpc` and the endpoint to `http://localhost:4317`.
- `micrometer-tracing-bridge-otel` + a `Tracer` bean gives you
  `Tracer.nextSpan().name("listings.create").start() … .end()`.
- In `otel/collector-config.yaml`, add an exporter
  (`otlp/tempo: { endpoint: "tempo:4317" }`) and reference it in the
  `traces` pipeline. `curl -X POST http://localhost:3200/api/search` is your
  verification (`docs/verification.md` §3 / "Traces").
- The Tempo backend is already up — **you only wire the Collector to it**.

---

## Step 5 - Collect logs → Promtail → Loki (with trace_id) `CORE`

> **User story** — *As an **operations engineer**, I want the API's **log
> lines** to be searchable in Loki, tagged with `service` and `level`, and I
> want each line to carry the **same `trace_id`** as its trace, so that I can
> **jump from a trace to its logs**.*

Split in two places:

| Where | What to do |
|---|---|
| **TP1** (`../springboot`) | add a **file appender** that writes to `logs/listings.log`, with `trace_id`/`span_id` in the log pattern |
| **This repo** (`monitoring/`) | make **Promtail** tail `logs/listings.log` → Loki (`promtail/promtail-config.yaml`) + add the **Loki** data source in Grafana |

### Changes to make in the TP1 (`../springboot`)

| What | From (TP1) | To (this step) |
|---|---|---|
| logging | console only | a **`FileAppender`** that writes `<repo-root>/logs/listings.log` (create the folder) |
| log pattern | (TP1) | include `trace=%X{traceId}` and `span=%X{spanId}` (these are populated by Micrometer tracing → MDC) and the `level` |

> **Important:** the log **file path is fixed** to `logs/listings.log` because
> that is what Promtail is configured to tail (see `docker-compose.yml`).

### Changes to make in this repo (`monitoring/`)

| File | From (skeleton) | To (this step) |
|---|---|---|
| `promtail/promtail-config.yaml` | no `listings` job | **add** the `listings` job: `__path__: /var/logs/listings.log`, `service` label, derive `level` from the line |
| `grafana/.../datasources.yml` | Prometheus only | **add** the Loki data source (`type: loki`, `url: http://loki:3100`) |

### Behavior

| # | Requirement |
|---|---|
| 5.1 | A request's log line is **stored in Loki**: the Loki query API for `service="listings-api"` returns it. |
| 5.2 | The line carries the **same `trace_id`** as the trace you looked at in Step 4 (correlation). |
| 5.3 | The line has a **`level`** label (e.g. `INFO`, `ERROR`) that can be filtered: `{service="listings-api"} |= "ERROR"`. |

### Acceptance checklist

- [ ] `curl -s "http://localhost:3100/loki/api/v1/query_range" --data-urlencode "query={service=\"listings-api\"}"` returns **≥ 1 line** after some traffic
- [ ] You can filter by level: add `| level=ERROR` (or `|= "ERROR"`) and an `ERROR` line from your app shows up
- [ ] The `trace_id` in the log line matches a real trace in Tempo (Step 4) — paste it into the Tempo API and it resolves

### Hints

- A Logback `FileAppender` with `<file>logs/listings.log</file>` under a
  pattern that includes `%X{traceId}` and `%X{spanId}` (Micrometer tracing
  writes these to the MDC — this is the **correlation** link).
- In `promtail/promtail-config.yaml`, the skeleton already shows the exact keys:
  `__path__: /var/logs/listings.log` plus a `regex` + `labels` pipeline stage
  to extract `level`.
- Add the **Loki** data source in Grafana (provisioning file **or** the UI).
- Verification: `docs/verification.md` §4 (Logs).

---

## Step 6 - Dashboards & alerts (the "look" of an incident) `CORE`

> **User story** — *As an **on-call engineer**, I want **one Grafana dashboard**
> that gives me the three golden signals + logs + traces, and I want a
> **Prometheus alert** to **fire** when the API is degraded or down, so that
> I **detect an incident without someone telling me**.*

This step is mostly in **this repo** (Grafana + Prometheus) — it is where the
three pillars you wired in Steps 3-5 **come together**.

| Where | What to do |
|---|---|
| **This repo** (`monitoring/`) | add the **Tempo + Loki** data sources, build a **dashboard** (golden signals + logs + traces), and write the **alert rules** |

### Behavior

| # | Requirement |
|---|---|
| 6.1 | A **Grafana dashboard** (that you build) shows: **request rate**, **p95 latency**, **error rate** (Prometheus); a **logs** panel (Loki) and a **traces** panel (Tempo). |
| 6.2 | Three alert rules exist and are **correct**: **api-down** (scrape fails), **high-5xx** (error rate spike), **high-p95** (latency above a threshold). |
| 6.3 | You can **prove at least one alert fires**: `curl :9090/api/v1/rules` shows a rule with `state: firing` under the condition. |

### Acceptance checklist

- [ ] A **single dashboard** you created shows all three metrics + the logs panel + the traces panel
- [ ] `curl -s http://localhost:9090/api/v1/rules` lists **3** rules for `listings-api`
- [ ] You trigger a real condition (e.g. 5xx from the app, or stop the scrape) and the matching rule's `state` becomes **`firing`**
- [ ] A log line with `level=ERROR` is visible in the dashboard's logs panel

### Hints

- Add the **Tempo** and **Loki** data sources (file or UI). Tempo → `http://tempo:3200`, Loki → `http://loki:3100`.
- Build the dashboard in Grafana (one, e.g. "Listings API") — 5-6 panels; pick the right data source per panel (Prometheus / Loki / Tempo).
- Alert rules go in `prometheus/rules.yml` (skeleton shows the SHAPe). Use the
  `http_server_requests_seconds_count` / `_bucket` metrics (the ones you saw in Step 3) and the `up` metric.
- To make an alert fire in the lab: call a failing endpoint (or generate a 5xx),
  or `docker compose stop listings` to test the `api-down` rule.
- Verification: `docs/verification.md` §6 (Alerts).

---

## Step 7 - Correlation & a first SLO `BONUS`

> **User story** — *As a **platform engineer**, I want to **jump from a slow
> trace to its logs and the metric window** around it, and I want one **SLO
> (e.g. 99% success, 30d-style)** computed from the metrics, so that I can
> answer "is the API meeting its commitment?" — not just "is it up?"*

### Behavior

| # | Requirement |
|---|---|
| 7.1 | From a Tempo trace, you **open the matching Loki logs** (via the `trace_id`) — demonstrate the link. |
| 7.2 | An **SLO panel**: availability / success-ratio computed from `http_server_requests_seconds_count` (e.g. 1 - (5xx / total)). |

### Acceptance checklist

- [ ] A Tempo → Loki "traces-to-logs" jump works (same `trace_id`)
- [ ] A Grafana panel shows a **success-rate / SLO** value between 0 and 1

### Hints

- Grafana Tempo datasource has a "traces to logs" mapping (see the Tempo datasource
  `jsonData.tracesToLogsV2` reference in `docs/verification.md`).
- SLO ratio PromQL: `1 - (sum(rate(http_server_requests_seconds_count{status=~"5.."}[24h])) / sum(rate(http_server_requests_seconds_count[24h])))`.

---

## Step 8 - Two services + a "slow" dependency `BONUS`

> **User story** — *As a **platform engineer**, I want the trace to **show a
> call to a second service** (or a deliberately slow step), so that I can
> **see the latency split** between the API and the callee — the classic reason
> we trace instead of only counting requests.*

### Behavior

| # | Requirement |
|---|---|
| 8.1 | A request spans **two services** (e.g. `listings-api` → `geo-service` or an in-app downstream step) and the Tempo trace shows both spans linked by the same `trace_id`. |
| 8.2 | You can **attribute** part of the total latency to the child span. |

### Acceptance checklist

- [ ] A Tempo trace shows **2+ spans** (parent `listings.create` + child) sharing one `trace_id`
- [ ] You can name which span took most of the time

### Hints

- The simplest "second service" is an **in-app downstream call** (a `RestTemplate`/`WebClient`
  to another endpoint that is also traced), or a real 2nd Spring Boot app in Docker
  that shares the same OTLP endpoint and `traceparent` propagation. Keep it small.

---

## Step 9 - Final Integration (delivery)

| # | Requirement |
|---|---|
| 9.1 | `docker compose up -d` → all six services healthy; the stack boots from the **student-filled** configs (no solution files present). |
| 9.2 | TP1 (`../springboot`) builds and runs on `8081`; generating traffic yields **metrics in Prometheus, traces in Tempo, logs in Loki**. |
| 9.3 | **One** Grafana dashboard shows the three pillars; **three** alert rules exist and at least one has been shown firing. |
| 9.4 | Your **config files you authored** (prometheus scrape/rules, collector, promtail, data sources) are in the repo; the **solutions** are **not** in the repo. |
| 9.5 | README documents: how to start the stack, the three pipelines, the ports, and one alert + one dashboard screenshot. |
| 9.6 | Oral (10 min + 5 min Q&A): *walk one slow request from the Grafana panel → the Tempo trace → the Loki log*, explaining which middleware produced each part. |

---

## Deliverables

| Item | Detail |
|---|---|
| Git repository (pair) | TP1 repo modified **+** this `monitoring/` folder with **your** pipeline configs |
| `docker compose up` | 6 services healthy from **student-filled** configs |
| `mvn verify` (TP1) | builds; instrumentation wired |
| Grafana | one dashboard, 3 data sources, 3 alerts (1 demonstrably firing) |
| README.md | stack run, the 3 pipelines, ports, 1 alert + 1 dashboard |

---

## Appendix A - The three pipelines at a glance

| Pillar | App produces | Middleware | Backend | Query lang |
|---|---|---|---|---|
| Metrics | `/actuator/prometheus` | **Prometheus scrape** | Prometheus | **PromQL** |
| Traces | OTLP gRPC (`localhost:4317`) | **OTel Collector** | Tempo | **TraceQL** / HTTP API |
| Logs | file `logs/listings.log` | **Promtail** | Loki | **LogQL** |

## Appendix B - Useful endpoints

| URL | Purpose |
|---|---|
| `http://localhost:9090/api/v1/query` | PromQL query |
| `http://localhost:9090/api/v1/rules` | alert rules + state |
| `http://localhost:9090/api/v1/targets` | scrape targets |
| `http://localhost:3100/loki/api/v1/query_range` | LogQL query |
| `http://localhost:3200/api/search` | Tempo trace search |
| `http://localhost:3200/api/traces/{traceId}` | one trace |
| `http://localhost:8081/actuator/prometheus` | app metrics (scrape source) |
| `http://localhost:3000` | Grafana (admin/admin) |

## Appendix C - What is NOT to be committed

- The **reference solutions** (live in `../../solutions/monitoring/`, used by the
  supervisor) must **not** be copied into this repo.
- `logs/listings.log` (runtime data) — it is gitignored.

