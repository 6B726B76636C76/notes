# Argo CD Architecture

## GitOps topology

```text
                 +----------------------+
                 | GitLab Git repository|
                 | devops/gitops.git    |
                 +----------+-----------+
                            |
                     HTTP Git / 8181
                            |
                            v
                 +----------------------+
                 | Argo CD repo-server   |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | gitops-root           |
                 | Application           |
                 +----------+-----------+
                            |
                    root/applications.yaml
                            |
                            v
                 +----------------------+
                 | Argo application      |
                 | controller            |
                 +----------+-----------+
                            |
                       Kubernetes API
                            |
                            v
                 +----------------------+
                 | Cluster workloads     |
                 +----------------------+
```

## Ingress topology

```text
Client
  |
  v
argocd.domain.com
  |
  v
Cilium Ingress / LoadBalancer
  |
  v
Argo CD server
```

TLS is terminated through the configured ingress/certificate path.

## Repository authentication

```text
Vault
  |
  v
Ansible variable
  |
  v
Kubernetes Secret gitlab-repo-creds
  |
  v
Argo CD repo-server
  |
  v
GitLab Workhorse :8181
```

The credential prefix is scoped to the GitLab repository path so that Argo CD can match credentials to the Application repository URL.

## Reconciliation model

```text
Git desired state
       |
       v
Argo CD compares desired vs live
       |
       +--> Synced / Healthy
       |
       +--> OutOfSync
                |
                v
        automated sync
                |
                v
        Kubernetes API
                |
                v
             live state
                |
                +------> self-heal
```

## Bootstrap vs steady state

There are two distinct phases:

### Bootstrap

Ansible installs Argo CD and creates the first Application/credential objects so that Argo CD can begin reading Git.

### Steady state

Git becomes the authoritative source for application declarations. Argo CD continuously reconciles that Git state.

The goal is to avoid a permanent dual-management situation where Ansible and GitOps continually overwrite the same Application object.

## Failure domains

- GitLab unavailable: repo-server cannot fetch desired state.
- Git credential invalid: repository fetch returns authentication errors.
- Repo URL wrong: connection goes to the wrong port/service.
- GitOps YAML invalid: repository fetch works but sync fails during rendering/application.
- Kubernetes API unavailable: Argo CD cannot apply state.
- Application unhealthy: sync may succeed while the workload remains unhealthy.

This separation is useful during incident response because repository, controller, API, and workload problems look different in Argo CD.
