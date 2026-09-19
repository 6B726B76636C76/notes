# GitOps Repository

## Purpose

This directory represents the dedicated Git repository consumed by Argo CD. It is separate from the Ansible repository and contains the declarative Kubernetes application state that Argo CD reconciles into the cluster.

The current Argo CD setup reads:

```text
http://gitlab-webservice-default.vps.svc:8181/devops/gitops.git
```

from revision `main`, with the root path:

```text
root
```

## Repository responsibility

The GitOps repository is the source of truth for application declarations after Argo CD bootstrap.

It should contain:

- the root Argo CD Application definition;
- child Application definitions;
- Kustomize bases/overlays where used;
- application manifests that are intended to be continuously reconciled;
- environment-specific changes tracked through Git history.

It should not contain:

- Ansible Vault secrets;
- Kubernetes credentials committed as plaintext;
- generated runtime state;
- temporary rendered Helm values from the Ansible repository.

## Current root structure

```text
root/
├── applications.yaml
└── kustomization.yaml
```

The root Application points Argo CD at this directory and uses automated sync, pruning, and self-healing.

## Current root repository URL

The GitLab Workhorse service is intentionally used for in-cluster Git HTTP:

```text
GitLab webservice service
        |
        +--> port 8181 / Workhorse
                |
                v
            Git HTTP
```

Do not replace it with the Rails service on `8080` for this Argo CD repository path.

## Bootstrap sequence

1. GitLab is available.
2. The Git repository exists.
3. Argo CD repository credentials are created by Ansible.
4. `gitops-root` is created.
5. Argo CD fetches `main`.
6. Argo CD reads `root/`.
7. Root declares child applications.
8. Argo CD reconciles them.

## Working with the repository

Normal change flow:

```text
developer
   |
   v
Git change
   |
   v
git commit
   |
   v
git push origin main
   |
   v
Argo CD detects revision
   |
   v
sync / self-heal
```

For a project using pull requests, the same flow can be wrapped in branch protection and review gates.

## Validation

From the repository itself:

```bash
git status
git log --oneline -10
```

After pushing a change, check:

```bash
kubectl get applications -n argocd
```

## Troubleshooting

### Argo CD does not see a commit

Check:

- the repository branch is `main`;
- the commit was pushed to the expected GitLab repository;
- the root Application points at the expected repo URL and path;
- Argo CD can authenticate to GitLab.

### Application is OutOfSync

Inspect the diff in Argo CD. Determine whether the desired state in Git is intentional before manually changing the cluster.

### Self-heal keeps reverting manual changes

That is expected GitOps behavior. Make the intended change in Git instead of changing the live object manually.

### Argo CD cannot read the repository

Verify the in-cluster Workhorse endpoint and repository credentials. A `connection refused` error points to endpoint/network problems; an authorization error points to the credential path.

## Security rules

- Never commit Vault secrets or CI credentials.
- Avoid embedding registry passwords in manifests.
- Use Kubernetes Secret references and external secret mechanisms where appropriate.
- Treat `root/applications.yaml` as a high-impact control-plane file.

## Suggested commit

```text
feat(gitops): document root application structure and reconciliation flow
```
