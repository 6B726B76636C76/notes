# Group Variables Architecture

## Global variable flow

```text
global inventory
      |
      v
host/group selection
      |
      v
group_vars/all.yml
      |
      +--------------------------+
      |                          |
      v                          v
service playbooks           Jinja templates
      |                          |
      +------------+-------------+
                   |
                   v
          target host / cluster
```

## Secret flow

```text
Encrypted secrets.yml
          |
          v
      Ansible Vault
          |
          v
    vault_* variables
          |
    +-----+------------------+
    |                        |
    v                        v
Kubernetes Secret       protected Helm/task input
    |                        |
    v                        v
runtime secret           service runtime
```

## Configuration categories

```text
all.yml
  |
  +--> network topology
  +--> domains/ports
  +--> namespaces
  +--> chart versions
  +--> storage/resource targets
  +--> feature flags
  +--> public identifiers

secrets.yml
  |
  +--> passwords
  +--> tokens
  +--> private keys
  +--> API credentials
```

## Why the separation matters

A global configuration file is consumed by many modules. Accidentally placing a secret in `all.yml` means every task/template that can read the variable now has access to a protected value and a future Git diff can expose it. The Vault boundary limits that risk.

## Change propagation

```text
change in all.yml
      |
      v
next relevant playbook run
      |
      v
template/task render
      |
      v
service reconciliation
```

There is no hidden dynamic synchronization. A variable change becomes operational only when the corresponding Ansible module is applied.

## Failure domains

- Wrong variable: service renders an incorrect configuration.
- Missing variable: Ansible fails with `undefined variable`.
- Wrong secret value: service starts but authentication fails.
- Vault unavailable/unreadable: protected variables cannot be loaded.
- Duplicate source of truth: a template or manual server edit conflicts with declared configuration.
