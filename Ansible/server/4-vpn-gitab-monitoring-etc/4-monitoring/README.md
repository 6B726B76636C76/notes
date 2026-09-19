# Monitoring

## Purpose

This directory manages the Prometheus/Grafana monitoring stack used to observe the Kubernetes cluster, the VPS host, and selected infrastructure components.

The current Kubernetes monitoring release is the `kube-prometheus-stack` Helm release named `monitoring` in namespace `monitoring`.

The current design enables Prometheus and Grafana, keeps the bundled Alertmanager disabled, keeps the chart's nodeExporter disabled because the host node_exporter is scraped separately, and enables kube-state-metrics.

## Main components

```text
Prometheus
  ├── Kubernetes metrics
  ├── kube-state-metrics
  ├── Cilium / service metrics where configured
  └── host node_exporter

Grafana
  ├── Prometheus datasource
  └── Loki datasource where configured

Host
  ├── node_exporter :9100
  └── log collector / Loki integration
```

The current Prometheus scrape configuration includes a static host target:

```text
91.114.34.211:9100
```

with labels identifying the VPS as `vpc` and the source as `vps`.

## Responsibilities

The module is responsible for:

- managing the Prometheus Community Helm repository;
- installing/upgrading `kube-prometheus-stack`;
- configuring Prometheus resource and retention parameters;
- enabling kube-state-metrics;
- configuring the host node_exporter scrape target;
- configuring Grafana and its datasources;
- keeping the monitoring release declarative through Ansible variables/templates;
- providing the foundation consumed by the separate Grafana Managed Alerting module.

## Important current choices

### Alertmanager

The bundled Alertmanager is disabled:

```yaml
alertmanager:
  enabled: false
```

Grafana Managed Alerting is used instead for the custom Telegram alerting flow.

### nodeExporter

The chart's nodeExporter component is disabled because the host already exposes node_exporter independently.

### Prometheus retention

Current retention is `3d`.

### Resource targets

Current starting targets are approximately:

```text
Grafana requests  200Mi
Grafana limit     500Mi
Prometheus req    300Mi
Prometheus limit  900Mi
```

## Prerequisites

Before applying:

1. Kubernetes is healthy.
2. Cilium is installed and working.
3. The host node_exporter is listening on `9100`.
4. The Prometheus namespace is available.
5. The StorageClass used by Grafana persistence exists.

## Apply

```bash
ansible-playbook \
  4-vpn-gitab-monitoring-etc/4-monitoring/monitoring.yml \
  --ask-vault-pass
```

## Validation

List monitoring resources:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get servicemonitors -n monitoring
```

Check Prometheus:

```bash
kubectl get prometheus -n monitoring
kubectl get sts -n monitoring | grep prometheus
```

Check the host exporter:

```bash
curl -s http://127.0.0.1:9100/metrics | head
```

Port-forward Prometheus when API access from the host is needed:

```bash
kubectl -n monitoring port-forward \
  svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Then:

```bash
curl -s http://127.0.0.1:9090/-/ready
curl -s http://127.0.0.1:9090/api/v1/targets
```

## Grafana persistence

Grafana must use persistent storage. The current installed state uses:

```text
PVC: monitoring-grafana
StorageClass: local-path
Size: 2Gi
Access mode: RWO
```

This matters because Grafana stores users, alert state, configuration metadata, and other local application data in its data directory. A restart must not recreate the database from scratch.

## Troubleshooting

### Prometheus pod is running but an alert says `NoData`

A running Prometheus process does not prove the alert query is correct. Test the exact PromQL against the Prometheus API.

For example:

```bash
kubectl -n monitoring port-forward \
  svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Then query the metric directly with `curl`.

This distinction matters because a selector that matches no series produces `NoData` even though Prometheus itself is healthy.

### Grafana password suddenly stops working after a restart

Check persistence first:

```bash
kubectl -n monitoring get pvc
helm get values monitoring -n monitoring
```

A missing Grafana PVC means the database may be ephemeral. Do not use destructive restarts until persistence is confirmed.

### Grafana cannot reach Prometheus

Validate the datasource URL and test it from inside the Grafana network path. Also check Service/endpoints:

```bash
kubectl -n monitoring get svc monitoring-kube-prometheus-prometheus
kubectl -n monitoring get endpoints monitoring-kube-prometheus-prometheus
```

### Host node_exporter target is down

Verify:

```bash
curl -s http://127.0.0.1:9100/metrics | head
sudo ss -lntup | grep ':9100'
```

Then check UFW rules for Prometheus source traffic.

### Prometheus is overloaded

Review target count, scrape interval, cardinality, and resource usage before simply increasing resources. The current additional scrape uses a `15s` interval.

## Interaction with Grafana Alerting

The monitoring module installs the base Grafana/Prometheus platform. The separate `8-grafana-alerting` module then:

```text
monitoring Helm release
        |
        +--> Grafana
        |
        +--> Prometheus datasource
        |
        v
Grafana Alerting provisioning
```

The alerting module intentionally reuses the existing monitoring release rather than installing a second Grafana.

## Suggested commit

```text
feat(monitoring): document Prometheus and Grafana platform setup
```
