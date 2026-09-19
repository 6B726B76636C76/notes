# Argo CD

## Purpose

This directory installs and configures Argo CD as the GitOps controller for the Kubernetes cluster. Argo CD watches the dedicated GitOps repository and reconciles the declared Kubernetes application state.

Current deployment:

```text
Namespace: argocd
Release: argocd
Helm chart: argo-cd 10.9.2
Application version: 3.5.3
Domain: argocd.domain.com
Ingress class: cilium
```

## Responsibilities

The module is responsible for:

- adding the Argo Helm repository;
- installing/upgrading the Argo CD Helm release;
- configuring ingress and TLS;
- enabling/configuring the local `vaclav` administration role;
- retaining the built-in `admin` account as an emergency fallback where configured;
- creating the Secret used for GitLab repository credentials;
- bootstrapping the `gitops-root` Application;
- pointing Argo CD at the GitLab GitOps repository through the internal Workhorse endpoint;
- configuring the desired revision and root path;
- enabling automated sync/prune/self-heal for the root Application.

## Current Git repository endpoint

The working in-cluster URL is:

```text
http://gitlab-webservice-default.vps.svc:8181/devops/gitops.git
```

The current branch/revision is `main` and the root application path is `root`.

This internal endpoint is preferred over the external GitLab URL for cluster-to-cluster traffic.

## Important credential model

Argo CD repository credentials are stored in a Kubernetes Secret created by Ansible. The token itself comes from encrypted Ansible Vault data.

The credential prefix must match the URL used by the Application:

```text
repoURL:
http://gitlab-webservice-default.vps.svc:8181/devops/gitops.git

credential prefix:
http://gitlab-webservice-default.vps.svc:8181/devops/
```

## Bootstrap workflow

The correct order is:

```text
1. Install Argo CD
2. Configure GitLab repository credentials
3. Create gitops-root Application
4. Argo CD clones GitOps repo
5. Argo CD renders root
6. Argo CD applies child Applications/resources
7. Argo CD continuously self-heals
```

The GitOps repository contains the declarative root application manifest. Avoid maintaining the same long-lived Application independently in two places once bootstrap is stable; choose a single source of truth for the steady state.

## Prerequisites

Before applying:

1. GitLab is healthy.
2. The GitLab Workhorse endpoint is reachable from the Argo repo-server.
3. The GitOps repository exists.
4. Vault contains the Argo Git credential.
5. cert-manager can issue the Argo TLS certificate.
6. The `argocd` namespace and cluster ingress path are available.

## Apply

```bash
ansible-playbook \
  4-vpn-gitab-monitoring-etc/6-argocd/argocd.yml \
  --ask-vault-pass
```

## Validation

```bash
kubectl get pods -n argocd
kubectl get applications -n argocd
kubectl get application gitops-root -n argocd
```

Expected steady state for `gitops-root`:

```text
SYNC STATUS   Synced
HEALTH STATUS Healthy
```

Check repository credentials without printing the token:

```bash
kubectl get secret gitlab-repo-creds -n argocd
```

## Git connectivity troubleshooting

If the repo-server cannot clone:

```bash
kubectl exec -n argocd deploy/argocd-repo-server -- \
  git ls-remote \
  "http://gitlab-webservice-default.vps.svc:8181/devops/gitops.git"
```

Authorization errors may be expected when credentials are not attached to the test command. The important distinction is whether DNS/network connectivity and the correct Workhorse endpoint exist.

### `connection refused` to the external `8443` address

Argo CD should normally use the internal service URL. If it tries `91.219.62.186:8443`, review the Application `repoURL`.

### `Nil JSON web token`

This previously occurred when Git HTTP was sent to GitLab Rails on `8080` instead of Workhorse on `8181`. Use the Workhorse service endpoint.

### Application stuck because repo URL is stale

Check:

```bash
kubectl get application gitops-root -n argocd -o yaml
```

The current expected repository URL is the internal `8181` endpoint. Prefer fixing the source-of-truth manifest rather than repeatedly patching the live Application.

## GitOps repository structure

Current root repository structure is centered on:

```text
gitops.git
└── root/
    ├── applications.yaml
    └── kustomization.yaml
```

The root Application points at `root/`, and child Applications/resources are declared from there.

## Security

- Keep Argo Git tokens in Vault/Kubernetes Secrets.
- Do not commit repository passwords.
- Do not expose the Argo server unnecessarily to the public Internet.
- Prefer internal GitLab service connectivity for repository access.
- Treat the root Application as a critical control-plane object.

## Suggested commit

```text
feat(argocd): document GitOps bootstrap and GitLab integration
```
