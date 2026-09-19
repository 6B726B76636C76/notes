# Firewall

## Purpose

This directory manages the host firewall and routing policy around the infrastructure. The current design uses UFW with a deny-by-default inbound posture while explicitly permitting public ingress, WireGuard, administrative VPN traffic, selected monitoring endpoints, and Kubernetes-related traffic required by the monitoring stack.

The firewall is a security boundary, not a replacement for Kubernetes network policy. Host-level policy and in-cluster Cilium policy are intentionally separate layers.

## Responsibilities

The playbook is responsible for declaring and applying:

- UFW installation and enablement;
- default inbound/outbound/routed policies;
- public HTTP/HTTPS access;
- WireGuard UDP access;
- SSH and GitLab SSH access according to the configured trust tier;
- private DNS access from the VPN networks;
- monitoring/control-plane metric access required by Prometheus;
- forwarding/routing prerequisites;
- any connection limiting or explicit service restrictions defined by the shared variables;
- logging needed for troubleshooting denied packets.

Exact rules remain defined by the playbook and `group_vars/all.yml`. The README documents the current architecture and validation commands; the playbook is the source of truth for the exact rule list.

## Security model

```text
Internet
  ├── TCP 80/443        → public web ingress
  └── UDP 51820         → WireGuard

Admin VPN 172.16.0.0/27
  └── administrative access

Limited VPN 172.16.1.0/24
  └── restricted SSH/GitLab/service access

HTTP-only 172.16.8.0/21
  └── HTTP/HTTPS only

Kubernetes pod network 10.244.0.0/16
  └── required host metrics/API paths
```

The access tiers are deliberately distinct. Do not merge them into one broad allow rule without redesigning the threat model.

## Important ports in the current design

Commonly used endpoints include:

| Port | Protocol | Role |
|---:|---|---|
| 22 | TCP | Host SSH, administrative path where allowed |
| 2424 | TCP | GitLab SSH |
| 53 | TCP/UDP | Private DNS |
| 80 | TCP | Public HTTP ingress |
| 443 | TCP | Public HTTPS ingress |
| 51820 | UDP | WireGuard |
| 6443 | TCP | Kubernetes API |
| 9100 | TCP | Host node_exporter |
| 10250 | TCP | kubelet metrics/API path used by monitoring |
| 10257 | TCP | kube-controller-manager metrics |
| 10259 | TCP | kube-scheduler metrics |
| 2381 | TCP | etcd metrics |

The exact source ranges for each port must be read from the playbook/variables rather than inferred from the port number alone.

## Prerequisites

Before changing the firewall:

1. Confirm the server has a working SSH or console path.
2. Confirm WireGuard is already working when VPN-only management is part of the target policy.
3. Confirm the Kubernetes pod CIDR is correct (`10.244.0.0/16` in the current cluster).
4. Confirm all public ingress ports required by Cilium Ingress are explicitly allowed.
5. Confirm GitLab SSH uses the configured `2424/tcp` port.

## Apply

```bash
ansible-playbook \
  4-vpn-gitab-monitoring-etc/2-firewall/firewall.yml \
  --ask-vault-pass
```

Prefer running the firewall module after WireGuard and before application exposure so that the security boundary is established deliberately.

## Validation

```bash
sudo ufw status verbose
sudo ufw status numbered
sudo ufw show raw
```

Check forwarding:

```bash
sysctl net.ipv4.ip_forward
```

Check the routing table:

```bash
ip route
```

Inspect nftables underneath UFW when necessary:

```bash
sudo nft list ruleset
```

## Functional tests

From an approved VPN client:

```bash
nc -vz <server-ip> 2424
nc -vz 172.16.0.1 53
```

For public ingress from an appropriate external source:

```bash
nc -vz 91.219.62.186 443
```

Use `curl`/`dig` for application-specific validation rather than relying only on TCP connect tests.

## Troubleshooting

### Locked out after a firewall change

Use console or an existing management session. Do not repeatedly add broad temporary rules from memory.

First inspect:

```bash
sudo ufw status numbered
```

If possible, disable only the specific bad rule rather than resetting the entire firewall.

### VPN handshake works but service traffic is blocked

Trace:

```text
client
  ↓
wg0
  ↓
UFW
  ↓
routing
  ↓
service listener
```

Commands:

```bash
sudo tcpdump -ni wg0 host <client-vpn-ip>
sudo journalctl -k --since "10 min ago" | grep -i ufw
sudo ss -lntup
```

### Monitoring cannot scrape a host metric

Confirm the relevant listener and the source network:

```bash
sudo ss -lntup | grep -E ':(9100|10250|10257|10259|2381)'
sudo ufw status numbered
```

Then verify whether Prometheus traffic originates from the expected pod CIDR or via another path.

### Private DNS works locally but not through VPN

The firewall must allow TCP/UDP 53 from the VPN source ranges. Validate with:

```bash
sudo tcpdump -ni wg0 port 53
sudo ufw status numbered
```

### Cilium NodePort/Ingress path fails

Do not assume a public firewall rule for `443` automatically allows every internal NodePort. Check the actual Cilium Service/Ingress path and only open host ports that are genuinely required.

## Safe change procedure

For production-like changes:

```text
1. Inspect current UFW state
2. Update variables/playbook
3. Syntax-check Ansible
4. Apply firewall
5. Test existing SSH session
6. Test WireGuard
7. Test public HTTPS
8. Test private DNS
9. Test monitoring
```

Keep an existing SSH session open while changing the firewall.

## Suggested commit

```text
feat(firewall): document UFW access tiers and verification workflow
```
