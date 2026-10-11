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
| Monitoring | OTel Collector 0.161 (DaemonSet) → vmauth → VictoriaMetrics single v1.153, Grafana. Namespace `monitoring`; see [Monitoring](#monitoring) |
| DNS | Cloudflare zone `ntcloud.dev`. `*.ntcloud.dev` is an `A` record to `100.114.165.114`, **DNS only** |

## Repository layout

| Path | Contents |
|---|---|
| `clusters/production/kustomization.yaml` | Root: `flux-system` and `overlays/production/{cilium,cert-manager,tailscale-operator,monitoring}` |
| `clusters/production/flux-system/` | HelmRepositories (`cilium`, `jetstack`, `tailscale`, `victoriametrics`, `otel-helm`, `prometheus-community`), Flux Operator OCIRepository and HelmRelease (`flux-operator.yaml`), HTTPRoute `fluxcd.ntcloud.dev` to `flux-operator:9080` |
| `clusters/production/base/cilium/` | Cilium HelmRelease (chart version) |
| `clusters/production/overlays/production/cilium/helmrelease-patch.yaml` | Cilium Helm values |
| `clusters/production/overlays/production/cilium/gateway.yaml` | LB IP pool, GatewayClass `cilium`, Certificate, Gateway `kube-system/gateway`, HTTPRoute `hubble.ntcloud.dev` |
| `clusters/production/overlays/production/cilium/gateway-tailscale.yaml` | Puts the Gateway on the tailnet (see below) |
| `clusters/production/overlays/production/cert-manager/` | cert-manager values and the ClusterIssuer |
| `clusters/production/base/tailscale-operator/` | Tailscale operator HelmRelease (chart version, OAuth `valuesFrom`) |
| `clusters/production/overlays/production/tailscale-operator/` | Namespace `tailscale` and operator values (proxy image tag) |
| `clusters/production/base/monitoring/agent/` | Monitoring agent, deployed in **every** cluster: OTel collector, node-exporter, kube-state-metrics |
| `clusters/production/base/monitoring/hub/` | Monitoring hub, deployed in **one** cluster: victoria-metrics-k8s-stack (VictoriaMetrics, vmauth, Grafana) |
| `clusters/production/overlays/production/monitoring/` | Namespace, hub values (`vm-stack-patch.yaml`), this cluster's collector settings (`otel-patch.yaml`), ingest HTTPRoute |
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
- Secret `monitoring/grafana-admin` (keys `admin-user`, `admin-password`): the Grafana login. Read the password
  with `kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d`.
  Recreate it with
  `kubectl -n monitoring create secret generic grafana-admin --from-literal=admin-user=admin --from-literal=admin-password="$(openssl rand -base64 24)"`.
- Secrets `monitoring/vmuser-<cluster>`: metrics ingest credentials, generated by the VictoriaMetrics operator
  for each VMUser (see [Monitoring](#monitoring)).
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

## Monitoring

```text
every cluster (base/monitoring/agent)                  hub, this cluster only (base/monitoring/hub)
node-exporter :9100 ───┐                               vmauth-vm-stack:8427 ──adds cluster=<name>──▶ vmsingle-vm-stack:8428
kube-state-metrics ────┼─scrape─▶ otel-collector ──OTLP + basic auth──▶ ▲                              (emptyDir, 7d)
kubelet / cAdvisor ────┘          (DaemonSet)                           │ https://metrics.ntcloud.dev       │
apps ──OTLP :4317/:4318──────────▶                                      │ (other clusters, tailnet-only)    ▼
                                                                                      Grafana: https://grafana.ntcloud.dev
```

- **Collector pipelines** (`base/monitoring/agent/otel.yaml`): `metrics` receives OTLP from apps (enriched by
  `k8s_attributes`); `metrics/scrape` scrapes the Prometheus exporters. VictoriaMetrics doesn't map
  `service.name`/`service.instance.id` back to `job`/`instance` and turns every resource attribute into a label,
  so `transform/prometheus` restores `job`/`instance` and drops the receiver's resource attributes. Scraped series
  then look exactly like a Prometheus scrape. Job names follow vm-stack conventions so its alert/recording rules
  will match later: `node-exporter`, `kube-state-metrics`, `kubelet` (plus `metrics_path`), `otel-collector`,
  and `victoria-metrics` (hub only).
- **VictoriaMetrics** (`vm-stack-patch.yaml`): `-opentelemetry.usePrometheusNaming=true` converts OTLP names and
  labels to Prometheus style (`k8s.pod.name` → `k8s_pod_name`, unit and `_total` suffixes); names that are already
  Prometheus-style pass through unchanged. `-opentelemetry.promoteScopeMetadata=false` keeps `otel_scope_*`
  labels off. Retention is `7d`.
- **No PVC.** Metrics are lost whenever the VMSingle pod is recreated (chart upgrade, config change, reboot).
  To persist them, remove the JSON6902 `storage: null` patch in `overlays/production/monitoring/kustomization.yaml`
  (the chart default is a 20Gi `local-path` PVC).
- **`cluster` label:** vmauth adds it. Each cluster has a VMUser whose route appends
  `?extra_label=cluster=<name>`. Collectors don't set it themselves, so one cluster can't write under another's name.
- **Grafana:** user `admin`, password in Secret `monitoring/grafana-admin`. It has three datasources:
  `VictoriaMetrics` (Prometheus type, the default), `VictoriaMetrics (DS)` (MetricsQL plugin) and
  `VictoriaMetrics (high-res)` (see [High-resolution metrics](#high-resolution-metrics)). The chart's
  sync-job downloads the dashboards from GitHub on every install and upgrade: Node Exporter Full,
  Kubernetes / Views / {Global, Namespaces, Nodes, Pods}, and VictoriaMetrics - single-node. Grafana isn't
  persistent either, so dashboards edited in the UI are lost on restart; keep them in Git.
- **Extra dashboards** live in `base/monitoring/hub/dashboards/` (Kubernetes: API Server, CoreDNS; Cilium: agent,
  operator, Hubble, Hubble L7 HTTP). A `configMapGenerator` makes one ConfigMap per JSON file, labeled
  `grafana_dashboard: "1"`, with a `grafana_folder` annotation that the sidecar turns into a Grafana folder.
  To add one, drop the JSON in a folder there and add a generator entry. Datasource references must be the
  default datasource, uid `VictoriaMetrics`, or a datasource variable without a saved `current`.
- **Apps** send OTLP to `http://otel-collector.monitoring.svc:4318` (HTTP) or `:4317` (gRPC).
- **Alerting** isn't deployed yet: vmalert, Alertmanager, the default rules and the kube-prometheus dashboards
  (which need recording rules) are disabled.

### Adding another cluster

1. On the hub, add a VMUser to `extraObjects` in `vm-stack-patch.yaml`. Copy `production`, then change the name,
   `username` and `cluster=<name>`. Once Flux applies it, the credentials are in Secret `monitoring/vmuser-<name>`.
2. On the new cluster, deploy `base/monitoring/agent` with an overlay that sets
   `VM_INGEST_URL=https://metrics.ntcloud.dev/opentelemetry/v1/metrics`, plus `VM_INGEST_USERNAME` and
   `VM_INGEST_PASSWORD` from a Secret holding those credentials. It needs no `dependsOn` and no `prometheus/hub`.
3. Its collector pods must reach the tailnet. Either its nodes run `tailscaled` and the CNI masquerades traffic
   leaving through `tailscale0` (as here), or it uses a Tailscale operator egress Service.
4. A cluster with more than one node must scrape kube-state-metrics from one collector only, for example a
   separate deployment-mode collector. Today every DaemonSet pod scrapes it.

### High-resolution metrics

VictoriaMetrics stores millisecond timestamps, and Grafana can query down to a 1 ms step. Use the
`VictoriaMetrics (high-res)` datasource (uid `VictoriaMetricsHighRes`) for such series:

- **Step floor:** Grafana's step is the time range divided by the panel width, but never below the panel's
  *Min interval*, or else the datasource's `timeInterval`. A Prometheus-type datasource without `timeInterval`
  (like `VictoriaMetrics`) uses 15s, so it can't zoom below 15s steps. The high-res datasource sets `1ms`.
- **Newest samples:** VictoriaMetrics hides the last 30s from queries (`-search.latencyOffset`). The high-res
  datasource sends `latency_offset=1s` with each query instead of changing the global flag.
- **Zoomed-out panels** return one sample per step, so short spikes disappear. Wrap the series in
  `max_over_time(m[$__interval])` (or `min_`/`avg_over_time`) to keep them.
- **Ingestion:** scrapes and OTel SDK exporters produce one point per collection interval. Write each sample with
  its own timestamp instead, e.g. JSON lines to `/api/v1/import` on `vmsingle-vm-stack:8428`. vmauth and the
  `metrics.ntcloud.dev` HTTPRoute only route `/opentelemetry/`, so pushing from outside the cluster needs another
  VMUser path and route rule. Don't set `-dedup.minScrapeInterval`: it would collapse the samples.

### Monitoring gotchas

- **Kustomize drops `null` in strategic-merge patches.** Helm "remove this chart default" values (`key: null`)
  must go in a JSON6902 patch, like the `storage: null` one, or in a base resource.
- **`${env:...}` in the collector config** is resolved by the collector at runtime. The `flux-system` Kustomization
  has no `postBuild` substitution, so it passes through untouched; escape it if substitution is ever enabled.
- **VMUser objects live in the chart's `extraObjects`,** so Helm creates the VictoriaMetrics CRDs first. A plain
  VMUser manifest would fail the whole Flux apply on a fresh cluster.
- **`alternateConfig` is the complete collector config;** it isn't merged with the chart's defaults. Presets still
  inject (`k8s_attributes` goes into the pipeline named `metrics`), so keep a pipeline with that name.
- **The collector's own `target_info`** would override `job`/`instance` on its self-scrape, so it is dropped in
  that scrape job.
- **Validate collector changes before pushing:** run `helm template` on the chart with the built values, then
  `otelcol-k8s validate --config=<rendered relay.yaml>` using the linux/arm64 binary from the collector releases.

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

# Monitoring (from a tailnet device)
curl -sS -o /dev/null -w '%{http_code}\n' https://grafana.ntcloud.dev/login                             # expect 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST https://metrics.ntcloud.dev/opentelemetry/v1/metrics  # expect 401
# Monitoring (on the node): every job up=1 with cluster="production"
kubectl -n monitoring get helmrelease,pods
VM=$(kubectl -n monitoring get svc vmsingle-vm-stack -o jsonpath='{.spec.clusterIP}')
curl -s -G http://$VM:8428/api/v1/query --data-urlencode 'query=up'
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
