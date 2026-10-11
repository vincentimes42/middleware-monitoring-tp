# TP Monitoring / Observability — Progressive Implementation (8h)

## Overview

This is a **follow-up** to the Spring Boot TP (TP1, `../springboot/`). We
reuse the **TP1 listings API** — unchanged domain — and wire it into a
full **observability stack** so every request can be *measured*, *traced*,
and *logged* across the three pillars of observability.

You do **not** write Java tests for this TP — you **prove each step with real
data in the backends** (a Prometheus query returns a value, a Grafana panel
renders, an alert rule fires). See "Conventions — how to validate each step"
below for the exact acceptance method per step. That evidence *is* your
deliverable.

The specification describes **behaviors, contracts, and observable signals**
(what a panel reads, what an alert fires on, what a query returns). It does
**not** prescribe internal implementation beyond the required dependencies
and configuration keys.

> **Prerequisite: TP1 is done and green.** The TP1 API (`../springboot`)
> passes its core checklists (Steps 1·2·3·5·6·8: register, CRUD, ownership,
> `?version=` V1/V2, metrics) and runs
> on **port `8080`** (unchanged — the app is on `8080` for every TP, even
> when the other follow-up, Keycloak, is active; Keycloak itself moves to
> `8081` in that case). If TP1 is not at that state, finish it first — it is
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
  histogram). Example: `http_server_requests_seconds_count{uri="/listings", status="200"}`.
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

**Data flow** (this is exactly what you will make work in Steps 3-6) — three
independent pipelines from the same TP1 app, all read by Grafana:

| Pillar | Path | Delivery |
|---|---|---|
| **Metrics** | app `/actuator/prometheus` → **Prometheus** | **pulled** (scraped) |
| **Traces** | app → OTel Collector → **Tempo** | **pushed** (OTLP gRPC) |
| **Logs** | app log file → **Promtail** → **Loki** | **pushed** (tailed) |

The **TP1 app is a container on the shared `middleware` network** (base
`../springboot/docker-compose.yml`), so it is reached by **container name**,
not `localhost`:

- the app pushes OTLP gRPC to `otel-collector:4317`;
- `Prometheus` scrapes the app at `listings-api:8080`;
- `Promtail` tails the log file on the shared volume `middleware-logs`
  (the app writes it there via `LOG_FILE`).

---

## Domain & data model (this TP)

The domain does **not** change from TP1 (actors, entities, ownership, V1/V2
visibility, endpoints, payloads). What changes is **only** the observation
layer:

| Concern | TP1 | This TP |
|---|---|---|
| How we *measure* the API | (none / basic actuator) | Micrometer → **Prometheus** |
| How we *trace* a request | (none) | OTel → **Collector → Tempo** |
| How we *collect logs* | console only | logback file → **Promtail → Loki** |
| Where we *look* in an incident | `tail -f` + guesswork | **Grafana** dashboards + **Prometheus alerts** |

> **Key idea**: the API keeps behaving exactly as TP1. You are adding a
> *lens* to see what is happening inside it — with **standard, open-source
> middleware**, not vendor code.

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
  OTLP `4317`/`4318`, app `8080` (this TP).
- **Metrics**: exposed by the app on `/actuator/prometheus` and scraped by
  Prometheus at `listings-api:8080`; all carry the `service="listings-api"` label.
