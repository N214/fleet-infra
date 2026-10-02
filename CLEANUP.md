# CLEANUP.md

This lists things in or around the cluster that aren't in use, with the evidence for each and how to
remove it. The inventory was taken on 2026-09-27 on node `vm` (k3s v1.35.2, Cilium 1.19.7, Tailscale
operator 1.98.4). **Nothing below has been removed yet.**

**How to use this:** handle one item at a time, then run the
[end-to-end checks in AGENT.md](AGENT.md#end-to-end-checks). Items that change Cilium values are made in
`clusters/production/overlays/production/cilium/helmrelease-patch.yaml`. After those, restart
`cilium-operator` and `cilium` (see AGENT.md, pitfall 3).

## Summary

| # | Item | Where | Risk |
|---|---|---|---|
| 1 | `tailscale-subnet` pod and its Secret, ServiceAccount, Role and RoleBinding | cluster (created by hand) | Low |
| 2 | Service `kube-system/hubble-ui-gateway` | cluster (created by hand) | None |
| 3 | Secret `kube-system/cloudflare-key` | cluster | Low |
| 4 | ClusterMesh API server | Cilium values | Low |
| 5 | ServiceMonitor CRD | Cilium values and cluster | Low |
| 6 | Namespace `monitoring` and the unused `hubble.dashboards` values | cluster and Cilium values | None |
| 7 | Egress Gateway | Cilium values | Low |
| 8 | Envoy settings left over from host-network mode | Cilium values | Low |
| 9 | `cilium-node-init` DaemonSet | Cilium values | Medium |
| 10 | `socketLB.useOriginalSourceAddress`, which does nothing | Cilium values | None |
| 11 | The Gateway's public LoadBalancer IP and NodePorts | Gateway | Medium |
| 12 | `local-path-provisioner` | k3s | Low |
| 13 | Host Tailscale route and DNS warning | host | Low |
| 14 | Unused container images and Docker leftovers | host | Low |
| 15 | Old app data and secret files in `~/Kube` | host | Low (back up first) |
| 16 | The `N214/argo-cd` repo | git | Low |
| 17 | Optional: settings added during troubleshooting | Cilium values | Test first |

## Cluster

### 1. `tailscale-subnet` pod and its RBAC

- **Evidence:** this is a bare Pod (no controller) created on 2026-09-05 from
  `ghcr.io/tailscale/tailscale:latest` with `TS_ROUTES=10.43.0.0/16`. That route was never approved in the
  tailnet, so the pod does nothing; tailnet access goes through the Gateway proxy. It runs as a user-owned
  device, so its key expires, and it uses Secret `tailscale-auth` and ServiceAccount `tailscale`, which
  nothing else uses. Its manifest is the untracked `tail-secret.yaml` at the root of this repo.
- **Remove:**
  ```bash
  kubectl -n tailscale get pods -o jsonpath='{range .items[*]}{.metadata.name}={.spec.serviceAccountName}{"\n"}{end}'  # only tailscale-subnet uses "tailscale"
  kubectl -n tailscale delete pod/tailscale-subnet secret/tailscale-auth \
    rolebinding/tailscale role/tailscale serviceaccount/tailscale
  ```
  Then remove the machine `tailscale-subnet` in the Tailscale admin console, revoke its auth key, and
  delete `tail-secret.yaml`. If tailnet devices ever need direct access to ClusterIPs, use the operator's
  `Connector` resource instead.

### 2. Service `kube-system/hubble-ui-gateway`

- **Evidence:** created on 2026-09-08 with `kubectl apply`. It has no selector and no endpoints, and no
  HTTPRoute references it; the `hubble` route points at `hubble-ui`.
- **Remove:** `kubectl -n kube-system delete svc hubble-ui-gateway`

### 3. Secret `kube-system/cloudflare-key`

- **Evidence:** an ACME account key managed by cert-manager, created 2026-04-05 03:49, nine minutes before
  `cert-manager/cloudflare-key`. The ClusterIssuer's `privateKeySecretRef: cloudflare-key` is resolved in
  the cluster resource namespace, `cert-manager`, and there are no Issuers in `kube-system`. Deleting it
  doesn't affect any certificate.
- **Remove:** `kubectl -n kube-system delete secret cloudflare-key`

### 4. ClusterMesh API server

- **Evidence:** this is a single cluster with `clustermesh.config.enabled: false` and no remote clusters,
  yet it still runs the `clustermesh-apiserver` Deployment (with etcd and kvstoremesh) and a ServiceMonitor.
  It also opens NodePort `32379` on every node address, including the tailnet address.
- **Change:** replace the `clustermesh:` block in the values with the following. `cluster.name` and
  `cluster.id` can stay.
  ```yaml
  clustermesh:
    useAPIServer: false
  ```

### 5. ServiceMonitor CRD (`servicemonitors.monitoring.coreos.com`)

- **Evidence:** this is the only Prometheus Operator CRD installed, and there's no Prometheus. The only
  ServiceMonitor is `kube-system/clustermesh-apiserver`, which item 4 removes.
- **Remove:** do this only after item 4, and after setting `prometheus.serviceMonitor.enabled: false`.
  Helm upgrades fail if the chart still renders a ServiceMonitor once the CRD is gone.
  ```bash
  kubectl get servicemonitors -A        # expect: No resources found
  kubectl delete crd servicemonitors.monitoring.coreos.com
  ```

### 6. Namespace `monitoring` and the `hubble.dashboards` values

- **Evidence:** namespace `monitoring` contains nothing but `kube-root-ca.crt`. The `hubble.dashboards`
  block in the values points at it, but Cilium's chart doesn't read that key (it reads
  `hubble.metrics.dashboards`), so the block has no effect.
- **Remove:** delete the `hubble.dashboards` block, then run `kubectl delete namespace monitoring`. If you
  plan to add Grafana, keep the namespace and rename the key to `hubble.metrics.dashboards` instead.

### 7. Egress Gateway

- **Evidence:** `egressGateway.enabled: true` (`enable-egress-gateway: "true"`), but no
  `CiliumEgressGatewayPolicy` exists.
- **Change:** delete `egressGateway` from the values, or set `enabled: false`.

### 8. Envoy settings left over from host-network mode

- **Evidence:** `envoy.securityContext.capabilities.keepCapNetBindService: true` and the `NET_BIND_SERVICE`
  capability let Envoy bind ports 80 and 443 on the host, which only host-network mode needs.
  `gatewayAPI.hostNetwork.enabled` has been `false` since `5328efc`.
- **Change:** remove `keepCapNetBindService` and `NET_BIND_SERVICE`, and keep `NET_ADMIN` and `SYS_ADMIN`.

### 9. `cilium-node-init` DaemonSet

- **Evidence:** `nodeinit.enabled: true` runs the `cilium-node-init` DaemonSet, which has restarted 8
  times. It exists to prepare nodes on managed platforms such as GKE and AKS. Self-managed k3s with
  `flannel-backend: none` doesn't need it.
- **Change:** set `nodeinit.enabled: false`. After the rollout, confirm the node is `Ready`, pods get IP
  addresses, and the end-to-end checks pass.

### 10. `socketLB.useOriginalSourceAddress`

- **Evidence:** added in `5aa24cc`, but no Cilium 1.19.7 chart template reads it. The real setting is
  `envoy.useOriginalSourceAddress`, which is already `false`.
- **Change:** delete the line.

### 11. The Gateway's public LoadBalancer IP and NodePorts

- **Evidence:** `CiliumLoadBalancerIPPool public-ip-pool` (`129.153.58.143/32`, defined in `gateway.yaml`)
  gives `cilium-gateway-gateway` the node's public IP, and the Service also has NodePorts `30219` and
  `32356`. Ports 80 and 443 on the public IP time out from the internet (blocked upstream, probably by the
  OCI security list), and tailnet traffic uses the ClusterIP, so neither is used.
- **Change (only if the Gateway should stay tailnet-only):** give GatewayClass `cilium` a ClusterIP-only
  config, then remove `public-ip-pool` from `gateway.yaml`. The tailnet path finds the Gateway by its
  Service's DNS name, so it keeps working; confirm with the end-to-end checks.
  ```yaml
  apiVersion: cilium.io/v2alpha1
  kind: CiliumGatewayClassConfig
  metadata:
    name: cilium-clusterip
    namespace: kube-system
  spec:
    service:
      type: ClusterIP
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: GatewayClass
  metadata:
    name: cilium
  spec:
    controllerName: io.cilium/gateway-controller
    parametersRef:
      group: cilium.io
      kind: CiliumGatewayClassConfig
      name: cilium-clusterip
      namespace: kube-system
  ```
- **Keep in mind:** opening 80 and 443 in the OCI security list would publish every HTTPRoute to the
  internet, and Hubble UI has no login.

### 12. `local-path-provisioner`

- **Evidence:** there are no PersistentVolumes or PersistentVolumeClaims.
- **Change (optional):** add `local-storage` to `disable:` in `/etc/rancher/k3s/config.yaml` and restart
  k3s. Skip this if you'll deploy stateful apps such as n8n or Actual Budget.

## Node (`vm`)

### 13. Host Tailscale

- The host advertises `10.0.0.188/32` as a subnet route. It was never approved and isn't needed, because
  the node is on the tailnet itself: `sudo tailscale set --advertise-routes=`
- `tailscale status` warns `Tailscale failed to set the DNS configuration of your device: running
  /usr/sbin/resolvconf ... Failed to resolve interface "tailscale"`. The node doesn't need MagicDNS, so
  either run `sudo tailscale set --accept-dns=false` or point `/etc/resolv.conf` at systemd-resolved.

### 14. Unused container images and Docker leftovers

- **containerd (k3s):** 48 images using 8.8 GB in `/var/lib/rancher/k3s/agent/containerd`.
  `sudo k3s crictl rmi --prune` removes images that no running container uses.
- **Docker:** only `hysteria-server` is running. There are 20 unused images (2.35 GB), 820 MB of build
  cache, 2 unused volumes, and networks left by old compose projects (`caddy_default`,
  `dns-bind9_default`, `frontend`, `kind`). If those projects are gone for good:
  ```bash
  docker image prune -a && docker builder prune -a && docker volume prune && docker network prune
  ```

### 15. Old app data and secret files in `~/Kube`

- `~/Kube/actualbudget/data`, `~/Kube/open-ui/data` and `~/Kube/apps/open-ui/data` are root-owned and
  hold data from apps that no longer run here. **Back them up** (the Actual Budget data in particular),
  then move them out of the repo.
- `~/Kube/apps/cert-manager/cloudflare-api-token.yaml` is an untracked copy of the Cloudflare token Secret
  (namespace `default`, which doesn't exist in the cluster). Delete it, and rotate the token if the file
  was ever pushed or shared.
- `tail-secret.yaml` in this repo (item 1): delete it, or at least add it to `.gitignore`.

## Git

### 16. `N214/argo-cd` (`~/Kube`)

Nothing deploys from this repo, because there's no Argo CD in the cluster.

- `apps/*` (argo-cd, monitoring, n8n, keda, actualbudget, open-ui) aren't deployed, and `argocdapps/`
  targets an Argo CD that isn't installed.
- `system-apps/cilium` has been replaced by this repo's HelmRelease, but Dependabot still bumps it. It
  merged a bump to Cilium 1.19.2 while the cluster runs 1.19.7.
- `gateway-api/` is scratch space: `test-gateway`, `tls-gateway` and `nginx-pod` aren't deployed. The
  `sys` file duplicates `clusters/production/flux-system/kustomization.yaml`.
- **Suggestion:** move anything worth keeping into this repo, turn off Dependabot there, and archive it.

## Optional: test first

### 17. Settings added during troubleshooting

`bpf.hostLegacyRouting: true` (`1bfa5b6`) and `socketLB.hostNamespaceOnly: true` (`5aa24cc`) were added
while chasing AGENT.md pitfall 1, and they weren't the fix. They're safe to keep. To revert them, change
one at a time and run the end-to-end checks after each. BPF host routing changes how pod traffic passes
through netfilter, which Tailscale's rules rely on.

## Follow-ups (not cleanup)

- Rotate the Cloudflare API token committed in `overlays/production/cert-manager/cluster-issuer.yaml`, then
  keep it out of Git (SOPS with Flux decryption, or a hand-made Secret). It's in the Git history, so treat it
  as exposed.
- The Tailscale operator moved to Flux on 2026-10-01 (see AGENT.md). Only its OAuth Secret is created by hand.
- Manage `hosts/vm/` with config management, or at least reinstall it whenever the node is rebuilt.

## Keep (in use)

- **Platform:** cert-manager, the Flux controllers and `flux-operator`, `cilium`, `cilium-envoy`,
  `cilium-operator`, CoreDNS and metrics-server.
- **Networking:** Hubble relay and UI (`hubble.ntcloud.dev`), the Tailscale operator, and the
  `ts-gateway-tailscale` proxy. Also the GatewayClass, Gateway, HTTPRoutes and Certificate, the
  CiliumNetworkPolicy `flux-system/allow-flux`, and namespace `cilium-secrets`.
- **Secrets:** `cert-manager/cloudflare-api-token`, `cert-manager/cloudflare-key` and
  `kube-system/cloudflare-cert`.
- **Other:** IngressClass `tailscale`, which is unused but ships with the operator chart, and the Docker
  container `hysteria-server`.
