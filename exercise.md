# TP Monitoring / Observability — Progressive Implementation

## Overview

This is a **follow-up** to the Spring Boot TP (TP1, `../springboot/`). We
reuse the **TP1 listings API** — unchanged domain — and wire it into a
full **observability stack** so every request can be *measured*, *traced*,
and *logged* across the three pillars of observability.

**Validation method (the contract):** each step's **`Expected behavior`** is
the contract. Its **`Verify`** section (Step 9: **`Final checks`**) is the
definition of done: **data you can show in a backend**, not a Java test.

| Level | What you assert | How |
|---|---|---|
| **Query** | a PromQL / TraceQL / LogQL expression returns the expected data | the `curl` of [`docs/verification.md`](docs/verification.md) |
| **Panel** | a Grafana panel renders a sensible value after real traffic | screenshot + the panel query string |
| **Alert** | a Prometheus rule **fires** under the condition you built | `curl :9090/api/v1/rules` shows `state: firing` |

Running the app is not enough: you show the *data* in the backend for the
scenario the step names.

New to observability, or to this stack? Read [`docs/primer.md`](docs/primer.md)
first (5 minutes).

The specification describes **behaviors, contracts, and observable signals**
(what a panel reads, what an alert fires on, what a query returns). It does
**not** prescribe internal implementation beyond the required dependencies
and configuration keys.

> **Prerequisite: TP1 is done and green.** The TP1 API (`../springboot`)
> passes its core checklists (Steps 1·2·3·5·6·8: register, CRUD, ownership,
> `?version=` V1/V2, metrics) and runs
> on **port `8080`** (unchanged — the app is on `8080` for every TP, even
> when the other follow-up, Keycloak, is active; Keycloak itself moves to
> `8081` in that case). If TP1 is not at that state, finish it first. This TP
> only adds *how we observe the API*, not new business behaviour.
>
> This is **one of four alternatives** for the second TP ([Keycloak](../keycloak/),
> Monitoring, [IA](../IA/), [Messaging](../messaging/)). Do not run two TP2
> stacks at the same time.

### How to read a step

| Section | What it gives you |
|---|---|
| **Outcome** | the observable result |
| **Why this step** | the concept taught and the real problem it solves |
| **How it fits** | which pipeline the step builds and which part you change (when several components are involved) |
| **Build it** | the changes, grouped by repo or component, in order; the `→` line under each item says what breaks without it |
| **Checkpoint** | a quick call to confirm a group works before starting the next one |
| **Expected behavior** | the contract: what a query returns, what a panel shows, what an alert does |
| **Verify** | the definition of done: the [`docs/verification.md`](docs/verification.md) section to run, plus the checks the contract cannot show |
| **If you are stuck** | folded: open it only when blocked; pointers to the file, key, or documentation, never the full config |

---

## Domain & data model (this TP)

The domain does **not** change from TP1 (actors, entities, ownership, V1/V2
visibility, endpoints, payloads). What changes is **only** the observation
layer:

| Concern | TP1 | This TP |
|---|---|---|
| How we *measure* the API | (none / basic actuator) | Micrometer → **Prometheus** |
| How we *trace* a request | (none) | Micrometer Tracing + OTel → **Collector → Tempo** |
| How we *collect logs* | console only | Spring Boot log file → **Alloy → Loki** |
| Where we *look* in an incident | `tail -f` + guesswork | **Grafana** dashboards + **Prometheus alerts** |

> **Key idea**: the API keeps behaving exactly as TP1. You are adding a
> *lens* to see what is happening inside it — with **standard, open-source
> middleware**, not vendor code.

### The three pipelines, in one picture

Every step from 3 to 6 builds or reads one arrow of this picture.

```
                       TP1 repo (../springboot)                 this repo (monitoring/)
                       ────────────────────────                 ───────────────────────
            ┌── /actuator/prometheus ◄──── scrape ──────────── Prometheus :9090 ──┐   (Step 3)
            │      (Micrometer)            prometheus/prometheus.yml               │
            │                                                                      │
listings-api├── OTLP HTTP ──────────────► OTel Collector :4318 ──► Tempo :3200 ─────┼──► Grafana :3000
  :8080     │      (OTel tracing)          otel/collector-config.yaml              │    datasources.yml
            │                                                                      │    + dashboard (UI)
            └── /listings-logs/listings.log ──► Alloy ─────────► Loki :3100 ───────┘    (Step 6a)
                   (Spring Boot log file,   alloy/config.alloy                         (Steps 4-5 add
                    volume middleware-logs)                                             the data sources)
                                                    Prometheus ── prometheus/rules.yml ── alerts (Step 6b)
```

The **TP1 app is a container on the shared `middleware` network** (base
`../springboot/docker-compose.yml`), so it is reached by **container name**,
not `localhost`:

