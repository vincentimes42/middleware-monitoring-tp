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
curl -fsS http://localhost:9090/-/healthy        # Prometheus Server is Healthy.
curl -fsS http://localhost:3100/ready            # ready
curl -fsS http://localhost:3200/ready            # ready
curl -fsS http://localhost:12345/-/ready         # Alloy is ready.
# Grafana up AND provisioned (/api/health alone says nothing about data sources)
curl -fsS -u admin:admin http://localhost:3000/api/datasources \
  | python3 -c 'import sys,json;print([d["name"] for d in json.load(sys.stdin)])'
# Collector accepts OTLP/HTTP (GET / answers 404: that is normal)
curl -s -o /dev/null -w "collector OTLP/HTTP -> %{http_code}\n" \
  -X POST http://localhost:4318/v1/traces -H 'Content-Type: application/json' -d '{}'
```

Expected: "healthy" / "ready" / `['Prometheus']` (more after Steps 4-5) / `200`.

---

## 2) Metrics — app → Prometheus

Step 2 (backend only, before the app is wired): Prometheus scrapes itself.

```bash
curl -s --get http://localhost:9090/api/v1/query --data-urlencode 'query=up' \
  | python3 -c 'import sys,json;[print(r["metric"]["job"], r["value"][1]) for r in json.load(sys.stdin)["data"]["result"]]'
# expected: prometheus 1 (and listings-api 1 once Step 3 is done)
```

From Step 3 on, generate a few real HTTP requests **first**:

```bash
for i in $(seq 1 10); do
  curl -s -o /dev/null http://localhost:8080/listings
done
```

On the app side (Step 3, items 1-3):

```bash
curl -s http://localhost:8080/actuator/prometheus | grep -c '^http_server_requests_seconds_bucket'   # > 0
curl -s http://localhost:8080/actuator/prometheus | grep '^listings_creations_total'
```

Then **scrape** via Prometheus:

```bash
# The app is scraped at every 10s cycle.  Wait at least 12s.
sleep 15

# Raw (should list many http_server_requests_seconds_* lines)
curl -s --get "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=http_server_requests_seconds_count{service="listings-api"}' \
  | python3 -m json.tool | head -25

# Total number of requests counted by the app (scraped value)
curl -s --get "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=sum(http_server_requests_seconds_count{service="listings-api"})' \
  | python3 -c 'import sys,json;d=json.load(sys.stdin);[print(r["value"][1]) for r in d["data"]["result"]]'
```

Expected: a number > 0 in the last query.

```bash
# the business counter and the scrape target
for q in 'listings_creations_total{service="listings-api"}' 'up{job="listings-api"}'; do
  curl -s --get "http://localhost:9090/api/v1/query" --data-urlencode "query=$q" \
    | python3 -c 'import sys,json;[print(r["metric"].get("__name__"), r["value"][1]) for r in json.load(sys.stdin)["data"]["result"]]'
done
```

---

## 3) Traces — app → Collector → Tempo

Step 2 (backend only): the search answers a well-formed object, `traces`
empty before Step 4.

```bash
curl -s "http://localhost:3200/api/search?limit=1" | python3 -c 'import sys,json;d=json.load(sys.stdin);print(sorted(d), len(d["traces"]))'
# expected: ['metrics', 'traces'] 0   (before Step 4)
```

From Step 4 on, create a listing first (Step 4 adds the `listings.create` span
to that path), then wait ~10 s for the batch to flush. Each Prometheus scrape
is a request too: `http get /actuator/prometheus` traces appear every 10 s.

```bash
# recent traces of the service (last 10 min)
NOW=$(date +%s)
curl -s --get http://localhost:3200/api/search \
  --data-urlencode 'tags=service.name=listings-api' \
  --data-urlencode "start=$((NOW - 600))" --data-urlencode "end=$NOW" --data-urlencode 'limit=20' \
  | python3 -c 'import sys,json;[print(t["traceID"],"|",t.get("rootTraceName"),"|",t.get("rootServiceName")) for t in json.load(sys.stdin).get("traces",[])]'

# TraceQL: traces that contain the explicit span
TID=$(curl -s --get http://localhost:3200/api/search --data-urlencode 'q={name="listings.create"}' --data-urlencode 'limit=1' \
  | python3 -c 'import sys,json;t=json.load(sys.stdin)["traces"];print(t[0]["traceID"] if t else "")')
echo "trace with listings.create: $TID"

# the spans of that trace
curl -s http://localhost:3200/api/traces/$TID | python3 -c '
import sys,json; d=json.load(sys.stdin)
print([s["name"] for b in d["batches"] for ss in b["scopeSpans"] for s in ss["spans"]])'
```

Expected: lines with `rootTraceName` like `http post /listings` and
`rootServiceName` = `listings-api`; the span list contains `listings.create`.

If empty, check what the app and the Collector say:

```bash
(cd "$TP1_DIR" && docker compose exec listings-api env | grep ^OTEL_)   # the 4 variables of README §0
(cd "$TP1_DIR" && docker compose logs --since 5m listings-api | grep -iE "otlp|export" | tail -5)
docker compose logs --since 5m otel-collector | tail -5
```

---

## 4) Logs — app (file) → Alloy → Loki

> Two ways to prove Loki stores and returns a stream:
> - (a) **before the app is wired** (Step 2): push a single line through the
>   Loki **push API** below — the simplest proof the backend works. It has
>   its own label (`service="step2-sanity"`), so it never answers the app's
>   queries of (b).
> - (b) **after the app is wired** (Step 5): query the `service="listings-api"`
>   stream that Alloy ships from the app's log file.

```bash
# (a) Push ONE synthetic line straight to Loki (no app required):
curl -s -o /dev/null -w "loki-push:%{http_code}\n" \
  -X POST http://localhost:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -d '{"streams":[{"stream":{"service":"step2-sanity","level":"INFO"},
       "values":[["'"$(date +%s%N)"'","TP2 Step 2 sanity line"]]}]}'
