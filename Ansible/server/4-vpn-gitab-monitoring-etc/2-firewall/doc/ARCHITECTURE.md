# Firewall Architecture

## Layered network policy

```text
                         Internet
                            |
                 +----------+----------+
                 |                     |
              80/443                UDP 51820
                 |                     |
          Cilium public IP          WireGuard
                 |                     |
                 |                   wg0
                 |                     |
                 |          +----------+----------+
                 |          |          |          |
                 |        Admin     Limited    HTTP-only
                 |        VPN       VPN        VPN
                 |          |          |          |
                 +----------+----------+----------+
                            |
                           UFW
                            |
          +-----------------+------------------+
          |                 |                  |
       Host SSH        Private DNS       Kubernetes host paths
          |                 |                  |
          +-----------------+------------------+
                            |
                    Kubernetes / Cilium
                            |
                      Services / Pods
```

## Trust boundaries

### Public edge

Only explicitly public services should be reachable from the Internet. In the current model those are principally HTTP/HTTPS ingress and the WireGuard UDP endpoint.

### Administrative VPN

`172.16.0.0/27` is the trusted management network. The firewall allows broader host/service access from this tier than from restricted tiers.

### Limited VPN

`172.16.1.0/24` is intended for restricted user/service access. It is not equivalent to host administration and should not receive host SSH unless deliberately configured.

### HTTP-only VPN

`172.16.8.0/21` is limited to web traffic by design.

### Kubernetes pod network

`10.244.0.0/16` is an internal cluster source range. It may need access to host control-plane and node-exporter endpoints for monitoring, even though those endpoints are not public services.

## Data path examples

### Public HTTPS

```text
Internet client
  -> 91.219.62.186:443
  -> Cilium LoadBalancer/Ingress
  -> ingress resource
  -> application service
  -> pod
```

### Private HTTPS

```text
VPN client
  -> wg0
  -> UFW
  -> private ingress path
  -> Cilium
  -> service
  -> pod
```

### Prometheus to host exporter

```text
Prometheus pod
  -> host endpoint :9100
  -> UFW source rule
  -> node_exporter
```

### Private DNS

```text
VPN client
  -> 172.16.0.1:53
  -> resolver
```

## State ownership

```text
Ansible variables + firewall tasks
            |
            v
          UFW rules
            |
            v
      Linux packet filter
```

Kubernetes/Cilium rules are a separate source of truth. Do not attempt to encode application-level Cilium policy in the host UFW module.

## Logging and incident workflow

```text
Dropped connection
      |
      v
Check UFW log
      |
      v
Check packet capture
      |
      v
Check route
      |
      v
Check listener/service
      |
      v
Check Kubernetes/Cilium policy
```

Useful tools:

```bash
sudo journalctl -k --since "10 min ago" | grep -i ufw
sudo tcpdump -ni any host <source-ip>
ip route get <destination-ip>
sudo ss -lntup
```

## Failure prevention

The firewall should not be treated as an isolated playbook. Changes to any of the following require cross-checking the firewall:

- WireGuard CIDRs;
- private DNS listener address;
- GitLab SSH port;
- Cilium Ingress NodePorts;
- node_exporter/control-plane metric ports;
- Kubernetes pod CIDR;
- management source address ranges.
