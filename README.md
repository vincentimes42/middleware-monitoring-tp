# TP Monitoring / Observability — identity stack

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

---

## Prerequisites

- Docker + Docker Compose (`docker compose version`)
- JDK 17 or later + Maven 3.9+ (already installed for TP1)
- Ports available on the host:
  - `3000` Grafana · `9090` Prometheus · `3100` Loki · `3200` Tempo
  - `4317` / `4318` OTel Collector (OTLP gRPC / HTTP) — the app pushes here
  - `8081` — the Spring Boot app

> **Tip:** this TP is one of two follow-ups after TP1. The other is
> [TP Keycloak](../keycloak/). They both use port `3000`, so **do not run
> them at the same time**.

## Run

### 1) The backend stack (boots as-is)

```bash
cd monitoring
docker compose up -d
docker compose ps          # all 6 up
```

```bash
# backend readiness (no auth needed)
curl -fsS http://localhost:9090/-/healthy
curl -fsS http://localhost:3100/ready
curl -fsS http://localhost:3200/ready
# Grafana
curl -fsS http://localhost:3000/api/health
```

- Grafana <http://localhost:3000> (`admin` / `admin`)
- Prometheus <http://localhost:9090> — Loki <http://localhost:3100> — Tempo <http://localhost:3200>

At this point the stack is **empty**: the three pipelines are **scaffolds**
you complete in Steps 3-6 (see `exercise.md`).

### 2) Instrument the API (TP1) — the real work

The instrumentation is **code in TP1** (not an agent). In `../springboot`:

- add the metrics + tracing dependencies (Step 3 & 4);
- expose `/actuator/prometheus` and record a business metric (Step 3);
- enable an OTel OTLP exporter to `localhost:4317` + one span (Step 4);
- add a Logback **file appender** to `logs/listings.log` (Step 5);
- run on port **8081**: `mvn spring-boot:run` (or `java -jar`).

Then generate traffic (`GET /listings`, `POST /listings`) — that is what the
three pipelines observe.

### 3) Stop

```bash
docker compose down        # keep data
docker compose down -v     # wipe metrics, traces, logs
```

## Tree

```
monitoring/
├── .gitignore
├── docker-compose.yml       # 6 services: prom, loki, tempo, collector, promtail, grafana
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
├── logs/                     # SHARED SINK — the app writes listings.log here (gitignored)
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
