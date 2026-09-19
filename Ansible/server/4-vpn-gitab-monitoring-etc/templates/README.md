# Templates

## Purpose

This directory is the shared Jinja2 template library for the infrastructure playbooks. It keeps rendered Helm values and other generated configuration in one predictable location instead of embedding templates inside each service module.

The design is:

```text
templates/
├── argocd/
├── gitlab/
├── harbor/
└── grafana/
    └── alerting/
```

The exact template inventory may grow as modules are added, but the ownership rule remains the same: templates are shared configuration inputs; service playbooks contain orchestration/tasks.

## Responsibilities

Templates convert Ansible variables into:

- Helm values files;
- WireGuard configuration;
- Grafana alerting provisioning files;
- other service configuration consumed by the corresponding playbook.

They should not contain operational logic that belongs in Ansible tasks.

## Rendering flow

```text
group_vars/all.yml
        +
group_vars/secrets.yml
        |
        v
Jinja2 template
        |
        v
Temporary rendered file
        |
        v
Helm / service task
        |
        v
Kubernetes / host state
```

Temporary values files should be removed after use when the playbook already implements cleanup.

## Important template groups

### Argo CD

Contains Helm values and related configuration for:

- ingress/domain/TLS;
- server behavior;
- repository integration settings.

### GitLab

Contains GitLab Helm values, including:

- external URL;
- SSH configuration;
- PostgreSQL/Valkey/object-storage references;
- resource settings;
- chart feature toggles.

### Harbor

Contains Harbor Helm values including:

- external URL;
- persistence sizes;
- replica counts;
- registry/core/portal/jobservice/Trivy settings;
- secret references.

### Grafana Alerting

Contains the provisioning files for:

```text
contact points
notification policies
node alerts
Kubernetes alerts
platform alerts
```

and the additional Helm values used to mount the provisioning ConfigMap and inject the Telegram secret.

## Change rules

- Do not hardcode real secrets.
- Prefer variables from `group_vars` over repeated literals.
- Keep Jinja logic simple and readable.
- Preserve valid YAML after rendering.
- Do not treat rendered server-side files as a second source of truth.
- When a template consumes a new variable, document and validate the variable in the appropriate `group_vars` file.

## Validation

At minimum:

```bash
ansible-playbook <playbook> --syntax-check
```

For service templates, use the module's `--check` path where safe and inspect the resulting task output. When a YAML template is especially sensitive, render it in a temporary file and validate the resulting YAML before applying it.

## Troubleshooting

### `undefined variable`

Add the missing non-secret variable to `group_vars/all.yml`, or the protected value to `group_vars/secrets.yml` if it is actually secret. Do not put secrets into `all.yml` merely to make templating convenient.

### Rendered YAML is invalid

Most common causes:

- Jinja conditional indentation;
- incorrect block scalar indentation;
- values that require YAML quoting;
- accidentally moving a field outside its parent object.

For Grafana alerting, pay special attention to nested `model`, `conditions`, and PromQL block indentation.

### A Helm value is ignored

Confirm the exact chart key for the installed chart version. Helm silently accepting a value does not guarantee the chart uses it.

## Suggested commit

```text
refactor(templates): document shared Jinja2 configuration ownership
```
