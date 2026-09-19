# Group Variables

## Purpose

This directory is the global configuration layer shared by the service modules under `4-vpn-gitab-monitoring-etc`.

Current files:

```text
group_vars/
├── all.yml
└── secrets.yml
```

## `all.yml`

`all.yml` contains non-secret configuration such as:

- domains and public/private addresses;
- WireGuard CIDRs and interface settings;
- firewall ports and access tiers;
- Kubernetes namespaces and pod CIDR;
- storage classes and sizes;
- Helm repositories/chart versions;
- GitLab/Argo CD/Harbor/Grafana names and endpoints;
- resource requests/limits;
- monitoring settings;
- GitLab Runner definitions that are safe to store as configuration;
- Grafana alerting folder/interval/configuration paths.

Examples from the current platform include:

```text
GitLab SSH port      2424
Argo CD domain       argocd.domain.com
Harbor domain        registry.domain.com
Grafana domain       grafana.domain.com
WG server address    172.16.0.1/27
Pod CIDR             10.244.0.0/16
```

## `secrets.yml`

`secrets.yml` contains protected values and must remain encrypted with Ansible Vault.

Typical secret classes in the current stack include:

- GitLab passwords/tokens;
- GitLab Runner tokens;
- Cloudflare API credentials;
- Argo CD Git credentials;
- PostgreSQL credentials;
- Valkey credentials;
- Harbor admin/core secrets;
- WireGuard private keys;
- Grafana Telegram bot token.

Never replace `secrets.yml` with a plaintext file in Git.

## Editing secrets

Use:

```bash
ansible-vault edit \
  4-vpn-gitab-monitoring-etc/group_vars/secrets.yml
```

View only when required:

```bash
ansible-vault view \
  4-vpn-gitab-monitoring-etc/group_vars/secrets.yml
```

Run playbooks with:

```bash
--ask-vault-pass
```

or with your established secure Vault password mechanism.

## Variable naming convention

Non-secret infrastructure variables use service-specific prefixes where practical:

```text
argocd_*
harbor_*
gitlab_*
grafana_*
runner_*
wg_*
```

Secret variables are similarly namespaced but use the `vault_` prefix:

```text
vault_*
```

This makes it obvious which values must not be logged or committed outside Vault encryption.

## Dependency management

When adding a new variable:

1. decide whether it is actually secret;
2. put it in `all.yml` only if it is non-secret;
3. put the protected value in `secrets.yml` if secret;
4. reference the variable from tasks/templates;
5. add validation/assertions where the variable is mandatory;
6. document the variable in the relevant module README.

## Validation

Basic Ansible syntax:

```bash
ansible-playbook <playbook> --syntax-check
```

For Vault variables:

```bash
ansible-vault view 4-vpn-gitab-monitoring-etc/group_vars/secrets.yml
```

Do not copy the decrypted contents into another file just to inspect them.

## Security rules

- Never commit a decrypted `secrets.yml`.
- Never paste secret values into README files.
- Never print secrets with debug tasks.
- Use Kubernetes Secret resources for runtime credentials.
- Treat Vault variable names and public identifiers as configuration; treat values such as passwords/tokens/private keys as sensitive.

## Suggested commit

```text
refactor(group-vars): document shared configuration and Vault boundaries
```
