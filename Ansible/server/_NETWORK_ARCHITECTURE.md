# Server Network Architecture

This document focuses only on the network layer of the infrastructure
from `Ansible/server/4-vpn-gitab-monitoring-etc`: WireGuard (transport),
UFW/firewall (host-level access control), private DNS (addressing), and
the Kubernetes network model (Cilium/MetalLB). Source: `group_vars/all.yml`
and the `1-wireguard`, `2-firewall`, `3-private-dns`, `3-basic-apps`
playbooks.

## 1. Network layers, top to bottom

```text
Layer 1: WireGuard      — who you are (identity + encrypted transport)
Layer 2: UFW/firewall   — what that IP is allowed to reach (host ACL)
Layer 3: Private DNS    — internal service naming
Layer 4: Cilium/MetalLB — what's allowed inside the cluster (K8s ACL)
```

Each layer is decoupled: WireGuard knows nothing about service ports,
UFW knows nothing about Kubernetes Services/Ingress, and DNS is not a
security mechanism at all — just addressing. Changing addressing/CIDRs
on one layer requires a matching change on the neighboring layers.

## 2. Addressing and overall topology

```text
                                Internet
                                   |
                    +--------------+--------------+
                    |                              |
                 80/443                       UDP 51820
                    |                              |
             Cilium (public IP)               WireGuard (wg0)
             LoadBalancer/Ingress            172.16.0.1/27
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
              +------------------+-------------------+-------------------+
              |                 |                     |                   |
          Host SSH        Private DNS :53      Kubernetes host paths   node_exporter
           :22/:2424          172.16.0.1        (kubelet/metrics)         :9100
              |                 |                     |                   |
              +------------------+-------------------+-------------------+
                                |
                        Kubernetes pod network
                          10.244.0.0/16
                                |
                            Cilium CNI
                                |
             +-----------+-----------+-----------+-----------+
             |           |           |           |           |
          GitLab      Argo CD      Harbor    Prometheus/   dev/staging/
        (vps ns)    (argocd ns)  (harbor ns)  Grafana        prod
```

### Subnets / addresses table

| Entity | Address/CIDR | Purpose |
|---|---|---|
| Host public IP | `public_ip` (e.g. `21.221.21.221`) | entry point for 80/443 and WireGuard UDP |
| `wg0` (server) | `172.16.0.1/27` | VPN gateway, also the private DNS resolver |
| Admin VPN | `172.16.0.0/27` | full administrative access |
| Developers/Limited VPN | `172.16.1.0/24` | restricted access (GitLab SSH, 8443) |
| HTTP-only VPN | `172.16.8.0/21` | 80/443 only |
| Kubernetes Pod CIDR | `10.244.0.0/16` | in-cluster pod/Cilium traffic |

## 3. WireGuard — transport layer

```text
group_vars/all.yml (wg_peers, CIDRs, wg_interface, wg_server_address)
group_vars/secrets.yml (peer private keys)
        |
        v
templates/wg0.conf.j2
        |
        v
/etc/wireguard/wg0.conf
        |
        v
wg-quick@wg0  →  wg0 interface (172.16.0.1/27) in the kernel
```

Key parameters (`group_vars/all.yml`):

```yaml
wg_interface: "wg0"
wg_server_address: "172.16.0.1/27"
wg_port: 51820
vpn_mtu: 1380
vpn_nat_interface: "eth0"        # outbound NAT for VPN subnets
wg_nat_subnets:
  - "172.16.0.0/27"
  - "172.16.1.0/24"
  - "172.16.8.0/21"

wg_peers:
  - id: "001"
    name: "vaclav"
    group: "admin"
    address: "172.16.0.2"
    persistent_keepalive: 25
  - id: "002"
    name: "vaclav-mobile"
    group: "developers"
    address: "172.16.1.2"
    persistent_keepalive: 25
```

Each peer gets a fixed address inside its own tier — the peer list is
itself the access matrix ("who is in which trust class"). Private keys
exist only in encrypted `secrets.yml` and are never committed in
plaintext.

Sequence of operations when deploying (or adding/removing a peer):

```text
1. validate variables
2. install the WireGuard package
3. render wg0.conf
4. enable/start wg0
5. verify the interface address
6. verify peer configuration
7. verify handshake
8. verify routing
9. apply the firewall policy
```

### Validation

```bash
ip -br addr show wg0
sudo wg show                       # interface state and every peer
systemctl status wg-quick@wg0
journalctl -u wg-quick@wg0 -b
ip route
```

For a given peer, check `latest handshake` and the RX/TX counters in
`wg show` — no recent handshake does not mean "the interface is broken."

### Monitoring quirk

