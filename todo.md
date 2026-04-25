# Forge — TODO

Active work backlog for the forge tier. Items are roughly ordered by
when they're likely to be tackled. Each item: *what*, *why now*, and
*what the finish line looks like*.

---

## Codify Vault PKI bootstrap into a playbook

**Status: not started.**

Phase 1-3 of the "identity + secrets + trust" loop went in via direct
`vault write …` calls (see commits `d370580…ec1be61` on this repo). The
end state is reproducible from the transcript but not from a single
playbook run, so a clean rebuild of mgmt-forge would mean re-typing the
sequence.

Author `forge/vault/playbooks/configure_pki.yml`, idempotent, mirroring
the existing `configure.yml`. It should:

1. Enable `pki/` (max_lease_ttl=87600h) if not present.
2. Generate root CA via `pki/root/generate/internal` if no CA cert
   currently issued. Capture PEM to `~/.vault-pki-ca.crt`.
3. Configure `pki/config/urls` with `vault.apps.mgmt-forge.engatwork.com`
   issuing + CRL paths.
4. Create role `platform-ingress` (allowed_domains=engatwork.com,
   allow_subdomains=true, max_ttl=2160h, require_cn=false,
   key_type=rsa, key_bits=2048).
5. Write policy `cert-manager-issuer` (sign + issue on
   `pki/sign/platform-ingress`).
6. Create k8s auth role `cert-manager-issuer` bound to SA
   `cert-manager` in namespace `cert-manager`.
7. Loop over a `vault_cluster_mounts` list (per-cluster auth mounts),
   each with name + apiserver URL + token-reviewer JWT + CA, and:
   - Enable `auth/kubernetes-{name}/` if not present.
   - Configure with that cluster's apiserver + CA + reviewer JWT.
   - Create `forge-reader` role bound to `external-secrets` SA in
     `external-secrets` namespace, policy `forge-reader`.

Tag layout matches `configure.yml`: `init`, `pki`, `auth-mounts`, `all`.

### Finish line

```bash
ansible-playbook playbooks/configure_pki.yml -e @vault_clusters.yml
```

reproduces the exact Vault state we have today, no manual `vault write`s.

### Watch out for

- `--diff` should be a no-op on a second run.
- Don't generate a new root CA on re-run — check `pki/cert/ca` first.
- Per-cluster auth mounts need each consumer cluster's
  `vault-token-reviewer` SA + ClusterRoleBinding to `system:auth-delegator`
  and a long-lived token Secret (kubernetes.io/service-account-token).
  That side-prep can be a sister playbook in `cluster/k8s-bstrp/` or a
  template that lives with `external-secrets` chart.

---

## Migrate Keycloak / ArgoCD UI / future Harbor onto Vault PKI certs

**Status: not started.**

Same pattern as the Vault ingress cert (this repo, commit `d370580`). Each
service still served by ingress-nginx's default self-signed cert is a
TLS-trust hole — anything trying to verify the cert chain (OIDC clients,
ESO, browsers without an exception) hits warnings or hard failures.

Per service, in its umbrella chart:

1. Add to ingress.annotations: `cert-manager.io/cluster-issuer: vault-pki`.
2. Add to ingress.tls: `[{ hosts: [<host>], secretName: <service>-tls }]`.
3. Verify cert-manager auto-creates the Certificate from the Ingress.
4. Verify ingress-nginx swaps to the new cert.

Order of operations (least → most disruptive):

