# /root/learning/forge

Manifests and values for services deployed to the **mgmt-forge** Kubernetes cluster.

Every subdirectory corresponds to one ArgoCD `Application` (or `ApplicationSet` target) managed by `argocd-sre` on OKD. The cluster destination is `mgmt-forge` (registered via the cluster Secret produced by bootstrap Phase 8).

> **Big picture**: the forge stack isn't a grab-bag of services — it's the three-piece SaaS control plane (identity + secrets + trust). See [Platform vision](#platform-vision--identity--secrets--trust-control-plane) at the bottom.

## Layout

| Subdir | Purpose | Sync wave |
|---|---|---|
| `cert-manager/` | CNCF TLS automation; Issuer CRDs + webhook | 0 |
| `ingress-nginx/` | Ingress controller (MetalLB LB IP on 10.10.5.x) | 0 |
| `external-secrets/` | ESO — syncs K8s Secrets from Vault | 1 |
| `vault/` | HashiCorp Vault (KV + PKI engines) | 1 |
| `trivy-operator/` | Cluster-wide image vuln scanning | 2 |
| `harbor/` | Container registry + Trivy integration | 2 |
| `keycloak/` | SSO / OIDC provider | 2 |
| `actions-runner-controller/` | Self-hosted GitHub Actions runners | 3 |
| `argo-workflows/` | Workflow DAGs for platform jobs | 3 |
| `tekton/` | OpenShift Pipelines (alt CI engine) | 3 |
| `sonarqube/` | Code quality / SAST gate (+ EDP sonar-operator) | 3 |

## Conventions (per subdir)

Each service directory will contain:

- `values.yaml` — Helm values (or `kustomization.yaml` for plain manifests)
- `application.yaml` — ArgoCD `Application` CR pointing at this directory (source repo = this GitHub repo, destination cluster = `mgmt-forge`)
- Optional: `certificate.yaml`, `ingress.yaml`, `externalsecret.yaml` for cross-cutting wiring

## Deployment flow

```
GitHub push → ArgoCD (argocd-sre on OKD) → Application resources applied to mgmt-forge
```

ArgoCD reconciles via the `cluster-admin` ServiceAccount `argocd-manager` created on mgmt-forge during bootstrap Phase 8.

## Sync waves rationale

- **Wave 0** = foundations every later app assumes (CRDs, ingress).
- **Wave 1** = secrets & identity plumbing (Vault reachable before ESO tries to connect).
- **Wave 2** = platform apps that consume waves 0-1 (Keycloak needs cert-manager + ESO-synced admin creds).
- **Wave 3** = workload-facing apps (CI runners, SonarQube, workflow engines).

Set via `argocd.argoproj.io/sync-wave` annotation on each Application.

---

# Platform vision — identity + secrets + trust control plane

The forge stack is structured around one architectural pattern that recurs in every real SaaS: **identity + secrets + trust**. Each wave-1 and wave-2 service sits on one of these three axes.

```
               ┌────────────────────┐
               │      IDENTITY      │  ← who you are + what roles
               │  (Keycloak)        │
               │  realms, users,    │
               │  groups, OIDC      │
               └─────────┬──────────┘
                         │ OIDC tokens (with claims)
       ┌─────────────────┴─────────────────┐
       ▼                                   ▼
┌──────────────┐                   ┌──────────────┐
│     APPS     │                   │   SECRETS    │  ← what you can access
│   (Harbor,   │                   │   (Vault)    │    dynamic creds, policies
│   SonarQube, │                   │              │
│   Jenkins,   │◀── short-TTL ─────│   OIDC auth, │
│   your own)  │    creds          │   k8s auth,  │
│              │                   │   PKI engine │
│              │                   └──────┬───────┘
└──────┬───────┘                          │
       │ OIDC login to app                │
       │ runtime fetch of secrets         ▼
       ▼                           ┌──────────────┐
  downstream services              │    TRUST     │  ← certs, mTLS, signing
  (DBs, APIs, queues)              │ (cert-manager│
                                   │  + Vault PKI)│
                                   │              │
                                   │  leaf certs  │
                                   │  per service │
                                   └──────────────┘
```

## Integration edges

The architecture is defined by how the three pillars COUPLE. Each edge has a canonical implementation plus alternatives.

| Edge | What | Our choice | Alternatives | When to pick alternative |
|---|---|---|---|---|
| **Keycloak → Vault** | Users log in via Keycloak, get Vault tokens with policies derived from Keycloak claims | Vault OIDC auth method with Keycloak as IdP | JWT auth (raw token validation), LDAP bridge | LDAP if existing AD is the identity source |
| **Vault → cert-manager** | Every ingress/service auto-gets a real TLS cert, rotated | Vault PKI engine + cert-manager's Vault Issuer | cert-manager + self-signed CA ClusterIssuer (no Vault), Let's Encrypt (if public DNS) | No-Vault for minimalism; LE for public-facing |
| **Vault → apps (runtime secrets)** | Pods fetch short-lived credentials at startup/per-request | Vault Agent sidecar, or ESO, or Vault CSI secret driver | Environment vars (bad), baked-in secrets (worst) | — |
| **Keycloak ↔ apps (SSO)** | User logs in once; each app validates bearer token | Per-app OIDC client in Keycloak realm | SAML (legacy enterprise), proxy SSO (oauth2-proxy) | SAML when consuming enterprise SaaS; proxy SSO for apps that don't speak OIDC |
| **Keycloak secrets ↔ Vault** | Keycloak's own DB password + client secrets stored in Vault | ESO ExternalSecrets materializing from Vault KV | sealed-secrets, k8s Secrets directly | Sealed-secrets when you don't want a Vault dependency; direct when you're OK with manual rotation |
| **Cross-cluster secret flow** | Vault on forge, consumers on other clusters need its data | ESO on each consumer reads remote Vault | Vault Agent Injector on each cluster, secret replication (Replicator) | Agent Injector for pod-native mounting; Replicator for minimalism |
| **Tenant onboarding automation** | Provision realm/group + Vault policy + PKI path + data schema | Custom control-plane app (we'd build this) | Terraform (Keycloak + Vault providers), Crossplane CRs, Ansible | Terraform if infra-team owns platform; Crossplane for fully-GitOps-native |

## Use-case catalog — candidates for every case

For each SaaS-level concern, what are the candidate tools/patterns and which one fits this stack.

### 1. Identity store (users + groups + authentication)

| Candidate | Notes | Fit here |
|---|---|---|
| **Keycloak** | OSS, self-hosted, realms for multi-tenancy, OIDC + SAML + LDAP federation | ✅ chosen — runs on forge |
| Zitadel | Newer OSS alternative; Rust/Go, simpler ops, multi-tenant first-class | Alternative if Keycloak's Java/Quarkus footprint becomes a pain |
| Dex | Lightweight OIDC federation service (no user store) | Use when identity lives elsewhere (GitHub, Google, LDAP) and you just need OIDC |
| Auth0 / Okta | Commercial SaaS | Skip for homelab; org-scale typical choice |
| Authentik | OSS, Django-based, easier UI than Keycloak | Alternative for friendlier UX; less mature |

### 2. Secrets / credentials store

| Candidate | Notes | Fit here |
|---|---|---|
| **HashiCorp Vault** | OSS, widely adopted, KV + PKI + dynamic creds + OIDC + k8s auth | ✅ chosen |
| OpenBao | Fork of Vault after license change; compatible API | Drop-in if Vault's BSL license becomes an issue |
| Infisical | OSS, developer-first UX, integrates well with Git | Alternative for smaller teams; less depth than Vault |
| AWS Secrets Manager / SSM | Cloud-native | On AWS only |
| Bitwarden / 1Password Secrets Automation | UX-first | Developer-targeted orgs |

### 3. PKI (internal CA + cert automation)

| Candidate | Notes | Fit here |
|---|---|---|
| **Vault PKI engine** + **cert-manager** | Vault issues certs; cert-manager automates lifecycle | ✅ target state |
| cert-manager + self-signed ClusterIssuer | Simpler, no Vault dependency | Use if you don't have Vault |
| Smallstep CA (step-ca) | Dedicated ACME CA; lighter than Vault | Alt when PKI is the only Vault use-case |
| Let's Encrypt | Public CA | Only works for internet-reachable DNS |
| FreeIPA / Active Directory CS | Enterprise PKI | Legacy enterprise integration |

### 4. Container registry + image scanning

| Candidate | Notes | Fit here |
|---|---|---|
| **Harbor** | Registry + Trivy scanner + replication + signing | ✅ planned wave 2 |
| GitHub Container Registry | Free, hosted, tied to GH | Alternative when images are already public and lifecycle is simple |
| Quay (OSS / Red Hat) | Similar to Harbor | If already on OCP; Red Hat ecosystem |
| JFrog Artifactory | Commercial, multi-artifact-type | Enterprise registry + more (Maven, npm, etc.) |
| Docker Hub | Public | Not an enterprise answer |

### 5. Runtime vulnerability scanning

| Candidate | Notes | Fit here |
|---|---|---|
| **Trivy Operator** (Aqua) | Scans running workloads, produces CRs | ✅ deployed |
| Falco | Runtime behavior detection (not CVE scanning; syscall monitoring) | Complement (different layer) |
| Kyverno policies | Admission-time policy (reject vulnerable images at deploy) | Complement |
| Sysdig Secure | Commercial, runtime + forensics | Enterprise |
| Wiz / Orca / Prisma Cloud | Commercial CNAPP | Enterprise multi-cloud |

### 6. Certificate-based workload identity (service-to-service mTLS)

| Candidate | Notes | Fit here |
|---|---|---|
| **Vault PKI + cert-manager per-workload Certificate** | Each workload gets a cert from Vault | Our target — simple and fits existing stack |
| SPIFFE / SPIRE | Workload identity standard; federation across clusters | Next-level if you have multi-cluster zero-trust needs |
| Istio / Linkerd mTLS | Service mesh auto-generates workload certs | If you already have a mesh |
| Cilium workload identity | eBPF-based, embedded in Cilium CNI | If on Cilium CNI (we are, on mgmt-storage) |

### 7. GitOps control plane (authoring + reconciling)

| Candidate | Notes | Fit here |
|---|---|---|
| **ArgoCD** | Web UI, Helm + Kustomize, ApplicationSet generators | ✅ installed on OKD (SRE + DevOps) |
| Flux | CLI-first, GitOps Toolkit, better multi-tenant | Alternative if you prefer CLI-first |
| Terraform + Atlantis | Infra-as-code + PR automation | Use for cloud infrastructure outside k8s |
| Crossplane | Compose cloud + k8s resources as k8s CRs | Use when platform-level API is the goal |

### 8. Tenant provisioning (the control-plane app)

This is the "build an app that does all this" — not a single product, but a pattern. Candidates:

| Candidate | Notes | Fit here |
|---|---|---|
| **Build custom** | Thin API + UI orchestrating Keycloak + Vault + k8s APIs | Likely path — you learn the integration |
| Teleport | Commercial access management; Keycloak SSO + short-lived certs | Inspiration; study the architecture |
| HashiCorp Boundary | PAM broker, OIDC → dynamic credentials | Similar inspiration |
| Backstage | Developer portal; plugins for Keycloak, Vault | If a dev portal UX is what you want |
| Zitadel | Combined Keycloak-like + per-tenant support | Use if you want less moving parts |
| Port | Commercial internal developer portal | Enterprise |

### 9. Observability of the control plane

Not a runtime component, but critical. How do you see what Keycloak / Vault / cert-manager are doing?

| Candidate | Our target |
|---|---|
| Prometheus + Grafana | Metrics dashboards for each service (Phase 6) |
| Loki + Promtail | Log aggregation across all three |
| Tempo + OTel | Distributed traces of OIDC → Vault → app chains |
| Audit log piping | Keycloak audit events → SIEM; Vault audit device → Loki |
| Falco | Runtime behavior monitoring |

### 10. Tenant-level isolation

What primitives isolate one customer from another across the stack?

| Layer | Isolation primitive | Notes |
|---|---|---|
| Identity | Keycloak realm per tenant (OR group within a shared realm) | Realms = hard isolation; groups = soft |
| Secrets | Vault namespace (Enterprise) or policy path prefix (OSS) | OSS uses path-based isolation, which is fine for most |
| PKI | Dedicated Vault PKI mount per tenant | Each tenant gets a clean cert authority |
| Data plane | Namespace / database schema / object-store prefix per tenant | App-layer choice |
| Network | NetworkPolicy / service mesh AuthorizationPolicy | Limit cross-tenant traffic |
| Observability | Tenant-id label on every log line, metric, trace | Grafana variables filter to per-tenant views |

## What we have today vs the target

| Capability | Status |
|---|---|
| Identity store (Keycloak) | ✅ running; one realm (`forge`) + one client (`argocd-sre`) |
| Secrets store (Vault) | ✅ running; KV engine active; k8s auth for ESO |
| ESO bridging Vault → k8s Secrets | ✅ running on forge |
| cert-manager | ⚠️ installed; **no Issuer configured** — every ingress still on `ingress.local` default cert |
| Vault OIDC auth method (Keycloak → Vault) | 📋 not configured |
| Vault PKI secrets engine | 📋 not enabled |
| Vault Issuer in cert-manager | 📋 not configured |
| Control-plane app | 📋 not built |
| Tenant isolation primitives | ⚠️ single-tenant today; schema for multi-tenant is TBD |
| Observability of the chain | ❌ no stack yet (Phase 6) |

## Learning path to mastery

This is the structured progression to go from "I have each piece running" to "I built the SaaS control plane."

### Step 1 — Each piece in isolation (≈ 80% done)
- Keycloak: ✅ realms, clients, users, groups, OIDC endpoints, token shape.
- Vault: ✅ KV engine, k8s auth method. ❌ PKI engine, OIDC auth method, policies.
- cert-manager: ⚠️ installed. ❌ Issuers not configured.

### Step 2 — Wire pairs (≈ 1 hour each)
- **2a. Keycloak → Vault OIDC auth.** Enable `vault auth oidc`, configure role pointing at Keycloak's `forge` realm, map Keycloak groups to Vault policies. Test: `vault login -method=oidc` → browser login → token with policy attached.
- **2b. Vault PKI + cert-manager Vault Issuer.** Enable Vault PKI engine, create a root CA (or intermediate off an existing root), configure cert-manager's Vault ClusterIssuer, issue a cert for `keycloak.apps.mgmt-forge.engatwork.com`. Solves today's TLS problem at the source.
- **2c. App → Vault via k8s auth.** Pick a trivial demo app (echo server), have it authenticate to Vault at startup using its k8s SA JWT, fetch a dummy secret, log it. Proves the workload-identity chain.

### Step 3 — End-to-end chain (≈ half day)
A single HTTP request flows through all three:
1. Request arrives with a Keycloak OIDC bearer token.
2. App validates token (signature via Keycloak JWKS, claims).
3. App exchanges token for a Vault token (OIDC auth).
4. App uses Vault token to fetch a short-lived per-tenant DB credential.
5. App opens DB connection, serves request.

### Step 4 — Tenant provisioning API (≈ 1-2 weeks)
Build a small service that exposes `POST /tenants`, `DELETE /tenants/:id`, etc. Each operation fans out to Keycloak (create realm/group + users), Vault (create policy + PKI mount), and the data plane (schema bootstrap). This is the "app that does all this."

### Step 5 — Harden + observe (ongoing)
- Short TTLs everywhere (Vault tokens, DB creds, TLS certs — all < 24h).
- Audit piping (Keycloak admin events + Vault audit device → Loki).
- Runtime tracing (OTel through the OIDC→Vault→DB chain).
- Admission control (Kyverno blocking manual secret creation in tenant namespaces).

## Reference apps to study

Open-source products that implement this pattern at production scale. Read their code and docs.

| Project | Why worth reading |
|---|---|
| **Teleport** | Keycloak SSO + short-lived cert-based access to infra. Clean architecture. |
| **HashiCorp Boundary** | PAM broker, OIDC login → dynamic credentials. HashiCorp-style policy engine. |
| **Backstage** (with Keycloak/Vault plugins) | Spotify's developer portal. How tenants are modeled as "components" and "resources." |
| **Zitadel** | Combined identity+tenant provider. Different take than Keycloak+Vault. |
| **Crossplane** | How to expose provisioning as k8s CRs. The "control plane as API" pattern. |

## Recommended reading order

1. [`oauth.com`](https://oauth.com) — Aaron Parecki, the OAuth 2.0 reference.
2. OIDC core spec — official.
3. HashiCorp Learn: Vault + OIDC + PKI tutorials.
4. NIST SP 800-207 (Zero Trust Architecture) — 50 pages, read once.
5. Teleport architecture docs.
6. SPIFFE / SPIRE whitepapers — the next level (workload identity federated across clusters).

---

## Cross-references

- `forge/keycloak/QA.md` — IAM/OIDC deep-dive (users, claims, flows) + Q11 on SaaS-grade OIDC automation patterns.
- `argo/README.md` — ApplicationSet patterns + selector design.
- `storage/README.md` / `storage/todo.md` — storage-layer equivalent of this architecture doc.
