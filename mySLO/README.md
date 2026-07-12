# mySLO: SLOs from Linkerd mesh metrics

SLO lab built on the OpenTelemetry Demo, meshed with Linkerd, scraped by kube-prometheus-stack. No app code changes: availability and latency SLIs come from the Linkerd proxy's `response_total` and `response_latency_ms_bucket` metrics.

## Layout

- `prometheus/kube-prometheus-stack.values.yaml`, Helm values; the key setting is `*SelectorNilUsesHelmValues: false` so our PodMonitor and rules are discovered
- `prometheus/slo-prometheus-rules.yaml`, PrometheusRule with SLI recording rules, a naive threshold alert (as the anti-pattern), and the burn-rate alert you actually ship
- `manifests/otel-demo-namespace.yaml`, namespace with `linkerd.io/inject: enabled`
- `manifests/linkerd-proxy-podmonitor.yaml`, scrapes the Linkerd proxy sidecars (native sidecar, port `linkerd-admin` 4191)
- `manifests/frontend-httproutes.yaml`, Gateway API HTTPRoutes so per-route SLOs get real `route_name` labels
- `manifests/slo-demo-faulty-service.yaml`, httpbin `orders-api` returning ~25% HTTP 500 to demo an SLO breach and burn-rate alert
- `dashboard/slo-from-mesh.dashboard.json`, Grafana dashboard for the SLIs, error budget, and burn rate

## Apply order

```sh
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace \
  -f prometheus/kube-prometheus-stack.values.yaml

kubectl apply -f manifests/otel-demo-namespace.yaml
# install the OpenTelemetry Demo into otel-demo, then:
kubectl apply -f manifests/linkerd-proxy-podmonitor.yaml
kubectl apply -f manifests/frontend-httproutes.yaml
kubectl apply -f manifests/slo-demo-faulty-service.yaml
kubectl apply -f prometheus/slo-prometheus-rules.yaml
```

Import `dashboard/slo-from-mesh.dashboard.json` into Grafana (login admin/admin, lab only).
