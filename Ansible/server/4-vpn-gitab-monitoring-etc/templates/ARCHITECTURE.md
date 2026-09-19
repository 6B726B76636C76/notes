# Templates Architecture

## Configuration generation

```text
                   group_vars
                 /            \
                /              \
           all.yml          secrets.yml
              |                  |
              +--------+---------+
                       |
                       v
                  Jinja2 engine
                       |
        +--------------+---------------+
        |              |               |
        v              v               v
     Helm values   host config     Grafana provisioning
        |              |               |
        v              v               v
      Helm          systemd          ConfigMap
        |                              |
        v                              v
   Kubernetes                  Grafana runtime
```

## Ownership model

Templates own **representation**. Tasks own **execution**.

```text
Template:
  What should the generated configuration look like?

Ansible task:
  Where should it be written?
  When should it be applied?
  How should success/failure be checked?
```

This separation reduces duplicated orchestration logic and keeps service directories small.

## Secret boundary

```text
Vault secret
   |
   v
Ansible variable
   |
   v
Template or task
   |
   +--> Kubernetes Secret / protected environment
```

The rendered artifact must not accidentally place a secret into a public ConfigMap, Git repository, or debug output.

## Grafana alerting template topology

```text
templates/grafana/alerting/
   |
   +--> contact-points.yaml.j2
   +--> notification-policies.yaml.j2
   +--> node-alerts.yaml.j2
   +--> kubernetes-alerts.yaml.j2
   +--> platform-alerts.yaml.j2
   +--> values.yaml.j2
```

The first five are mounted into Grafana provisioning. `values.yaml.j2` modifies the Helm deployment so the provisioning directory and Telegram Secret are available to Grafana.