- the app pushes OTLP over HTTP to `otel-collector:4318`;
- `Prometheus` scrapes the app at `listings-api:8080`;
- `Alloy` tails the log file on the shared volume `middleware-logs`
  (Spring Boot writes it there: `LOGGING_FILE_NAME` in the base compose).

---

## Scope

| Track | Steps |
|---|---|
| **Core** (required) | 1 · 2 · 3 · 4 · 5 · 6a · 6b |
| **Bonus** (start only when the core `Verify` sections are green) | 7 (correlation & first SLO) · 8 (two services / a slow dependency) |
| **Delivery** | 9 (final integration) |

Bonus steps are optional but count for the grade if you deliver them.

---

## Conventions

- **Service name** used everywhere (labels, tags, panel queries): `listings-api`.
- **Ports** (host): Grafana `3000`, Prometheus `9090`, Loki `3100`, Tempo `3200`,
  OTLP `4317`/`4318`, app `8080` (this TP).
- **Metrics**: exposed by the app on `/actuator/prometheus` and scraped by
  Prometheus at `listings-api:8080`. The `service="listings-api"` label is
  added by the **scrape job** (`prometheus.yml`), not by the app: a `service`
  tag set in the app too would clash and be renamed `exported_service`.
- **Business metric**: the TP1 counter `listings.creations`, published as
  `listings_creations_total`.
