# AGENTS.md — fleet-kustomize

Fleet GitOps repo managing infrastructure operators + applications across 3 k3s clusters.

## Clusters & Labels

| Label | Values |
|---|---|
| `sigaint.au/cluster-id` | `k3s-lhm-prod`, `k3s-lhm-devel`, `rancher-syd-prod` |
| `sigaint.au/environment` | `production`, `development` |

## Structure

- `infrastructure/` — platform operators (cert-manager, external-secrets, metallb, traefik, velero, cloudnative-pg, harvester-csi-driver)
- `applications/` — workloads (forgejo, grafana, falcosidekick, thelounge, netbird, adguard)
- VictoriaMetrics not yet committed despite README mention

## Fleet Patterns

Every `fleet.yaml` follows these conventions:

1. **Catch-all block** — trailing `doNotDeploy: true` with `clusterSelector: {}` excludes local mgmt cluster + unknowns. Every bundle must have one.
2. **Component labels** — `labels: component: foo` enables `dependsOn` targeting.
3. **Two-bundle pattern** for apps: (a) Helm install bundle, (b) supporting kustomize config bundle. Helm bundle `dependsOn` config bundle.
4. **`applications/fleet.yaml`** — `doNotDeploy: true` grouping-only bundle prevents Fleet from treating subdir YAML as raw objects.
5. **Kustomize isolation** — all kustomize `dir:` paths must be local to the `fleet.yaml` parent. Cross-bundle `../../components` paths do not work in Fleet.
6. **Cluster targeting** — `matchExpressions` with `In` operator for targeting multiple clusters, `matchLabels` for single-cluster or env targeting.

## Deployment Details

- **MetalLB**: `k3s-lhm-*` only, BGP mode, `takeOwnership: true` on Helm, per-cluster kustomize overlays for IP pools/peers. Webhook `diff.comparePatches` in operator fleet.yaml.
- **Traefik**: all 3 clusters, LoadBalancer on k3s-lhm, hostNetwork DaemonSet on rancher-syd-prod. `dependsOn` metallb on k3s-lhm clusters. `rancher-syd-prod` uses `experimentalChannel: true` — requires Gateway API experimental CRDs (see README for install).
- **Harvester CSI Driver**: k3s-lhm clusters only, two bundles: CRDs (prerequisite) → Helm chart via `dependsOn`.
- **cert-manager**: all clusters, DNS-01 with RFC2136 TSIG via Doppler. Webhook comparePatches. Config bundle depends on external-secrets.
- **external-secrets**: all clusters, Doppler ClusterSecretStore. Config bundle per env (production/development).
- **Velero**: all clusters, Backblaze B2 via S3-compat plugin. Config bundle per env.
- **CloudNativePG**: all clusters, Helm only. Webhook comparePatches.
- **Grafana**: k3s-lhm clusters per env, OIDC (Zitadel), VictoriaMetrics datasource. Config bundle with ExternalSecrets.
- **Falcosidekick**: k3s-lhm clusters per env. Config bundle with NetworkPolicies.
- **Forgejo**: rancher-syd-prod only, OCI chart. Config bundle overlays per cluster.
- **The Lounge**: rancher-syd-prod only. Avoid TrueCharts helm repo (file name with space breaks Fleet post-render). Config bundle with NetworkPolicies.
- **NetBird**: rancher-syd-prod only, pure kustomize (base/). No app.kubernetes.io/name in commonLabels — 3 sub-apps need distinct values.
- **AdGuard Home**: rancher-syd-prod only, pure kustomize (base/). Uses IngressRouteUDP (Traefik does not implement Gateway API UDPRoute).

## Kustomize Patterns

- `commonLabels` vs `labels` with `includeSelectors: false` — netbird uses `commonLabels` (with caveat), adguard uses `labels` block with `includeSelectors: false` for label-only without selector pollution.
- Overlay kustomizations typically just `resources: - ../../base`.

## Key Commands / One-Time Actions

- **Gateway API experimental CRDs** for rancher-syd-prod Traefik: `kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/${GATEWAY_API_VERSION}/experimental-install.yaml`
- **Doppler bootstrap**: initial auth secret must be created manually on each cluster before first Fleet sync.
- **k3s bootstrap**: see `docs/installation.md` (Traefik disabled, WireGuard-native flannel).
- **Harvester CSI one-time setup**: see `docs/harvester-csi-driver.md`.
- **AdGuard host firewall**: `sudo firewall-cmd --permanent --zone=public --add-port=53/udp` on Traefik nodes.

## Existing Instruction Files

README.md covers component table, cluster targeting, secrets/bootstrapping, Gateway API CRDs. Keep it as ground truth. AGENTS.md provides agent workflow patterns not in README.