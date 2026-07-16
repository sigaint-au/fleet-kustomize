# Fleet Kustomize

Declarative GitOps configuration for Rancher-managed k3s clusters using [Fleet](https://fleet.rancher.io/).

## Overview

This repository defines the desired state of infrastructure operators and applications across multiple k3s clusters. Fleet continuously reconciles the state defined here to the clusters based on labels.

## Repository Structure

```
infrastructure/          # Core platform components
  cert-manager-operator/
  cloudnative-pg/
  external-secrets-operator/
  harvester-csi-driver/
  metallb/
  traefik/
  velero/

applications/            # Workload applications
  falcosidekick/
  grafana/
  victoria-metrics/
```

## Components

| Component                  | Deployment          | Scope                  | Notes |
|----------------------------|---------------------|------------------------|-------|
| cert-manager               | Helm + Kustomize    | All clusters           | DNS-01 (RFC2136) with Doppler-managed TSIG |
| external-secrets           | Helm + Kustomize    | All clusters           | Doppler ClusterSecretStore |
| MetalLB                    | Helm + Kustomize    | `k3s-lhm-*`            | BGP mode; cluster-specific IP pools & peers |
| Traefik                    | Helm                | All clusters           | LoadBalancer on k3s-lhm clusters; hostNetwork DaemonSet on rancher-syd-prod |
| Velero                     | Helm + Kustomize    | All clusters           | Backups to Backblaze B2 via AWS-compatible plugin |
| CloudNativePG              | Helm                | All clusters           | PostgreSQL operator |
| Harvester CSI Driver       | Helm                | `k3s-lhm-*`            | Enables Harvester-backed storage in guest clusters (requires one-time Harvester-side setup — see [docs/harvester-csi-driver.md](docs/harvester-csi-driver.md)) |
| Falcosidekick              | Helm                | `k3s-lhm-*`            | Event forwarder for Falco + Web UI; per-env hostnames + Ingress; Doppler-managed config/secrets |
| VictoriaMetrics Cluster    | Helm                | By environment         | Metrics storage & querying |
| Grafana                    | Helm                | By environment         | Observability dashboards with OIDC (Zitadel) and VictoriaMetrics/VictoriaLogs datasources |

## Cluster Targeting

Components are deployed selectively using Fleet `targetCustomizations` and `dependsOn` driven by these labels:

| Label                        | Values                                      | Used For |
|------------------------------|---------------------------------------------|----------|
| `sigaint.au/cluster-id`      | `k3s-lhm-prod`, `k3s-lhm-devel`, `rancher-syd-prod` | Cluster-specific config (MetalLB peers, Traefik mode, etc.) |
| `sigaint.au/environment`     | `production`, `development`                 | Environment tuning (retention, replicas, resources) |

## Adding a Component

1. Create a new directory under `infrastructure/` or `applications/`.
2. Add a `fleet.yaml` that declares the Helm chart or kustomize resources.
3. Use `targetCustomizations` for any cluster- or environment-specific values.
4. Declare `dependsOn` relationships where ordering matters.
5. Register the path with your Fleet `GitRepo` resource(s).

## Secrets & Bootstrapping

- Long-lived secrets are delivered via External Secrets Operator from Doppler.
- The initial Doppler authentication secret must be created manually on each cluster before the first sync.
- k3s clusters are bootstrapped with Traefik disabled and WireGuard-native flannel. See [docs/installation.md](docs/installation.md) for the exact commands used in this environment.

## Gateway API CRDs (Traefik experimental channel)

`rancher-syd-prod` Traefik enables the Kubernetes Gateway API provider with **`experimentalChannel: true`**. That makes Traefik list/watch experimental resources such as **TCPRoute** and **TLSRoute**.

Those types are **not** part of the standard Gateway API install (and are not always present on a fresh k3s cluster). If the experimental CRDs are missing, Traefik logs errors like:

```text
failed to list *v1.TLSRoute: the server could not find the requested resource
(get tlsroutes.gateway.networking.k8s.io)
```

### Install (one-time per cluster)

Apply the **experimental** channel bundle from [kubernetes-sigs/gateway-api](https://github.com/kubernetes-sigs/gateway-api/releases). This includes standard CRDs **plus** experimental ones (TCPRoute, TLSRoute, UDPRoute, …).

Prefer server-side apply for CRDs:

```bash
# Pin a release that matches your Traefik/Gateway API expectations.
# Check https://github.com/kubernetes-sigs/gateway-api/releases and
# https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-gateway/
GATEWAY_API_VERSION=v1.5.1

kubectl apply --server-side -f \
  "https://github.com/kubernetes-sigs/gateway-api/releases/download/${GATEWAY_API_VERSION}/experimental-install.yaml"
```

If the cluster already has an older **standard** install, upgrading to experimental is supported; still use a pinned version and review the release notes.

### Verify

```bash
kubectl get crd | grep gateway.networking.k8s.io

# Experimental types required when experimentalChannel: true
kubectl get crd tlsroutes.gateway.networking.k8s.io
kubectl get crd tcproutes.gateway.networking.k8s.io
```

Optional: confirm Traefik no longer logs TLSRoute reflector errors:

```bash
kubectl -n traefik logs -l app.kubernetes.io/name=traefik --tail=50 | grep -i tlsroute || true
```

### When to run this

| Cluster | Traefik `experimentalChannel` | Action |
|---------|-------------------------------|--------|
| `rancher-syd-prod` | `true` (see `infrastructure/traefik/values/rancher-syd-prod.yaml`) | Install experimental CRDs **before** or immediately after Traefik first rolls out |
| `k3s-lhm-prod` / `k3s-lhm-devel` | `false` | Standard Gateway CRDs only (if any); experimental install optional |

Install experimental CRDs as part of cluster bootstrap (after k3s is up, before or alongside the first Fleet Traefik sync). See also [docs/installation.md](docs/installation.md).

### Notes

- **UDP DNS** still uses Traefik **IngressRouteUDP** (`kubernetesCRD`). Traefik does not implement Gateway API **UDPRoute** for this stack.
- Experimental channel is useful for future **TCPRoute** / **TLSRoute** (e.g. DoT) frontends; HTTP(S) Ingress and IngressRoute continue to work without those resources existing as objects—only the **CRDs** must exist so the watch succeeds.

## Requirements

- Rancher with Fleet
- Target clusters labeled according to the scheme above
- Network access from clusters to configured chart repositories and backup storage
- For `rancher-syd-prod` Traefik: Gateway API **experimental** CRDs installed (see above)

See [docs/installation.md](docs/installation.md) for k3s bootstrap instructions.
