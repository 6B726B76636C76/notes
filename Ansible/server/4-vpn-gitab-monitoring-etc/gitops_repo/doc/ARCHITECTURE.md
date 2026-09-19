# GitOps Repository Architecture

## Repository-to-cluster flow

```text
                         GitLab
                           |
                    gitops.git / main
                           |
                           v
                 +---------------------+
                 | Argo CD repo-server  |
                 +----------+----------+
                            |
                            v
                  gitops-root Application
                            |
                            v
                    root/applications.yaml
                            |
                            v
                child Applications / resources
                            |
                            v
                    Kubernetes API server
                            |
                            v
                       live workloads
```

## Root directory model

```text
root/
  |
  +--> applications.yaml
  |      |
  |      +--> declares child Applications
  |
  +--> kustomization.yaml
         |
         +--> builds the root resource set
```

The root is intentionally small. Its job is to establish the GitOps control tree rather than embed every workload manifest in a single file.

## Desired-state model

```text
Git
 |
 | desired state
 v
Argo CD comparison
 |
 +--> equal ----------------> Synced
 |
 +--> different ------------> OutOfSync
                                 |
                                 v
                          automated sync
                                 |
                                 v
                         Kubernetes live state
                                 |
                                 +--> self-heal
```

## Control-plane dependency chain

```text
GitLab availability
       |
       v
Git HTTP / Workhorse
       |
       v
Argo repo-server
       |
       v
Application Controller
       |
       v
Kubernetes API
       |
       v
Workloads
```

A repository failure prevents reconciliation but does not automatically imply that already-running workloads are broken. Conversely, a successful repository fetch does not guarantee workload health.

## Ownership boundary

```text
Ansible
  |
  +--> installs/configures Argo CD
  +--> bootstrap credentials
  +--> bootstrap root Application

GitOps repository
  |
  +--> steady-state application declarations

Argo CD
  |
  +--> reconciliation
```

The intended steady state avoids duplicate ownership of the same resource between Ansible and GitOps.

## Security boundary

Repository content is not a secret store. Sensitive data should arrive through Kubernetes Secrets, external secret systems, or other protected mechanisms rather than plaintext Git.
