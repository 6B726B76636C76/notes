# Harbor Architecture

## Component topology

```text
                         registry.domain.com
                                  |
                                  v
                         Cilium Ingress / TLS
                                  |
                                  v
                       +---------------------+
                       |    Harbor Portal    |
                       +----------+----------+
                                  |
                                  v
                       +---------------------+
                       |    Harbor Core      |
                       +----+-----------+----+
                            |           |
                         Registry    Jobservice
                            |           |
                            |        Redis
                            |
                            v
                    Registry storage PVC

Core dependencies
  |
  +--> PostgreSQL / database PVC
  +--> Redis PVC

Harbor scanning
  |
  +--> Trivy
  +--> Trivy cache/data PVC
```

## CI data flow

```text
GitLab CI job
    |
    | docker login
    v
Harbor ingress
    |
    v
Harbor Core / Registry
    |
    v
OCI image layers
    |
    v
Registry persistent storage
    |
    +--> Trivy scan path
```

## State boundaries

### Stateless-ish services

Portal/Core/Jobservice can be recreated by Kubernetes, subject to their dependency state.

### Stateful data

Registry contents, database state, Redis state where persistence is configured, and Trivy data require persistent storage appropriate to the component.

## Authentication model

```text
Vault
  |
  +--> Harbor admin password
  +--> Harbor core secret

GitLab CI secret store
  |
  +--> Harbor robot account credentials
```

Separate those two secret domains. Harbor installation secrets should not be embedded in the GitLab application repository, and CI robot credentials should not be committed to Ansible variables.

## Failure domains

- Ingress/TLS failure: URL unavailable before Harbor Core.
- Core failure: portal/auth/API operations fail.
- Registry failure: login may work while pushes/pulls fail.
- Storage failure: registry operations become unreliable or fail.
- Database failure: Harbor control plane loses persistent metadata.
- Redis failure: queue/session/cache behavior may degrade.
- Trivy failure: scanning features degrade while the registry may remain available.
