# TP Monitoring / Observability

Follow-up to the Spring Boot TP (TP1, `../springboot/`): we **reuse the
TP1 listings API** and wire it into a **full observability stack** so every
request can be *measured*, *traced*, and *logged* across the three pillars.

- **Metrics** — Spring Actuator → (you write the scrape) → **Prometheus** → Grafana
- **Traces**  — Micrometer / OTel → (you wire the collector) → **Tempo** → Grafana
- **Logs**    — Logback file → (you wire Promtail) → **Loki** → Grafana

What you are given vs what you build:

| Given (boots out of the box) | **You build** (the point of the TP) |
|---|---|
| The 6 backend services (`docker-compose.yml`) | The **Prometheus scrape job** for the API |
| The three **stores** (Prometheus, Loki, Tempo) | The **Collector → Tempo** trace pipeline |
| Grafana (UI) | The **Promtail → Loki** log pipeline |
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
| this | `monitoring` | `prometheus`, `loki`, `tempo`, `otel-collector`, `promtail`, `grafana` | `9090/3100/3200/4317/4318/3000` |

The app is on **8080** for this TP (and for every TP — even the Keycloak one,
where Keycloak itself takes `8081`). Services discover each other by
**container name** over the shared network:

- **Prometheus** scrapes the app at `listings-api:8080` (Step 3).
- **The app** pushes OTLP gRPC to `otel-collector:4317` (Step 4).
- **Promtail** tails the log file on the shared volume `middleware-logs`
  (Step 5) — the app writes it there (see "required base edit" below).
- **The Collector** relays spans to `tempo:4317` (Step 4).

## Prerequisites

- Docker + Docker Compose (`docker compose version`)
- JDK 27 + Maven 3.9+ (already installed for TP1)
- Ports free on the host: `3000` Grafana · `9090` Prometheus · `3100` Loki ·
  `3200` Tempo · `4317/4318` Collector · `8080` the API · `5435` the API's Postgres

> **Tip:** Keycloak and Monitoring are **alternatives** — both start the base
> `listings-api` (same container name), so run **one** at a time.

## Run

### 0) Required base edit — `../springboot/docker-compose.yml`

So the app pushes traces to the collector and writes the shared log file, add
to the `listings-api` service's `environment` (the `LOG_FILE` env and the
`listings-logs` volume already exist in the base):

```yaml
  listings-api:
    environment:
      # push traces to the collector over the shared network name (not localhost)
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4317"
      OTEL_SERVICE_NAME: "listings-api"
      # (already present in the base) LOG_FILE: "/listings-logs/listings.log"
```

### 1) Start the base (the API + its Postgres)

```bash
cd ../springboot
docker compose up -d
curl -fsS http://localhost:8080/actuator/health     # API on 8080
```

### 2) Start the observability stack (this folder)

```bash
cd monitoring
docker compose up -d
docker compose ps          # all 6 up + healthy
```

Backend readiness + Grafana/Prometheus/Loki/Tempo URLs are checked in
[`docs/verification.md` §1](docs/verification.md).

At this point the stack is **empty**: the three pipelines are **scaffolds**
you complete in Steps 3-6 (see `exercise.md`).

### 3) Instrument the API (TP1) — the real work

The instrumentation is **code in TP1** (`../springboot`), not an agent:

- add the metrics + tracing dependencies (Step 3 & 4);
- expose `/actuator/prometheus` and record a business metric (Step 3);
- enable the OTel OTLP exporter to `otel-collector:4317` + one span (Step 4);
- add a Logback **file appender** to `${LOG_FILE}` =
  `/listings-logs/listings.log` (the shared volume Promtail tails) in Step 5.

`docker compose restart listings-api` (from `../springboot`) to pick up
changes. Then generate traffic (`GET /listings`) — that is what the three
pipelines observe.

### 4) Stop

```bash
cd monitoring   && docker compose down        # keep data
cd ../springboot && docker compose down       # stop base (add -v to wipe DB)
```

## Tree

```
monitoring/
├── .gitignore
├── docker-compose.yml       # 6 backend services, attached to the shared `middleware`
├── prometheus/
│   ├── prometheus.yml        # SCAFFOLD — you add the listings-api scrape job (Step 3)
│   └── rules.yml             # SCAFFOLD — you add the alert rules (Step 6)
├── otel/
│   └── collector-config.yaml # SCAFFOLD — you add the OTLP→Tempo exporter (Step 4)
├── promtail/
│   └── promtail-config.yaml  # SCAFFOLD — you add the listings log job (Step 5)
├── grafana/
│   └── provisioning/datasources/datasources.yml  # SCAFFOLD — you add Loki+Tempo (Step 5/6)
├── loki/loki-config.yaml     # store (fixed) — file storage
├── tempo/tempo.yaml          # store (fixed) — file storage + OTLP receiver
├── docs/verification.md      # live per-pillar check + copy-paste curl
├── README.md                 # this file
└── exercise.md               # the statement (8h, core 1-6 + bonus 7-8 + delivery 9)
```

> **Solutions** (supervisor only, NOT shipped): `../../solutions/monitoring/`.

## Useful links

- Prometheus: <https://prometheus.io/docs/introduction/overview/>
- Loki: <https://grafana.com/docs/loki/latest/>
- Tempo: <https://grafana.com/docs/tempo/latest/>
- OpenTelemetry Collector: <https://opentelemetry.io/docs/collector/>
- Micrometer Tracing + OTel bridge: <https://micrometer.io/docs/tracing>
- Grafana data sources: <https://grafana.com/docs/grafana/latest/datasources/>

> TP rule: no `TODO`, no stub, no secret in production code.
> The `admin/admin` credentials are for the academic environment only.
