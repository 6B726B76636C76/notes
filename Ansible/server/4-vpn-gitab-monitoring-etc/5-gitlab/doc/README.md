# GitLab

## Purpose

This directory deploys and manages the self-hosted GitLab platform used as the source-code, CI, and Git transport system for the infrastructure.

The current GitLab release is deployed with the official GitLab Helm chart. The working environment uses the `vps` namespace, GitLab external URL `https://gitlab.domain.com:8443`, and GitLab SSH on port `2424`.

## Current platform model

```text
GitLab
├── Webservice
├── Sidekiq
├── Gitaly
├── Toolbox
├── GitLab Shell
├── PostgreSQL (external CNPG)
├── Valkey (external)
└── Object storage (Garage)
```

The bundled GitLab Runner is disabled because the project uses separate Kubernetes Runner Manager releases.

## Current versions and endpoints

```text
Helm chart: 10.3.2
GitLab application: 19.3.2
Namespace: vps
External URL: https://gitlab.domain.com:8443
SSH: 2424/tcp
```

The exact chart/application relationship is controlled by the variables and should be rechecked when upgrading the Helm chart.

## Responsibilities

The GitLab module is responsible for:

- adding the GitLab Helm repository;
- rendering GitLab Helm values from global variables and Vault data;
- configuring the external URL and ingress/TLS behavior;
- configuring the external PostgreSQL dependency;
- configuring external Valkey;
- configuring object storage for Rails/application data;
- configuring GitLab Shell SSH exposure on the custom port;
- sizing webservice, Sidekiq, Gitaly, and toolbox resources;
- disabling the bundled GitLab Runner;
- applying the release declaratively.

The module does not own the PostgreSQL cluster, Valkey installation, or Garage storage implementation when those are provisioned by separate modules.

## Prerequisites

Before running GitLab:

1. Kubernetes is healthy.
2. cert-manager and the `letsencrypt-cloudflare` ClusterIssuer are working.
3. PostgreSQL/CNPG is available.
4. Valkey is available.
5. Garage/object-storage credentials are available in the expected Kubernetes Secret.
6. Required DNS/Ingress paths are available.
7. Vault contains the required GitLab credentials and tokens.

## Apply

```bash
ansible-playbook \
  4-vpn-gitab-monitoring-etc/5-gitlab/gitlab.yml \
  --ask-vault-pass
```

## Validation

Check the release:

```bash
helm list -n vps
helm status gitlab -n vps
```

Check workloads:

```bash
kubectl get pods -n vps
kubectl get svc -n vps
kubectl get ingress -n vps
```

Check GitLab webservice:

```bash
kubectl get deploy -n vps | grep webservice
```

Check the public/private endpoint according to the active DNS path:

```bash
curl -k -I https://gitlab.domain.com:8443
```

Check SSH:

```bash
nc -vz <gitlab-host> 2424
```

## Important Git HTTP detail

Inside the cluster, GitLab's HTTP path that Argo CD uses terminates through GitLab Workhorse on port `8181`:

```text
Git client / Argo CD
        |
        v
GitLab Workhorse :8181
        |
        v
GitLab application services
```

Do not replace the internal Workhorse endpoint with the Rails service port when configuring Git HTTP clients that expect GitLab's normal web authorization flow. A previous failure used the wrong internal port and resulted in `Nil JSON web token` authorization errors.

## SSH model

External GitLab SSH uses:

```text
client
  |
  +--> gitlab.domain.com:2424
             |
             v
       GitLab Shell
```

Keep this port aligned across GitLab, UFW, DNS/ingress configuration, and client URLs.

## Resource model

Current target settings include:

```text
Webservice  request 1Gi   limit 1.5Gi
Sidekiq     request 500Mi limit 800Mi
Gitaly      request 300Mi limit 700Mi
Toolbox     request 200Mi limit 400Mi
```

These are starting resource targets, not guarantees. Monitor actual usage before scaling.

## Troubleshooting

### GitLab pods are healthy but the browser cannot connect

Trace:

```text
DNS → ingress → service → webservice
```

Check:

```bash
kubectl get ingress -n vps
kubectl get svc -n vps
kubectl get endpoints -n vps
```

### Argo CD cannot clone GitLab repository

Check the internal repository URL used by Argo CD. The currently working endpoint is the GitLab Workhorse service on port `8181`, not the external `8443` path and not the Rails service on `8080`.

### SSH clone fails

Check:

```bash
sudo ufw status numbered
nc -vz <gitlab-host> 2424
kubectl get svc -n vps | grep -i shell
```

Then verify the external port mapping in GitLab values.

### PostgreSQL problems

Do not debug only the GitLab pod. Verify the external database service and credentials used by the Helm release:

```bash
kubectl get pods -A | grep -i postgres
kubectl get svc -n vps | grep postgres
```

### Object storage errors

Check that the expected Kubernetes Secret exists and that the key configured by GitLab matches the secret key expected by the chart. Do not print secret contents into logs.

## Upgrade procedure

1. Review GitLab chart release notes.
2. Verify compatibility of PostgreSQL, Valkey, and object storage.
3. Update the version in `group_vars/all.yml`.
4. Run Ansible syntax-check.
5. Apply the GitLab playbook.
6. Watch rollout.
7. Verify web UI and SSH.
8. Verify Git operations.
9. Verify downstream Argo CD repository access.

## Suggested commit

```text
feat(gitlab): document Helm deployment and service dependencies
```
