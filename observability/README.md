# forge/observability

Observability backends deployed to **mgmt-observability** (role=observ).
This directory holds the umbrella charts for every component of the
central telemetry stack. Agents that scrape/ship telemetry FROM
workload clusters TO these backends are Pattern A — they live
separately and target every workload cluster via ApplicationSet.

## Why split, and why on mgmt-observability

### Split (one chart per component) instead of kube-prometheus-stack

`kube-prometheus-stack` is a popular monolith chart — Prometheus +
Alertmanager + Grafana + node-exporter + kube-state-metrics + 30
dashboards + ServiceMonitors in one install. It's fine for a single
cluster that observes only itself. We chose NOT to use it because:

1. **Multi-cluster observability** requires the backend to accept
   remote-write from other clusters. The monolith chart assumes a
   single-cluster deployment. Splitting makes the boundary obvious.
2. **Swap-ability**: when you outgrow Prometheus single-instance,
   you swap to Mimir without touching Grafana's chart. The monolith
   wires the components together tightly — unwinding is painful.
3. **Different lifecycle per component.** Grafana UI updates don't
   necessitate Prometheus restarts. The monolith couples them.
4. **Sync wave ordering**: Prometheus (the data source) must be up
   and accepting remote-write before Grafana tries to query it.
   Split charts + Argo sync waves make that explicit.
5. **Different ownership at scale**: in real orgs, the metrics team
   owns Prometheus/Mimir while a separate team owns Grafana
   dashboards. One chart per team.

Trade-off: more charts to author and keep in sync. Acceptable — each
chart is small (thin umbrella over upstream), and they compose by
convention (Grafana datasources point at well-known Service DNS
names like `prometheus.observability.svc.cluster.local`).

### Why mgmt-observability hosts the backends

Observability is a CENTRAL SERVICE — one Grafana, one Prometheus
store, one Alertmanager. Putting these on a cluster dedicated to
observability:

- Isolates failure domain: if mgmt-forge's Vault thrashes, it doesn't
  impact Prometheus queries.
- Sizing independence: Prometheus needs large disks; dashboards are
  memory-heavy. A dedicated cluster lets you right-size.
- Clear access model: observability team gets kubeadm-level access
  on mgmt-observability without being near platform secrets.
- Network posture: backends can be firewalled to receive only
  remote-write / push from workload clusters; no public exposure.

mgmt-observability = `tier=mgmt role=observ`. ApplicationSets in
`argo/sre/observability/` use Pattern C selector (`role: observ`).

## Architecture

```
  workload clusters              mgmt-observability (role=observ)
  (mgmt-forge, OKD,              ─────────────────────────────────
   dev-*, mgmt-core)
  ─────────────────              ┌─────────────────────────────┐
                                 │                             │
  Prometheus agent  ──remote-write────→  Prometheus  (metrics) │
  (Pattern A, fut)                 │     ↑                     │
                                   │     │ datasource          │
  Promtail daemon  ──push────────────→   Loki        (logs)    │
  (Pattern A, fut)                 │     ↑                     │
                                   │     │ datasource          │
  OTel Collector   ──OTLP traces────→    Tempo       (traces)  │
  agent daemon                     │     ↑                     │
  (Pattern A, fut)                 │     │ datasource          │
                                   │     │                     │
                                   │   Grafana  (UI)           │
                                   │     │                     │
                                   │     ↓ rule eval           │
                                   │   Alertmanager ──→ Slack /│
                                   │                   PagerDuty│
                                   │                   email   │
                                   └─────────────────────────────┘
```

## Components in this directory

| Component | Role | Session |
|---|---|---|
| `prometheus/` | Metrics storage (with remote-write receiver enabled) | 1 — today |
| `alertmanager/` | Alert routing (Slack, email, PagerDuty) | 1 — today |
| `grafana/` | UI fronting all data sources | 1 — today |
| `loki/` | Log aggregation backend | 2 — next session |
| `tempo/` | Distributed trace backend | 2 — next session |
| `otel-gateway/` | OTel Collector in gateway mode (receives from agents) | 2 — next session |
| `mimir/` | Horizontally-scalable Prometheus replacement | when Prometheus single-instance is outgrown |

### What lives elsewhere (not in this directory)

