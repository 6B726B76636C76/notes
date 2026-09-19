# Monitoring Architecture

## Platform topology

```text
                           Kubernetes cluster
                                  |
             +--------------------+--------------------+
             |                    |                    |
         Prometheus          kube-state-metrics     Cilium metrics
             |                    |                    |
             +--------------------+--------------------+
                                  |
                          metric time series
                                  |
                                  v
                         +----------------+
                         |    Grafana     |
                         | namespace:     |
                         | monitoring     |
                         +-------+--------+
                                 |
                    +------------+------------+
                    |                         |
               Prometheus                  Loki
               datasource               datasource*

Host VPS
  |
  +--> node_exporter :9100
          |
          +------> Prometheus static scrape target

* Loki availability depends on the broader monitoring/logging deployment.
```

## Prometheus data flow

```text
Kubernetes objects / exporters
          |
          v
ServiceMonitor / scrape configuration
          |
          v
Prometheus
          |
          +--> rule evaluation
          |
          v
Grafana datasource
```

The host scrape is explicit:

```text
Prometheus pod
   -> 91.219.62.186:9100
   -> node_exporter
```

The firewall therefore has to permit this source path without making port `9100` public.

## Grafana state

```text
Grafana pod
   |
   +--> /var/lib/grafana
            |
            v
      PVC monitoring-grafana
            |
            v
       local-path storage
```

This persistent volume protects Grafana's local database across pod recreation and Helm upgrades.

## Alerting architecture

Alerting is deliberately split:

```text
Prometheus
   |
   +--> stores/evaluates metrics
   |
   v
Grafana Managed Alerting
   |
   +--> rule evaluation
   +--> notification policy
   +--> Telegram contact points
```

Bundled Alertmanager is disabled in the current design.

## Service dependencies

```text
Kubernetes API / exporters
          |
          v
      Prometheus
          |
          v
       Grafana
          |
          v
  Grafana Managed Alerting
          |
          v
        Telegram
```

A Grafana alert can fail because the monitored metric is wrong even when Prometheus itself is healthy. Therefore alert troubleshooting must test the query independently.
