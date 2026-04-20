# Cluster Conifiguration

Installs and configures helm charts and operators.

## Cluster Install.

Install the server.
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --cluster-init \
  --disable traefik \
  --flannel-backend=wireguard-native \
  --node-external-ip=139.99.210.170 \
  --node-name rancher-afe5.syd.prod.sigaint.au \
  --tls-san=139.99.210.170,139.99.210.89,45.32.191.253,rancher-afe5.syd.prod.sigaint.au,rancher-4e60.syd.prod.sigaint.au,rancher-a163.syd.prod.sigaint.au" sh -
```

Install the other nodes.

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://139.99.210.170:6443 K3S_TOKEN=<token> \
INSTALL_K3S_EXEC="server \
--disable traefik \
--flannel-backend=wireguard-native \
--node-external-ip=139.99.210.89 \
--node-name rancher-4e60.syd.prod.sigaint.au \
--tls-san=139.99.210.170,139.99.210.89,45.32.191.253,rancher-afe5.syd.prod.sigaint.au,rancher-4e60.syd.prod.sigaint.au,rancher-a163.syd.prod.sigaint.au" sh -
```