# `Ansible/server` — Project Architecture

Repository: `notes/Ansible/server` — a set of Ansible playbooks covering the
full lifecycle of a single VPS host: from the very first root login to a
production stack of Kubernetes + GitLab + Argo CD + Harbor + monitoring,
reachable only over VPN.

## 1. Overview

Everything is organized around four sequential, idempotent stages. Each
next stage assumes the previous one has already been applied. The source
of truth is Ansible variables (`group_vars`, `host_vars`) and Jinja
templates — manual changes on the server do not survive the next run.

```text
1-ssh-access   → base access (users, keys, sudo+TOTP)
2-security     → OS hardening (UFW, CrowdSec, SSH, auditd)
3-basic-apps   → Kubernetes: kubeadm + Cilium + MetalLB + cert-manager + external-dns
4-vpn-gitab-monitoring-etc → the service layer on top of the cluster
```

Stage 4 has its own internal order, since each module depends on the
network/platform layer built by the previous one:

```text
1-wireguard   → private VPN network (transport)
2-firewall    → UFW policy on top of WireGuard / public ports
3-private-dns → private DNS for internal names (VPN-only)
4-monitoring  → Prometheus + Grafana (kube-prometheus-stack)
5-gitlab      → self-hosted GitLab (Helm, external Postgres/Valkey/S3)
6-argocd      → Argo CD, GitOps controller
7-harbor      → private container registry for CI
8-grafana-alerting → Grafana Managed Alerting → Telegram
gitops_repo/  → template of the separate git repo that Argo CD reads
```

## 2. Network topology

```text
                                Internet
                                   |
                    +--------------+--------------+
                    |                              |
                 80/443                       UDP 51820
                    |                              |
             Cilium (public IP)               WireGuard (wg0)
                    |                       172.16.0.1/27
                    |                              |
                    |           +------------------+------------------+
                    |           |                  |                  |
                    |        admin           developers/limited   http-only
                    |     172.16.0.0/27       172.16.1.0/24       172.16.8.0/21
                    |           |                  |                  |
                    +-----------+------------------+------------------+
                                |
                              UFW
                                |
              +------------------+-------------------+
              |                 |                     |
          Host SSH        Private DNS :53      Kubernetes host paths
              |                 |               (kubelet/metrics)
              +------------------+-------------------+
                                |
                        Kubernetes / Cilium
                                |
             +-----------+-----------+-----------+-----------+
             |           |           |           |           |
          GitLab      Argo CD      Harbor    Prometheus/   (dev/staging/
        (vps ns)    (argocd ns)  (harbor ns)  Grafana     prod namespaces)
                                            (monitoring ns)
```

Three VPN trust tiers (defined in `group_vars/all.yml`, must only be
changed together with the firewall):

| Tier | CIDR | Purpose |
|---|---|---|
| Admin | `172.16.0.0/27` | full administrative access |
| Developers/Limited | `172.16.1.0/24` | restricted service access |
| HTTP-only | `172.16.8.0/21` | web traffic only |

The only genuinely public entry points are `80/443` (Cilium Ingress) and
`UDP 51820` (WireGuard). Everything internal (GitLab, Argo CD, Harbor,
Grafana) is resolved via private DNS and reachable only from the VPN.

## 3. Stage 1 — `1-ssh-access`

Bootstrap playbook for a bare server (`bootstrap.yml`):

- generates ed25519 keys on the controller for each user
  (`~/.ssh/bootstrap-vpc/<user>/key`);
- creates `service_users` (`ansible` — passwordless sudo, for automation)
  and `human_users` (`vaclav` — sudo via TOTP/Google Authenticator, no
  password at all);
- pushes public keys into `authorized_keys`.

The first login happens as `root`/password from `host_vars/<host>.yml`
(stored encrypted with Ansible Vault: `mkpasswd` → `ansible-vault
encrypt_string`). After the first run, access is key-only + a TOTP code
for sudo.

## 4. Stage 2 — `2-security`

OS hardening (`playbook.yml.example`), no Kubernetes specifics:

- **UFW**: default deny incoming / allow outgoing / forward=ACCEPT
  (required for the CNI); rules per VPN tier (admin — full, limited —
  selected ports, http-only — 80/8443 only); rate-limit + connlimit on
  the public `80/443`;
- **SSH hardening**: `PermitRootLogin no`, `PasswordAuthentication no`,
  `KbdInteractiveAuthentication no`;
- **CrowdSec** + iptables bouncer with the `sshd`, `linux` collections;
- **unattended-upgrades**: security updates only, no auto-reboot;
- **auditd**: watches `/etc/passwd`, `/etc/shadow`, `sudoers`,
  `sshd_config`, privileged execve, etc.

## 5. Stage 3 — `3-basic-apps` (Kubernetes)

