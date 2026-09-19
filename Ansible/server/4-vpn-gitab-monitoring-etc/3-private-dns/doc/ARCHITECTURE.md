# Private DNS Architecture

## Resolver topology

```text
                    +-------------------+
                    |   VPN clients     |
                    | 172.16.x.x        |
                    +---------+---------+
                              |
                         TCP/UDP 53
                              |
                              v
                    +-------------------+
                    | 172.16.0.1        |
                    | Private DNS        |
                    +----+----------+----+
                         |          |
                internal|          |recursive
                records  |          |queries
                         |          v
                         |     upstream DNS
                         |
                         v
                internal service names
```

## Record flow

```text
group_vars/all.yml
      |
      v
private DNS variables
      |
      v
Ansible task/template
      |
      v
resolver configuration
      |
      v
DNS daemon
      |
      v
VPN clients
```

## Isolation

The resolver is private because the route to it is private and the firewall allows DNS only from approved VPN networks. Public clients should not be able to query the internal records directly.

## Integration with other modules

```text
WireGuard
   |
   +--> provides private transport
   |
Private DNS
   |
   +--> resolves internal names
   |
Firewall
   |
   +--> permits DNS only from approved sources
   |
Ingress / services
   |
   +--> terminate actual application traffic
```

DNS is therefore an addressing/namespace layer; it is not an access-control mechanism.

## Failure domains

- WireGuard failure: client cannot reach `172.16.0.1`.
- Firewall failure: packets are dropped at UFW.
- Resolver failure: port 53 is unreachable or the daemon has no usable configuration.
- Record failure: DNS answers but contains the wrong address.
- Application failure: DNS is correct but the service behind the name is unavailable.

Each failure domain should be tested independently.
