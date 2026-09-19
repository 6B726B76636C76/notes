# GitLab Architecture

## Deployment topology

```text
                         Internet / VPN
                              |
                 +------------+-------------+
                 |                          |
              HTTPS 8443                SSH 2424
                 |                          |
                 v                          v
          GitLab web ingress          GitLab Shell
                 |                          |
                 +------------+-------------+
                              |
                              v
                         GitLab platform
                              |
        +---------------------+----------------------+
        |                     |                      |
    Webservice             Sidekiq                Gitaly
        |                     |                      |
        +---------------------+----------------------+
                              |
              +---------------+---------------+
              |               |               |
          PostgreSQL        Valkey          Garage
             (CNPG)          cache         object store
```

## Kubernetes boundary

GitLab is deployed in namespace `vps`. Supporting databases/caches may live in the same or separate namespaces depending on the module responsible for them. The GitLab Helm values reference those services rather than assuming the bundled database/cache components are enabled.

## Git operation paths

### SSH

```text
Developer
  -> gitlab.domain.com:2424
  -> GitLab Shell
  -> repository storage / Gitaly
```

### Internal Git HTTP used by Argo CD

```text
Argo CD repo-server
  -> gitlab-webservice-default.vps.svc:8181
  -> GitLab Workhorse
  -> GitLab application
  -> repository
```

The `8181` path is important because Workhorse participates in GitLab's HTTP authorization/request handling.

## Data dependencies

```text
GitLab Webservice
      |
      +--> PostgreSQL  <--- persistent application metadata
      |
      +--> Valkey      <--- cache/session/background coordination
      |
      +--> Garage      <--- object storage
      |
      +--> Gitaly      <--- Git repository storage
```

The failure of one external dependency can present as a GitLab application failure even when the webservice pod itself is running.

## Storage model

Git repository data is stored through Gitaly. Application/object data uses the configured object storage. PostgreSQL and Valkey are treated as external services from the GitLab Helm chart's perspective.

## CI relationship

```text
GitLab
  |
  v
Kubernetes Runner Managers
  |
  v
temporary CI job pods
  |
  v
build/test/deploy
```

The bundled Runner is disabled; Runner Managers are deployed separately and are documented in their own configuration.
