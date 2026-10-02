# AGENT.md

Guidance for humans and coding agents working on **fleet-infra**, the Flux GitOps repo for the
single-node k3s cluster on the `vm` host. Read [Known pitfalls](#known-pitfalls) before debugging
Gateway, Envoy or Tailscale networking.

## Scope

- Flux 2.8.x, run by Flux Operator 0.61.0 and defined by
  [`clusters/production/flux-system/flux-instance.yaml`](clusters/production/flux-system/flux-instance.yaml)
  (self-managed: Flux applies its own FluxInstance), syncs **`clusters/production`** from
  `main` every 10 minutes with pruning on. **Pushing to `main` deploys, and deleting a manifest deletes
  the object.**
- Some pieces live outside Git and are managed by hand; see [Not in this repo](#not-in-this-repo).
- `~/Kube` on the node (`N214/argo-cd`) is the old Argo CD layout. Nothing deploys from it.

## Cluster facts

| | |
|---|---|
| Node | `vm`: Oracle Cloud, arm64, Ubuntu 26.04. Node IP `10.0.0.188/24` (VCN), public IP `129.153.58.143`, tailnet IP `100.122.203.7` |
| k3s | `flannel-backend: none`, `disable-kube-proxy`, `disable-network-policy`, `disable: [traefik, servicelb]` (`/etc/rancher/k3s/config.yaml`) |
| Cilium | 1.19.x via HelmRelease. kube-proxy replacement, native routing, pod CIDR `10.0.0.0/16`, Gateway API (not host-network), external `cilium-envoy` DaemonSet, Hubble relay and UI |
| TLS | cert-manager, ClusterIssuer `cloudflare-issuer` (ACME DNS-01 via Cloudflare), Certificate `kube-system/cloudflare-cert` for `*.ntcloud.dev` |
| Tailscale | Tailnet `tail8d4c27.ts.net`. Host `tailscaled` 1.102.4 (apt). Kubernetes operator 1.102.4 via Flux HelmRelease `tailscale/tailscale-operator`; proxies run `tailscale/tailscale:v1.102.5` |
| DNS | Cloudflare zone `ntcloud.dev`. `*.ntcloud.dev` is an `A` record to `100.114.165.114`, **DNS only** |

## Repository layout

| Path | Contents |
|---|---|
| `clusters/production/kustomization.yaml` | Root: `flux-system`, `overlays/production/cilium`, `overlays/production/cert-manager` |
| `clusters/production/flux-system/` | HelmRepositories (`cilium`, `jetstack`, `tailscale`), Flux Operator OCIRepository and HelmRelease (`flux-operator.yaml`), HTTPRoute `fluxcd.ntcloud.dev` to `flux-operator:9080` |
| `clusters/production/base/cilium/` | Cilium HelmRelease (chart version) |
| `clusters/production/overlays/production/cilium/helmrelease-patch.yaml` | Cilium Helm values |
| `clusters/production/overlays/production/cilium/gateway.yaml` | LB IP pool, GatewayClass `cilium`, Certificate, Gateway `kube-system/gateway`, HTTPRoute `hubble.ntcloud.dev` |
| `clusters/production/overlays/production/cilium/gateway-tailscale.yaml` | Puts the Gateway on the tailnet (see below) |
| `clusters/production/overlays/production/cert-manager/` | cert-manager values and the ClusterIssuer |
| `clusters/production/base/tailscale-operator/` | Tailscale operator HelmRelease (chart version, OAuth `valuesFrom`) |
| `clusters/production/overlays/production/tailscale-operator/` | Namespace `tailscale` and operator values (proxy image tag) |
| `hosts/vm/` | Host config that Flux does **not** apply; install it by hand |
| [`CLEANUP.md`](CLEANUP.md) | Unused resources and how to remove them |

### Not in this repo

- Secret `tailscale/tailscale-operator-oauth` (keys `clientId`, `clientSecret`): the operator's Tailscale OAuth
  client, fed to the HelmRelease through `valuesFrom`. The chart renders it into `operator-oauth`. To recreate it,
  make an OAuth client in the Tailscale admin console (Devices write and Auth Keys write scopes, tag
  `tag:k8s-operator`), then run
  `kubectl -n tailscale create secret generic tailscale-operator-oauth --from-literal=clientId=... --from-literal=clientSecret=...`.
- **Warning:** `overlays/production/cert-manager/cluster-issuer.yaml` *does* contain the Cloudflare DNS-01 API
  token in plain text. Rotate it and move it out of Git (SOPS, or a hand-made Secret like the one above).
  Don't commit new secrets.
- Host configuration: `tailscaled`, the k3s config, `/etc/sysctl.d` (a copy lives in `hosts/vm/`).

## Tailnet access to cluster services

```text
tailnet device -> 100.114.165.114:80/443     Tailscale device "kube-system-gateway-tailscale" (tag:k8s)
                  pod tailscale/ts-gateway-tailscale-*-0 (created by the Tailscale operator)
                    | TS_EXPERIMENTAL_DEST_DNS_NAME = cilium-gateway-gateway.kube-system.svc.cluster.local
                    v
                  Service kube-system/gateway-tailscale (ExternalName, tailscale.com/expose: "true")
                    v
                  Service kube-system/cilium-gateway-gateway -> Cilium L7 LB -> cilium-envoy
                    v  TLS *.ntcloud.dev, routed by HTTPRoute hostname
                  hubble-ui, flux-operator, ...
```

- **Add an app:** create an HTTPRoute with a `*.ntcloud.dev` hostname and
  `parentRefs: [{name: gateway, namespace: kube-system}]`. Nothing changes on the Tailscale side.
- **Who can reach it:** `100.114.165.114` is a tailnet address. Only devices signed in to this tailnet,
  and allowed by its access rules, can connect. Keep the Cloudflare record **DNS only**, because
  Cloudflare's proxy can't reach a tailnet address. Hubble UI has no login, so the tailnet access rules
  are the only access control.
- **IP stability:** the device identity is stored in Secret `tailscale/ts-gateway-tailscale-6jgq4-0`, so
  the IP survives reboots and pod restarts. It changes only when the device is recreated. That happens if
  the `gateway-tailscale` Service or its annotation is removed (including deleting the manifest from this
  repo, since Flux prunes), the Secret is deleted, the operator is uninstalled, or the machine is removed
  in the Tailscale admin console. If it happens, update the DNS record, or give the new device the old IP
  in the admin console (**Machines → Edit IP address**).

### Upgrading Flux

- **Flux Operator:** bump `ref.tag` in `clusters/production/flux-system/flux-operator.yaml`. Flux upgrades the
  operator itself through a HelmRelease that runs as the operator's own ServiceAccount. The CRDs have
  `helm.sh/resource-policy: keep`. Release notes:
  https://github.com/controlplaneio-fluxcd/flux-operator/releases
- **Flux controllers:** these come from `spec.distribution.version` in
  `clusters/production/flux-system/flux-instance.yaml` (`2.8.x` picks up patch releases automatically). Commit
  the change; Flux applies the FluxInstance and the operator rolls out the controllers.
- **Self-management:** the FluxInstance also defines the `flux-system` GitRepository and Kustomization that
  apply it. A bad `spec.sync` (URL, ref, path, secret) or `distribution` can stop Flux from pulling the fix,
  so review those carefully. It has `kustomize.toolkit.fluxcd.io/prune: disabled`, so deleting the file
  leaves Flux running.
- **Recovery:** if a bad operator release breaks Flux, run `helm -n flux-system rollback flux-operator`. If a
  bad FluxInstance breaks it, apply a good version from Git by hand with
  `kubectl apply --server-side --field-manager=kustomize-controller -f clusters/production/flux-system/flux-instance.yaml`.
  Then fix Git.

### Upgrading Tailscale

- **Operator:** bump `spec.chart.spec.version` in `base/tailscale-operator/helmrelease.yaml`. Versions are
  listed at `https://pkgs.tailscale.com/helmcharts/index.yaml`.
- **Proxies:** by default they use the operator's version. `proxyConfig.image.tag` in the overlay pins them
  separately; it was set because 1.102.5 shipped only the proxy image. Remove the pin once the chart
  catches up.
- **What a bump restarts:** the proxy pods, for a few seconds. The tailnet IPs stay the same because device
  state lives in Secrets.
- **Host `tailscaled`:** upgrade it separately with `sudo apt-get install --only-upgrade tailscale`.

## Known pitfalls

### 1. Tailscale's `src_valid_mark=1` breaks Cilium's L7 proxy (503s and timeouts)

- **Symptom:** the Gateway answers, but every request returns
  `503 upstream connect error or disconnect/reset before headers. reset reason: connection timeout`.
  Pods time out when connecting to the Gateway. Other Cilium L7 features, such as L7 and DNS policies,
  break the same way.
- **Cause:** with netfilter mode on (checked on v1.102), `tailscaled` sets `net.ipv4.conf.all.src_valid_mark=1`.
  Cilium marks proxy traffic `0x200`, and `ip rule 9: fwmark 0x200/0xf00 lookup 2004` resolves to
  `local default dev lo`. Once the mark is included in the reverse-path check, the kernel sees the pod's
  source address as local and drops the packet (`IP_LOCAL_SOURCE`). Upstream:
  [cilium/cilium#48706](https://github.com/cilium/cilium/issues/48706) and
  [tailscale/tailscale#19796](https://github.com/tailscale/tailscale/issues/19796).
- **Fix (installed):** [`hosts/vm/etc/sysctl.d/90-cilium-lxc-accept-local.conf`](hosts/vm/etc/sysctl.d/90-cilium-lxc-accept-local.conf)
  sets `accept_local=1` on Cilium's pod network interfaces (`lxc*`) only. systemd-sysctl applies it to
  each new interface through udev. Leave Tailscale's setting alone.
- **Not the cause:** `bpf.hostLegacyRouting`, `socketLB.*` and `gatewayAPI.hostNetwork` were changed while
  chasing this (commits `1bfa5b6`, `5aa24cc`, `5328efc`). None of them fixed it.
- **Check:**
  ```bash
  sysctl net.ipv4.conf.all.src_valid_mark                          # 1 while tailscaled runs
  cat /proc/sys/net/ipv4/conf/lxc*/accept_local | sort | uniq -c   # every value must be 1
  # Broken: "RTNETLINK answers: Invalid argument". Fixed: "local 10.0.0.5 ... table 2004".
  ip route get 10.0.0.5 from <pod-ip> iif <pod's lxc interface> mark 0x200
  ```
- **On a rebuilt host:**
  ```bash
  sudo install -m 0644 hosts/vm/etc/sysctl.d/90-cilium-lxc-accept-local.conf /etc/sysctl.d/
  sudo /usr/lib/systemd/systemd-sysctl /etc/sysctl.d/90-cilium-lxc-accept-local.conf
  ```

### 2. Don't route to the Gateway through its NodePorts

With kube-proxy replacement, Cilium can't chain one Service into another Service's NodePort. A selectorless
Service whose EndpointSlice points at `10.0.0.188:<nodePort>` gets its connections refused. To expose the
Gateway, point an **ExternalName** Service at `cilium-gateway-gateway.kube-system.svc.cluster.local` (the
Tailscale operator supports ExternalName Services), or set `spec.infrastructure.annotations` on the Gateway.

### 3. Cilium config changes need an operator and agent restart

The chart doesn't restart pods when `cilium-config` changes. After `gatewayAPI.hostNetwork` was turned off
on Sep 7, the operator kept the old behavior until the Sep 26 reboot. After that reboot, the 1.19.7 agent
left Envoy listening on the old `0.0.0.0:80/443` addresses while sending traffic to a new proxy port, so
every connection to the Gateway was refused.

- **After changing Gateway, Envoy or proxy values**, run
  `kubectl -n kube-system rollout restart deployment/cilium-operator daemonset/cilium`, or set
  `rollOutCiliumPods: true` and `operator.rollOutPods: true` in the values.
- **Verify** that both commands show the same `127.0.0.1:<port>`:
  ```bash
  sudo curl -s --unix-socket /var/run/cilium/envoy/sockets/admin.sock http://localhost/listeners | grep gateway
  GW=$(kubectl -n kube-system get svc cilium-gateway-gateway -o jsonpath='{.spec.clusterIP}')
  kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg service list | grep "$GW:443"
  ```

### 4. Overlapping IP ranges

The pod CIDR `10.0.0.0/16` (this node's `/24` is `10.0.0.0/24`) overlaps the VCN subnet `10.0.0.0/24` on
`enp0s3`. That works on a single node but blocks traffic to other hosts in the VCN. Don't advertise pod or
VCN ranges into the tailnet.

## End-to-end checks

```bash
# From a tailnet device
curl -sS -o /dev/null -w '%{http_code}\n' https://hubble.ntcloud.dev/     # expect 200 and valid TLS
curl -sS -o /dev/null -w '%{http_code}\n' \
  --resolve hubble.ntcloud.dev:443:100.114.165.114 https://hubble.ntcloud.dev/   # bypasses DNS

# On the node
GW=$(kubectl -n kube-system get svc cilium-gateway-gateway -o jsonpath='{.spec.clusterIP}')
curl -sS -o /dev/null -w '%{http_code}\n' --resolve hubble.ntcloud.dev:443:$GW https://hubble.ntcloud.dev/   # expect 200
kubectl get gateway,httproute -A
kubectl -n tailscale get pods
kubectl -n kube-system exec ds/cilium -c cilium-agent -- hubble observe --since 5m --verdict DROPPED
```

## Rules for agents

- Change cluster state through this repo. Flux reapplies everything it manages, so a `kubectl edit` on
  those objects doesn't last. Run `kubectl kustomize clusters/production` before pushing.
- Never commit secrets. Stage files explicitly with `git add <path>`, never `git add -A`, because the
  working tree contains untracked secrets.
- Diagnose read-only first, and keep changes small and reversible.
- After changing Cilium Gateway or Envoy values, restart the operator and agent (pitfall 3) and run the
  end-to-end checks.
- Record new findings in this file.
