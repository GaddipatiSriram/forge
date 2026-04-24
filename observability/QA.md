# Observability Learning Q&A

Study log for understanding the observability stack we're building.
Questions to come back to as we iterate — each answered enough to
get started, linked to primary sources for depth.

---

## Q1. What does "observability" mean, and how is it different from "monitoring"?

**A.** Monitoring = "I set up alerts and dashboards for things I know
can go wrong." Observability = "I can ask arbitrary questions about
what the system is doing, including things I didn't think to measure
in advance."

The distinction matters because modern distributed systems fail in
ways nobody predicted. Observability is an infrastructure property:
your tooling needs to be rich enough to let you *explore* without
re-instrumenting.

Standard definition (Honeycomb / Charity Majors): observability is
the ability to explain any state your system can get into, based on
the output (metrics + logs + traces + events) without having to ship
new code.

---

## Q2. What are the "three pillars"?

**A.** Metrics, logs, traces. Each answers a different question:

| Pillar | Answers | Cardinality | Storage shape |
|---|---|---|---|
| **Metrics** | "how many / how fast / how full?" over time | low (hundreds of labels) | time-series DB (Prometheus, Mimir, VictoriaMetrics) |
| **Logs** | "what exactly happened?" for one event | infinite (full-text) | log aggregator (Loki, Elasticsearch) |
| **Traces** | "how did one request flow through services?" | per-request, hierarchical | trace DB (Tempo, Jaeger) |

Use together: a dashboard shows a latency spike (metric) → jump to
logs for that service in that window → find the trace ID of a slow
request → open trace to see which downstream call was the bottleneck.

### Newer additions
- **Events** (discrete occurrences: deployments, config changes).
  Often first-class in platforms like Honeycomb; ad-hoc in Loki.
- **Profiles** (CPU/memory flame graphs). Grafana Pyroscope /
  Parca. Gaining traction; 4th pillar in some models.

---

## Q3. Why not just use kube-prometheus-stack for everything?

**A.** The monolith chart bundles Prometheus + Grafana + Alertmanager
+ node-exporter + kube-state-metrics in one install. Great for single-
cluster POC; limiting for a real platform:

- **Multi-cluster**: the monolith assumes one cluster observes only
  itself. Receiving remote-write from N clusters needs the backend
  reconfigured.
- **Component lifecycle**: Grafana updates shouldn't restart
  Prometheus. Monolith couples them.
- **Team ownership**: at scale, metrics infra and dashboards are
  different teams.
- **Component swap**: outgrow Prometheus single-instance, swap to
  Mimir. Monolith makes this harder than it should be.

The LGTM stack (Loki/Grafana/Tempo/Mimir) from Grafana Labs is the
canonical "split backends" reference architecture. Our stack mirrors
it structurally (Prometheus can stand in for Mimir at homelab scale).

---

## Q4. Pull vs push — which does Prometheus use, and why does it matter?

**A.** Prometheus is **pull-based**: its server periodically scrapes
HTTP endpoints (`/metrics`) exposed by each target. This is unusual —
most other monitoring systems (Datadog, New Relic, Graphite) are push.

Reasons pull is a good default:

1. **Discovery**: Prometheus uses k8s API for service discovery; it
   knows what exists and whether it's up from the outside.
2. **Up/down signal**: if scrape fails, that IS a signal. With push,
   a silent app looks identical to a healthy app not sending.
3. **Backpressure**: Prometheus controls scrape rate; an app can't
   flood it.
4. **No agent on every node**: centralized server does the work.

Downsides:
1. **Firewalls**: Prometheus needs to reach every target. Problematic
   for short-lived jobs or NAT boundaries.
2. **Batch jobs**: a job that runs 10 seconds then exits can't be
   scraped in time. Solution: **Pushgateway** — the job pushes to
   Pushgateway, Prometheus scrapes Pushgateway.

**Remote-write** is the push-like mode: Prometheus forwards its
scraped data to a remote store (Mimir, Thanos, another Prometheus).
Use when you need multi-cluster aggregation. This is what agents on
workload clusters will do.

---

## Q5. Prometheus vs Mimir vs Thanos vs VictoriaMetrics?

**A.** All Prometheus-compatible long-term-storage options, different
trade-offs:

