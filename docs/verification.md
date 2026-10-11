# Monitoring / observability — stack verification

This file is a **live verification** of the full observability stack.
After `docker compose up -d` (see `../README.md`), run each section
in order. Every section ends with a `curl` that should return data —
that is the evidence that the pipeline actually works.

> **Prerequisite:** the Spring Boot TP1 app is instrumented (metrics, tracing,
> log file) and running on the host at `http://localhost:8081/actuator/health`.
> See `../README.md` §2 for the instrumentation. If the app is not up yet, run
> Section 1 only — the stack starts regardless, and the backends stay healthy
> with zero inbound data.

---

## 1) All backends up

```bash
docker compose ps
curl -fsS http://localhost:3000/api/health
curl -fsS http://localhost:9090/-/healthy
curl -fsS http://localhost:3100/ready
curl -fsS http://localhost:3200/ready
curl -fsS -o /dev/null -w "collector-OTLP-HTTP-%{http_code}\n" http://localhost:4318/
```

Expected: all 200 / "healthy" / "ready".

---

## 2) Metrics — app → Prometheus

Generate a few real HTTP requests **first**:

```bash
for i in $(seq 1 10); do
  curl -s -o /dev/null http://localhost:8081/listings
  curl -s -o /dev/null http://localhost:8081/create
done
```

Then **scrape** via Prometheus:

```bash
# The app is scraped at every 10s cycle.  Wait at least 12s.
sleep 15

# Raw (should list many http_server_requests_seconds_* lines)
curl -s --get "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=http_server_requests_seconds_count{service="listings-api"}' \
  | python3 -m json.tool | head -25

# Sum of requests since first scrape
curl -s --get "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=sum(increase(http_server_requests_seconds_count{service="listings-api"}[5m]))' \
  | python3 -c 'import sys,json;d=json.load(sys.stdin);[print(r["value"][1]) for r in d["data"]["result"]]'
```

Expected: a number > 0 in the last query.

---

## 3) Traces — app → Collector → Tempo

Wait ~10s after generating traffic for the batch to flush.

```bash
# Search for recent traces in Tempo
NOW=$(date -u +%s)
START=$((NOW - 600))
curl -s -X POST http://localhost:3200/api/search \
  -H "Content-Type: application/json" \
  -d '{"start":"'"$(date -u -d "@$START" +%FT%TZ)"'","end":"'"$(date -u -d "@$NOW" +%FT%TZ)"'\
       "limit":5,"tags":{"service.name":"listings-api"}}' \
  | python3 -c 'import sys,json;d=json.load(sys.stdin);[print(t.get("traceID") or t.get("traceId"),"|",t.get("rootTraceName"),"|",t.get("rootServiceName")) for t in d.get("traces",[])]'
```

Expected: at least one line with `traceID`, `rootTraceName`
(e.g. `GET /create`), `rootServiceName` = `listings-api`.

If empty, check whether the app is actually exporting:

```bash
# From the host, look at the app log for OTel SDK / exporter lines
grep -i "otel\|otlp\|exporter\|trace" <tp1-app.log> 2>/dev/null | tail -10 || \
  echo "No OTel lines yet - check the app's tracing deps + otel.* config (Step 4)."
```

---

## 4) Logs — app (file) → Promtail → Loki

```bash
# Check that Promtail is running and watching
docker logs --tail 20 mon-promtail 2>&1 | grep -i "tailer\|new\|watch"

# Query Loki for any line with service=listings-api
curl -s 'http://localhost:3100/loki/api/v1/query_range' \
  --data-urlencode "query={service=\"listings-api\"}" \
  --data-urlencode "direction=BACKWARD" \
  --data-urlencode "limit=10" \
  | python3 -c '
import sys,json
d=json.load(sys.stdin)["data"]["result"]
print("streams:", len(d))
for s in d:
    print("  ", s["stream"])
    for ts,l in s.get("values",[])[:3]:
        print("   ", l[:120])
'
```

Expected: >= 1 stream labelled `service=listings-api`, with log
lines matching what the app printed in the terminal.

If **empty**, the app has not written to the shared `logs/listings.log`.
Verify:

```bash
ls -la logs/
tail -5 logs/listings.log 2>/dev/null || echo "file missing - not yet instrumented"
docker logs mon-promtail 2>&1 | tail -10
```

---

## 5) Grafana dashboard (starter)

Open <http://localhost:3000> with `admin`/`admin`. You should see a
folder **"Listings TP"** containing the **3-pillars starter dashboard**:

- Panel "Requests / Errors / p95 latency / JVM heap" (metrics, from Prometheus)
- Panel "Logs (listings-api)" (logs, from Loki)
- Panel "Traces" (traces, from Tempo)

Generate traffic again (Section 2) and refresh the dashboard: the
"Requests" panel should start showing time series.

- To **build your own panel**, use Data source picker (Prometheus / Loki / Tempo)
  and any expression (PromQL / LogQL / TraceQL). See exercise.md Step 5.

---

## 6) Alerts — Prometheus rules

```bash
# List loaded alert rules + state
curl -s http://localhost:9090/api/v1/rules | python3 -c '
import sys,json
d=json.load(sys.stdin)["groups"]
for g in d:
    for r in g.get("rules",[]):
        print(g["name"], ":", r.get("name"), "->", r.get("state"), "l=", r["labels"])
'
```

Expected: 3 scaffold rules in `listings-api-alerts`:
`ApiHighErrorRate`, `ApiSlow`, `ApiDown`.

To **force a firing** during the lab:

```bash
# Trigger a real 500 from the app (if TP1 has a /internal/fail endpoint)
# or generate enough traffic so a rule fires.
# Then the rule's `state` will move: pending -> firing
# and the alert will appear in Prometheus UI: Alerts/Active.
```

---

## 7) Clean state

```bash
docker compose down          # keep data
docker compose down -v       # wipe
```
