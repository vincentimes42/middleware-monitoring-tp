# TP Monitoring / Observability

Follow-up to the Spring Boot TP (TP1, `../springboot/`): we **reuse the
TP1 listings API** and wire it into a **full observability stack** so every
request can be *measured*, *traced*, and *logged* across the three pillars.

- **Metrics** — Spring Actuator → (you write the scrape) → **Prometheus** → Grafana
- **Traces**  — Micrometer Tracing / OTel → (you wire the collector) → **Tempo** → Grafana
- **Logs**    — Spring Boot log file → (you wire Grafana Alloy) → **Loki** → Grafana

What you are given vs what you build:

| Given (boots out of the box) | **You build** (the point of the TP) |
|---|---|
| The 6 backend services (`docker-compose.yml`) | The **Prometheus scrape job** for the API |
| The three **stores** (Prometheus, Loki, Tempo) | The **Collector → Tempo** trace pipeline |
| Grafana (UI) | The **Alloy → Loki** log pipeline |
| `exercise.md`, `docs/verification.md` | The **datasources**, one **dashboard**, the **alert rules** |
| | The **TP1 instrumentation** (deps, code, log file) |

The **reference solutions** are **not in this repo** — they live in
`../../solutions/monitoring/` (supervisor use only).

## How the pieces fit (one shared network)

Like the Keycloak TP, this is **two Docker Compose projects on ONE shared
network** (`middleware`, created by the base in `../springboot/docker-compose.yml`):

| Project | Folder | Services | Host ports |
|---|---|---|---|
| base | `../springboot` | `listings-api`, `listings-postgres` | API `8080`, pg `5435` |
| this | `monitoring` | `prometheus`, `loki`, `tempo`, `otel-collector`, `alloy`, `grafana` | `9090/3100/3200/4317/4318/12345/3000` |

The app is on **8080** for this TP (and for every TP — even the Keycloak one,
where Keycloak itself takes `8081`). Services discover each other by
**container name** over the shared network:

- **Prometheus** scrapes the app at `listings-api:8080` (Step 3).
- **The app** pushes OTLP over HTTP to `otel-collector:4318` (Step 4).
- **Alloy** tails the log file on the shared volume `middleware-logs`
  (Step 5) — the app writes it there (see "required base edit" below).
- **The Collector** relays spans to `tempo:4317` (Step 4).

## Prerequisites

