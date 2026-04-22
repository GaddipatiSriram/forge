# Vault post-install playbook (`configure.yml`)

Imperative bootstrap steps that can't live in pure GitOps:

- `vault operator init` — generates 5 unseal keys + root token (one-shot, security-critical)
- Unseal all pods
- Enable `kv/` (v2) secrets engine
- Enable `kubernetes/` auth method + configure host
- Write Vault policies
- Write Kubernetes auth roles (binding ServiceAccounts to policies)
- Optional: seed smoke-test secrets

Runs against the Vault StatefulSet already deployed by the ArgoCD `vault-mgmt-forge` Application.

## Why not pure GitOps

Vault's security model requires manual custody of the 5 unseal keys + root token. Auto-init in-cluster means the platform holds those keys — defeats the point. See `/root/learning/README.md` for enterprise alternatives (AWS KMS auto-unseal, transit auto-unseal via secondary Vault).

## Usage

### Full fresh bootstrap (first time)

```
ansible-playbook configure.yml --tags all
```

This runs: init → unseal → configure → policies → roles. Init output is written to `~/.vault-keys-mgmt-forge.json` (mode 0600). **Copy to offline storage, then `shred -u` the file.**

### Just configure (Vault already init'd + unsealed)

```
ansible-playbook configure.yml --tags configure
```

Assumes `~/.vault-keys-mgmt-forge.json` exists for the root token, or pass `-e vault_root_token=hvs...` to override.

### Seed smoke-test secrets (optional, destructive for existing paths)

```
ansible-playbook configure.yml --tags seed
```

### Individual phases

- `--tags init` — only `vault operator init`
- `--tags unseal` — only unseal (requires keys file)
- `--tags configure` — KV + k8s auth + policies + roles
- `--tags seed` — smoke-test secrets

## Variables you'd override

| Var | Default | When to change |
|---|---|---|
| `vault_kubectl_context` | `mgmt-forge` | Running against a different cluster |
| `vault_namespace` | `vault` | Vault deployed in a different ns |
| `vault_pods` | `[vault-0, vault-1, vault-2]` | HA replica count changed |
| `vault_keys_file` | `~/.vault-keys-mgmt-forge.json` | Different key storage location |
| `vault_root_token` | (from keys file) | Explicit override |
| `kv_mount` | `kv` | Different KV mount path |
| `vault_policies` | `{forge-reader: ...}` | Add more policies |
| `vault_roles` | `[{forge-reader, external-secrets/external-secrets}]` | Add more roles |
| `vault_seed_secrets` | `[{kv/forge/hello: message=hello}]` | Different test data |

## Idempotency

| Phase | Idempotent? |
|---|---|
| `init` | Errors if already initialized — playbook tolerates via seal-status pre-check |
| `unseal` | Errors if already unsealed — tolerated |
| `configure` | Yes — Vault's `secrets enable`, `auth enable`, `policy write`, `auth/role write` are all upsert |
| `seed` | No — `kv put` increments the version on every run |

## Dependencies

- `kubectl` configured for `mgmt-forge` context (our merged `~/.kube/config`)
- Vault StatefulSet deployed (ArgoCD `vault-mgmt-forge` Application Synced+Healthy)
- Ansible ≥ 2.14

## File layout

```
forge/vault/
├── Chart.yaml                      ← Helm umbrella (deployed via ArgoCD)
├── values.yaml                     ← Helm values (deployed via ArgoCD)
└── playbooks/
    ├── ansible.cfg
    ├── inventory.ini
    ├── configure.yml               ← THIS PLAYBOOK
    └── README.md
```

## What this playbook does NOT do

- Does not install Vault itself (ArgoCD + Helm does that).
- Does not manage `ExternalSecret` / `ClusterSecretStore` CRs — those belong to ESO (`forge/external-secrets/`) as pure GitOps manifests.
- Does not rotate or re-key — separate operational runbook.

## Security

- **Keys file** (`~/.vault-keys-mgmt-forge.json`) is mode 0600 and in `.gitignore`. Delete it after copying to offline storage. If it leaks, every secret Vault holds is compromised.
- **Root token** should be revoked after creating a scoped admin policy + user. This playbook does not do that yet — enterprise TODO.
- **`no_log: true`** is set on every task that handles keys/tokens so playbook output doesn't leak them.