| Thing | Where | Why not here |
|---|---|---|
| Prometheus agent (on workload clusters) | TBD, Pattern A ApplicationSet in `argo/sre/observability-agents/` | Different selector (every cluster, not just observ) |
| Promtail (logs agent) | Same | Same |
| OTel agent (traces agent) | Same | Same |
| node-exporter, kube-state-metrics | Same | Same |

The split: `forge/observability/` = backends. A sibling directory
(to be authored) = agents. Both deployed by Argo ApplicationSets but
with different selectors.

## GitOps wiring

```
forge/observability/<component>/    ← umbrella chart per component
  └── Chart.yaml + values.yaml + optional templates/

argo/sre/observability/             ← Argo wiring
  ├── prometheus-appset.yaml        ← Pattern C, selector role=observ
  ├── grafana-appset.yaml
  ├── alertmanager-appset.yaml
  └── ...
```

Each ApplicationSet:
- `selector.matchLabels: {role: observ}` — generates one Application
  per cluster labeled role=observ (today: mgmt-observability only).
- `source.path: observability/<component>` in the forge repo.
- `destination.namespace: observability` — single namespace for all
  backends, keeps them on one network policy boundary and Grafana
  can reach peers via `<service>.observability.svc.cluster.local`.
- Sync-waves enforce ordering: Prometheus=0, Alertmanager=0,
  Grafana=1 (depends on datasources), Loki=0, Tempo=0.

## Integrations with the rest of the platform

| External piece | How it plugs in |
|---|---|
| **cert-manager** | Issues TLS certs for Grafana ingress, Prometheus remote-write endpoint. Today ingress-nginx default cert; target: Vault PKI Issuer. |
| **Keycloak** | Grafana OIDC client in `forge` realm. Users log in via Keycloak, RBAC via group claim. |
| **Vault** | Grafana admin password, SMTP credentials, PagerDuty API keys. ExternalSecret materializes them in the `observability` ns. |
| **Rook-Ceph-External** | ceph-rbd StorageClass for Prometheus/Loki/Tempo PVCs. Must be installed on mgmt-observability before backends deploy. |
| **Ingress-nginx** | Exposes Grafana + Alertmanager webhooks. Must exist on mgmt-observability (Pattern A — broaden cert-manager/ingress-nginx/ESO selectors when we onboard). |

## Deployment order (this session + next)

1. **Register mgmt-observability in argocd-sre** (one-time)
   — `bootstrap-sa.yaml` + `cluster-secret.yaml` with labels
   `tier=mgmt role=observ env=nonprod`.
2. **Cephx bootstrap** from
   `storage/rook-ceph-external/generated/mgmt-observ-bundle.yaml`.
3. **rook-ceph-external ApplicationSet auto-fires** (matches
   `tier In [mgmt,dev]` + `role NotIn [storage]`). CSI on
   mgmt-observability; `ceph-rbd` + `cephfs` StorageClasses appear.
4. **Broaden cert-manager / ingress-nginx / external-secrets
   selectors** (storage/todo.md task) — once changed, they
   auto-install on mgmt-observability too.
5. **Deploy Prometheus** — Pattern C ApplicationSet; PVC on
   ceph-rbd; remote-write receiver enabled.
6. **Deploy Alertmanager** — alongside Prometheus, wired to it for
   alert definitions.
7. **Deploy Grafana** — depends on Prometheus + Alertmanager
   existing as ClusterIP services; datasources provisioned via
   values.yaml.
8. **Session 2**: Loki, Tempo, OTel gateway, then agents on
   workload clusters.

## Reference apps worth reading

- **kube-prometheus-stack** — even though we're not using it, its
  values.yaml is the single best reference for Prometheus operator
  configuration (ServiceMonitor CRDs, alert rule bundles, etc.).
- **Grafana Tanka** — the LGTM stack (Loki, Grafana, Tempo, Mimir)
  reference deployment. Canonical for this split pattern.
- **Grafana OnCall** — alert routing alternative to Alertmanager
  with incident management built in.
- **Victoria Metrics** — Prometheus-compatible, simpler ops
  story than Mimir for mid-scale.

## Cross-references

- `forge/observability/QA.md` — learning Q&A (pillars, tool
  comparisons, cardinality, multi-cluster patterns).
- `argo/README.md` — selector patterns (A/B/C/D); this stack is
  Pattern C for backends, Pattern A for agents.
- `forge/README.md` — the broader platform vision, where
  observability sits in the identity+secrets+trust model.
- `storage/rook-ceph-external/README.md` — why cephx bootstrap is
  a prerequisite per cluster.
