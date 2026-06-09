# K3s Cluster Installation

This document contains the bootstrap commands used to install k3s on the managed clusters in this environment.

> **Important**: These commands are environment-specific. They contain real IPs, hostnames, and TLS SANs for the Sydney production infrastructure. Use them only for nodes in this deployment.

## Prerequisites

- Root access on the target nodes
- Outbound internet access to download k3s and required images
- Firewall rules allowing:
  - Kubernetes API (6443)
  - Flannel/WireGuard (UDP 51820 by default)
  - etcd (if using embedded etcd)
- The first node must be designated as the initial server

## Install the First Server

Run on the primary node:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --cluster-init \
  --disable traefik \
  --flannel-backend=wireguard-native \
  --node-external-ip=139.99.210.170 \
  --node-name rancher-afe5.syd.prod.sigaint.au \
  --tls-san=139.99.210.170,139.99.210.89,45.32.191.253,rancher-afe5.syd.prod.sigaint.au,rancher-4e60.syd.prod.sigaint.au,rancher-a163.syd.prod.sigaint.au" sh -
```

Key flags explained:

| Flag                        | Purpose |
|-----------------------------|---------|
| `--cluster-init`            | Initializes a new cluster (embedded etcd) |
| `--disable traefik`         | Disables the default Traefik ingress (we deploy our own via Fleet) |
| `--flannel-backend=wireguard-native` | Uses WireGuard for pod networking (required for this environment) |
| `--node-external-ip`        | Advertises the public IP of the node |
| `--node-name`               | Sets a stable, DNS-resolvable node name |
| `--tls-san`                 | Adds all known IPs and hostnames to the API server certificate |

## Install Additional Servers

On each additional control-plane node, run the following (replace `<token>` with the token from the first server):

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://139.99.210.170:6443 K3S_TOKEN=<token> \
INSTALL_K3S_EXEC="server \
--disable traefik \
--flannel-backend=wireguard-native \
--node-external-ip=139.99.210.89 \
--node-name rancher-4e60.syd.prod.sigaint.au \
--tls-san=139.99.210.170,139.99.210.89,45.32.191.253,rancher-afe5.syd.prod.sigaint.au,rancher-4e60.syd.prod.sigaint.au,rancher-a163.syd.prod.sigaint.au" sh -
```

## Post-Installation Steps

After the cluster is up:

1. Retrieve the kubeconfig from `/etc/rancher/k3s/k3s.yaml` on any server node.
2. Import the cluster into Rancher (if not already managed).
3. Apply the required Fleet labels to the cluster so that resources from this repository are deployed:

   ```yaml
   sigaint.au/cluster-id: k3s-lhm-prod      # or k3s-lhm-devel / rancher-syd-prod
   sigaint.au/environment: production       # or development
   ```

4. Ensure the initial Doppler token secret is created in the `external-secrets` namespace before Fleet attempts to deploy `external-secrets-operator`.

5. For `k3s-lhm-prod` and `k3s-lhm-devel` (Harvester guest clusters), enable the Harvester CSI driver so the clusters can consume storage from the underlying Harvester. This requires both the Fleet component and a one-time registration step on the Harvester cluster using the `generate_addon_csi.sh` script. See [Harvester CSI Driver](harvester-csi-driver.md) for the full procedure (including bastion commands).

## Verification

- Check node status: `kubectl get nodes`
- Verify WireGuard interfaces are up: `ip link show type wireguard`
- Confirm Traefik is not running: `kubectl get pods -n kube-system | grep traefik`

## Notes

- All nodes run as servers (HA control plane). There are currently no dedicated agent-only nodes in this setup.
- The `--tls-san` list must include every IP and hostname that will be used to reach the API server.
- After initial bootstrap, further configuration (CNI, ingress, storage, operators) is managed declaratively via Fleet from this repository.