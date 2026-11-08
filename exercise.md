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
| **Bonus** (start only when the core `Verify` sections are green) | 7 (correlation & first SLO) · 8 (two services / a slow dependency) |
| **Delivery** | 9 (final integration) |

Each core step's `Verify` section is your "definition of done".
Bonus steps have their own `Verify` section; doing them is optional
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

## Step 1 — Bring up the observable stack   CORE

**Outcome** — The six observability services (`mon-prometheus`, `mon-loki`,
`mon-tempo`, `mon-otel-collector`, `mon-promtail`, `mon-grafana`) are **up**
in this repo, each answering its backend health/ready endpoint. This is the
**bootstrap**: before any app wiring, the three backends + the collector +
the UI are reachable, and you can prove it with a `curl`.

### Build it
> **Pre-req** — the **TP1 base** (`../springboot`) is up on `8080`
> (TP1's `docker compose up`). This step only starts **this repo's** stack
> (the six `mon-*` services).

1. **Start** the stack in this repo (`monitoring/`):
   `cd "$MON_DIR" && docker compose up -d`.
2. **Wait** for the services to settle (a few seconds; Promtail/Collector
   take longer than the backends). Then **check** each health endpoint
   (the Verify list).
3. **If one is not ready**, tail its logs
   (`docker compose -f <compose> logs <service>`), fix the compose / path
   issue, and re-check.

### Expected behavior
- `docker compose ps` shows **six** services, all **`Up`** (or `healthy`).
- The four backend readiness endpoints answer (Prometheus `/-/healthy`,
  Loki `/ready`, Tempo `/ready`, OTel Collector's OTLP gRPC listener on
  `4317`).
- Grafana is **up** and already has **at least one** provisioned data source
  (the Prometheus one — the *look* of the rest comes in Step 6).

### Verify
- [ ] `docker compose ps` shows `mon-prometheus`, `mon-loki`, `mon-tempo`, `mon-otel-collector`, `mon-promtail`, `mon-grafana` all `Up`
- [ ] `curl -fsS http://localhost:9090/-/healthy` → `Prometheus Server is Healthy.`
- [ ] `curl -fsS http://localhost:3100/ready` → `ready`
- [ ] `curl -fsS http://localhost:3200/ready` → `ready`
- [ ] `curl -fsS http://localhost:3000/api/health -u admin:admin` → `database: ok` (Grafana up, not "no data sources")

### If you are stuck
- The exact health `curl`s: [`docs/verification.md` §1](docs/verification.md).
- All backends answer readiness **without auth** — no login needed for these checks (only Grafana needs `admin`/`admin`).
- `docker compose -f <compose> logs -f <service>` to watch a service start; check the port mapping if a ready endpoint times out.

---

## Step 2 — Understand the three pillars (read the backends, don't trust)   CORE

**Outcome** — Each of the three backends **stores and returns** a signal,
**before the app is wired in**: Prometheus returns a `PromQL` query, Loki
returns a pushed `LogQL` stream, and Tempo returns a valid (possibly empty)
trace search. You can **name** the three query languages. No app code is
involved yet — this proves **each pipeline is real** on its own.

### Build it
> This step is **read-only on the backends**. You do **not** change any app
> or config file — you **run `curl`** against the three backends (no Java, no
> config edits).

1. **Query** Prometheus directly (`curl :9090/api/v1/query`) and read your own
   `up` / `prometheus_...` series.
2. **Push** one line to Loki's **push API**, then **query** it back with
   `LogQL` (`{service="listings-api"}`).
3. **Call** Tempo's search API (last 10 min) and confirm it returns a **valid
   JSON object with a `traces` key** (it may legitimately be **empty**).
4. **Name** (in your own words) the three query languages: **PromQL**,
   **LogQL**, **TraceQL**.

### Expected behavior
- **Prometheus** can **store and query** a series — a `PromQL` query returns
  its own `up` / `prometheus_*` metrics.
- **Loki** can **store and query** a stream — after you push one line, the
  `LogQL` query `returns` it.
- **Tempo** can **store and return** a search result — a search returns a
  **well-formed** JSON object with a `traces` key (**empty is correct** at
  this stage).
- "Empty but valid" is the expected starting state: a stack with **no data**
  still returns **well-formed** JSON. **The contract is real even before the
  app is connected.**

### Verify
- [ ] `curl -s http://localhost:9090/api/v1/query --data-urlencode "query=up"` returns a JSON `result` with an `up` series (value `1`)
- [ ] Push one line to the Loki **push API** (one-liner in [`docs/verification.md` §4](docs/verification.md)) and the Loki query `{service="listings-api"}` **returns it** — proves Loki stores + returns a stream **before the app is wired**
- [ ] A Tempo search (last 10 min) returns a valid JSON object with a `traces` key (may legitimately be **empty**)
- [ ] You can name (in your own words) the three query languages: **PromQL**, **LogQL**, **TraceQL**

### If you are stuck
- Full copy-pasteable `curl` recipes for each backend: [`docs/verification.md`](docs/verification.md) §2, §4, §3.
- If Loki/Tempo return an **empty `traces`/`[]`** — that is **correct**, not a failure. You are proving the **shape** of the response, not that data exists.

---

## Step 3 — Expose metrics → Prometheus → Grafana   CORE

**Outcome** — The API **exposes** HTTP metrics + **one business metric**
(`listings_created_total`) in Prometheus text form, **Prometheus scrapes**
them (visible via the Prometheus query API), and the series are tagged
`service="listings-api"`. The Grafana panel for them comes in **Step 6** —
here you only prove the scrape works by asking **Prometheus** directly.

### Build it
This step is **split in two repos** — both must be done:

**In the TP1 repo (`../springboot`):**
1. **Add** `io.micrometer:micrometer-registry-prometheus` to `pom.xml`
   (version-managed by Boot — no `<version>`).
2. **Expose** `/actuator/prometheus` in `application.yml` (`management.endpoints
   .web.exposure.include: …, prometheus`) and keep the existing
   `management.metrics.tags.service: listings-api`.
3. **Record** a **business** `Counter` in the create path:
   `Counter.builder("listings_created_total").tag("service","listings-api")
   .register(meterRegistry)` and **increment** it on each successful
   `POST /listings`.

**In this repo (`monitoring/`):**
4. **Add** the `listings-api` **scrape job** in `prometheus/prometheus.yml`
   (the skeleton already shows the exact keys). **Reload** with
   `curl -X POST http://localhost:9090/-/reload` (or
   `docker compose restart prometheus`).
5. **Rebuild + restart** the app so it picks up the dependency + endpoint
   (`cd "$TP1_DIR" && docker compose up -d --build listings-api`), then
   generate a `POST /listings`.

### Expected behavior
- `GET http://localhost:8080/actuator/prometheus` returns Prometheus
  **text exposition** format (`# HELP` / `# TYPE` lines).
- The default HTTP metrics (`http_server_requests_seconds_*`) are present,
  tagged `service="listings-api"`, `uri=`, `status=`.
- The **business** metric `listings_created_total{service="listings-api"}`
  **increases by one per successful `POST /listings`**.
- **Prometheus scrapes** the app: the series are visible via the **Prometheus
  query API** (not only on the app's actuator).

### Verify
- [ ] `curl -s http://localhost:8080/actuator/prometheus` shows `# HELP http_server_requests_seconds`
- [ ] After 1 `POST /listings`, `listings_created_total` is **1 higher** than before the call
- [ ] `curl -s "http://localhost:9090/api/v1/query" --data-urlencode 'query=listings_created_total{service="listings-api"}'` returns a value (i.e. **Prometheus now has the series** — your scrape job works)

### If you are stuck
- The `Counter` is a **`micrometer`** object — inject `MeterRegistry`, build it once, keep a reference, and `.increment()` in the create-path service.
- If the Prometheus query returns `[]`, the **scrape job** is the usual cause — confirm the job's `static_configs` points at `listings-api:8080` (container name, not `localhost`), then **reload**.
- (Panel-building is **Step 6** — here you only prove the series is **scraped** by asking Prometheus directly, not Grafana.)

---

## Step 4 — Wire distributed tracing → Collector → Tempo   CORE

**Outcome** — Every request is **traced**: a `POST /listings` produces a
**business span named `listings.create`** (not only the auto HTTP span), that
span is **stored in Tempo** (returned by `…/api/traces/<traceId>` and by a
search), and every trace keeps a stable **`trace_id`**. You open a slow
request in Tempo and see *exactly which operation took the time*.

### Build it
This step is **split in three places** — all three must be done:

**In the TP1 repo (`../springboot`):**
1. **Add** the tracing dependency: `micrometer-tracing-bridge-otel` + the
   OpenTelemetry **Spring Boot starter** (`opentelemetry-spring-boot-starter`).
2. **Set** the OTel env vars on `listings-api` (in the base `docker-compose`
   or `application.yml`): `OTEL_SERVICE_NAME=listings-api`,
   `OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317`, 100% sampling.
   The canonical form is in **this repo's `README` → "0) Required base edit"**.
3. **Record an explicit span** on the create path via the `Tracer`:
   `Tracer.nextSpan().name("listings.create").start() … .end()`.

**In this repo (`monitoring/`) — Collector:**
4. **Add** the `otlp/tempo` **exporter** in
   `otel/collector-config.yaml` (`endpoint: tempo:4317`) and put it on the
   `traces` pipeline.

**In this repo (`monitoring/`) — Grafana:**
5. **Add** the **Tempo** data source in
   `grafana/provisioning/datasources/datasources.yml`
   (`type: tempo`, `url: http://tempo:3200`), or in the Grafana UI.
6. **Rebuild + restart** the app (`cd "$TP1_DIR" && docker compose up -d --build
   listings-api`) so it picks up the starter + env vars, then create a listing.

### Expected behavior
- A `POST /listings` produces a **span named `listings.create`** (you added it
  explicitly — it is **not** only the auto-generated HTTP span).
- That span is **stored in Tempo**: `GET …:3200/api/traces/<traceId>` returns
  it, and a Tempo **search** (last 10 min, `service.name=listings-api`)
  returns the trace.
- Each trace keeps a **stable `trace_id`** for the whole request.
- The Tempo backend is **already up** (Step 1) — you **only wire the Collector
  to it** and **add the data source**; you do not bring Tempo up.

### Verify
- [ ] `curl -s "http://localhost:3200/api/search?tags=service.name%3Dlistings-api"` (last 10 min) returns **≥ 1 trace** after you create a listing (`POST /listings`)
- [ ] Opening one traceId in Tempo shows a span named **`listings.create`** (the explicit one, not only the HTTP span)
- [ ] `grep -rn "listings.create\|nextSpan\|Tracer" "$TP1_DIR/src"` — the explicit span is in **your** code, not only the auto HTTP span

### If you are stuck
- The OTel Spring Boot starter reads the **standard** env vars (`OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`). Point the endpoint at the **Collector** (`otel-collector:4317`), **not** at Tempo directly.
- The Collector's `traces` pipeline must list the tempo exporter in its **`exporters`** key. If the search returns 0 traces, check the pipeline `exporters` line and that the app is actually sending OTLP (watch the Collector logs for `exporter/tempo`).
- `curl -X POST http://localhost:3200/api/search` (or the GET above) is the verification — the full form is in [`docs/verification.md` §3](docs/verification.md) (Traces).

---

## Step 5 — Collect logs → Promtail → Loki (with trace_id)   CORE

**Outcome** — The API's **log lines** are searchable in Loki, tagged with
`service` and a filterable `level`, and **each line carries the same
`trace_id`** as its Tempo trace (Step 4) — so you can **jump from a trace to
its logs**.

### Build it
This step is **split in two repos** — both must be done:

**In the TP1 repo (`../springboot`):**
1. **Add** a Logback **`FileAppender`** that writes to
   `/listings-logs/listings.log` (**create the folder** — and the shared
   `middleware-logs` volume in the base compose, so Promtail can read it).
2. **Put** the correlation fields in the log pattern: `trace=%X{traceId}`,
   `span=%X{spanId}` (Micrometer tracing writes these to the **MDC** — that is
   the correlation link), plus the `level`.

> **Important:** the log **file path is fixed** to
> `/listings-logs/listings.log` because that is what **Promtail is configured
> to tail** (see this repo's `docker-compose.yml`). Do not change it here.

**In this repo (`monitoring/`):**
3. **Add** the `listings` job in `promtail/promtail-config.yaml`:
   `__path__: /listings-logs/listings.log`, a `service` label, and a
   `regex` + `labels` pipeline stage that **derives `level`** from the line.
4. **Add** the **Loki** data source in
   `grafana/provisioning/datasources/datasources.yml`
   (`type: loki`, `url: http://loki:3100`), or in the Grafana UI.
5. **Restart** the app so it writes the file
   (`cd "$TP1_DIR" && docker compose up -d --build listings-api`), then
   generate some traffic.

### Expected behavior
- A request's **log line is stored in Loki**: the Loki query
  `{service="listings-api"}` returns it.
- The line carries the **same `trace_id`** as the trace from Step 4
  (**correlation**).
- The line has a **filterable `level`** (`INFO` / `ERROR`) —
  `{service="listings-api"} |= "ERROR"` works.

### Verify
- [ ] `curl -s "http://localhost:3100/loki/api/v1/query_range" --data-urlencode 'query={service="listings-api"}'` returns **≥ 1 line** after some traffic
- [ ] Filtering by level works: add `| level=ERROR` (or `|= "ERROR"`) and an `ERROR` line from your app shows up
- [ ] The `trace_id` in the log line **matches a real Tempo trace** (Step 4) — paste it into the Tempo API and it resolves

### If you are stuck
- The MDC keys are **`traceId`** and **`spanId`** (Micrometer tracing sets them). The Logback pattern uses `%X{traceId}` / `%X{spanId}` — that is **what** makes a log line linkable to a Tempo trace.
- In `promtail-config.yaml`, the skeleton already shows the exact keys: `__path__: /listings-logs/listings.log` plus a `regex` + `extract`/`labels` stage to build `level`.
- If Loki has **no** lines, the usual causes: the app is **not writing the file** (check the app logs for the FileAppender) or Promtail is **not tailing the right path** (tail this repo's logs for `promtail` scrape errors).
- Full copy-pasteable recipe: [`docs/verification.md` §4](docs/verification.md) (Logs).

---

## Step 6 — Dashboards & alerts (the "look" of an incident)   CORE

**Outcome** — **One Grafana dashboard** shows the three golden signals
(request rate, p95 latency, error rate) **plus** a logs panel and a traces
panel, **and** **three Prometheus alerts** (`ApiDown`, `ApiHighErrorRate`,
`ApiSlow`) exist in config-as-code — with at least **one proven firing**.
This is where the three pillars you wired in Steps 3-5 **come together**.

### Build it
This step is **in this repo** (Grafana UI + Prometheus rules file):

1. **Build the dashboard in the Grafana UI** (Dashboards → New). Add panels
   one by one and pick the **right data source per panel**: **request rate**,
   **p95 latency**, **error rate** → **Prometheus**; a **logs** panel →
   **Loki**; a **traces** panel → **Tempo**. The data sources are **already
   wired** (Prometheus pre-provisioned, Tempo added Step 4, Loki added Step 5)
   — do **not** re-add them here.
2. **Write the three alert rules** in `prometheus/rules.yml` (`ApiDown`,
   `ApiHighErrorRate`, `ApiSlow`), in **one group**
   (`listings-api`). Each rule has `alert`, `expr`, `for`, `labels`,
   `annotations.summary`.
3. **Reload** Prometheus: `curl -X POST http://localhost:9090/-/reload` (or
   `docker compose restart prometheus`).
4. **Prove one rule fires** — the easiest is `ApiDown` (see below). Then
   **screenshot** the dashboard into your team README.

**The two deliverables** of this step:

| Artifact | Where | Why |
|---|---|---|
| **Grafana dashboard** | In the **Grafana UI** (screenshot in your README) | Assembled from the 3 wired data sources; shipping JSON adds nothing to the pedagogy. |
| **Prometheus alert rules** | `prometheus/rules.yml` (this repo) | Config-as-code; they must survive any new `docker compose up`. |

### Expected behavior
- **One dashboard** (in the UI) with **5-6 self-built panels**: request rate,
  p95 latency, error rate (Prometheus) + a **logs** panel (Loki) + a
  **traces** panel (Tempo).
- **Three alert rules** in `prometheus/rules.yml`:
  - **`ApiDown`** — no scrape (`up{service="listings-api"} == 0`);
  - **`ApiHighErrorRate`** — 5xx spike
    (`sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
      / sum(rate(http_server_requests_seconds_count[5m])) > 0.05 for 1m`);
  - **`ApiSlow`** — p95 above 500 ms
    (`histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket
      {service="listings-api"}[5m])) by (le)) > 0.5 for 2m`).
- You **prove** at least one alert **fires**: after the condition,
  `curl :9090/api/v1/rules` shows a rule with **`state: firing`**.

### Verify
- [ ] A **single dashboard** (in the UI) shows the three metrics panels **and** the logs panel **and** the traces panel — **screenshot saved to the team README**
- [ ] `curl -s http://localhost:9090/api/v1/rules | python3 -m json.tool` lists the **3** rules — **`ApiDown`**, **`ApiHighErrorRate`**, **`ApiSlow`** — in a group called `listings-api` (or similar)
- [ ] `cd "$TP1_DIR" && docker compose stop listings-api` (to make the scrape fail) → wait `for` + ~30 s → `curl :9090/api/v1/rules` shows **`ApiDown` in `state: firing`** → then `docker compose start listings-api` to restore
- [ ] (Optional, if a 5xx can be induced — e.g. an endpoint that throws in TP1) you show **`ApiHighErrorRate` firing** the same way
- [ ] A log line with `level=ERROR` (or `INFO`) is visible in the dashboard's **logs** panel

### If you are stuck
- **`ApiDown` is the easiest to prove in a lab** — the *target* goes down (`docker compose stop listings-api`), and after the rule's `for:` elapses its `state` flips to **`firing`**. Use a short `for: 1m` (or 30 s) so the proof is not slow.
- The three rules can share **one group** in `rules.yml`; the skeleton already shows the shape.
- Full copy-pasteable walk to see each rule in `state: firing`: [`docs/verification.md` §6](docs/verification.md) (Alerts).

---

## Step 7 — Correlation & a first SLO   BONUS

**Outcome** — From a slow Tempo trace you **jump to its logs** (same
`trace_id`) and to the metric window around it, and you show **one SLO
panel** (a success-ratio between 0 and 1) computed from the metrics — so you
can answer **"is the API meeting its commitment?"**, not just "is it up?".

### Build it
> Bonus — build **both** pieces, on the existing wiring from Steps 4-6.

1. **Demonstrate the Tempo → Loki jump**: open a Tempo trace, then use the
   Tempo data source's **traces-to-logs** mapping (via the `trace_id`) to
   open the matching Loki logs. Confirm the `trace_id` matches.
2. **Add an SLO panel** in the dashboard: a success-ratio from
   `http_server_requests_seconds_count` (e.g. `1 - (5xx / total)`).

### Expected behavior
- **7.1** A Tempo trace **opens the matching Loki logs** (via the `trace_id`)
  — the link is demonstrated, not just asserted.
- **7.2** An **SLO panel** shows availability / success-ratio computed from
  `http_server_requests_seconds_count` (e.g. `1 - (5xx / total)`).

### Verify
- [ ] A Tempo → Loki **"traces-to-logs" jump works** (same `trace_id`)
- [ ] A Grafana panel shows a **success-rate / SLO** value **between 0 and 1**

### If you are stuck
- The Grafana Tempo data source has a **traces-to-logs** mapping — see the `jsonData.tracesToLogsV2` reference in [`docs/verification.md`](docs/verification.md).
- SLO ratio PromQL: `1 - (sum(rate(http_server_requests_seconds_count{status=~"5.."}[1h])) / sum(rate(http_server_requests_seconds_count[1h])))` — use a window that matches your data: `[1h]` in a lab, `[24h]` / `[30d]` in production.

---

## Step 8 — Two services + a "slow" dependency   BONUS

**Outcome** — A single request spans **two services** (or an in-app downstream
step), and the Tempo trace shows **both spans linked by the same `trace_id`**.
You can **attribute** part of the total latency to the child span — the
classic reason we **trace** instead of only counting requests.

### Build it
> Bonus — add one extra hop, keep it small.

1. **Extend** the traced path of Step 4 so a request makes **one more hop**:
   the simplest is an **in-app downstream call** (a `RestTemplate` /
   `WebClient` to another endpoint that is also traced), or a **real 2nd
   Spring Boot app** in Docker that shares the same OTLP endpoint and
   `traceparent` propagation.
2. **Keep** the same `service.name` tracing setup, then generate one request
   and **open** the Tempo trace.

### Expected behavior
- **8.1** A request spans **two services** (e.g. `listings-api` →
  `geo-service`, or an in-app downstream step); the Tempo trace shows **both
  spans linked by the same `trace_id`**.
- **8.2** You can **attribute** part of the total latency to the child span
  (name which span took most of the time).

### Verify
- [ ] A Tempo trace shows **2+ spans** (parent `listings.create` + child) sharing one `trace_id`
- [ ] You can name **which span took most of the time** (the child / downstream)

### If you are stuck
- The simplest "second service" is an **in-app downstream call** — a `RestTemplate`/`WebClient` call to another endpoint, also traced. A full 2nd Spring Boot app in Docker works too (same OTLP endpoint, `traceparent` propagation), but keep it small.

---

## Step 9 — Final Integration   DELIVERY

**Outcome** — The whole stack boots cleanly from **your own configs** (no
solution files present): all six services healthy, TP1 on `8080`, traffic
yields **metrics in Prometheus, traces in Tempo, logs in Loki**, one dashboard
shows the three pillars, and `ApiDown` has been shown **firing**. The
evaluator can start the stack and **trust the output**.

### Final checks
- [ ] `cd "$MON_DIR" && docker compose up -d` → **all six services healthy**; the stack boots from the **student-filled** configs (no solution files present)
- [ ] `cd "$TP1_DIR" && mvn verify && docker compose up -d` → TP1 builds and runs on `8080`; generating traffic yields **metrics in Prometheus, traces in Tempo, logs in Loki**
- [ ] **One** Grafana dashboard (in the UI, screenshot in the README) shows the three pillars; **three** alert rules in `prometheus/rules.yml` and **`ApiDown`** has been shown `firing`
- [ ] Your **five config files** (`prometheus/prometheus.yml`, `prometheus/rules.yml`, `otel/collector-config.yaml`, `promtail/promtail-config.yaml`, `grafana/provisioning/datasources/datasources.yml`) are in **this** repo; the **reference solutions** are **not**
- [ ] The team **README** documents: how to start the stack, the three pipelines, the ports, and one alert + one dashboard **screenshot**
- [ ] **Oral** (10 min + 5 Q&A): walk **one slow request** from the Grafana panel → the Tempo trace → the Loki log, explaining which middleware produced each part

### If you are stuck
- The **team README** is where you write **what** you wired and **why** (not a copy of this exercise statement) — it is your **oral-prep**. If you cannot say *which* middleware produced the metrics / the trace / the log, write it down and re-check.
- The **smoke bar** is `docker compose up` green + the three backends returning data — you do **not** need new Java tests (see Overview).

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

