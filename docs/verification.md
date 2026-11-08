# Monitoring / observability — stack verification

This file is a **live verification** of the full observability stack.
After `docker compose up -d` (see `../README.md`), run each section
in order. Every section ends with a `curl` that should return data —
that is the evidence that the pipeline actually works.

> **Prerequisite:** the Spring Boot TP1 app (base compose in your `springboot`
> repo) is up at `http://localhost:8080/actuator/health` and instrumented
> (metrics, tracing, log file). See **README → Run** for the required base
> edits. If the app is not up yet, run Section 1 only — the stack starts
> regardless, and the backends stay healthy with zero inbound data.
>
> **Paths:** the commands below use two variables, set once per terminal
> (from the "Run" box of [`README.md`](../README.md)):
>
> ```bash
> export TP1_DIR="/path/to/your/tps-1-repo"     # the Spring Boot app repo
> export MON_DIR="/path/to/monitoring-repo"     # this monitoring repo
> ```

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
  curl -s -o /dev/null http://localhost:8080/listings
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
(e.g. `http_server / GET /listings`), `rootServiceName` = `listings-api`.

If empty, check whether the app is actually exporting:

```bash
# From the host, look at the app log for OTel SDK / exporter lines
grep -i "otel\|otlp\|exporter\|trace" <tp1-app.log> 2>/dev/null | tail -10 || \
  echo "No OTel lines yet - check the app's tracing deps + otel.* config (Step 4)."
```

---

## 4) Logs — app (file) → Promtail → Loki

> Two ways to prove Loki stores and returns a stream:
> - (a) **before the app is wired** (Step 2): push a single line through the
>   Loki **push API** below — the simplest proof the backend works.
> - (b) **after the app is wired** (Step 5): query the `service="listings-api"`
>   stream the app's file appender feeds Promtail.

```bash
# (a) Push ONE synthetic line straight to Loki (no app required):
curl -s -o /dev/null -w "loki-push:%{http_code}\n" \
  -X POST http://localhost:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -d '{"streams":[{"stream":{"service":"listings-api","level":"INFO"},
       "values":[["1700000000000000000","TP2 Step 2 sanity line"]]}]}'

# (b) Query Loki for any line with service=listings-api (Step 5 stream):
docker logs --tail 20 mon-promtail 2>&1 | grep -i "tailer\|new\|watch"

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

Expected: >= 1 stream labelled `service=listings-api` (the synthetic push line
in (a) is guaranteed; the app stream in (b) appears once Step 5 is done).

If **empty**, the app has not written to the shared log volume
(`/listings-logs/listings.log`). Verify:

```bash
docker compose \
  --project-name monitoring-tp exec -T mon-promtail sh \
  -c "ls -l /listings-logs && tail -5 /listings-logs/listings.log" \
  || echo "file missing - not yet instrumented"
docker logs --tail 20 mon-promtail 2>&1
```

---

## 5) Grafana dashboards (you build them)

> **No starter dashboard JSON ships in this repo** — you create the dashboards
> as part of exercise.md **Step 6** ("Dashboards & alerts"). The `providers.yml`
> only configures the "Listings TP" **folder** and the datasource wiring.

Open <http://localhost:3000> with `admin`/`admin`. In **Step 6** you will
create one dashboard in the "Listings TP" folder with panels for:
request rate, p95 latency, error rate (Prometheus); a logs panel (Loki); and
a traces panel (Tempo).

Build each panel with the **Data source picker** (Prometheus / Loki / Tempo)
and an expression (PromQL / LogQL / TraceQL). See exercise.md **Step 6**.

---

## 6) Alerts — Prometheus rules

These rules are the ones **you author in Step 6** (`prometheus/rules.yml`),
named `ApiDown`, `ApiHighErrorRate`, `ApiSlow`.

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

Expected: the **3 rules you wrote** in `listings-api` (or a group of your
choosing): **`ApiDown`**, **`ApiHighErrorRate`**, **`ApiSlow`**.

To **prove one fires** during the lab (the easiest is `ApiDown`):

```bash
# Stop the target -> the scrape goes down -> `up{job="listings-api"}` == 0
cd "$TP1_DIR" && docker compose stop listings-api
# Wait for the rule's `for:` (e.g. 1m) + ~30s, then check the state:
sleep 90
curl -s http://localhost:9090/api/v1/rules | python3 -m json.tool | grep -A1 ApiDown

# Restore the app:
docker compose start listings-api
```

---

## 7) Stop

Use the runbook in `../README.md` → "Stop" (`docker compose down` to keep data,
`docker compose down -v` to wipe and re-import the provision on next start).
