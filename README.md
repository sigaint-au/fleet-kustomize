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
  metallb/
  traefik/
  velero/

applications/            # Workload applications
  perses/
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
| VictoriaMetrics Cluster    | Helm                | By environment         | Metrics storage & querying |
| Perses                     | Helm                | Production             | Observability dashboards |

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
- k3s clusters are bootstrapped with Traefik disabled and WireGuard-native flannel (exact commands are environment-specific and maintained separately).

## Requirements

- Rancher with Fleet
- Target clusters labeled according to the scheme above
- Network access from clusters to configured chart repositories and backup storage
