# Grafana Alerting Architecture

## End-to-end notification path

```text
                    Prometheus
                        |
                        | datasource UID: prometheus
                        v
              +----------------------+
              | Grafana Unified      |
              | / Managed Alerting   |
              +----------+-----------+
                         |
                rule evaluation
                         |
                         v
                Infrastructure folder
                         |
                         v
               Notification policy
                         |
                         v
                  Telegram contact
                         |
                         v
                  Telegram Bot API
                         |
                         v
                    Chat IDs
```

## Provisioning architecture

```text
group_vars/all.yml
        |
        +--> alerting settings
        |
group_vars/secrets.yml
        |
        +--> Telegram bot token
        |
        v
Ansible tasks
   |
   +--> Secret grafana-telegram
   |
   +--> render contact-points.yaml
   +--> render notification-policies.yaml
   +--> render node-alerts.yaml
   +--> render kubernetes-alerts.yaml
   +--> render platform-alerts.yaml
   |
   v
ConfigMap grafana-alerting
   |
   v
Grafana provisioning mount
/etc/grafana/provisioning/alerting
   |
   v
Grafana database / alerting runtime
```

## Rule organization

```text
Infrastructure
├── Node alerts (4)
│   ├── CPU
│   ├── Memory
│   ├── Filesystem
│   └── WireGuard interface
│
├── Kubernetes alerts (3)
│   ├── Node not ready
│   ├── CrashLoopBackOff
│   └── Cilium unavailable
│
└── Platform alerts (9)
    ├── Prometheus
    ├── Grafana
    ├── GitLab Webservice
    ├── Harbor Core
    ├── Harbor Registry
    ├── Harbor Jobservice
    ├── Argo CD Repo Server
    ├── Argo CD Server
    └── Argo CD Application Controller
```

## Grafana persistence architecture

```text
Grafana container
      |
      v
/var/lib/grafana
      |
      v
PVC monitoring-grafana
      |
      v
local-path storage
```

Alert provisioning files are configuration inputs and do not replace Grafana's persistent database. Both are required: ConfigMap for declarative provisioning, PVC for runtime/application state.

## Credential isolation

```text
Bot token
   |
   v
Kubernetes Secret
   |
   v
Grafana environment

Chat IDs
   |
   v
Contact point provisioning
   |
   v
Grafana alerting configuration
```

The bot token is protected as a secret; chat IDs are configuration metadata.

## State transitions

```text
Rule query
   |
   +--> data available + condition false --> Normal
   |
   +--> data available + condition true  --> Pending/Firing
   |
   +--> no data                          --> depends on noDataState
```

The custom rules currently use explicit `noDataState` behavior. This makes an incorrect selector operationally significant, as demonstrated by the previous Prometheus selector problem.

## Failure domains

1. Prometheus datasource failure.
2. Query/selector error.
3. Grafana rule provisioning failure.
4. Notification policy mismatch.
5. Contact point configuration failure.
6. Telegram API/network failure.
7. Grafana persistence failure.

Troubleshoot in that order from evaluation to delivery instead of treating every Telegram alerting problem as a bot problem.