1. **ArgoCD UI** (Keycloak OIDC discovery currently fails because
   ArgoCD's own ingress is on the self-signed cert and its OIDC client
   can't verify back).
2. **Keycloak** (chicken-and-egg with ArgoCD: ArgoCD OIDC needs
   Keycloak's discovery URL to be trusted, Keycloak's discovery URL is
   served by the same self-signed ingress).
3. **Harbor** when it lands.

### Finish line

`curl https://keycloak.apps.mgmt-forge.engatwork.com/auth/realms/master/.well-known/openid-configuration`
succeeds without `--insecure` from any cluster, because every consumer
trusts the engatwork.com Root CA via the `caBundle` already embedded in
the external-secrets chart.

### Watch out for

- Keycloak's reverse-proxy headers (`KC_PROXY_HEADERS=xforwarded`)
  matter for the OIDC issuer URL — make sure the upgrade preserves them.
- ArgoCD config-map needs an `oidc.config` block referencing Keycloak's
  discovery URL once Keycloak's cert is trusted; current sessions parked
  this with TLS verify disabled.

---

## Tighten ESO ApplicationSet for cluster-onboarding race

**Status: nice-to-have.**

A new cluster's `auth/kubernetes-{name}/` mount has to exist on Vault
*before* the AppSet generates an ESO Application that points at it,
otherwise the ClusterSecretStore goes `InvalidProviderConfig` until the
mount is created. Today that's a manual step (or a side effect of the
PKI playbook re-running).

Either:

a. **k8s-bstrp Phase 9** — extend `bootstrap-k8s.yml` to (1) create the
   `vault-token-reviewer` SA + CRB + token Secret on the new cluster,
   (2) call back to mgmt-forge's Vault to enable + configure the new
   `auth/kubernetes-{name}/` mount + role.
b. **A separate ApplicationSet job** — a one-shot Job per cluster that
   reads the Vault root token from a Secret and runs the auth-mount
   creation as a Helm post-install hook.

Option (a) is more proper (cluster bootstrap stays in one place); (b)
keeps secret-engine config in the GitOps layer.

### Finish line

A brand-new cluster registered via Phase 8 is fully ESO-functional within
the same Argo sync cycle that generates its Pattern A ApplicationSet
children — no manual Vault config step.

---

## Phase 8 of `cluster/k8s-bstrp/bootstrap-k8s.yml` — pin `--context`

**Status: small but burned a session today.**

`bootstrap-k8s.yml` Phase 8 runs `kubectl apply -n argocd-sre -f …`
against whatever the *current* kubectl context happens to be. Today the
current context was `dev-apps` (a dead cluster), so all 8 phases of a
fresh bootstrap completed and then Phase 8 silently failed for 60s
trying to reach the wrong apiserver. Pinning `--context "{{ okd_context
| default('admin') }}"` makes the playbook self-contained.

Lives in the `cluster` repo, but flagging here so it doesn't get lost.

---

## Delete stale `argo/sre/mgmt-observability/cluster-secret.yaml`

**Status: cosmetic.**

After the mgmt-observability rebuild, Phase 8 of `bootstrap-k8s.yml`
generated and applied a fresh `argocd-sre/mgmt-observability` Secret
with a current SA token. The file
`argo/sre/mgmt-observability/cluster-secret.yaml` carries an outdated
token that's no longer applied anywhere — it was a manual-bootstrap
artifact from before the cluster was Phase-8-registered. Delete it from
the argo repo so future readers don't think it's authoritative.

Lives in the `argo` repo, flagging here for cross-reference.

---

## Author observability backends (Prometheus / Alertmanager / Grafana)

**Status: this was the original goal that triggered everything in the
last few sessions.**

mgmt-observability is now stable, ESO works, ceph-rbd PVCs bind,
Pattern A baseline is healthy. Time to actually deploy the backends.
Tracked under `observability/` repo — see its README + QA — but linking
here because Grafana's OIDC client lives in the forge realm and pulls
its admin password via ESO.

### Per-component

- **Prometheus**: umbrella over kube-prometheus-stack's prometheus
  subchart with remote-write receiver enabled. PVC on ceph-rbd.
  Pattern C AppSet (role=observ).
- **Alertmanager**: same chart family, configures Slack/email/PagerDuty
  receivers; admin secrets via ExternalSecret pulling from `kv/forge/
  alertmanager`.
- **Grafana**: provisioned datasources point at the cluster-local
  Prometheus + Loki + Tempo Services. OIDC: Keycloak `forge` realm
  (depends on Keycloak ingress cert migration above).

Sync-waves: Prometheus=0, Alertmanager=0, Grafana=1 (depends on the
others being ClusterIP services).
