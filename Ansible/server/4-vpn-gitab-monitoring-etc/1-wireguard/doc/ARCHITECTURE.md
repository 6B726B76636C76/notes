# WireGuard Architecture

## High-level topology

```text
                         Internet
                            |
                      UDP 51820
                            |
                    +----------------+
                    | Linux VPS host |
                    | 91.122.33.122  |
                    +--------+-------+
                             |
                            wg0
                    172.16.0.1/27
                             |
          +------------------+------------------+
          |                  |                  |
     Admin peers       Limited peers       HTTP-only peers
     172.16.0.0/27     172.16.1.0/24       172.16.8.0/21
          |                  |                  |
          +------------------+------------------+
                             |
                         UFW policy
                             |
                    Services / Kubernetes
```

## Control flow

```text
Ansible inventory
      |
      v
group_vars/all.yml -----------+
                                 |
group_vars/secrets.yml ----------+--> Jinja template
                                 |        |
                                 |        v
                                 |   /etc/wireguard/wg0.conf
                                 |        |
                                 |        v
                                 +--> wg-quick@wg0
                                          |
                                          v
                                      kernel wg0
                                          |
                              +-----------+-----------+
                              |                       |
                           routing                    peers
                              |                       |
                              +-----------+-----------+
                                          |
                                          v
                                        UFW
```

## Addressing model

The server owns `172.16.0.1/27` on `wg0`. The peer addresses are explicitly declared by Ansible and should remain unique.

The broader infrastructure distinguishes three trust tiers:

| Tier | CIDR | Intended use |
|---|---|---|
| Admin | `172.16.0.0/27` | Full administrative VPN access |
| Developers/Limited | `172.16.1.0/24` | Restricted service access |
| HTTP-only | `172.16.8.0/21` | Web-only access |

The VPN layer provides identity and transport. It does not alone decide which services each tier may reach.

## Trust boundaries

```text
VPN peer identity
       |
       v
VPN subnet / source address
       |
       v
Host firewall (UFW)
       |
       +--> host services
       |
       +--> Kubernetes entry points
                   |
                   v
             Cilium / Service
                   |
                   v
               Workload
```

## Failure domains

### WireGuard configuration failure

The interface cannot start, or peers are malformed.

### Network reachability failure

The UDP endpoint is unreachable before WireGuard itself can authenticate a packet.

### Peer configuration failure

The tunnel endpoint is reachable but key/address/AllowedIPs configuration is inconsistent.

### Post-handshake policy failure

The tunnel works, but UFW or Kubernetes policy denies the desired application connection.

These domains should be tested separately instead of treating every VPN problem as a key problem.

## Monitoring model

The interface-level Grafana alert checks:

```promql
max(node_network_info{device="wg0",adminstate="up"}) or vector(0)
```

This is intentionally different from `node_network_up`, because Linux reports WireGuard `operstate=unknown` on this host. The alert therefore measures administrative interface presence/state rather than relying on the generic network-operstate metric.

The correct next layer for monitoring is peer liveness based on WireGuard handshake age:

```text
interface up
    ≠
peer reachable
    ≠
application reachable
```

## Recovery model

On a damaged host:

1. Preserve console or SSH access.
2. Validate `/etc/wireguard/wg0.conf` ownership/permissions.
3. Run the Ansible playbook.
4. Verify `wg0`.
5. Verify a peer handshake.
6. Verify routing.
7. Reapply/validate firewall policy.
8. Verify the dependent service path.

The intended source of truth remains Ansible variables and templates.
