# Private DNS

## Purpose

This directory configures the private DNS layer used by VPN clients to resolve internal infrastructure names without publishing the private addresses to the public Internet.

The resolver is reachable through the WireGuard network. The current private gateway address is `172.16.0.1`, and the internal domain is `domain.com`.

## Responsibilities

The playbook and templates are responsible for:

- installing/configuring the selected DNS resolver;
- binding DNS service to the private VPN gateway address;
- creating internal DNS records required by the infrastructure;
- configuring upstream DNS servers where recursive resolution is required;
- keeping private records declarative through Ansible;
- enabling the resolver service at boot;
- integrating the DNS listener with the firewall policy.

Private DNS is intentionally not exposed on the public interface.

## Current internal records

The current design includes private names for services such as:

```text
gitlab.domain.com    -> private gateway / ingress path
grafana.domain.com   -> private gateway / ingress path
```

Other internal records should be declared through the shared variable structure instead of being manually edited in the resolver configuration.

## Network path

```text
VPN client
    |
    | DNS TCP/UDP 53
    v
172.16.0.1
    |
private DNS resolver
    |
    +--> internal records
    |
    +--> upstream DNS
```

## Prerequisites

Before deployment:

1. WireGuard `wg0` must be available.
2. `private_dns_bind_address` must match the VPN gateway address.
3. Internal names and addresses in `group_vars/all.yml` must be correct.
4. UFW must allow TCP/UDP 53 from the intended VPN networks.

## Apply

```bash
ansible-playbook \
  4-vpn-gitab-monitoring-etc/3-private-dns/private-dns.yml \
  --ask-vault-pass
```

## Validation

Find the listener:

```bash
sudo ss -lntup | grep ':53 '
```

Check the resolver service:

```bash
systemctl --type=service | grep -Ei 'dns|dnsmasq|unbound|bind'
```

From a VPN client:

```bash
dig @172.16.0.1 gitlab.domain.com A
dig @172.16.0.1 grafana.domain.com A
dig @172.16.0.1 gitlab.domain.com A +tcp
```

The answer should contain the intended private address/path and should not depend on public DNS publishing the private record.

## Troubleshooting

### DNS timeout

Trace the complete path:

```text
VPN client
  -> wg0
  -> UFW
  -> UDP/TCP 53 listener
  -> resolver
```

Check:

```bash
sudo tcpdump -ni wg0 port 53
sudo ufw status numbered
sudo ss -lntup | grep ':53 '
```

If no packet is seen on `wg0`, investigate WireGuard/routing. If a request arrives with no response, investigate the resolver.

### Listener is only on localhost

This causes local queries to work while VPN queries fail. The resolver must bind to the private gateway address.

### Internal name resolves publicly

Verify the private DNS record and client DNS server selection. A successful `dig` against an external resolver does not prove that private DNS is working.

### TCP works but UDP fails

Check UFW separately for protocol-specific rules and inspect packet captures on `wg0`.

## Security

- Never expose the private DNS listener directly to the public Internet.
- Keep private records in the internal configuration source.
- Treat the DNS server address as part of the VPN design.
- When moving a service, update DNS and ingress/firewall dependencies together.

## Suggested commit

```text
feat(private-dns): document VPN-scoped DNS records and resolver flow
```