On this host the WireGuard interface always reports
`operstate=unknown`, so the standard `node_network_up{device="wg0"}`
metric stays at `0` even with a working tunnel. The health check is
built on:

```promql
max(node_network_info{device="wg0",adminstate="up"}) or vector(0)
```

Peer liveness is tracked separately (handshake age), not as the
interface's overall state:

```text
interface up  ≠  peer reachable  ≠  application reachable
```

## 4. UFW/Firewall — host-level access control

Default policy: **deny incoming / allow outgoing**, `forward=ACCEPT`
(required for CNI packet forwarding between pods).

```text
group_vars/all.yml (my_ip, admin_subnet, limited_subnet, http_only_subnet,
                     wg_port, ingress_ports, connlimit_above)
        |
        v
2-firewall/playbook.yml.example (community.general.ufw tasks)
        |
        v
UFW rules → nftables → kernel packet filter
```

### Rules per tier

```text
Outside world
  ├── TCP 80/443   → allow, rate-limited (basic flood protection)
  └── UDP 51820    → allow (WireGuard endpoint)

my_ip (personal static IP)
  └── TCP 22       → allow (SSH fallback outside WireGuard)

172.16.0.0/27 (admin)
  └── allow everything (admin_all_tcp: true)

172.16.1.0/24 (developers/limited)
  └── allow only developer_ports (2424 — GitLab SSH, 8443 — private web)

172.16.8.0/21 (http-only)
  └── allow only http_only_ports (80, 443)
```

A `connlimit` rule is also added to `before.rules` — no more than
`connlimit_above` (default 20) concurrent connections per source IP on
80/443, to blunt simple abuse.

### Port table (full, with sources)

| Port | Protocol | Who can reach it | Role |
|---:|---|---|---|
| 22 | TCP | `my_ip` (fallback), admin VPN | host SSH |
| 2424 | TCP | limited VPN, admin VPN | GitLab SSH (GitLab Shell) |
| 53 | TCP/UDP | all VPN subnets | private DNS |
| 80 | TCP | internet (rate-limited), all VPN | public HTTP → Cilium ingress |
| 443 | TCP | internet (rate-limited), all VPN | public HTTPS → Cilium ingress |
| 51820 | UDP | internet | WireGuard endpoint |
| 6443 | TCP | (internal) | Kubernetes API |
| 9100 | TCP | pod CIDR (Prometheus) | node_exporter |
| 10250 | TCP | pod CIDR | kubelet metrics/API |
| 10257 | TCP | pod CIDR | kube-controller-manager metrics |
| 10259 | TCP | pod CIDR | kube-scheduler metrics |
| 2381 | TCP | pod CIDR | etcd metrics |