- Linux or WSL2, Bash terminal ([details](../springboot/README.md#operating-system-and-shell)).
- Docker + Docker Compose (`docker compose version`)
- JDK 27 + Maven 3.9+ (already installed for TP1)
- Ports free on the host: `3000` Grafana · `9090` Prometheus · `3100` Loki ·
  `3200` Tempo · `4317/4318` Collector · `12345` Alloy (UI) · `8080` the API ·
  `5435` the API's Postgres

> **Tip:** the four TP2s (Keycloak, Monitoring, IA, Messaging) are
> **alternatives** — they all use the base `listings-api` (same container
> name), so run **one** TP2 stack at a time.

## Run

> **Two repos, two folders.** The TP1 repo (the Spring Boot app) and this
> monitoring repo are **side by side** (any folder names). Set these once per
> terminal so the commands below do not need hard-coded paths:
>
> ```bash
> export TP1_DIR="/path/to/your/tps-1-repo"     # the Spring Boot app repo
> export MON_DIR="/path/to/monitoring-repo"     # this monitoring repo
> ```

### 0) Required base edit — `../springboot/docker-compose.yml`

So the app pushes traces to the collector, add to the `listings-api`
service's `environment` (the `LOGGING_FILE_NAME` env and the `listings-logs`
volume, used by Step 5, already exist in the base):

```yaml
  listings-api:
    environment:
      OTEL_SERVICE_NAME: "listings-api"
      # OTLP over HTTP (Spring Boot's default transport) to the collector's
      # HTTP port, by network name (not localhost, not the gRPC port 4317)
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4318"
      # trace every request (Spring Boot samples 10 % by default)
      OTEL_TRACES_SAMPLER: "always_on"
      # metrics go to Prometheus by scrape, not over OTLP
      OTEL_METRICS_EXPORTER: "none"
      # (already present in the base) LOGGING_FILE_NAME: "/listings-logs/listings.log"
```

Spring Boot 4 maps these standard `OTEL_*` variables to its own settings
(`management.opentelemetry.map-environment-variables`, on by default): there
is no tracing property to add in `application.yml`. They take effect with the
tracing starter of Step 4 (item 1); setting them now is harmless, and Step 4
item 2 only checks them.

### 1) Start the base (the API + its Postgres)

```bash
cd "$TP1_DIR"
docker compose up -d
curl -fsS http://localhost:8080/actuator/health     # API on 8080
```

### 2) Start the observability stack (this folder)

```bash
cd "$MON_DIR"
docker compose up -d
docker compose ps          # all 6 up + healthy
```

Backend readiness + Grafana/Prometheus/Loki/Tempo URLs are checked in
[`docs/verification.md` §1](docs/verification.md).

At this point the stack is **empty**: the three pipelines are **scaffolds**
you complete in Steps 3-6 (see `exercise.md`).

### 3) Instrument the API (TP1) — the real work

The instrumentation is **code in TP1** (`../springboot`), not an agent:

- add `micrometer-registry-prometheus`, expose `/actuator/prometheus`, and
  turn on the HTTP latency histogram (Step 3);
- add `spring-boot-starter-opentelemetry` + the `OTEL_*` env of §0 + one
  explicit span (Step 4);
- nothing to code for logs: Spring Boot already writes
  `/listings-logs/listings.log` (`LOGGING_FILE_NAME`), with the trace id in
  each line once tracing is on (Step 5).

`docker compose up -d --build listings-api` (from `../springboot`) to pick up
changes (a plain `restart` reruns the old image). Then generate traffic (`GET /listings`) — that is what the three
pipelines observe.

### 4) Stop

```bash
cd "$MON_DIR"    && docker compose down       # keep data (-v also deletes the Grafana dashboard built in the UI)
cd "$TP1_DIR"    && docker compose down       # stop base (add -v to wipe DB)
```

## Tree

```
monitoring/
├── .gitignore
├── docker-compose.yml       # 6 backend services, attached to the shared `middleware`
├── prometheus/
│   ├── prometheus.yml        # SCAFFOLD — you add the listings-api scrape job (Step 3)
│   └── rules.yml             # SCAFFOLD — you add the alert rules (Step 6b)
├── otel/
│   └── collector-config.yaml # SCAFFOLD — you add the OTLP→Tempo exporter (Step 4)
├── alloy/
│   └── config.alloy          # SCAFFOLD — you add the log pipeline to Loki (Step 5)
├── grafana/
│   └── provisioning/datasources/datasources.yml  # SCAFFOLD — you add Loki+Tempo (Step 5/6)
├── loki/loki-config.yaml     # store (fixed) — file storage
├── tempo/tempo.yaml          # store (fixed) — file storage + OTLP receiver
├── docs/primer.md            # observability, the six services, what not to do
├── docs/verification.md      # live per-pillar check + copy-paste curl
├── README.md                 # this file
└── exercise.md               # the statement (core 1-6 + bonus 7-8 + delivery 9)
```

> **Solutions** (supervisor only, NOT shipped): `../../solutions/monitoring/`.

## Useful links

- Prometheus: <https://prometheus.io/docs/introduction/overview/>
- Loki: <https://grafana.com/docs/loki/latest/>
- Tempo: <https://grafana.com/docs/tempo/latest/>
- OpenTelemetry Collector: <https://opentelemetry.io/docs/collector/>
- Spring Boot 4.1 — metrics: <https://docs.spring.io/spring-boot/4.1/reference/actuator/metrics.html>
- Spring Boot 4.1 — tracing (OpenTelemetry): <https://docs.spring.io/spring-boot/4.1/reference/actuator/tracing.html>
- Spring Boot 4.1 — logging (file, correlation): <https://docs.spring.io/spring-boot/4.1/reference/features/logging.html>
- Grafana data sources: <https://grafana.com/docs/grafana/latest/datasources/>
- Grafana Alloy (log pipeline, replaces Promtail): <https://grafana.com/docs/alloy/latest/>

> TP rule: no `TODO`, no stub, no secret in production code.
> The `admin/admin` credentials are for the academic environment only.
