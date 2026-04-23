# forge/okd/jenkins

Jenkins on OKD (`admin` / in-cluster), deployed via ArgoCD `argocd-sre`.
Umbrella Helm chart wrapping the official `jenkinsci/jenkins` chart.

## Files

- `Chart.yaml` — pins `jenkins/jenkins` version.
- `values.yaml` — Jenkins config (image tag, ingress, persistence,
  admin creds handling, OKD SCC tweaks). Gotchas documented inline.

## Storage — read this before changing persistence settings

Jenkins persists to `ceph-rbd` (20 GiB RWO) via the **central Rook-Ceph
cluster on `mgmt-storage`** consumed through `rook-ceph-external` on OKD.
See `storage/rook-ceph/` for the consumer setup.

### Why this matters for Jenkins specifically

Jenkins was the first stateful workload we tried to stand up on OKD
against `ceph-rbd`. Issue #1 in `storage/rook-ceph/README.md` — OSDs
on the central cluster advertising pod-CIDR addresses instead of host
IPs — surfaced exactly here: PVC stuck `Pending`, CSI provisioner
looping with `DeadlineExceeded`. That issue is now CLOSED; if PVC
binding ever starts hanging again, that's the first place to look.

### If storage breaks again

Fallback order (what we tried when debugging issue #1):

1. **`cephfs`** — tried first, same failure mode. Not a workaround.
2. **`local-path`** — fails on OKD: helper pod gets "Permission denied"
   at `/opt/local-path-provisioner` because SCOS treats `/opt` as
   read-only immutable FS. See `storage/rook-ceph/README.md` #3.
3. **`emptyDir`** (`persistence.enabled: false`) — always works but
   loses all Jenkins data on pod restart. Acceptable only for a
   disposable learning install.

### Jenkins-specific OKD gotchas

1. **SCC**: the jenkinsci chart uses an init container that runs as
   UID 0 to `chown /var/jenkins_home`. OKD's `restricted-v2` SCC
   forbids fixed UIDs, so the `jenkins` ServiceAccount must be granted
   the `anyuid` SCC:

   ```
   oc adm policy add-scc-to-user anyuid -z jenkins -n jenkins
   ```

2. **Pod Security Admission** — the `jenkins` namespace needs
   `pod-security.kubernetes.io/enforce=privileged` to allow the init
   container. This is separate from SCC and is K8s-level:

   ```
   kubectl label namespace jenkins \
     pod-security.kubernetes.io/enforce=privileged \
     pod-security.kubernetes.io/warn=privileged \
     pod-security.kubernetes.io/audit=privileged --overwrite
   ```

3. **Image tag must match chart plugin defaults.** The chart ships a
   pinned default plugin list, and a given chart version's plugins
   assume a specific minimum Jenkins version. Jenkins 2.479.2 (chart
   5.8.1 default) doesn't satisfy plugins that later require
   2.541.1+. Pin `controller.image.tag` explicitly to an LTS that
   matches the chart's plugin set:

   - Chart 5.9.18 works with `jenkins:2.541.1-jdk17`.
   - Never leave the image tag at chart default without verifying plugin
     compat — the init container will silently fail at the plugin
     install step with `requires a greater version of Jenkins (X) than Y`.

4. **Admin credentials** — chart-generated for now, in Secret
   `jenkins`:

   ```
   kubectl --context admin -n jenkins get secret jenkins \
     -o jsonpath='{.data.jenkins-admin-password}' | base64 -d
   ```

   Migrate to Vault via ESO (same pattern as `keycloak`) before moving
   off the disposable-learning posture.

## Argo Application

Lives at `argo/sre/okd/jenkins-app.yaml` in the `learning` workspace.
Destination: `https://kubernetes.default.svc` (OKD in-cluster).
`argocd-sre` is namespace-scoped, so the `jenkins` namespace must be
pre-labeled with `argocd.argoproj.io/managed-by=argocd-sre` before
Argo can reconcile resources into it.