### SSH hardening (same playbook, not a separate port, but part of network security)

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no
```

Plus CrowdSec + `crowdsec-firewall-bouncer-iptables` with the
`crowdsecurity/sshd` and `crowdsecurity/linux` collections — dynamic
bans layered on top of the static UFW rules.

### Validation

```bash
sudo ufw status verbose
sudo ufw status numbered
sudo ufw show raw
sudo nft list ruleset            # what's actually in nftables under UFW
sysctl net.ipv4.ip_forward
ip route
```

Functional tests from a VPN client:

```bash
nc -vz <server-ip> 2424          # GitLab SSH
nc -vz 172.16.0.1 53             # private DNS
```

From outside:

```bash
nc -vz <public-ip> 443
```

### Safe rule-change procedure

```text
1. inspect current UFW state
2. update variables/playbook
3. ansible-playbook --syntax-check
4. apply the firewall
5. verify the current SSH session (don't break it!)
6. verify WireGuard
7. verify public HTTPS
8. verify private DNS
9. verify monitoring
```

Keeping a working SSH session open while making changes is the standard
safeguard against locking yourself out.

## 5. Private DNS

```text
group_vars/all.yml → private_dns_bind_address, private_dns_domain,
                      private_dns_records, private_dns_upstreams
        |
        v
templates/dnsmasq-private.conf.j2 (or dnsmasq-main.conf.j2)
        |
        v
resolver bound to 172.16.0.1:53
```

The resolver listens **only** on the private VPN address, never
publicly. Current records (`private_dns_records`):

```text
gitlab.domain.com    → public/ingress address (reachable over VPN)
grafana.domain.com   → public/ingress address
argocd.domain.com    → public/ingress address
registry.domain.com  → public/ingress address
```

Upstream for every other domain is a public resolver (`1.1.1.1`,
`9.9.9.9`).

```text
VPN client
    | TCP/UDP 53
    v
172.16.0.1
    |
    +--> internal A records (gitlab/grafana/argocd/registry)
    +--> recursive queries to upstream
```

### Validation

```bash
sudo ss -lntup | grep ':53 '
dig @172.16.0.1 gitlab.domain.com A
dig @172.16.0.1 gitlab.domain.com A +tcp
```

### Common problems

- **Timeout** — trace `client → wg0 → UFW → 53 listener → resolver`; if
  the packet never shows up on `wg0` (`tcpdump -ni wg0 port 53`), the
  issue is WireGuard/routing; if the packet arrives but gets no reply,
  the issue is the resolver.
- **Bound to localhost only** — local queries work, VPN queries don't;
  the resolver must bind to `172.16.0.1`, not `127.0.0.1`.
- **TCP works, UDP doesn't** — check protocol-specific UFW rules
  separately.

DNS is purely an addressing layer: resolving a name successfully does
not by itself grant access to the service — UFW and Cilium still decide
that.

## 6. Kubernetes network model (Cilium/MetalLB)

```text
containerd/CNI
     |
     v
Cilium 1.19.8
  ├── full kube-proxy replacement (kubeProxyReplacement=true)
  ├── socketLB.hostNamespaceOnly=false
  ├── ipam.mode=kubernetes
  ├── built-in Ingress Controller (loadbalancerMode=shared)
  │      insecureNodePort 32283, secureNodePort 31140
  └── k8sServiceHost = control-plane address, k8sServicePort=6443
     |
     v
MetalLB 0.14.9 (L2, no FRR)
  ├── IPAddressPool: {{ public_ip }}/32
  └── L2Advertisement
```

After installation, the Cilium Ingress Service is **explicitly
patched** with `loadBalancerSourceRanges`, restricting sources to the
VPN subnets only:

```yaml
loadBalancerSourceRanges:
  - "{{ admin_subnet }}"      # 172.16.0.0/27
  - "{{ limited_subnet }}"    # 172.16.1.0/24
  - "{{ http_only_subnet }}"  # 172.16.8.0/21
```

The playbook then asserts the applied value matches the expected one
(`ansible.builtin.assert`) — so the public IP formally "listens" on
80/443, but Cilium drops packets from sources outside the VPN ranges at
its own layer, before they ever reach the Ingress resource.

```text
Public client → 80/443 (UFW allow) → Cilium LB
                                         |
                                src IP in a VPN CIDR? ── no ──> drop
                                         | yes
                                         v
                                  Ingress resource
                                         |
                                         v
                                    Service → Pod
```

So actual public access to HTTP services is only possible over
WireGuard, even though UFW formally opens 80/443 to the whole internet
— the VPN restriction is enforced at the Cilium layer, not by UFW.

## 7. End-to-end traffic examples

### Private HTTPS (over VPN)

```text
VPN client → wg0 → UFW → Cilium LoadBalancer (source verified)
           → Ingress → Service → Pod
```

### Prometheus → host exporter

```text
Prometheus pod (10.244.x.x) → host's public IP on port 9100
                             → UFW (allows the pod CIDR as source)
                             → node_exporter
```

### GitLab SSH over VPN

```text
VPN client (172.16.1.x) → wg0 → UFW (allow 2424 for limited/admin)
                        → GitLab Shell Service (K8s)
```

### Private DNS

```text
VPN client → wg0 → UFW (allow 53 for the VPN CIDR) → 172.16.0.1:53 → resolver
```

## 8. Ownership boundaries and what can't be changed in isolation

```text
WireGuard   → identity/transport
Firewall    → host-level access (source IP → port)
Cilium      → in-cluster access (source IP/namespace → Service)
Private DNS → naming only, not security
```

Changing any of the following requires a matching change in the other
modules:

- any VPN tier's CIDR (`admin_subnet`, `limited_subnet`,
  `http_only_subnet`);
- the private DNS resolver's address;
- the GitLab SSH port (`gitlab_ssh_port`, currently `2424`);
- Cilium Ingress NodePorts (`insecureNodePort`, `secureNodePort`);
- metrics ports (`9100`, `10250`, `10257`, `10259`, `2381`);
- the Kubernetes Pod CIDR (`10.244.0.0/16`).

## 9. Recovering from a network failure

```text
1. keep a console/SSH path that doesn't depend on the VPN
2. check ownership/permissions of /etc/wireguard/wg0.conf
3. run the WireGuard Ansible playbook
4. verify wg0 (ip -br addr show wg0)
5. verify a handshake with at least one peer
6. verify routing (ip route)
7. reapply/verify the firewall
8. verify the end service (DNS → Ingress → Service → Pod)
```

In every case the source of truth is the Ansible variables and
templates; manual edits to `wg0.conf` / UFW rules survive only until
the next playbook run.
