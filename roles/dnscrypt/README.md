# `flyoverhead.docker.dnscrypt`

`DNSCrypt Proxy` docker service deployment.

Runs `dnscrypt-proxy` as the encrypted upstream resolver. Pairs with the
[pihole](../pihole/README.md) role, which forwards to it on port 5353 by
default.

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `dnscrypt_docker_config` | Docker configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `dnscrypt_service_config` | Service configuration | Definition example in [defaults/main.yml](defaults/main.yml) |

`dnscrypt_sysctl_config_ipv6` in [vars/main.yml](vars/main.yml) lists the sysctl
keys toggled from `dnscrypt_service_config.ipv6_servers`. It is a role internal,
not a knob.

## Host-level changes

Two things outside the container are managed, both driven by facts gathered in
`detect.yml`:

- `avahi-daemon.service` and `avahi-daemon.socket` are stopped and masked when
  `dnscrypt_service_config.listening_port` is 5353, because avahi owns the mDNS
  port on a stock Debian install. Skipped when the units are not present.
- IPv6 is disabled or enabled via sysctl to match
  `dnscrypt_service_config.ipv6_servers`.

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `dnscrypt_container_exists` | Whether the container is already present |
| `dnscrypt_container_update` | Whether the running container's image differs from the configured tag |

## Dependencies

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.docker` | [README.md](../docker/README.md) |

## Example playbook

```yaml
- hosts: docker
  gather_facts: true
  roles:
    - role: flyoverhead.docker.docker
    - role: flyoverhead.docker.dnscrypt
```

## Service configuration example

```yaml
dnscrypt_service_config:
  listening_addresses:
    - 0.0.0.0
  listening_port: 5353
  # empty means "use every registered server matching the require_* filters"
  public_resolvers: []
  ipv4_servers: true
  ipv6_servers: false
  public_servers: true
  doh_servers: true
  odoh_servers: true
  anon_servers: true
  quad_servers: true
  dnssec_servers: true
  nolog_servers: true
  nofilter_servers: true
  disabled_public_resolvers: []
  upstream_servers:
    - 1.1.1.1
    - 8.8.8.8
    - 9.9.9.9
  netprobe_address: 9.9.9.9
  block_unqualified: true
  block_undelegated: true
  dns_cache: true
  dns_cache_size: 4096
```

`upstream_servers` are bootstrap resolvers only: they are used to fetch the
resolver lists and are never sent user queries.

## Check mode

`--check --diff` reports drift in the rendered configuration and the compose
file against a host where the stack is already deployed. The restart and
recreate handlers report what they would do without touching the container.

Detection reads the running container through
`community.docker.docker_container_info`, so a check run needs a reachable
docker daemon: against a host without the engine it fails in `detect.yml`. Run
the `docker` role for real once first.

## Tags

| Tag | Purpose |
| :--- | :--- |
| `dnscrypt.detect` | Container and service state detection only |
| `dnscrypt.install` | First deployment |
| `dnscrypt.update` | Image tag change: pull and recreate |
| `dnscrypt.config` | `dnscrypt-proxy.toml`, avahi masking and IPv6 sysctls |

## License

GPL-3.0-only

## Author Information

fLy0v3rH34d
