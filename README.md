# k8s-homelab-gitops

GitOps source of truth for the [k8s-homelab](https://github.com/martinezsaweczko/k8s-homelab) Kubernetes cluster.

Managed by [Flux CD](https://fluxcd.io/). Every commit to `main` is automatically reconciled by the cluster.

## Repository Structure

```
cluster/
├── flux-system/              # Flux self-management and layer orchestration
│   ├── gotk-components.yaml  # Flux controllers + CRDs
│   ├── cluster.yaml          # Root Kustomization
│   ├── infrastructure.yaml   # Envoy Gateway
│   ├── cert-manager.yaml
│   ├── cert-manager-webhook-ionos.yaml
│   ├── cert-manager-issuers.yaml
│   ├── gateway-config.yaml
│   └── apps.yaml
├── infrastructure/
│   ├── gateway-api/          # Envoy Gateway HelmRelease
│   ├── cert-manager/         # cert-manager HelmRelease
│   ├── cert-manager-webhook-ionos/   # IONOS DNS webhook for Let's Encrypt
│   ├── cert-manager-issuers/         # ClusterIssuer + Certificates
│   └── gateway-config/       # Gateway, HTTPRoutes, EnvoyProxy
└── apps/
    └── homelab/              # Applications deployed to the homelab cluster
        ├── notifierwhatsapp/
        └── monitoring/       # Prometheus + Grafana
```

## Flux Kustomization Layers & Dependencies

### Why separate layers?

This repository is split into multiple Flux `Kustomization` resources instead of one large Kustomization. The reasons are:

- **Ordering**: some layers install CRDs that later layers need (e.g. cert-manager CRDs before `Certificate` resources).
- **Dependencies**: Envoy Gateway HTTPS listener needs the `grafana-tls` certificate to exist first.
- **Isolation**: infrastructure, cert-manager, gateway config, and apps can reconcile independently.

### How ordering is maintained

Each Flux `Kustomization` uses `spec.dependsOn` to declare what must be ready before it reconciles. If a dependency is not ready, Flux retries until it is.

### Current dependency graph

```
cluster (root Kustomization, reconciles everything)
├── infrastructure        # Envoy Gateway
├── cert-manager          # cert-manager core
│   └── cert-manager-webhook-ionos
│       └── cert-manager-issuers  # ClusterIssuer + Certificate
└── gateway-config        # depends on infrastructure + cert-manager-issuers
    └── apps              # Grafana, Prometheus, notifierwhatsapp
```

### Layer descriptions

| Layer | Path | Purpose | Depends on |
|-------|------|---------|------------|
| `cluster` | `./cluster` | Root Kustomization that applies all other layers | — |
| `infrastructure` | `./cluster/infrastructure` | Installs Envoy Gateway | — |
| `cert-manager` | `./cluster/infrastructure/cert-manager` | Installs cert-manager | — |
| `cert-manager-webhook-ionos` | `./cluster/infrastructure/cert-manager-webhook-ionos` | Installs IONOS DNS webhook for DNS-01 challenges | `cert-manager` |
| `cert-manager-issuers` | `./cluster/infrastructure/cert-manager-issuers` | Creates ClusterIssuer and Certificates | `cert-manager-webhook-ionos` |
| `gateway-config` | `./cluster/infrastructure/gateway-config` | Configures Gateway, listeners, HTTPRoutes, redirects | `infrastructure`, `cert-manager-issuers` |
| `apps` | `./cluster/apps` | Deploys applications (Grafana, Prometheus, notifierwhatsapp) | `gateway-config` |

### Why `infrastructure/kustomization.yaml` only includes `gateway-api`

`cluster/infrastructure/kustomization.yaml` is consumed by the `infrastructure` Flux Kustomization. It only includes `gateway-api` because cert-manager and its webhook were split into separate Flux Kustomizations (`cert-manager.yaml`, `cert-manager-webhook-ionos.yaml`, `cert-manager-issuers.yaml`) so we can enforce the order:

1. cert-manager must be installed before its CRDs are used.
2. The webhook must be installed before the `ClusterIssuer` references it.
3. The certificate must be ready before the Gateway HTTPS listener references it.

Putting everything under `infrastructure/kustomization.yaml` would apply them all at once without ordering, causing random failures on fresh installs.

### Fresh install behavior

On a fresh cluster:

1. `gotk-components.yaml` installs Flux itself.
2. `cluster.yaml` creates the child Kustomizations.
3. Each child waits for its `dependsOn` parents before reconciling.
4. Temporary failures (e.g. CRD not ready yet) are retried automatically.

You can watch progress with:

```bash
flux get kustomizations
```

## Secrets Management (SOPS + Age)

This repository is **public**. All Kubernetes Secrets are encrypted with [Mozilla SOPS](https://github.com/getsops/sops) + [Age](https://github.com/FiloSottile/age) before being committed.

### Required Tools

- `sops` — [releases](https://github.com/getsops/sops/releases)
- `age` — `dnf install age` or [releases](https://github.com/FiloSottile/age/releases)

### One-Time Setup: Create `.sops.yaml`

Before encrypting any secrets, you must create a `.sops.yaml` file in the root of this repository. This file tells SOPS which Age public key to use for encryption.

```bash
# Get your Age public key (generated by Ansible during cluster setup)
AGE_KEY=$(grep "Public key" ~/.config/sops/age/keys.txt | awk '{print $3}')

# Create the SOPS config
cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: cluster/.*\.sops\.yaml$
    age: $AGE_KEY
EOF

git add .sops.yaml
git commit -m "chore: add sops creation rules"
git push
```

> **IMPORTANT:** Back up `~/.config/sops/age/keys.txt` in your password manager. If you lose this file, all encrypted secrets are **unrecoverable**.

### Encrypting a New Secret

```bash
# Create a plaintext Secret YAML
cat > secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
stringData:
  password: my-secret-value
EOF

# Rename to .sops.yaml convention
mv secret.yaml secret.sops.yaml

# Encrypt (uses .sops.yaml in this repo root)
sops --encrypt --in-place secret.sops.yaml
```

### Editing an Existing Encrypted Secret

```bash
sops secret.sops.yaml
```

This opens your `$EDITOR` with the decrypted contents. Save and exit to re-encrypt automatically.

### Rotating or Replacing the Age Key

If the Age private key is lost or rotated (for example, after recreating the cluster without preserving the previous key), the existing encrypted secrets can no longer be decrypted. You must re-encrypt them with the new key.

#### 1. Update `.sops.yaml` with the new public key

```bash
# Get your new Age public key
AGE_KEY=$(grep "Public key" ~/.config/sops/age/keys.txt | awk '{print $3}')

# Update the SOPS config
cat > .sops.yaml <<EOF
creation_rules:
  - path_regex: cluster/.*\.sops\.yaml$
    age: $AGE_KEY
EOF

git add .sops.yaml
git commit -m "chore: rotate age key"
```

#### 2. Re-encrypt existing secrets (only if you still have the old key)

```bash
for f in $(find cluster -name '*.sops.yaml'); do
  sops --decrypt --in-place "$f"
  sops --encrypt --in-place "$f"
done
```

#### 3. If the old key is lost, recreate the secrets

When the old key is gone, the encrypted files are unrecoverable. Recreate each secret from scratch, encrypt it with the new key, and commit it.

For example, for the Grafana admin secret:

```bash
cat > cluster/apps/homelab/monitoring/grafana-admin-secret.sops.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: grafana-admin
  namespace: monitoring
type: Opaque
stringData:
  admin-user: admin
  admin-password: <new-password>
EOF

sops --encrypt --in-place cluster/apps/homelab/monitoring/grafana-admin-secret.sops.yaml
```

#### 4. Push and reconcile

```bash
git add .
git commit -m "chore: re-encrypt secrets with new age key"
git push

flux reconcile kustomization cluster --with-source
```

Make sure the new Age private key is installed in the cluster as the `sops-age` secret in `flux-system` so Flux can decrypt the files.

### How Decryption Works in the Cluster

The Age private key is stored in the cluster as a Kubernetes Secret `sops-age` in the `flux-system` namespace. Flux uses it to decrypt `.sops.yaml` files during reconciliation. The private key **never** enters this repository.

## Upgrading Components

All upgrades in this repository are done through Git. Flux reconciles the changes automatically after you push.

### Upgrading a Helm chart

Each Helm chart version is pinned in the corresponding `HelmRelease` under `spec.chart.spec.version`.

For example, to upgrade cert-manager:

```yaml
# cluster/infrastructure/cert-manager/helmrelease.yaml
spec:
  chart:
    spec:
      version: "1.16.0"   # change this
```

Then commit and push:

```bash
git add cluster/infrastructure/cert-manager/helmrelease.yaml
git commit -m "chore(cert-manager): upgrade cert-manager to v1.16.0"
git push
```

Flux will upgrade the release automatically.

### Helm charts managed in this repo

| Component | File to edit |
|-----------|--------------|
| cert-manager | `cluster/infrastructure/cert-manager/helmrelease.yaml` |
| cert-manager-webhook-ionos | `cluster/infrastructure/cert-manager-webhook-ionos/helmrelease.yaml` |
| Envoy Gateway | `cluster/infrastructure/gateway-api/envoy-gateway-helmrelease.yaml` |
| kube-prometheus-stack | `cluster/apps/homelab/monitoring/helmrelease.yaml` |
| prometheus-snmp-exporter | `cluster/apps/homelab/monitoring/snmp-exporter-helmrelease.yaml` |

### Upgrading Flux components

Flux itself is installed from `cluster/flux-system/gotk-components.yaml`. To upgrade Flux:

```bash
flux install \
  --version=v2.9.5 \
  --components-extra=image-reflector-controller,image-automation-controller \
  --export > cluster/flux-system/gotk-components.yaml
```

Then commit and push.

> **Important:** `gotk-components.yaml` must always be generated with `--components-extra=image-reflector-controller,image-automation-controller` because the `notifierwhatsapp` app uses `ImageRepository`, `ImagePolicy`, and `ImageUpdateAutomation` CRDs.

### Upgrading raw Kubernetes manifests

For resources like `Gateway`, `HTTPRoute`, `Deployment`, `ConfigMap`, etc., edit the manifest directly and push.

### Upgrading container images

`notifierwhatsapp` image updates are automated via Flux `ImageUpdateAutomation`. The workflow is:

1. `ImageRepository` scans the registry.
2. `ImagePolicy` selects the latest matching tag.
3. `ImageUpdateAutomation` updates the deployment image and pushes to the `flux-image-updates` branch.
4. A GitHub Actions workflow opens a PR.

You can add the same pattern to other applications by creating `ImageRepository`, `ImagePolicy`, and `ImageUpdateAutomation` resources.

### Verify the upgrade

After pushing, force reconciliation and watch the status:

```bash
flux reconcile source git k8s-homelab-gitops
flux reconcile kustomization cluster
flux get kustomizations
flux get helmreleases -A
```

Check that the new version is running:

```bash
kubectl get deployment -n <namespace> <deployment-name> -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## Adding a New Application

1. Create a new directory under `cluster/apps/homelab/<app-name>/`
2. Add your manifests (Deployment, Service, etc.)
3. If you have Secrets, encrypt them as `*.sops.yaml`
4. Add the directory to `cluster/apps/homelab/kustomization.yaml`
5. Commit and push — Flux will deploy automatically

## Image Updates

Flux `ImageUpdateAutomation` watches the GitHub Container Registry (GHCR) for new image tags and automatically updates this repository. See the `image-policies/` directory for configuration.

## Disaster Recovery

If the cluster is completely rebuilt:

1. Run the [k8s-homelab](https://github.com/martinezsaweczko/k8s-homelab) Ansible playbooks
2. If `flux bootstrap` fails due to Deploy Keys being disabled, follow the manual wiring steps in the [infrastructure repo's Flux troubleshooting guide](https://github.com/martinezsaweczko/k8s-homelab/blob/main/docs/FLUX_GITOPS.md)
3. When bootstrapping Flux manually, always include the image automation controllers:

   ```bash
   flux bootstrap github \
     --owner=martinezsaweczko \
     --repository=k8s-homelab-gitops \
     --branch=main \
     --path=cluster \
     --components-extra=image-reflector-controller,image-automation-controller
   ```

4. The cluster pulls this repo and self-heals to the current desired state


## Troubleshooting

How to get syncronization progress

```
flux get kustomizations -A
```

```
flux-system     cluster                 False           False   ImageUpdateAutomation/notifierwhatsapp/notifierwhatsapp dry-run failed: failed to create typed patch object (notifierwhatsapp/notifierwhatsapp; image.toolkit.fluxcd.io/v1, Kind=ImageUpdateAutomation): .spec.policy: field not declared in schema
```

Check the different options via flux --help