| Tool | Storage model | Scale | Ops complexity |
|---|---|---|---|
| **Prometheus** (single) | local disk / PVC | ~1 cluster | low |
| **Mimir** (Grafana) | object store + ring | horizontal scale | high (many microservices) |
| **Thanos** | object store + sidecars | horizontal scale | medium |
| **VictoriaMetrics** | own columnar store | vertical + horizontal | low (monolith deploy) |

For homelab → single Prometheus. For "we have 5 clusters but
retention < 6 months" → VictoriaMetrics single-node. For "we have
50 clusters, 2-year retention, 100 engineers querying" → Mimir.

Our current path: Prometheus on mgmt-observability (single instance)
with remote-write RECEIVE enabled. When we outgrow it (many clusters
remote-writing, long retention), swap backend to Mimir by changing
one chart. Grafana datasource URL stays the same.

---

## Q6. Loki vs Elasticsearch / OpenSearch for logs?

**A.** Both aggregate logs; they have very different storage models.

| Loki | Elasticsearch |
|---|---|
| Indexes labels only (namespace, pod, app) | Indexes every word |
| Full-text search = grep-over-compressed-blobs | Full-text search = fast inverted index |
| Cheap — ~10x less storage than ES | Expensive — shard-heavy |
| Query language LogQL (Prometheus-like) | Query language ES DSL / Lucene |
| Built by Grafana; fits LGTM stack | Generalist, fits ELK/Elastic stack |

**Rule of thumb**: if you mostly filter by known metadata (service,
namespace, severity) and only occasionally grep, Loki wins on cost.
If you run complex text searches routinely (security, forensics),
Elasticsearch is worth the cost.

For platform logs (our use case), **Loki**.

---

## Q7. Tempo vs Jaeger for traces?

**A.** Both are distributed trace backends. Tempo is the Grafana one;
Jaeger is the CNCF one (originally Uber).

| Tempo | Jaeger |
|---|---|
| Object-store backed | Cassandra / Elasticsearch |
| Trace-ID lookup only (correlated via metrics/logs) | Full query UI (service, tags, duration) |
| Simpler ops | More flexible querying |
| Integrates natively with Grafana | Has its own UI |

Modern pattern: **Tempo for storage, Grafana for UI, OpenTelemetry
for instrumentation**. Jaeger is older; still fine, but Tempo wins on
cost at scale.

---

## Q8. What's OpenTelemetry, and why does everyone talk about it?

**A.** OpenTelemetry (OTel) is **the vendor-neutral standard** for
generating and shipping telemetry (metrics, logs, traces — all
three). Before OTel: each app used the SDK of whatever backend you
picked (Datadog SDK, Jaeger SDK, Prometheus client). Changing
backends meant re-instrumenting. OTel standardizes:

- **SDK**: same instrumentation library across languages. Emit
  traces/metrics/logs in OTel format.
- **Collector**: a daemon that receives in OTel format and exports
  to anything (Prometheus, Loki, Tempo, Datadog, Splunk, etc.).
- **OTLP (OpenTelemetry Protocol)**: gRPC/HTTP wire format for
  telemetry.

Collector runs in two modes:
- **Agent** (DaemonSet on every workload cluster) — receives from
  pods, maybe pre-processes, forwards to gateway.
- **Gateway** (StatefulSet on mgmt-observability) — central point
  that fans out to backends (Loki, Tempo, Prometheus remote-write).

Our architecture uses both: agents push to gateway, gateway fans out.

**Why people talk about it**: it's the first truly vendor-neutral
telemetry standard that has caught on. Apps instrumented with OTel
can swap backends without code changes. Ten years from now,
instrumentation will be OTel-native by default; proprietary SDKs
will die.

---

## Q9. What's "cardinality" and why do people warn about it?

**A.** Cardinality = number of unique combinations of label values.

Prometheus stores each unique label combination as a separate
time-series. A metric `http_requests{method, status, path}` with
say 5 methods × 10 status codes × 1000 paths = 50,000 series.

If `path` is unbounded (includes user IDs or request IDs), you get a
cardinality explosion — billions of series — and Prometheus OOMs or
grinds to a halt.

**Rules**:
- **Never put unbounded values (UUIDs, user IDs, request IDs) in
  metric labels.** Put them in LOGS or TRACES, not metrics.
- Keep cardinality per metric under ~100k series in most cases.
- If you need per-user metrics, pre-aggregate: one counter per
  important bucket, not per user.

