# /root/learning/forge

Manifests and values for services deployed to the **mgmt-forge** Kubernetes cluster.

Every subdirectory corresponds to one ArgoCD `Application` (or `ApplicationSet` target) managed by `argocd-sre` on OKD. The cluster destination is `mgmt-forge` (registered via the cluster Secret produced by bootstrap Phase 8).

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
# forge
