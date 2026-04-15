# ad-otel

Full-stack k8s observability for the sandbox-east cluster.

Metrics and logs are collected by OpenTelemetry Collector, stored in VictoriaMetrics and VictoriaLogs, and visualised in Grafana. 

Currently deployed with Helm CLI.

ArgoCD deployment is planned (see [Planned: ArgoCD](#planned-argocd-deployment)).

## Stack

| Component               | Role                                                     |
|-------------------------|----------------------------------------------------------|
| OpenTelemetry Collector | DaemonSet on every node - collects all signals           |
| VictoriaMetrics         | Metrics storage - 1-month retention                      |
| VictoriaLogs            | Log storage - 30-day retention                           |
| Grafana                 | Unified UI - datasources and dashboards auto-provisioned |

## Telemetry layers

Each layer has dedicated receivers, its own processor chain, and a `layer` label for clean separation in Grafana.

#### OS - `layer=infra`
Host-level collection via direct mounts (`/proc`, `/sys`, `/var/log`).
Receivers: `hostmetrics` (CPU, memory, disk, filesystem, network, processes), `filelog/system` (syslog).

#### Kubernetes `layer=kuber`

Workload signals enriched with full k8s metadata. 
Receivers: `kubeletstats` (pod CPU/memory/network from kubelet API on port 10250), `k8s_events` (cluster-wide events). 
Processor: `k8sattributes` adds namespace, pod, deployment, and node labels to every signal.


#### Application `layer=app`

Zero-config annotation-driven discovery across all namespaces. 
Any pod with `prometheus.io/scrape: "true"` is scraped automatically. 
Receivers: `prometheus/app` (request rate, latency, custom metrics), `filelog/app` (CRI-parsed container logs from all pods with namespace/pod/container attribution).

## Pipelines

```
hostmetrics + kubeletstats  →  resourcedetection · k8sattributes · attributes · batch  →  VictoriaMetrics
prometheus/app              →  resource/app · resourcedetection · k8sattributes · batch  →  VictoriaMetrics

filelog/system   →  resource/infra · resourcedetection · k8sattributes · batch  →  VictoriaLogs
k8s_events                  →  resource/kuber · resourcedetection · batch                  →  VictoriaLogs
filelog/app                 →  resource/app · resourcedetection · k8sattributes · batch    →  VictoriaLogs
```

## Dashboards

Three dashboards ship pre-provisioned as ConfigMaps and no manual import required.

| Dashboard                 | Panels                                              |
|---------------------------|-----------------------------------------------------|
| Node Infrastructure       | CPU, memory, disk, network per node · system logs   |
| Kubernetes Infrastructure | Pod CPU/memory/network · k8s events log             |
| App in all namespaces     | Request rate, p95/p99 latency · application logs    |

All log panels filter by `layer` label: `{layer="infra"}`, `{layer="kuber"}`, `{layer="app"}`.

App metric panels require pods to expose `http_request_duration_seconds` histogram and carry `prometheus.io/scrape: "true"` annotation.

## Deploy (current Helm CLI)

```bash
CONTEXT=sandbox-east
NS=ad-otel

helm upgrade --install ad-otel-victoriametrics helm/victoriametrics -n $NS --create-namespace --kube-context $CONTEXT
helm upgrade --install ad-otel-victorialogs    helm/victorialogs    -n $NS --create-namespace --kube-context $CONTEXT
helm upgrade --install ad-otel-otelcol         helm/otelcol         -n $NS --create-namespace --kube-context $CONTEXT
helm upgrade --install ad-otel-grafana         helm/grafana         -n $NS --create-namespace --kube-context $CONTEXT
```

Deploy VictoriaMetrics and VictoriaLogs before otelcol on first install so exporters can connect immediately.
Subsequent upgrades can run in any order.

## Access Grafana

Quick local access:

```bash
kubectl port-forward svc/ad-otel-grafana 3000:80 -n ad-otel --context sandbox-east
# → http://localhost:3000
```

Via ingress — add to `/etc/hosts`:

```
<nginx-ingress-external-ip>  grafana.ad-otel.local
```

Then open `http://grafana.ad-otel.local`. Anonymous admin access is enabled and no login required.

## Enabling app-level metrics for a pod

Add these annotations to any pod spec:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"   # port where /metrics is exposed
```

The OTel Collector picks it up within 30 seconds. 
Metrics appear in VictoriaMetrics with `job="all-namespaces"` and `namespace`, `pod` labels. 
Logs from the same pod appear automatically via `filelog/app`.

## Directory structure

```
helm/
  grafana/
    templates/
      configmap.yaml        # grafana.ini + datasources.yaml + dashboards.yaml provider
      dashboard-infra.yaml  # Node Infrastructure dashboard (layer=infra)
      dashboard-kuber.yaml  # Kubernetes Infrastructure dashboard (layer=kuber)
      dashboard-app.yaml    # App dashboard (layer=app, all namespaces)
      deployment.yaml       # Single replica, securityContext uid/gid 472 (Longhorn compat)
      ingress.yaml          # NGINX ingress on grafana.ad-otel.local
      pvc.yaml              # 2Gi Longhorn PVC
      service.yaml          # ClusterIP :80 → :3000

  otelcol/
    templates/
      configmap.yaml        # Full OTel config: receivers, processors, exporters, pipelines
      daemonset.yaml        # Root + privileged, mounts: /proc /sys /var/log
      serviceaccount.yaml
      clusterrole.yaml      # ClusterRole + Binding (kubelet, pods, events, nodes)

  victoriametrics/
    templates/
      deployment.yaml       # strategy: Recreate, 1-month retention
      pvc.yaml              # 10Gi Longhorn PVC
      service.yaml          # ClusterIP :8428

  victorialogs/
    templates/
      deployment.yaml       # strategy: Recreate, 30d retention, --memory.allowedPercent=60
      pvc.yaml              # 10Gi Longhorn PVC
      service.yaml          # ClusterIP :9428

argocd/                     # Planned — see below
  app-of-apps.yaml
  apps/
    application-grafana.yaml
    application-otelcol.yaml
    application-victorialogs.yaml
    application-victoriametrics.yaml
```

## Planned: ArgoCD deployment

ArgoCD is available on sandbox-east. 
The `argocd/` manifests are prepared and point to this GitLab repo. 
Once the stack is validated via Helm, the plan is to switch to GitOps.


## Design notes

Static otelcol config - no Helm `{{ }}` templating inside the block scalar. 
Component addresses are hardcoded Kubernetes service DNS names (`ad-otel-*.ad-otel.svc.cluster.local`).

ConfigMap-provisioned dashboards - datasources and dashboards survive pod restarts without re-importing. 
Dashboard JSON is embedded in Helm templates as a raw string block.

Projected volume for dashboards - all three dashboard ConfigMaps and the provisioning config are merged into a single directory at `/etc/grafana/provisioning/dashboards` using a projected volume.

Grafana securityContext - uid/gid 472 required for Longhorn PVCs. Without it Grafana cannot write to `/var/lib/grafana` and crashes on startup.

Deployment strategy: Recreate - VictoriaMetrics and VictoriaLogs use RWO PVCs with an exclusive file lock. 
RollingUpdate would start the new pod before the old one releases the lock, causing a crash. `Recreate` terminates the old pod first.

Memory sizing - VictoriaLogs limit is 1Gi with `--memory.allowedPercent=60`. 
OtelCol limit is 512Mi. Both use `start_at: end` for log receivers to avoid reading all historical logs on restart. 
If OOM kills reappear (`Exit Code 137` in pod describe), increase the limits in `values.yaml`.

`layer` label as routing key - one `resource/*` processor per pipeline inserts `layer=infra|kuber|app`. 
This single attribute cleanly separates all three signal tiers in every Grafana dashboard.

DaemonSet runs as root - required to read `/proc`, `/sys`, `/var/log/pods` from the host.

Cluster-wide monitoring - `prometheus/app` uses cluster-wide Kubernetes SD (no namespace filter). 
`filelog/app` pattern is `/var/log/pods/*/*/*.log`. 
Both were scoped to a single namespace in the reference repo and were widened here to cover all workloads on sandbox-east.