A single-node kubeadm cluster (`playbook.yml`, detailed in `_README.md`).
Installs:

```text
Docker CE + containerd (SystemdCgroup=true)
kubeadm/kubelet/kubectl 1.31 (apt-mark hold)
     |
kubeadm init (--skip-phases=addon/kube-proxy, control-plane untainted)
     |
Helm 3
     |
Cilium 1.19.8   — CNI, full kube-proxy replacement, built-in Ingress
MetalLB 0.14.9  — L2 LoadBalancer (no FRR)
cert-manager    — ClusterIssuer letsencrypt-cloudflare (DNS-01)
external-dns    — automatic DNS records in Cloudflare
```

Notable details:
- swap is not disabled but supported by kubelet (`failSwapOn: false`,
  `LimitedSwap`) — used on a single-node cluster;
- Cilium Ingress is deliberately restricted via
  `loadBalancerSourceRanges` to the VPN subnets only
  (admin/limited/http-only) — it is not reachable from the public
  internet directly;
- control-plane metrics (`kube-controller-manager`, `kube-scheduler`,
  `etcd`) are bound to `0.0.0.0` for Prometheus scraping.

## 6. Stage 4 — the service layer

### 6.1 WireGuard (`1-wireguard`)

```text
group_vars/all.yml (wg_peers, CIDRs) + secrets.yml (private keys)
        → wg0.conf.j2 → /etc/wireguard/wg0.conf → wg-quick@wg0
```

Server: `172.16.0.1/27` on `wg0`. The module is responsible for the VPN
layer only; access control is the job of the firewall and (for the
cluster) Cilium. Monitoring quirk: the `wg0` interface reports
`operstate=unknown`, so the health check is built on
`node_network_info{adminstate="up"}` rather than `node_network_up`.

### 6.2 Firewall (`2-firewall`)

UFW policy driven by the WireGuard tiers (port table from
`group_vars/all.yml`):

| Port | Protocol | Role |
|---:|---|---|
| 22 | TCP | SSH (admin fallback + VPN) |
| 2424 | TCP | GitLab SSH |
| 53 | TCP/UDP | private DNS |
| 80/443 | TCP | public HTTP/HTTPS ingress |
| 51820 | UDP | WireGuard |
| 6443 | TCP | Kubernetes API |
| 9100/10250/10257/10259/2381 | TCP | node_exporter / control-plane metrics for Prometheus |

The firewall is not a replacement for Cilium NetworkPolicy — it's a
separate layer below it.

### 6.3 Private DNS (`3-private-dns`)

Resolver on `172.16.0.1:53`, reachable only over WireGuard. Serves
internal names (`gitlab.domain.com`, `grafana.domain.com`,
`argocd.domain.com`, `registry.domain.com`) pointing at the
private/ingress address; never published externally.

### 6.4 Monitoring (`4-monitoring`)

`kube-prometheus-stack` Helm release (ns `monitoring`):
- Prometheus (3d retention, ~300Mi request / 900Mi limit) +
  kube-state-metrics;
- bundled Alertmanager **disabled** — replaced by Grafana Managed
  Alerting (see 6.8);
- the chart's nodeExporter **disabled** — a separate host
  `node_exporter:9100` is used instead, added as a static scrape
  target;
- Grafana with a persistent PVC `monitoring-grafana` (`local-path`,
  2Gi) — without it, users/settings get reset on restart.

### 6.5 GitLab (`5-gitlab`)

Self-hosted GitLab, Helm chart 10.3.2 / app 19.3.2, ns `vps`,
`https://gitlab.domain.com:8443`, SSH on `2424/tcp`.

```text
GitLab (webservice, Sidekiq, Gitaly, Shell, Toolbox)
   ├── PostgreSQL — external (CloudNativePG)
   ├── Valkey     — external
   └── Object storage — Garage (S3-compatible)
```

The bundled GitLab Runner is disabled — runners are deployed separately
(Runner Manager Helm releases, see `group_vars/all.yml`: `runners`).

**Important detail**: internal Git HTTP (used by Argo CD) must go
through the GitLab Workhorse endpoint on `:8181`, not the Rails service
on `:8080` — otherwise it fails with `Nil JSON web token`.

### 6.6 Argo CD (`6-argocd`)

Ns `argocd`, chart `argo-cd` 10.9.2, domain `argocd.domain.com`.

GitOps flow:

```text
group_vars/secrets.yml (git token)
        → Kubernetes Secret (repo credentials)
        → Application "gitops-root"
        → repoURL: http://gitlab-webservice-default.vps.svc:8181/devops/gitops.git
          revision: main, path: root
        → Argo CD clones and renders root/
        → applications.yaml declares child Applications
        → sync + prune + self-heal (automated)
```

A local `vaclav` admin role is used; the built-in `admin` account stays
as an emergency fallback.

