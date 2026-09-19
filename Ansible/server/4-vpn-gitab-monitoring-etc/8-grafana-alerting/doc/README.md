# Grafana Alerting

## Purpose

This directory adds Grafana Managed Alerting to the existing `monitoring` Helm release and routes alert notifications to Telegram.

It is deliberately implemented as an Ansible-managed provisioning layer on top of the existing Grafana deployment rather than as a separate monitoring stack.

## Current architecture

```text
Prometheus
    |
    v
Grafana Managed Alerting
    |
    +--> Alert rules
    +--> Notification policies
    +--> Telegram contact points
    |
    v
Telegram Bot API
```

The bundled Prometheus Alertmanager remains disabled. Telegram delivery is handled by Grafana itself.

## Responsibilities

The playbook and templates are responsible for:

1. validating the alerting configuration;
2. ensuring the Prometheus Community Helm repository is available;
3. reading the current `monitoring` release/version;
4. creating the Kubernetes Secret `grafana-telegram` with the bot token;
5. rendering five Grafana provisioning files:
   - contact points;
   - notification policies;
   - node alerts;
   - Kubernetes alerts;
   - platform alerts;
6. creating the `grafana-alerting` ConfigMap;
7. rendering additional Helm values;
8. upgrading the existing `monitoring` release with `--reuse-values`;
9. mounting the ConfigMap at `/etc/grafana/provisioning/alerting`;
10. restarting Grafana after alerting configuration changes;
11. waiting for the Grafana rollout to complete;
12. cleaning temporary rendered files.

## Secret model

The Telegram bot token is stored only in encrypted Ansible Vault data and becomes a Kubernetes Secret in the monitoring namespace.

Conceptually:

```text
group_vars/secrets.yml
      |
      v
vault_grafana_telegram_bot_token
      |
      v
Kubernetes Secret grafana-telegram
      |
      v
Grafana environment
```

Telegram chat IDs are configuration data and can be rendered into the contact-point provisioning file. The bot token must never appear in the rendered ConfigMap.

## Current variable set

The alerting layer uses variables equivalent to:

```yaml
grafana_alerting_namespace: monitoring
grafana_alerting_release: monitoring
grafana_alerting_configmap: grafana-alerting
grafana_alerting_telegram_secret: grafana-telegram
grafana_alerting_prometheus_datasource_uid: prometheus
grafana_alerting_folder: Infrastructure
grafana_alerting_interval: 60s
grafana_alerting_config_dir: /tmp/grafana-alerting
```

Grafana persistence is also enabled through the Helm values used by this module. The current installed PVC is `monitoring-grafana`, using `local-path` and `2Gi`.

## Alert inventory

The current custom configuration contains 16 rules:

```text
Node alerts         4
Kubernetes alerts   3
Platform alerts     9
--------------------
Total              16
```

The platform layer monitors components such as Prometheus, Grafana, GitLab Webservice, Harbor, and Argo CD.

## Important rule corrections already made

### Prometheus selector

The real Prometheus StatefulSet is:

```text
prometheus-monitoring-kube-prometheus-prometheus
```

The alert must use that exact label value. An earlier selector omitted the `prometheus-` prefix and produced `NoData` even though Prometheus was healthy.

### WireGuard interface check

The host reports:

```text
adminstate="up"
operstate="unknown"
node_network_up{device="wg0"} = 0
```

Therefore the current alert uses:

```promql
max(node_network_info{device="wg0",adminstate="up"}) or vector(0)
```

This avoids a false positive caused by the generic `node_network_up` metric.

## Apply

```bash
ansible-playbook \
  4-vpn-gitab-monitoring-etc/8-grafana-alerting/alerting.yml \
  --ask-vault-pass
```

## Validation

Check the Grafana pod:

```bash
kubectl get pods -n monitoring | grep grafana
```

Check the Secret without printing it:

```bash
kubectl get secret grafana-telegram -n monitoring
```

Check the ConfigMap:

```bash
kubectl get configmap grafana-alerting -n monitoring
```

Check Grafana logs for provisioning:

```bash
kubectl logs deploy/monitoring-grafana -n monitoring --since=10m \
  | grep -Ei 'provision|alert|telegram|error'
```

Successful provisioning includes messages equivalent to:

```text
starting to provision alerting
finished to provision alerting
```

The current deployment has also reported 16 initialized alert rules.

## UI validation

In Grafana:

```text
Alerting → Alert rules
Alerting → Contact points
Alerting → Notification policies
```

Expected custom state:

```text
Folder: Infrastructure
Contact point: telegram
Receiver routing: telegram
```

Send a test Telegram notification from the contact point before declaring the integration complete.

## Persistence requirement

This module performs a Grafana restart when configuration changes. Grafana must therefore have persistent storage before routine upgrades/restarts.

Current state:

```text
PVC                 monitoring-grafana
Status              Bound
StorageClass        local-path
Size                2Gi
Access mode         RWO
```

Without persistence, restarting Grafana can recreate its local database and reset users/configuration.

## Troubleshooting

### Alert fires with `NoData` while the service is healthy

Test the PromQL directly against Prometheus. A selector that matches no series is not equivalent to a failed Prometheus server.

Example:

```bash
kubectl -n monitoring port-forward \
  svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Then query the exact expression through `/api/v1/query`.

### Telegram message is delivered but Grafana is inaccessible

Alert delivery and web authentication are separate systems. First check the Grafana pod, Service, Ingress, and persistent volume. If the password changed after a restart, verify that the PVC was mounted.

### Grafana provisioning logs show `invalid suffix '..data'`

These warnings come from Kubernetes ConfigMap's projected-volume internals. Grafana skips the `..data` and timestamped helper paths and processes the `.yaml` files.

### `sh` is missing in the Grafana container

Use a container-compatible inspection method; do not assume every image contains `sh`. This does not indicate a provisioning failure by itself.

### Rules are present but notifications do not arrive

Check in order:

```text
rule firing
  -> notification policy
  -> contact point
  -> Telegram bot token
  -> chat ID
  -> network access to Telegram API
```

A successful alert evaluation does not prove message delivery.

## Maintenance

The provisioning files are managed by Ansible. Do not make long-lived manual GUI changes to provisioned resources and expect them to survive the next playbook run.

When modifying a rule:

1. edit the appropriate Jinja template;
2. syntax-check the playbook;
3. apply with `--ask-vault-pass`;
4. inspect Grafana provisioning logs;
5. inspect the rule in the UI;
6. test notification delivery.

## Suggested commit

```text
feat(grafana-alerting): document managed alerting and Telegram provisioning
```