- **Traces**: spans are exported over **OTLP gRPC** to `otel-collector:4317`;
  `service.name=listings-api` (that's the **Tempo** label key).
- **Logs**: the app writes the file `/listings-logs/listings.log` on the shared
  volume `middleware-logs`; Promtail tails it and ships to Loki with labels
  `service="listings-api"` and `level` derived from the line.
- **Query languages**: PromQL (metrics), **TraceQL** (traces), **LogQL** (logs).
- Error JSON shape of the **API** is the same as TP1 — monitoring never changes
  the API's response format.

> **Label naming — don't mix these up:** the *same* service shows up as
> **different label keys** in the three backends. This is the classic beginner
> gotcha when building panels:
>
> | Backend | Label key for the service |
> |---|---|
> | **Prometheus** (scrape job) | `job="listings-api"` or `service="listings-api"` (depending on your scrape config) |
> | **Loki** (stream labels) | `service="listings-api"` |
> | **Tempo** (TraceQL / tags) | **`service.name="listings-api"`** |
>
> So a Tempo search tag is `"service.name"`, a Loki selector is `{service="listings-api"}`,
> and a PromQL series is `…{service="listings-api"}` (or `…{job="listings-api"}`).
> They all refer to the **same app**.


---

## Conventions — how to validate each step

Each step's acceptance is **data you can show in a backend**, not a Java test
suite. The three ways:

| Level | What you assert | How |
|---|---|---|
| **Query** | a PromQL / TraceQL / LogQL expression returns the expected data | run the `curl` in `docs/verification.md` |
| **Panel** | a Grafana panel renders a sensible value after real traffic | screenshot / visual check + the panel query string |
| **Alert** | a Prometheus rule **fires** under the condition you built | `curl :9090/api/v1/rules` shows `state: firing` |

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
- [ ] You can push one log line directly to the Loki **push API** (the one-liner in
      `docs/verification.md` §4) and the Loki query API (`{service="listings-api"}`)
      returns it — this proves Loki stores and returns a stream **before the app is wired**
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
| 3.1 | `GET http://localhost:8080/actuator/prometheus` returns Prometheus **text exposition** format (`# HELP` / `# TYPE` lines). |
| 3.2 | The default HTTP metrics (`http_server_requests_seconds_*`) are present, tagged with `service="listings-api"`, `uri=`, `status=`. |
| 3.3 | A **business metric** `listings_created_total{service="listings-api"}` **increases by one per successful `POST /listings`**. |
| 3.4 | Prometheus **scrapes** the app: the series are visible via the Prometheus query API (not only on the app). |

### Acceptance checklist

- [ ] `curl -s http://localhost:8080/actuator/prometheus` shows `# HELP http_server_requests_seconds`
- [ ] After 1 `POST /listings`, `listings_created_total` value is **1 higher** than before the call
- [ ] `curl -s "http://localhost:9090/api/v1/query" --data-urlencode "query=listings_created_total{service="listings-api"}"` returns a value (i.e. **Prometheus now has the series** — the scrape job you added works)

### Hints

- Dependency: `io.micrometer:micrometer-registry-prometheus` (version-managed by Boot).
- In `application.yml` keep the existing `management.metrics.tags.service` and add
  `management.endpoints.web.exposure.include: ... prometheus`.
- A clean way to record the business counter: inject `MeterRegistry`, build
  `Counter.builder("listings_created_total").tag("service","listings-api").register(...)`.
- In `prometheus/prometheus.yml` you **add** the `listings-api` scrape job —
  the skeleton already shows the exact keys to set. To reload after editing:
  `curl -X POST http://localhost:9090/-/reload` (or `docker compose restart prometheus`).
- (Panel-building happens in **Step 6** — here you only prove the series is scraped
  by asking Prometheus directly, not Grafana.)

---

## Step 4 - Wire distributed tracing → Collector → Tempo `CORE`

> **User story** — *As a **platform developer**, I want each request to be
> **traced across the API**, with a span for the business operation, so that
> I can **open any slow request in Tempo** and see *exactly which operation
> took the time*.*

This step is **split in three places** and all three must be done:

| Where | What to do |
|---|---|
| **TP1** (`../springboot`) | add tracing deps + a `Tracer` usage + an OTel SDK that exports OTLP to `otel-collector:4317` |
| **This repo** (`monitoring/`) — Collector | point the **Collector** at **Tempo** (`otel/collector-config.yaml`) |
| **This repo** (`monitoring/`) — Grafana | add the **Tempo** data source (`grafana/provisioning/datasources/datasources.yml`, or in the Grafana UI) |

### Changes to make in the TP1 (`../springboot`)

| What | From (TP1) | To (this step) |
|---|---|---|
| `pom.xml` | (no tracing) | **add** `micrometer-tracing-bridge-otel` + OpenTelemetry **Java agent / Spring Boot starter** |
| `docker-compose` of the base (or `application.yml`) | (none) | set **OTEL_EXPORTER_OTLP_ENDPOINT** = `http://otel-collector:4317` and **OTEL_SERVICE_NAME** = `listings-api` (env vars), sampling 100% |
| Business code | (none) | record an explicit span (e.g. `listings.create`) on the create path via `Tracer` |

> The exact dependency set and the OTel env vars are documented in
> **`../monitoring/README.md` § "0) Required base edit"**, which is the
> canonical place for what `docker-compose` needs to add on `listings-api`.

### Changes to make in this repo (`monitoring/`)

| File | From (skeleton) | To (this step) |
|---|---|---|
| `otel/collector-config.yaml` | receiver + `debug` sink | **add** the `otlp/tempo` **exporter** and put it on the traces pipeline |
| `grafana/provisioning/datasources/datasources.yml` | Prometheus only | **add** the Tempo data source (`type: tempo`, `url: http://tempo:3200`) |

### Behavior

| # | Requirement |
|---|---|
| 4.1 | A `POST /listings` (create a listing) produces a **span named `listings.create`** (not only the auto-generated HTTP span). |
| 4.2 | That span is **stored in Tempo**: `curl .../api/traces/<traceId>` returns the span; and a Tempo **search** (last 10 min, `service.name=listings-api`) returns traces. |
| 4.3 | Each trace keeps a **`trace_id`** that is stable for the whole request. |

### Acceptance checklist

- [ ] `curl -s "http://localhost:3200/api/search" (last 10min, tags service.name=listings-api)` returns **≥ 1 trace** after you create a listing (`POST /listings`)
- [ ] Opening one traceId in Tempo shows a span named **`listings.create`**
- [ ] In the TP1 code you created an explicit span (not only relying on the auto HTTP span)

### Hints

- The **OTel Spring Boot starter** (`opentelemetry-spring-boot-starter`) reads
  the standard OTel env vars: set **`OTEL_SERVICE_NAME`** to `listings-api`
  and **`OTEL_EXPORTER_OTLP_ENDPOINT`** to `http://otel-collector:4317`
  (see `../monitoring/README.md` § "0) Required base edit" for the canonical
  form to put in `docker-compose.yml`).
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
| **TP1** (`../springboot`) | add a **file appender** that writes to `/listings-logs/listings.log`, with `trace_id`/`span_id` in the log pattern |
| **This repo** (`monitoring/`) | make **Promtail** tail `/listings-logs/listings.log` → Loki (`promtail/promtail-config.yaml`) + add the **Loki** data source in Grafana |

### Changes to make in the TP1 (`../springboot`)

| What | From (TP1) | To (this step) |
|---|---|---|
| logging | console only | a **`FileAppender`** that writes `/listings-logs/listings.log` (create the folder) |
| log pattern | (TP1) | include `trace=%X{traceId}` and `span=%X{spanId}` (these are populated by Micrometer tracing → MDC) and the `level` |

> **Important:** the log **file path is fixed** to `/listings-logs/listings.log` because
> that is what Promtail is configured to tail (see `docker-compose.yml`).

### Changes to make in this repo (`monitoring/`)

| File | From (skeleton) | To (this step) |
|---|---|---|
| `promtail/promtail-config.yaml` | no `listings` job | **add** the `listings` job: `__path__: /listings-logs/listings.log`, `service` label, derive `level` from the line |
| `grafana/provisioning/datasources/datasources.yml` | Prometheus only | **add** the Loki data source (`type: loki`, `url: http://loki:3100`) |

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

- A Logback `FileAppender` with `<file>/listings-logs/listings.log</file>` under a
  pattern that includes `%X{traceId}` and `%X{spanId}` (Micrometer tracing
  writes these to the MDC — this is the **correlation** link).
- In `promtail/promtail-config.yaml`, the skeleton already shows the exact keys:
  `__path__: /listings-logs/listings.log` plus a `regex` + `labels` pipeline stage
  to extract `level`.
- Add the **Loki** data source in Grafana (provisioning file **or** the UI).
- Verification: `docs/verification.md` §4 (Logs).

---

## Step 6 - Dashboards & alerts (the "look" of an incident) `CORE`

> **User story** — *As an **on-call engineer**, I want **one Grafana dashboard**
> that gives me the three golden signals + logs + traces, and I want a
> **Prometheus alert** to **fire** when the API is degraded or down, so that
> I **detect an incident without someone telling me**.*

This step is **in this repo** (Grafana UI + Prometheus rules file) — it is
where the three pillars you wired in Steps 3-5 **come together**.

**What you ship here** (the two separate deliverables):

| Artifact | Where it lives | Why |
|---|---|---|
| **Grafana dashboard** | In the **Grafana UI** (commit a screenshot of it into your README) | The dashboard is assembled from 3 data sources (prom/lok/tempo) already wired; shipping it as JSON in the repo is *possible* but adds nothing to the pedagogy of Step 6. |
| **Prometheus alert rules** | In this repo, `prometheus/rules.yml` | Alert rules are config-as-code; they must reload on any new `docker compose up`. |

### Behavior

| # | Requirement |
|---|---|
| 6.1 | A **Grafana dashboard** (in the UI, "Listings TP" folder or "General") shows 5-6 panels you build yourself: **request rate**, **p95 latency**, **error rate** (Prometheus), a **logs** panel (Loki), and a **traces** panel (Tempo). |
| 6.2 | Three alert rules exist and are **correct** in `prometheus/rules.yml`, named **`ApiDown`** (no scrape), **`ApiHighErrorRate`** (5xx spike), **`ApiSlow`** (p95 above threshold). |
| 6.3 | You **prove** at least one alert fires: after the condition, `curl :9090/api/v1/rules` shows a rule with `state: firing`. The simplest to prove in a lab is **`ApiDown`** (§6.3 hint says how). |

### Acceptance checklist

- [ ] A **single dashboard** (in the UI) shows the three metrics panels **and** the logs panel **and** the traces panel — screenshot saved to the team README
- [ ] `curl -s http://localhost:9090/api/v1/rules | python3 -m json.tool` lists the **3** rules — **`ApiDown`**, **`ApiHighErrorRate`**, **`ApiSlow`** — in a group called `listings-api` (or similar)
- [ ] You `docker compose stop listings-api` (from `../springboot`) to make the scrape fail, wait `for` + ~30 s, and `curl :9090/api/v1/rules` shows **`ApiDown` in `state: firing`**. Then `docker compose start listings-api` to restore it.
- [ ] (Optional, if a 5xx can be induced — e.g. an endpoint that throws in TP1) you show **`ApiHighErrorRate` firing** in the same way
- [ ] A log line with `level=ERROR` (or `INFO`) is visible in the dashboard's **logs** panel

### Hints

- You build the dashboard **in the UI** (Grafana → Dashboards → New). Add panels one by one; pick the right data source per panel (Prometheus / Loki / Tempo). The data sources themselves were wired in **Steps 3-5** (Prometheus pre-provisioned, **Tempo added in Step 4**, **Loki added in Step 5**) — you do **not** re-add them here.
- **`ApiDown`** is the easiest to prove in a lab: the *target* goes down (`docker compose stop listings-api`), and after the rule's `for:` elapses its `state` flips to **`firing`**. Use a short `for: 1m` (or 30 s) so the proof is not too slow.
- **`ApiHighErrorRate`**: `sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) > 0.05 for 1m` — 5% of requests are 5xx.
- **`ApiSlow`**: `histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{service="listings-api"}[5m])) by (le)) > 0.5 for 2m` — p95 above 500 ms.
- Rules belong in `prometheus/rules.yml` (skeleton already shows the shape). To reload after editing: `curl -X POST http://localhost:9090/-/reload` — or `docker compose restart prometheus`.
- The 3 rules can share **one** group in `rules.yml`. Each one has: `alert`, `expr`, `for`, `labels`, `annotations.summary`.
- Verification: `docs/verification.md` §6 (Alerts) walks through the exact `curl` you need to see each rule in `state: firing`.

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
- SLO ratio PromQL: `1 - (sum(rate(http_server_requests_seconds_count{status=~"5.."}[1h])) / sum(rate(http_server_requests_seconds_count[1h])))`
  (use a window that matches your data: `[1h]` in a lab; `[24h]` / `[30d]` in production).

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
| 9.2 | TP1 (`../springboot`) builds and runs on `8080`; generating traffic yields **metrics in Prometheus, traces in Tempo, logs in Loki**. |
| 9.3 | **One** Grafana dashboard (in the UI, screenshot in the README) shows the three pillars; **three** alert rules exist in `prometheus/rules.yml` and **`ApiDown`** has been shown `firing`. |
| 9.4 | Your **five config files** (`prometheus/prometheus.yml`, `prometheus/rules.yml`, `otel/collector-config.yaml`, `promtail/promtail-config.yaml`, `grafana/provisioning/datasources/datasources.yml`) are in the repo; the **reference solutions** are **not** in the repo. |
| 9.5 | README documents: how to start the stack, the three pipelines, the ports, and one alert + one dashboard screenshot. |
| 9.6 | Oral (10 min + 5 min Q&A): *walk one slow request from the Grafana panel → the Tempo trace → the Loki log*, explaining which middleware produced each part. |

---

## Deliverables

| Item | Detail |
|---|---|
| Git repository (pair) | TP1 repo modified **+** this `monitoring/` folder with **your** pipeline configs |
| `docker compose up` | 6 services healthy from **student-filled** configs |
| `mvn verify` (TP1) | builds; instrumentation wired |
| Grafana | **One dashboard (in the UI; screenshot in README), 3 data sources (in `datasources.yml`), 3 alert rules in `rules.yml` (`ApiDown` demonstrably firing)** |
| README.md | stack run, the 3 pipelines, ports, 1 alert + 1 dashboard **screenshot** (not the dashboard JSON) |

---

## Appendix A - The three pipelines at a glance

| Pillar | App produces | Middleware | Backend | Query lang |
|---|---|---|---|---|
| Metrics | `/actuator/prometheus` | **Prometheus scrape** | Prometheus | **PromQL** |
| Traces | OTLP gRPC (`otel-collector:4317`, network name) | **OTel Collector** | Tempo | **TraceQL** / HTTP API |
| Logs | file `/listings-logs/listings.log` | **Promtail** | Loki | **LogQL** |

## Appendix B - Useful endpoints

All the backend URLs (PromQL, LogQL, TraceQL, Tempo search/traces, alert rules,
Grafana) are listed run-ready in [`docs/verification.md`](docs/verification.md).

## Appendix C - What is NOT to be committed

- The **reference solutions** (live in `../../solutions/monitoring/`, used by the
  supervisor) must **not** be copied into this repo.
- Log files: the app writes to the shared volume `middleware-logs` (see the
  base compose in `../springboot`) — do not copy the file into the repo.

