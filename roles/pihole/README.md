# `flyoverhead.docker.pihole`

`Pi-hole` docker service deployment.

## Upgrading from Pi-hole v5

This role targets **Pi-hole v6**. The previous version pinned `2024.07.0`, the
v5 line, and v6 renamed every environment variable:

| v5 | v6 |
| :--- | :--- |
| `WEBPASSWORD` | `FTLCONF_webserver_api_password` |
| `WEB_PORT` | `FTLCONF_webserver_port` |
| `WEBTHEME` | `FTLCONF_webserver_interface_theme` |
| `PIHOLE_DNS_` | `FTLCONF_dns_upstreams` |
| `DNSMASQ_LISTENING` | `FTLCONF_dns_listeningMode` |
| `DNSSEC` | `FTLCONF_dns_dnssec` |
| `QUERY_LOGGING` | `FTLCONF_dns_queryLogging` |
| `DNS_BOGUS_PRIV` | `FTLCONF_dns_bogusPriv` |
| `DNS_FQDN_REQUIRED` | `FTLCONF_dns_domainNeeded` |
| `FTLCONF_LOCAL_IPV4` | removed |
| `IPv6` | removed -- the role disables IPv6 via sysctl instead |

`pihole_service_config` follows suit: `dnsmasq_listening_address` became
`dns_listening_mode` (values `LOCAL`, `SINGLE`, `BIND`, `ALL`, `NONE`), and
`web_address` is gone.

Anything set through the environment is **read-only** in the v6 web UI and CLI,
which is what makes this role the single source of truth for those settings.

Set `pihole_service_config.etc_dnsmasq_d: true` for the first v6 start on a host
upgraded from v5, so custom `dnsmasq.d` snippets are migrated into
`pihole.toml`. Turn it back off afterwards.

The container migrates the v5 data directory in place on first start. Back up
`<pihole_docker_config.path>/data` before the upgrade.

## Requirements

Blocklists and domain lists are read from this role's `files` directory:

| File | Purpose | `domainlist.type` |
| :--- | :--- | :--- |
| `adlist.list` | Blocklist source URLs, one per line | n/a (`adlist` table) |
| `domain_black.list` | Regex deny patterns, one per line | `3` |
| `domain_white.list` | Regex allow patterns, one per line | `2` |

Both domain files hold **regexes**, not literal domains -- that is why they map
to the regex list types. The `sqlite3` package is installed on the host to
manage the rows.

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `pihole_docker_config` | Docker configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `pihole_service_config` | Service configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `pihole_lists` | Expected row counts, derived from the `*.list` files | Definition example in [defaults/main.yml](defaults/main.yml) |
| `pihole_required_packages` | Host packages needed to manage `gravity.db` | `['sqlite3']` |

`pihole_service_config.web_password` defaults to a placeholder. Override it, and
keep the value in `ansible-vault`.

`pihole_gravity_db` in [vars/main.yml](vars/main.yml) is the host path to
`gravity.db`. It is a role internal, not a knob.

## How list convergence works

`config.yml` counts the rows in `gravity.db` and compares them with the number
of entries in the corresponding `*.list` file. A mismatch marks the check task
changed, which fires the handlers that wipe and repopulate the table, then run
`pihole reloadlists` (domain lists) or `pihole -g` (adlists).

This means the `*.list` files are declarative: removing a line removes the rule.
It also means an out-of-band edit through the web UI is reverted on the next
run, since only the count is compared -- not the contents.

## Host-level changes

- `DNSStubListener=no` in `/etc/systemd/resolved.conf`, so Pi-hole can bind
  `:53`. Skipped when `systemd-resolved` is not present.
- `dnsmasq.service` stopped and masked, for the same reason. Skipped when not
  present.
- IPv6 disabled or enabled via sysctl to match
  `pihole_service_config.ipv6_support`.

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `pihole_container_exists` | Whether the container is already present |
| `pihole_container_update` | Whether the running container's image differs from the configured tag |

## Dependencies

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.docker` | [README.md](../docker/README.md) |

Optional, and only for the default `pihole_service_config.upstream_dns`, which
points at port 5353 on the same host:

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.dnscrypt` | [README.md](../dnscrypt/README.md) |

## Example playbook

```yaml
- hosts: docker
  gather_facts: true
  roles:
    - role: flyoverhead.docker.docker
    - role: flyoverhead.docker.dnscrypt
    - role: flyoverhead.docker.pihole
      vars:
        pihole_service_config:
          dns_listening_mode: ALL
          upstream_dns:
            - '{{ ansible_host }}#5353'
          web_password: '{{ vault_pihole_web_password }}'
          web_port: 8080
          dnssec: true
          query_logging: true
          dns_bogus_priv: true
          dns_fqdn_required: true
          web_theme: default-dark
          ipv6_support: false
          etc_dnsmasq_d: false
```

## Check mode

`--check --diff` reports drift in the rendered configuration and the compose
file against a host where the stack is already deployed, and it reports list
drift too. The three `SELECT COUNT(*)` probes in `config.yml` carry
`check_mode: false` and run for real: they only read, and a skipped `command`
has no `stdout` for `changed_when` to compare against, so without this a check
run would call the lists converged whatever the `*.list` files hold. The
handlers that wipe and repopulate `gravity.db` are `command` tasks with no
`creates`/`removes`, so they skip in a check run and the database is never
written.

Detection reads the running container through
`community.docker.docker_container_info`, so a check run needs a reachable
docker daemon: against a host without the engine it fails in `detect.yml`.

## Tags

| Tag | Purpose |
| :--- | :--- |
| `pihole.detect` | Container and service state detection only |
| `pihole.install` | Host preparation and first deployment |
| `pihole.update` | Image tag change: pull and recreate |
| `pihole.config` | Blocklist and domain list convergence |

## License

GPL-3.0-only

## Author Information

fLy0v3rH34d