# expected: loki-push:204 (a fixed old timestamp is rejected with 400: Loki
# refuses entries older than one week)
curl -s --get 'http://localhost:3100/loki/api/v1/query_range' --data-urlencode 'query={service="step2-sanity"}' \
  | python3 -c 'import sys,json;[print(v[1]) for s in json.load(sys.stdin)["data"]["result"] for v in s["values"]]'
# expected: TP2 Step 2 sanity line

# (b) Query Loki for any line with service=listings-api (Step 5 stream):
curl -fsS http://localhost:12345/-/ready                      # Alloy is ready.
# every component of your pipeline must be "healthy"
curl -s http://localhost:12345/api/v0/web/components \
  | python3 -c 'import sys,json;[print(c["localID"], c["health"]["state"]) for c in json.load(sys.stdin)]'

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

Expected (Step 5): >= 1 stream labelled `service=listings-api`, with the
`filename` / `job` labels of the Alloy pipeline (nothing before Step 5).

If **empty**, the app has not written to the shared log volume
(`/listings-logs/listings.log`). Verify:

```bash
(cd "$TP1_DIR" && docker compose exec -T listings-api tail -3 /listings-logs/listings.log) \
  || echo "file missing - check LOGGING_FILE_NAME in the base compose"
docker compose logs --tail 20 alloy

# level label (Step 5): both must return lines. ERROR needs a 500 logged at
# ERROR (Step 5 item 2); cause one: the database down for one request
(cd "$TP1_DIR" && docker compose stop listings-postgres)
curl -s -o /dev/null -m 40 -w "%{http_code}\n" http://localhost:8080/listings   # 500 (after ~30 s)
(cd "$TP1_DIR" && docker compose start listings-postgres); sleep 10
for lvl in INFO ERROR; do
  curl -s --get 'http://localhost:3100/loki/api/v1/query_range' \
    --data-urlencode "query={service=\"listings-api\", level=\"$lvl\"}" --data-urlencode 'limit=1' \
    | python3 -c "import sys,json;r=json.load(sys.stdin)['data']['result'];print('$lvl:', r[0]['values'][0][1][:140] if r else 'no line')"
done
```

---

## 5) Grafana dashboards (you build them)

> **No starter dashboard JSON ships in this repo** — you create the dashboards
> as part of exercise.md **Step 6a** ("One dashboard"). The `providers.yml`
> only configures the "Listings TP" **folder** and the datasource wiring.

Open <http://localhost:3000> with `admin`/`admin`. In **Step 6a** you will
create one dashboard in the "Listings TP" folder with panels for:
request rate, p95 latency, error rate (Prometheus); a logs panel (Loki); and
a traces panel (Tempo).

Build each panel with the **Data source picker** (Prometheus / Loki / Tempo)
and an expression (PromQL / LogQL / TraceQL). See exercise.md **Step 6a**.

---

## 6) Alerts — Prometheus rules

These rules are the ones **you author in Step 6b** (`prometheus/rules.yml`),
named `ApiDown`, `ApiHighErrorRate`, `ApiSlow`.

```bash
# List loaded alert rules + state
curl -s http://localhost:9090/api/v1/rules | python3 -c '
import sys,json
d=json.load(sys.stdin)["data"]["groups"]
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
(cd "$TP1_DIR" && docker compose stop listings-api)
# The rule's `for:` (1m) + up to 10 s to miss a scrape + up to 10 s to
# evaluate: poll until `firing` (pending first), at most 3 min
for i in $(seq 18); do
  sleep 10
  S=$(curl -s http://localhost:9090/api/v1/rules \
    | python3 -c 'import sys,json;print(" ".join(r["name"]+"="+r["state"] for g in json.load(sys.stdin)["data"]["groups"] for r in g["rules"]))')
  echo "$((i*10))s: $S"; case "$S" in *ApiDown=firing*) break;; esac
done

# Restore the app:
(cd "$TP1_DIR" && docker compose start listings-api)
```

`ApiHighErrorRate` (optional): a real dependency failure under load. With the
database down, every request waits 30 s for a connection, then answers 500
(an unknown path is a 404, not a 5xx). `ApiSlow` fires too: the p95 is the
30 s wait.

```bash
(cd "$TP1_DIR" && docker compose stop listings-postgres)
for w in $(seq 8); do        # 8 clients in parallel for 3 min
  ( end=$((SECONDS + 180)); while [ $SECONDS -lt $end ]; do
      curl -s -o /dev/null -m 40 http://localhost:8080/listings; done ) &
done; wait
curl -s http://localhost:9090/api/v1/rules \
  | python3 -c 'import sys,json;[print(r["name"],r["state"]) for g in json.load(sys.stdin)["data"]["groups"] for r in g["rules"]]'
# expected: ApiHighErrorRate firing (ApiSlow pending, then firing)
(cd "$TP1_DIR" && docker compose start listings-postgres)
```

---

## 7) Stop

Use the runbook in `../README.md` → "Stop" (`docker compose down` to keep data,
`docker compose down -v` to wipe and re-import the provision on next start).
`-v` also deletes the Grafana volume, hence the dashboard you built in the UI
(Step 6a): take its screenshot first.
