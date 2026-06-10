# Harvester CSI Driver

The `k3s-lhm-prod` and `k3s-lhm-devel` clusters are guest clusters running on Harvester. To allow pods in these clusters to consume persistent storage backed by the Harvester cluster (Longhorn), the Harvester CSI driver must be installed and registered.

Enabling the driver consists of two parts:

1. **Declarative deployment into the guest clusters** (via Fleet).
2. **One-time registration / configuration on the Harvester side** (performed from a bastion host).

## 1. Fleet Deployment (Guest Cluster Side)

The Helm chart is deployed declaratively by Fleet:

- Path: `infrastructure/harvester-csi-driver/`
- Chart: `harvester/harvester-csi-driver` from `https://charts.harvesterhci.io/`
- Namespace: `kube-system`
- Scope: Only clusters with `sigaint.au/cluster-id` matching `k3s-lhm-prod` or `k3s-lhm-devel`.

See [infrastructure/harvester-csi-driver/fleet.yaml](../infrastructure/harvester-csi-driver/fleet.yaml) for the exact configuration and version pin.

Once Fleet has synced the chart, the CSI driver pods will be present in `kube-system` on the guest cluster.

## 2. Harvester-Side Registration (Bastion Steps)

Before (or in conjunction with) the CSI driver pods becoming useful, Harvester must be told about the guest cluster. This is done by creating a dedicated ServiceAccount + RBAC in the Harvester cluster and generating a cloud config (kubeconfig) that the guest cluster uses to talk back to Harvester.

These steps are run on the **bastion** (a host with administrative `kubectl` access to the Harvester cluster, usually via `~/.kube/config`).

### Prerequisites

- `kubectl` configured with a context that has permissions on the Harvester cluster.
- `jq` installed (the script uses it to detect the Kubernetes server version).
- Internet access (to download the generation script).

### Steps

```bash
# 1. Download and prepare the generator script
curl -LO https://raw.githubusercontent.com/harvester/harvester-csi-driver/master/deploy/generate_addon_csi.sh
chmod +x generate_addon_csi.sh

# 2. Install jq (example for openSUSE/SLE-based bastion)
sudo zypper install jq

# 3. Generate configuration for each guest cluster
# Usage: ./generate_addon_csi.sh <service_account_name> <namespace> k3s

# For the development guest cluster
./generate_addon_csi.sh k3s-lhm-devel k3s-lhm-devel k3s

# For the production guest cluster
./generate_addon_csi.sh k3s-lhm-prod k3s-lhm-prod k3s
```

### What the script does

- Creates a ServiceAccount (named after the guest cluster) inside a namespace (also named after the guest cluster) on the Harvester cluster.
- Creates a RoleBinding (for the `harvesterhci.io:cloudprovider` ClusterRole) and a ClusterRoleBinding (for the `harvesterhci.io:csi-driver` ClusterRole).
- Creates a token Secret for the ServiceAccount (Kubernetes 1.24+ style).
- Discovers the Harvester API endpoint from the `harvester-system/vip` ConfigMap.
- Generates a kubeconfig that authenticates as the new ServiceAccount against Harvester.
- For k3s/RKE2 targets, it prints:
  - The raw kubeconfig under a "cloud-config" heading.
  - A `cloud-init` `write_files` snippet (for reference when using cloud-init based provisioning).

The script does **not** modify the guest cluster directly. It only creates identity material on the Harvester side and emits the configuration the guest needs.

### Applying the Generated Configuration

The kubeconfig emitted by the script (the content under `cloud-config`) is the credential the Harvester CSI driver in the guest cluster will use to communicate with Harvester.

In this environment the driver is installed via the Fleet-managed Helm chart. The generated cloud-config is used to populate the necessary secret or configuration that the chart (or the underlying deployment manifests) expects, typically a secret named `harvester-csi-config` in `kube-system` on the guest cluster, or by placing the file at the location the driver looks for cloud provider configuration.

After the configuration is in place on the guest cluster and the Fleet-deployed CSI driver pods are running, Harvester storage should become available.

## Verification

Run the following inside the guest cluster (`k3s-lhm-devel` or `k3s-lhm-prod`):

```bash
# Check that the CSI driver is running
kubectl get pods -n kube-system | grep -E 'harvester-csi|longhorn'

# Look for Harvester-provided StorageClasses
kubectl get storageclass | grep -i harvester

# Optional: create a test PVC (adjust storageClassName as needed)
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-harvester-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: harvester   # or the specific class provided by Harvester
EOF
```

## Notes

- The namespace and ServiceAccount name on the Harvester side are conventionally set to the guest cluster's name (as shown in the commands above). This keeps per-guest identity isolated.
- The script must be re-run (or the resulting resources updated) if the guest cluster's identity needs to be rotated or if the Harvester VIP changes.
- Because the k3s-lhm clusters are already bootstrapped, the cloud-config output is typically applied after the fact (via secret, Rancher addon mechanism, or direct manifest) rather than at initial provisioning time.
- The Fleet `harvester-csi-driver` component only deploys the driver into the guest. The Harvester-side ServiceAccount/RBAC step is a prerequisite for that driver to successfully authenticate and provision volumes.

## References

- Harvester CSI Driver repository: https://github.com/harvester/harvester-csi-driver
- Generation script: `deploy/generate_addon_csi.sh` in the above repo
- Fleet component: `infrastructure/harvester-csi-driver/fleet.yaml` in this repository