This is the single most common mistake in new Prometheus deployments.

---

## Q10. How does multi-cluster observability actually work?

**A.** Two patterns, with trade-offs:

### Pattern A — each cluster owns its telemetry, central federation

```
  Workload cluster 1:                    mgmt-observability:
    Prometheus (full) ────────federation──→ Prometheus (central)
    retains locally, serves local queries
```

Central Prometheus queries N Prometheus servers with PromQL
federation. Problems: brittle (remote Prometheus down = gap),
expensive (same data stored in 2 places), slow queries.

### Pattern B — agents with remote-write (what we'll use)

```
  Workload cluster 1:                    mgmt-observability:
    Prometheus agent ───remote-write───→   Prometheus / Mimir
    (no local retention,                   (central store)
     scrape + forward only)
```

Lightweight agent on each workload cluster. Central backend holds
ALL metrics. Queries are fast (one place). Retention policy central.

Pattern B is the modern default. Loki + Promtail, Tempo + OTel work
the same shape: ship from workload clusters to central backends.

---

## Q11. What's an SLI / SLO / SLA, and how do I pick one?

**A.** Google SRE book primer distilled:

- **SLI** (Service Level Indicator) — a metric that measures a
  user-visible quality. "Fraction of requests that succeed in
  <200ms."
- **SLO** (Service Level Objective) — the target you want to hit.
  "99.9% of requests succeed in <200ms over 30 days."
- **SLA** (Service Level Agreement) — contractual commitment,
  often money-backed. "If we miss 99.9%, you get a credit."

Rules of thumb:
- Pick SLIs the USER cares about, not what's easy to measure.
  Latency + success rate > CPU%.
- SLO < SLA always (always). The gap is your buffer.
- SLO targets 99% / 99.9% / 99.99% — each 9 is 10x harder. Don't
  promise 4 nines unless you have the architecture for it.
- **Error budget** = 1 - SLO. If SLO=99.9%, budget = 0.1% of 30
  days = 43 min of downtime/month. When you burn budget, freeze
  feature deploys and fix reliability.

Our homelab SLOs: don't have any yet. Eventually: "Keycloak login
success rate >99%" + "PVC provisioning <30s p95." Measure from
Prometheus; alert when burn rate is unsustainable.

---

## Q12. What's an ServiceMonitor vs PodMonitor vs ScrapeConfig?

**A.** Prometheus Operator CRDs for declaring scrape targets.

- **ServiceMonitor**: scrapes all pods behind a k8s Service. Most
  common; use when the app is exposed via Service.
- **PodMonitor**: scrapes pods directly, matched by labels. Use
  when no Service exists or you need pod-specific scrape config.
- **ScrapeConfig**: raw Prometheus scrape config. Escape hatch
  when ServiceMonitor/PodMonitor aren't expressive enough.

Most Helm charts ship ServiceMonitor CRDs alongside the workload
(behind `serviceMonitor.enabled: true` in values). trivy-operator
has this — we set `serviceMonitor.enabled: false` until Prometheus
is up; flip when wiring trivy-operator metrics into Grafana.

---

## Q13. Alertmanager vs Grafana OnCall vs PagerDuty / Opsgenie?

**A.** Three layers, all valid depending on scale:

- **Alertmanager**: OSS, dedupes + groups + routes alerts. Routes to
  Slack/email/PagerDuty webhooks. Config in YAML.
- **Grafana OnCall**: OSS PagerDuty alternative. Schedule, escalation,
  ack, runbook links. For teams that want incident management in the
  same tool as dashboards.
- **PagerDuty / Opsgenie / VictorOps**: SaaS incident management.
  Alertmanager/OnCall route TO these as terminal destination.

At homelab/small scale, Alertmanager → Slack webhook is enough. At
team scale, add Grafana OnCall. At org scale, route to PagerDuty
(money-backed SLAs to care about).

---

## Q14. Where should dashboards come from — configured by hand, or as code?

**A.** **As code.** Three patterns, in order of maturity:

1. **Sidecar loader** (`grafana-sidecar` container watches for
   ConfigMaps labeled `grafana_dashboard: "1"`, imports them). Most
   common. Dashboards are JSON in Git.
2. **Grafana Operator**: CRDs like `GrafanaDashboard`, `Grafana
   DataSource`. Argo-manages them like any other k8s resource.