### 6.7 Harbor (`7-harbor`)

Private registry for CI, ns `harbor`, `registry.domain.com`, chart
1.19.2 / app 2.15.2. Storage: registry 10Gi, database 5Gi, Redis 1Gi,
Trivy (scanning) 5Gi. The core secret must use the key `secretKey` (not
`secret`).

```text
GitLab CI → docker push → registry.domain.com → Harbor Core
                                                    ├── Registry (storage)
                                                    └── Trivy (scanning)
```

### 6.8 Grafana Alerting (`8-grafana-alerting`)

Not a separate stack — a provisioning layer on top of the existing
Grafana release (`--reuse-values`):

```text
Prometheus → Grafana Managed Alerting → Telegram Bot API
```

16 rules (4 node + 3 kubernetes + 9 platform); the bot token lives only
in Vault → Kubernetes Secret `grafana-telegram`, chat IDs go into the
ConfigMap `grafana-alerting`. The bundled Alertmanager stays disabled
(see 6.4).

### 6.9 GitOps repository (`gitops_repo/`)

Not an Ansible module — a template of a **separate** git repository
(`devops/gitops.git`) that Argo CD reads:

```text
root/
├── applications.yaml   # child Applications
└── kustomization.yaml
```

It must not contain Vault secrets or Helm values from the Ansible repo
— only the declarative application state.

## 7. Variables and secrets model

```text
group_vars/all.yml      — non-secret configuration (domains, CIDRs, ports,
                           namespaces, chart versions, resource limits)
group_vars/secrets.yml  — Ansible Vault: GitLab passwords/tokens, Cloudflare
                           API, Argo CD Git credentials, Postgres/Valkey,
                           Harbor admin/core secret, WireGuard private keys,
                           Telegram bot token
host_vars/*.yml.example — host-specific secrets (initial root password, etc.)
```

Naming convention: non-secret variables use a service prefix (`argocd_*`,
`harbor_*`, `gitlab_*`, `wg_*`); secret ones use `vault_*`. Playbooks are
run with `--ask-vault-pass` / `--vault-password-file`.

## 8. Bootstrap order (from scratch)

```bash
ansible-playbook 1-ssh-access/bootstrap.yml
ansible-playbook 2-security/playbook.yml.example
ansible-playbook 3-basic-apps/playbook.yml -e @secrets/k8s.yml

ansible-playbook 4-vpn-gitab-monitoring-etc/1-wireguard/wireguard.yml --ask-vault-pass
ansible-playbook 4-vpn-gitab-monitoring-etc/2-firewall/firewall.yml --ask-vault-pass
ansible-playbook 4-vpn-gitab-monitoring-etc/3-private-dns/private-dns.yml --ask-vault-pass
ansible-playbook 4-vpn-gitab-monitoring-etc/4-monitoring/deploy-monitoring.yml --ask-vault-pass
ansible-playbook 4-vpn-gitab-monitoring-etc/5-gitlab/gitlab.yml --ask-vault-pass
ansible-playbook 4-vpn-gitab-monitoring-etc/6-argocd/argocd.yml --ask-vault-pass
ansible-playbook 4-vpn-gitab-monitoring-etc/7-harbor/harbor.yml --ask-vault-pass
ansible-playbook 4-vpn-gitab-monitoring-etc/8-grafana-alerting/alerting.yml --ask-vault-pass
```

WireGuard and the firewall come before any applications — an
independent SSH path is kept until the VPN is fully verified. GitLab
comes before Argo CD (the repository must already exist). Harbor and
monitoring aren't order-critical, but it's convenient to install them
after GitLab/Argo CD.

## 9. Ownership boundaries

```text
WireGuard   → identity/transport (who you are)
Firewall    → host-level network access control (what that IP may reach)
Cilium      → in-cluster access control (NetworkPolicy, Ingress source ranges)
Private DNS → naming/addressing only, not a security mechanism
```

Changing any of: WireGuard CIDRs, the DNS resolver address, the GitLab
SSH port, Cilium Ingress NodePorts, metrics ports, the pod CIDR —
requires a matching change in the firewall module.

## 10. Known gotchas documented in the repo

- `node_network_up{device="wg0"}` is always `0` on this host — use
  `node_network_info{adminstate="up"}` for liveness instead.
- Argo CD's Git HTTP must go through GitLab Workhorse `:8181`, not
  Rails `:8080` (otherwise `Nil JSON web token`).
- Harbor core secret — the key must be named `secretKey`.
- Grafana must have a PVC (`monitoring-grafana`) before any restarts —
  otherwise users/alerts get lost.
- The Prometheus StatefulSet is named
  `prometheus-monitoring-kube-prometheus-prometheus` — a selector
  missing the `prometheus-` prefix returns `NoData` even though
  Prometheus is healthy.
