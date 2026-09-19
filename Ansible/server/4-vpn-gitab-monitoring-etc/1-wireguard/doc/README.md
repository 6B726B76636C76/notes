# WireGuard

## Purpose

This directory manages the WireGuard VPN layer used as the private administrative and service-access network for the server and Kubernetes infrastructure.

The current server interface is `wg0` with the server VPN address `172.16.0.1/27`. Peer configuration is generated from Ansible variables. Public keys belong in normal configuration; private keys must remain in encrypted Ansible Vault data and must never be committed as plaintext.

The module is intentionally limited to VPN provisioning. Access control is enforced separately by the firewall and, where applicable, Kubernetes/Cilium policy.

## Responsibilities

The playbook and its supporting files are responsible for:

- installing the WireGuard package and required host dependencies;
- creating the `wg0` interface;
- assigning the server VPN address;
- rendering the server WireGuard configuration from Ansible variables;
- configuring peers from the shared `wg_peers` data structure;
- enabling the kernel networking settings required by the VPN design;
- enabling persistent startup of `wg0` through the system service;
- validating peer configuration before applying it;
- safely reapplying the declarative configuration when peer membership changes.

The module does **not** define the complete host firewall policy. The firewall module owns UFW policy, and Kubernetes networking remains under Kubernetes/Cilium.

## Configuration model

The configuration is split between the global variables and the encrypted variables:

```text
group_vars/all.yml
    ├── wg_interface
    ├── wg_server_address
    ├── wg_port
    ├── access-tier CIDRs
    └── wg_peers (public information only)

 group_vars/secrets.yml
    └── private peer material / other protected values

        ↓

 templates/wg0.conf.j2
        ↓

 /etc/wireguard/wg0.conf
        ↓

 wg-quick@wg0
```

The authoritative state is Ansible. Manual edits to `/etc/wireguard/wg0.conf` are not a supported long-term configuration method because a later playbook run will render the declared state again.

## Access tiers used by the wider infrastructure

The current network design uses separate VPN ranges for different trust levels:

```text
172.16.0.0/27   administrative VPN
172.16.1.0/24   limited/developer VPN
172.16.8.0/21   HTTP-only VPN class
```

These ranges are consumed by firewall policy and must not be changed casually. Changing them is a network design change because routing, UFW rules, DNS access, and service policies may all depend on them.

## Prerequisites

Before running this module, verify:

1. The target host exists in the global inventory.
2. The host has a working management path that does not depend exclusively on the new VPN.
3. `group_vars/all.yml` contains the correct WireGuard interface, server address, port, and peer public keys.
4. Required private values are stored only in the encrypted Vault file.
5. The intended access-tier CIDRs match the firewall configuration.

## Initial setup

From the Ansible repository root:

```bash
ansible-playbook \
  4-vpn-gitab-monitoring-etc/1-wireguard/wireguard.yml \
  --ask-vault-pass
```

Run this layer before relying on VPN-only administration. On a new system, keep an independent SSH/console path available until the VPN has been verified end to end.

## Validation after deployment

Check the interface:

```bash
ip -br addr show wg0
```

Check WireGuard state:

```bash
sudo wg show
```

Check service state:

```bash
systemctl status wg-quick@wg0
```

Check boot-time logs:

```bash
journalctl -u wg-quick@wg0 -b
```

Check routing:

```bash
ip route
```

For an individual peer, verify `latest handshake` and RX/TX counters in `wg show`. A configured peer with no recent handshake is not the same thing as a broken WireGuard interface.

## Operational sequence

A reliable deployment order is:

```text
1. Validate variables
2. Install WireGuard
3. Render wg0.conf
4. Enable/start wg0
5. Verify interface address
6. Verify peer configuration
7. Test handshake
8. Test VPN routing
9. Apply firewall policy
```

The firewall should be validated only after a working VPN path exists when the firewall is going to restrict administrative traffic to the VPN.

## Troubleshooting

### `wg0` is missing

```bash
systemctl status wg-quick@wg0
journalctl -u wg-quick@wg0 -b --no-pager
sudo wg show
```

Then inspect:

```bash
ls -l /etc/wireguard/
sudo sed -n '1,220p' /etc/wireguard/wg0.conf
```

Do not copy private keys into chat, tickets, commits, or debug logs.

### The interface exists but the peer has no handshake

Trace the path:

```text
client
  ↓
Internet / NAT
  ↓ UDP WireGuard port
server network interface
  ↓
wg0
  ↓
peer configuration
```

Useful commands:

```bash
sudo wg show
sudo tcpdump -ni any udp port 51820
```

A missing handshake usually points to endpoint reachability, key mismatch, firewall policy, or client-side configuration.

### Handshake exists but application traffic fails

WireGuard has established the cryptographic tunnel, so continue with:

```text
wg0 → routing → UFW → service listener → application
```

Use:

```bash
ip route
sudo ufw status numbered
sudo ss -lntup
sudo tcpdump -ni wg0 host <client-vpn-ip>
```

### `operstate` says `unknown`

This is expected for a WireGuard interface on this host. Do not use `node_network_up{device="wg0"}` as a liveness signal in Prometheus because this host reports `operstate="unknown"` and consequently exposes `node_network_up=0` even while the interface is administratively up.

The current Grafana alert instead checks `node_network_info` with `adminstate="up"`. Tunnel health should be monitored separately with peer handshake age.

### Configuration changes disappear

That is expected when the change was made manually. Reapply the desired Ansible state and make the persistent change in the variables/templates rather than editing `/etc/wireguard/wg0.conf` directly.

## Security rules

- Never commit WireGuard private keys.
- Never paste private keys into incident documentation.
- Treat peer removal as an access-revocation operation.
- Keep administrative VPN CIDRs stable unless the firewall and routing design are updated together.
- Keep the WireGuard UDP port closed to sources that are not supposed to establish tunnels.

## Suggested commit

```text
feat(wireguard): document VPN provisioning and peer operations
```