- **Traces**: spans are exported over **OTLP/HTTP** to `otel-collector:4318`
  (Spring Boot's default OTLP transport is HTTP; 4317 is the gRPC port);
  `service.name=listings-api` (that's the **Tempo** label key). All tracing
  settings are `OTEL_*` environment variables (README §0).
- **Logs**: Spring Boot writes the file `/listings-logs/listings.log` on the
  shared volume `middleware-logs`; with tracing on, each line carries
  `[<traceId>-<spanId>]`. Alloy tails it and ships to Loki with labels
  `service="listings-api"` and `level` derived from the line.
- **Query languages**: PromQL (metrics), **TraceQL** (traces), **LogQL** (logs).
- Error JSON shape of the **API** is the same as TP1 — monitoring never changes
  the API's response format.
- **Rebuild after every change to the TP1 repo** (code, `pom.xml`, or base
  compose env): `cd "$TP1_DIR" && docker compose up -d --build listings-api`.
  A new dependency or env var only applies to a recreated container.

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

## Step 1 — Bring up the observable stack   CORE

**Outcome** — The six observability services (`mon-prometheus`, `mon-loki`,
`mon-tempo`, `mon-otel-collector`, `mon-alloy`, `mon-grafana`) are **up**
in this repo, each answering its backend health/ready endpoint. This is the
**bootstrap**: before any app wiring, the three backends + the collector +
the UI are reachable, and you can prove it with a `curl`.

**Why this step** — An observability stack is six services that each can fail
on its own. Checking each one is up *before* wiring the app means that, later,
"no data" points at your pipeline config, not at a backend that never started.

### Build it

> **Pre-req** — the **TP1 base** (`../springboot`) is up on `8080`
> (TP1's `docker compose up`). This step only starts **this repo's** stack
> (the six `mon-*` services).

1. **Start** the stack in this repo (`monitoring/`):
   `cd "$MON_DIR" && docker compose up -d`.
   → The stack attaches to the `middleware` network created by TP1: if the
   TP1 base is not up, the network does not exist and the start fails.
2. **Wait** for the services to settle (a few seconds; Alloy and Grafana
   start after Loki is healthy). Then **check** each health endpoint
   (the Verify list).
   → `Up` in `docker compose ps` only means the process started; the ready
   endpoint means the backend can accept data.
3. **If one is not ready**, tail its logs
   (`docker compose -f <compose> logs <service>`), fix the compose / path
   issue, and re-check.
   → A backend that fails now fails silently later: its pipeline just shows
   "no data".

### Expected behavior
- `docker compose ps` shows **six** services, all **`Up`** (or `healthy`).
- The four backend readiness endpoints answer (Prometheus `/-/healthy`,
  Loki `/ready`, Tempo `/ready`, OTel Collector's OTLP/HTTP receiver on
  `4318`).
- Grafana is **up** and already has **at least one** provisioned data source
  (the Prometheus one — the *look* of the rest comes in Step 6a).

### Verify

- [ ] [`docs/verification.md` §1](docs/verification.md): the six services are `Up` and every ready endpoint answers as in *Expected behavior*

<details>
<summary><strong>If you are stuck</strong></summary>

- The exact health `curl`s: [`docs/verification.md` §1](docs/verification.md).
- All backends answer readiness **without auth** — no login needed for these checks (only Grafana needs `admin`/`admin`).
- `docker compose -f <compose> logs -f <service>` to watch a service start; check the port mapping if a ready endpoint times out.

</details>

---

## Step 2 — Understand the three pillars (read the backends, don't trust)   CORE

**Outcome** — Each of the three backends **stores and returns** a signal,
**before the app is wired in**: Prometheus returns a `PromQL` query, Loki
returns a pushed `LogQL` stream, and Tempo returns a valid (possibly empty)
trace search. You can **name** the three query languages. No app code is
involved yet — this proves **each pipeline is real** on its own.

**Why this step** — Each backend has its own API and query language. Talking
to them directly once, with no app in between, gives you the tool you will use
in every later step to find out *which* side of a pipeline is broken.

### Build it

> This step is **read-only on the backends**. You do **not** change any app
> or config file — you **run `curl`** against the three backends (no Java, no
> config edits).

1. **Query** Prometheus directly (`curl :9090/api/v1/query`) and read your own
   `up` / `prometheus_...` series.
   → Prometheus scrapes itself: this is a known-good series to compare with
   when the app's series are missing in Step 3.
2. **Push** one line to Loki's **push API**, then **query** it back with
   `LogQL` (`{service="listings-api"}`).
   → If this works and Step 5 shows nothing, the problem is Alloy or the
   log file, not Loki.
3. **Call** Tempo's search API (last 10 min) and confirm it returns a **valid
   JSON object with a `traces` key** (it may legitimately be **empty**).
   → An empty but valid answer proves Tempo is queryable; Step 4 fills it.
4. **Name** (in your own words) the three query languages: **PromQL**,
   **LogQL**, **TraceQL**.
   → Each panel of Step 6a is written in one of them.

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

- [ ] [`docs/verification.md` §2, §3, §4](docs/verification.md) (the backend-only reads: `up`, the Tempo search, the Loki push and read-back): every result matches *Expected behavior*
- [ ] You can name (in your own words) the three query languages: **PromQL**, **LogQL**, **TraceQL**

<details>
<summary><strong>If you are stuck</strong></summary>

- Full copy-pasteable `curl` recipes for each backend: [`docs/verification.md`](docs/verification.md) §2, §4, §3.
- If Loki/Tempo return an **empty `traces`/`[]`** — that is **correct**, not a failure. You are proving the **shape** of the response, not that data exists.

</details>

---

## Step 3 — Expose metrics → Prometheus → Grafana   CORE

**Outcome** — The API **exposes** HTTP metrics (with latency histograms) and
the TP1 business counter (`listings_creations_total`) in Prometheus text form,
**Prometheus scrapes** them (visible via the Prometheus query API), and the
series carry `service="listings-api"`. The Grafana panel for them comes in
**Step 6a** — here you only prove the scrape works by asking **Prometheus**
directly.

**Why this step** — Metrics are **pulled**: the app only *exposes* numbers,
and Prometheus decides when to collect them. This is the cheapest pillar
(numbers, not text) and the one alerts are built on.

### How it fits

```
TP1 repo                                         this repo
listings-api :8080                               Prometheus :9090
  Micrometer registry (group A, items 1-2)  ◄─── scrape every 10 s ──── job listings-api (group B, item 4)
  GET /actuator/prometheus  (text format)        adds the label service="listings-api"
```

### Build it

**A. In the TP1 repo (`../springboot`) — expose**

1. **Add** `io.micrometer:micrometer-registry-prometheus` to `pom.xml`
   (version-managed by Boot — no `<version>`).
   → Without this registry, Micrometer keeps the metrics in memory but has
   no Prometheus text format to serve.
2. **Configure** `application.yml`: add `prometheus` to
   `management.endpoints.web.exposure.include`, and turn on the percentile
   histogram of the HTTP server metric
   (`management.metrics.distribution.percentiles-histogram.http.server.requests: true`).
   → Actuator endpoints are hidden unless listed. Without the histogram, the
   app publishes no `http_server_requests_seconds_bucket` series, and the p95
   panel and the `ApiSlow` alert of Steps 6a-6b have nothing to compute.
3. **Check** the TP1 business counter (`listings.creations`, TP1 Step 8) in
   the output: Prometheus names it `listings_creations_total`. Do **not**
   create a second counter, and do **not** add a `service` tag in the app.
   → Two meters with the same Prometheus name collide (the second one is
   rejected with a WARN). The `service` label comes from the scrape job
   (item 4); the same tag set in the app would be renamed `exported_service`.

> **Checkpoint** — after a rebuild,
> `curl -s http://localhost:8080/actuator/prometheus | grep -E "^listings_creations_total|_bucket" | head -3`
> shows the counter and bucket lines.

**B. In this repo (`monitoring/`) — scrape**

4. **Add** the `listings-api` **scrape job** in `prometheus/prometheus.yml`
   (the skeleton already shows the exact keys, including the `service`
   label). **Reload** with `curl -X POST http://localhost:9090/-/reload` (or
   `docker compose restart prometheus`).
   → Prometheus only scrapes the targets it is told about; without the job,
   the app's metrics exist but nobody stores them.

### Expected behavior
- `GET http://localhost:8080/actuator/prometheus` returns Prometheus
  **text exposition** format (`# HELP` / `# TYPE` lines).
- The default HTTP metrics are present (`http_server_requests_seconds_count`,
  `_sum`, and `_bucket`), with `uri=`, `status=`, `application="listings"`.
- The **business** metric `listings_creations_total` **increases by one per
  successful `POST /listings`**.
- **Prometheus scrapes** the app: the series are visible via the **Prometheus
  query API** with `service="listings-api"` (added by the scrape job).

### Verify

- [ ] [`docs/verification.md` §2](docs/verification.md): every result matches *Expected behavior*
- [ ] `up{job="listings-api"}` returns `1` (the scrape job works, not only the endpoint)

<details>
<summary><strong>If you are stuck</strong></summary>

- If the Prometheus query returns `[]`, the **scrape job** is the usual cause — confirm the job's `static_configs` points at `listings-api:8080` (container name, not `localhost`), then **reload**. The Prometheus UI page *Status → Targets* shows the scrape state and the last error.
- `percentiles-histogram` is a map keyed by meter name: set it in `application.yml`. As an environment variable, the dots of `http.server.requests` cannot be expressed and the app fails to start.
- A meter name ending in `created` is published without that suffix (`_created` is reserved in the Prometheus format): that is why TP1 uses `listings.creations`.
- Reference: <https://docs.spring.io/spring-boot/4.1/reference/actuator/metrics.html>
- (Panel-building is **Step 6a** — here you only prove the series is **scraped** by asking Prometheus directly, not Grafana.)

</details>

---

## Step 4 — Wire distributed tracing → Collector → Tempo   CORE

**Outcome** — Every request is **traced**: a `POST /listings` produces a
**business span named `listings.create`** (not only the auto HTTP span), that
span is **stored in Tempo** (returned by `…/api/traces/<traceId>` and by a
search), and every trace keeps a stable **`trace_id`**. You open a slow
request in Tempo and see *exactly which operation took the time*.

**Why this step** — A metric says *that* requests are slow; a trace says
*where* the time went in one request. Traces are **pushed** through a
collector, so the app does not need to know which tracing backend is used.

### How it fits

```
TP1 repo                                 this repo
listings-api ── OTLP/HTTP ──► OTel Collector :4318 ── traces pipeline ──► Tempo :3200 ◄── Grafana
 (group A: starter,           receivers: otlp        exporters: otlp/tempo           data source
  OTEL_* env, span)           (group B, item 4)      (group B, item 4)              (group C, item 5)
```

### Build it

**A. In the TP1 repo (`../springboot`) — produce spans**

1. **Add** the Spring Boot starter `spring-boot-starter-opentelemetry` to
   `pom.xml` (no `<version>`).
   → It brings Micrometer Tracing, its OpenTelemetry bridge, and the OTLP
   exporter in one dependency. Without a tracer on the classpath, no span is
   created and nothing is sent.
2. **Set** the four `OTEL_*` environment variables on `listings-api` in the
   base compose (block in **README → "0) Required base edit"**): service
   name, OTLP endpoint `http://otel-collector:4318`, sampler `always_on`, and
   no OTLP metrics export.
   → Spring Boot maps these standard variables to its own settings. The
   endpoint is the Collector's **HTTP** port by network name (`localhost` is
   the app's own container; 4317 speaks gRPC only). Without `always_on`,
   Spring Boot samples **10 %** of requests and your test request is probably
   not traced. Without disabling metrics export, the app also pushes OTLP
   metrics that the Collector has no pipeline for.
3. **Record an explicit span** named `listings.create` around the create
   path in `ListingServiceImpl`, with Micrometer's `Tracer`.
   → The automatic HTTP span only says "POST /listings took 120 ms"; a
   business span isolates the part of the work you care about.

**B. In this repo (`monitoring/`) — relay to Tempo**

4. **Add** the `otlp/tempo` **exporter** in
   `otel/collector-config.yaml` (`endpoint: tempo:4317`, TLS disabled) and
   put it on the `traces` pipeline.
   → The Collector receives spans but drops them if no exporter is listed in
   the pipeline. Between the Collector and Tempo, gRPC on 4317 is fine.

> **Checkpoint** — after a `POST /listings`,
> `curl -s "http://localhost:3200/api/search?tags=service.name%3Dlistings-api"`
> returns at least one trace.

**C. In this repo (`monitoring/`) — read in Grafana**

5. **Add** the **Tempo** data source in
   `grafana/provisioning/datasources/datasources.yml`
   (`type: tempo`, `url: http://tempo:3200`), or in the Grafana UI.
   → Tempo stores traces whether or not Grafana knows it exists; the data
   source is what lets the Step 6a dashboard show them.

### Expected behavior
- A `POST /listings` produces a **span named `listings.create`** (you added it
  explicitly — it is **not** only the auto-generated HTTP span
  `http post /listings`).
- That span is **stored in Tempo**: `GET …:3200/api/traces/<traceId>` returns
  it, and a Tempo **search** (last 10 min, `service.name=listings-api`)
  returns the trace.
- Each trace keeps a **stable `trace_id`** for the whole request.
- Every request is traced (`always_on`), not a sample.
- The Tempo backend is **already up** (Step 1) — you **only wire the Collector
  to it** and **add the data source**; you do not bring Tempo up.

### Verify

- [ ] [`docs/verification.md` §3](docs/verification.md): every result matches *Expected behavior*
- [ ] `grep -rn "listings.create\|nextSpan\|Tracer" "$TP1_DIR/src"` — the explicit span is in **your** code, not only the auto HTTP span

<details>
<summary><strong>If you are stuck</strong></summary>

- `Tracer#nextSpan().name(…)`, `start()`, `end()` in a `finally`, and `Tracer#withSpan(span)` so the log lines written inside carry its id. `@Observed` (with the `ObservedAspect`) is an alternative. Reference: <https://docs.spring.io/spring-boot/4.1/reference/actuator/tracing.html>
- If the search returns 0 traces: check the endpoint port (**4318**), the sampler (`always_on`), and the Collector pipeline's `exporters` line; then watch the Collector logs (`docker compose logs otel-collector`).
- A TraceQL search on the span name: `curl -s "http://localhost:3200/api/search" --data-urlencode 'q={name="listings.create"}' -G`.

</details>

---

## Step 5 — Collect logs → Alloy → Loki (with trace_id)   CORE

**Outcome** — The API's **log lines** are searchable in Loki, tagged with
`service` and a filterable `level`, and **each line carries the same
`trace_id`** as its Tempo trace (Step 4) — so you can **jump from a trace to
its logs**.

**Why this step** — Logs hold the detail metrics and traces leave out (the
error message, the input that failed). Centralised in Loki and stamped with
the `trace_id`, they stop being a file you `tail` on one machine and become
the third view of the same request.

### How it fits

```
TP1 repo                                 shared volume              this repo
listings-api ── Spring Boot file log ──► middleware-logs ──► Alloy ───── push ──► Loki :3100 ◄── Grafana
  LOGGING_FILE_NAME (base compose)       /listings-logs/            file_match → source.file      data source
  [traceId-spanId] in each line          listings.log               → process (level) → write    (group B, item 3)
  (group A)                                                         (group B, item 2)
```

### Build it

**A. In the TP1 repo (`../springboot`) — check the file**

1. **Check** that the app writes `/listings-logs/listings.log`. The base
   compose already sets `LOGGING_FILE_NAME` and mounts the shared volume
   `middleware-logs`; with Step 4 done, Spring Boot adds `[traceId-spanId]`
   to every line. No Logback XML is needed.
   → Alloy reads files, not the container's console: no file on the shared
   volume, nothing to ship. The `traceId` in the line is the correlation link
   with Tempo.

> **Important:** the log **file path is fixed** to
> `/listings-logs/listings.log` because that is the path **Alloy's container
> mounts** (see this repo's `docker-compose.yml`). Do not change it.

> **Checkpoint** — after some traffic,
> `docker compose exec listings-api tail -3 /listings-logs/listings.log`
> (from `$TP1_DIR`) shows lines like
> `2026-…Z  INFO 1 --- [listings] [http-nio-8080-exec-1] [<traceId>-<spanId>] c.a.l… : …`.

**B. In this repo (`monitoring/`) — ship and read**

2. **Add** the log pipeline in `alloy/config.alloy`, in front of the
   provided `loki.write` component (the skeleton shows the shape):
   `local.file_match` (the file + a `service` label), `loki.source.file`
   (tail it), and `loki.process` with a `stage.regex` that captures `level`
   and a `stage.labels` that turns it into a label.
   → An Alloy component does nothing until another one forwards to it: each
   block must point to the next with `forward_to`. Loki indexes labels only:
   without `service` the Loki query cannot select the stream, without
   `level` you cannot filter errors cheaply.
3. **Add** the **Loki** data source in
   `grafana/provisioning/datasources/datasources.yml`
   (`type: loki`, `url: http://loki:3100`), or in the Grafana UI.
   → Needed by the logs panel of Step 6a and the trace-to-logs jump of
   Step 7.
4. **Reload** Alloy (`curl -X POST http://localhost:12345/-/reload`), then
   generate some traffic on the app.
   → Alloy reads its configuration at start or on reload; a syntax error is
   reported in the reload answer and the previous config keeps running.

### Expected behavior
- A request's **log line is stored in Loki**: the Loki query
  `{service="listings-api"}` returns it.
- The line carries the **same `trace_id`** as the trace from Step 4
  (**correlation**).
- The line has a **filterable `level`** label (`INFO` / `WARN` / `ERROR`) —
  `{service="listings-api", level="ERROR"}` works.

### Verify

- [ ] [`docs/verification.md` §4](docs/verification.md): every result matches *Expected behavior*
- [ ] The `trace_id` of a log line **resolves** in the Tempo API (Step 4)

<details>
<summary><strong>If you are stuck</strong></summary>

- Alloy references: `local.file_match`, `loki.source.file`, `loki.process` in <https://grafana.com/docs/alloy/latest/reference/components/>.
- Spring Boot pads the level to 5 characters (`" INFO"`, `"ERROR"`): between the timestamp and the level there are **one or two** spaces. A regex with a single space misses every INFO and WARN line. Use `\s+`.
- `{service="listings-api"} |= "ERROR"` searches the text; `{…, level="ERROR"}` uses the label (faster, and only works if the regex stage extracted it).
- If Loki has **no** lines: the file is missing (check `LOGGING_FILE_NAME` in the base compose; `LOG_FILE` is not read by Spring Boot) or the Alloy pipeline is incomplete: the UI <http://localhost:12345> shows every component, its health, and its arguments (`docker compose logs alloy` for errors).
- Reference: <https://docs.spring.io/spring-boot/4.1/reference/features/logging.html>

</details>

---

## Step 6a — One dashboard over the three pillars   CORE

**Outcome** — **One Grafana dashboard** shows the three golden signals
(request rate, p95 latency, error rate) **plus** a logs panel and a traces
panel. This is where the three pillars you wired in Steps 3-5 **come
together**.

**Why this step** — Collected data is only useful if someone looks at it. The
dashboard is where you *diagnose* an incident, with the three pillars on one
screen.

### How it fits

```
Prometheus ──┬──► Grafana dashboard (UI): rate, p95, error rate     (Step 6a)    
             │
             └──► rules.yml: ApiDown, ApiHighErrorRate, ApiSlow     (Step 6b)          
                   evaluated every interval ── condition true for `for:` ──► state: firing
Loki  ───────────► Grafana dashboard: logs panel                    (Step 6a)    
Tempo ───────────► Grafana dashboard: traces panel                  (Step 6a)    
```

### Build it

This step is **in the Grafana UI** (no JSON committed):

1. **Build the dashboard in the Grafana UI** (Dashboards → New). Add panels
   one by one and pick the **right data source per panel**: **request rate**,
   **p95 latency**, **error rate** → **Prometheus**; a **logs** panel →
   **Loki**; a **traces** panel → **Tempo**. The data sources are **already
   wired** (Prometheus pre-provisioned, Tempo added Step 4, Loki added Step 5)
   — do **not** re-add them here.
   → Rate, latency, and errors are the "golden signals": together they tell
   whether users are affected; the logs and traces panels give the *why* on
   the same screen.
2. **Screenshot** the dashboard into your team README.
   → The dashboard lives in the Grafana UI only: the screenshot is the
   evidence that it existed and showed data.

### Expected behavior
- **One dashboard** (in the UI) with **5-6 self-built panels**: request rate,
  p95 latency, error rate (Prometheus) + a **logs** panel (Loki) + a
  **traces** panel (Tempo).

### Verify

- [ ] [`docs/verification.md` §5](docs/verification.md): every panel shows data after real traffic — **screenshot saved to the team README**

<details>
<summary><strong>If you are stuck</strong></summary>

- The p95 panel needs the `_bucket` series of Step 3 (percentile histogram): without them `histogram_quantile` returns nothing.

</details>

---

## Step 6b — Alerts as code   CORE

**Outcome** — **Three Prometheus alerts** (`ApiDown`, `ApiHighErrorRate`,
`ApiSlow`) exist in config-as-code, with at least **one proven firing**.

**Why this step** — Nobody watches a dashboard all night: alerts are for
*being told* an incident is happening. A rule is only trusted once it has been
seen firing.

### Build it

This step is **in this repo** (`prometheus/rules.yml`):

1. **Write the three alert rules** in `prometheus/rules.yml` (`ApiDown`,
   `ApiHighErrorRate`, `ApiSlow`), in **one group**
   (`listings-api`). Each rule has `alert`, `expr`, `for`, `labels`,
   `annotations.summary`.
   → Rules in a file are versioned and survive a `docker compose down`;
   `for:` avoids paging someone for a single bad scrape.
2. **Reload** Prometheus: `curl -X POST http://localhost:9090/-/reload` (or
   `docker compose restart prometheus`).
   → Prometheus reads `rules.yml` at start or on reload only; an edited file
   is otherwise ignored.
3. **Prove one rule fires** — the easiest is `ApiDown` (see below).
   → A rule that has never fired may have a wrong label in its `expr` and
   never fire in production either.

### Expected behavior
- **Three alert rules** in `prometheus/rules.yml`:
  - **`ApiDown`** — no scrape (`up{job="listings-api"} == 0`, `for: 1m`);
  - **`ApiHighErrorRate`** — 5xx spike
    (`sum(rate(http_server_requests_seconds_count{service="listings-api", status=~"5.."}[5m]))
      / sum(rate(http_server_requests_seconds_count{service="listings-api"}[5m])) > 0.05 for 1m`);
  - **`ApiSlow`** — p95 above 500 ms
    (`histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket
      {service="listings-api"}[5m])) by (le)) > 0.5 for 2m`).
- You **prove** at least one alert **fires**: after the condition,
  `curl :9090/api/v1/rules` shows a rule with **`state: firing`**.

### Verify

- [ ] [`docs/verification.md` §6](docs/verification.md): `ApiDown` reaches `state: firing` after `docker compose stop listings-api` (then `docker compose start listings-api`)
- [ ] (Optional) **`ApiHighErrorRate` firing**: send a steady mix of normal and failing requests for ~2 min (any TP1 call that answers 5xx; an **authenticated** request to an unknown path does, because TP1's catch-all handler maps it to 500 — anonymous, it is a 401)

<details>
<summary><strong>If you are stuck</strong></summary>

- **`ApiDown` is the easiest to prove in a lab** — the *target* goes down (`docker compose stop listings-api`), and after the rule's `for:` elapses its `state` flips to **`firing`**. Use a short `for: 1m` (or 30 s) so the proof is not slow.
- The three rules can share **one group** in `rules.yml`; the skeleton already shows the shape.
- `ApiSlow` and the p95 panel need the `_bucket` series of Step 3 (percentile histogram): without them `histogram_quantile` returns nothing and the alert can never fire.
- Full copy-pasteable walk to see each rule in `state: firing`: [`docs/verification.md` §6](docs/verification.md) (Alerts).

</details>

---

## Step 7 — Correlation & a first SLO   BONUS

**Outcome** — From a slow Tempo trace you **jump to its logs** (same
`trace_id`) and to the metric window around it, and you show **one SLO
panel** (a success-ratio between 0 and 1) computed from the metrics — so you
can answer **"is the API meeting its commitment?"**, not just "is it up?".

**Why this step** — In an incident, the time goes into switching tools and
copying identifiers. A one-click trace-to-logs link removes that step. An SLO
turns "the API is up" into a measurable promise: "99 % of requests succeed".

### How it fits

```
Tempo trace (trace_id = abc…) ── tracesToLogs mapping (Tempo data source) ──► Loki {service="listings-api"} |= "abc…"
Prometheus http_server_requests_seconds_count ──► SLO panel: 1 - (5xx / total)
```

### Build it

> Bonus — build **both** pieces, on the existing wiring from Steps 4-6.

1. **Demonstrate the Tempo → Loki jump**: open a Tempo trace, then use the
   Tempo data source's **traces-to-logs** mapping (via the `trace_id`) to
   open the matching Loki logs. Confirm the `trace_id` matches.
   → Without the mapping, Grafana shows the trace and the logs separately,
   and you copy the identifier by hand.
2. **Add an SLO panel** in the dashboard: a success-ratio from
   `http_server_requests_seconds_count` (e.g. `1 - (5xx / total)`).
   → Error *counts* grow with traffic; a *ratio* is comparable from one day
   to the next and against a target.

### Expected behavior
- **7.1** A Tempo trace **opens the matching Loki logs** (via the `trace_id`)
  — the link is demonstrated, not just asserted.
- **7.2** An **SLO panel** shows availability / success-ratio computed from
  `http_server_requests_seconds_count` (e.g. `1 - (5xx / total)`).

### Verify

- [ ] Both lines of *Expected behavior* are demonstrated in Grafana: the trace-to-logs jump (same `trace_id`) and an SLO value between 0 and 1

<details>
<summary><strong>If you are stuck</strong></summary>

- The Grafana Tempo data source has a **traces-to-logs** setting (`jsonData.tracesToLogsV2`): `datasourceUid` of Loki, `filterByTraceID: true`, and `tags` mapping the span attribute `service.name` to the Loki label `service` (the label keys differ, see *Conventions*). Reference: <https://grafana.com/docs/grafana/latest/datasources/tempo/configure-tempo-data-source/provision/>
- SLO ratio PromQL: `1 - (sum(rate(http_server_requests_seconds_count{status=~"5.."}[1h])) / sum(rate(http_server_requests_seconds_count[1h])))` — use a window that matches your data: `[1h]` in a lab, `[24h]` / `[30d]` in production.

</details>

---

## Step 8 — Two services + a "slow" dependency   BONUS

**Outcome** — A single request spans **two services** (or an in-app downstream
step), and the Tempo trace shows **both spans linked by the same `trace_id`**.
You can **attribute** part of the total latency to the child span — the
classic reason we **trace** instead of only counting requests.

**Why this step** — "Distributed" tracing earns its name across services: the
`trace_id` travels in the `traceparent` header, so one request through two
services is still one trace, and you see which service is slow.

### How it fits

```
POST /listings ──► listings-api (span listings.create)
                       │  HTTP call, header traceparent: <trace_id>-<span_id>
                       ▼
                   downstream (child span) ── both spans exported via the Collector ──► Tempo: one trace
```

### Build it

> Bonus — add one extra hop, keep it small.

1. **Extend** the traced path of Step 4 so a request makes **one more hop**:
   the simplest is an **in-app downstream call** (a `RestTemplate` /
   `WebClient` to another endpoint that is also traced), or a **real 2nd
   Spring Boot app** in Docker that shares the same OTLP endpoint and
   `traceparent` propagation.
   → The HTTP client must be the one Spring builds (instrumented): a client
   created with `new` does not add the `traceparent` header, and the child
   starts a new, unrelated trace.
2. **Keep** the same `service.name` tracing setup, then generate one request
   and **open** the Tempo trace.
   → Both spans must reach the same Tempo through the same Collector to be
   shown as one tree.

### Expected behavior
- **8.1** A request spans **two services** (e.g. `listings-api` →
  `geo-service`, or an in-app downstream step); the Tempo trace shows **both
  spans linked by the same `trace_id`**.
- **8.2** You can **attribute** part of the total latency to the child span
  (name which span took most of the time).

### Verify

- [ ] Both lines of *Expected behavior* are shown on one Tempo trace: 2+ spans sharing one `trace_id`, and the span that took most of the time

<details>
<summary><strong>If you are stuck</strong></summary>

- The simplest "second service" is an **in-app downstream call** — a `RestTemplate`/`WebClient` call to another endpoint, also traced. A full 2nd Spring Boot app in Docker works too (same OTLP endpoint, `traceparent` propagation), but keep it small.

</details>

---

## Step 9 — Final Integration   DELIVERY

**Outcome** — The whole stack boots cleanly from **your own configs** (no
solution files present): all six services healthy, TP1 on `8080`, traffic
yields **metrics in Prometheus, traces in Tempo, logs in Loki**, one dashboard
shows the three pillars, and `ApiDown` has been shown **firing**. The
evaluator can start the stack and **trust the output**.

**Why this step** — The evaluator starts your stack in your place. This step
proves it works from your committed configuration alone, and that you can
follow one request through the three pillars.

### Final checks
- [ ] `cd "$MON_DIR" && docker compose up -d` → **all six services healthy**; the stack boots from the **student-filled** configs (no solution files present)
- [ ] `cd "$TP1_DIR" && mvn verify && docker compose up -d` → TP1 builds and runs on `8080`; generating traffic yields **metrics in Prometheus, traces in Tempo, logs in Loki**
- [ ] **One** Grafana dashboard (in the UI, screenshot in the README) shows the three pillars; **three** alert rules in `prometheus/rules.yml` and **`ApiDown`** has been shown `firing`
- [ ] Your **five config files** (`prometheus/prometheus.yml`, `prometheus/rules.yml`, `otel/collector-config.yaml`, `alloy/config.alloy`, `grafana/provisioning/datasources/datasources.yml`) are in **this** repo; the **reference solutions** are **not**
- [ ] The team **README** documents: how to start the stack, the three pipelines, the ports, and one alert + one dashboard **screenshot**
- [ ] **Oral** (10 min + 5 Q&A): walk **one slow request** from the Grafana panel → the Tempo trace → the Loki log, explaining which middleware produced each part

<details>
<summary><strong>If you are stuck</strong></summary>

- The **team README** is where you write **what** you wired and **why** (not a copy of this exercise statement) — it is your **oral-prep**. If you cannot say *which* middleware produced the metrics / the trace / the log, write it down and re-check.
- The **smoke bar** is `docker compose up` green + the three backends returning data — you do **not** need new Java tests (see Overview).

</details>

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
| Traces | OTLP/HTTP (`otel-collector:4318`, network name) | **OTel Collector** | Tempo | **TraceQL** / HTTP API |
| Logs | file `/listings-logs/listings.log` (Spring Boot, `LOGGING_FILE_NAME`) | **Grafana Alloy** | Loki | **LogQL** |

## Appendix B - Useful endpoints

All the backend URLs (PromQL, LogQL, TraceQL, Tempo search/traces, alert rules,
Grafana) are listed run-ready in [`docs/verification.md`](docs/verification.md).

## Appendix C - What is NOT to be committed

- The **reference solutions** (live in `../../solutions/monitoring/`, used by the
  supervisor) must **not** be copied into this repo.
- Log files: the app writes to the shared volume `middleware-logs` (see the
  base compose in `../springboot`) — do not copy the file into the repo.