3. **Grafonnet / jsonnet**: generate dashboards programmatically.
   Advanced — organizations that templatize dashboards across
   services.

Dashboards in Git survive Grafana wipes, cluster rebuilds, accidental
UI edits. UI edits are ephemeral until exported — always export +
commit.

---

## Q15. Retention — how long should I keep metrics / logs / traces?

**A.** Depends on use case:

| Data | Typical retention | Why |
|---|---|---|
| **Metrics** | 90d hot + 1-2y warm (downsampled) | Trend analysis, capacity planning |
| **Logs** | 14-30d hot, 90d warm | Incident investigation, compliance |
| **Traces** | 3-7d | Debugging; if a bug's not found in a week, trace is stale |

Homelab: 15d Prometheus, 7d logs, 3d traces. Plenty.
Compliance environments (PCI-DSS, HIPAA): log retention is
contractually long (90d+) — often the primary cost driver.

---

## Q16. How much storage do I need?

**A.** Rough budgets for a 3-node k8s cluster:

| Data | Per day (light load) | 30 days |
|---|---|---|
| Prometheus metrics | 200 MB - 2 GB | 6 - 60 GB |
| Logs | 500 MB - 5 GB | 15 - 150 GB |
| Traces | 100 MB - 1 GB | 3 - 30 GB |

These scale roughly linearly with cluster size and traffic. Size
PVCs generously — retention you're not using is cheap; running out
of space is a pager at 3am.

---

## Q17. What's the difference between pull-model agents and push-model agents?

**A.** In our architecture:

- **Prometheus agent** — RUNS a scraper locally, pulls from local
  targets, pushes via remote-write to central. "Pull then push."
- **Promtail** — watches node filesystem for container logs, reads
  them, pushes to Loki. Pure push.
- **OTel agent** — receives from apps (apps push to localhost),
  forwards to OTel gateway. "Push then push."

All three use push at the boundary between cluster and central
backend because pull across that boundary is brittle (firewalls,
NAT, etc.).

---

## Q18. What about Falco / runtime security events?

**A.** Falco is a runtime security detector — watches syscalls via
eBPF and emits alerts ("shell opened in production container",
"network connection to unknown IP"). NOT observability in the
metrics/logs/traces sense, but its events often get piped into the
same observability stack.

Typical pattern: Falco → OTel → Loki (events indexed as logs) AND
Prometheus (event rate as metrics). Alertmanager routes critical
Falco events the same way as CPU alerts.

Not wiring today; consider for Phase 7+ (security hardening).

---

## Glossary

- **PromQL** — Prometheus Query Language. `rate(http_requests_total{}[5m])`.
- **LogQL** — Loki's PromQL-like language for logs.
- **OTLP** — OpenTelemetry Protocol. gRPC or HTTP. The wire format.
- **Histogram** (Prometheus) — distribution of values via buckets.
  Used for latency SLIs.
- **Summary** — like histogram but pre-calculates percentiles on
  the target. Less flexible than histogram; prefer histograms.
- **Exemplar** — a trace ID embedded in a metric data point. Lets
  Grafana "jump from this metric spike to the trace that caused it."
- **Recording rule** — pre-computed query. Speeds up slow
  aggregations. `job:http_errors:rate5m`.
- **Alert rule** — query that fires when threshold crossed.
- **Label (metric)** — key-value attached to a metric. Dimensional.
- **Cardinality** — unique label combinations (see Q9).
- **Retention** — how long data is kept.
- **Federation** — Prometheus hierarchical scraping. Legacy.
- **Remote write** — Prometheus pushes to another store. Modern.
- **Service discovery** — Prometheus finds targets via k8s API.

## Resources

- **Google SRE Book** — free online. Chapters 4 (SLOs), 6 (Monitoring).
- **Grafana Labs blog** — LGTM stack architecture deep-dives.
- **Charity Majors** (Honeycomb) — "Observability Engineering" book.
  Philosophy + practical techniques.
- **Prometheus docs** — the "BEST PRACTICES" section on label design.
- **OpenTelemetry spec** — opentelemetry.io/docs/specs/otel/.
- **Kubernetes monitoring with Prometheus** — official prometheus-operator docs.
- **Grafana's "Mimir from scratch" talk** — YouTube, explains the
  horizontal-scale remote-write receiver pattern.
