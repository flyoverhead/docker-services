# `flyoverhead.docker.singbox`

`Sing-box VPN` (VLESS + REALITY) docker service deployment.

Generates the server configuration on the host and one client profile per
configured client on the controller.

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `singbox_docker_config` | Docker configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `singbox_server_config` | Server configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `singbox_clients_config` | Clients configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `singbox_client_inbounds` | Per-client `inbounds` override, keyed by client name | `Example below` |
| `singbox_restarter` | Periodic restart sidecar | `Example below` |
| `singbox_tuning_config` | Server kernel optimization parameters | Definition example in [defaults/main.yml](defaults/main.yml) |
| `singbox_force_update_secrets` | Force regeneration of server keys and client secrets | `false` |

`singbox_docker_config.network_mode` is deliberately `host` rather than
`{{ docker_network_mode }}`: a REALITY server behind a docker bridge sees the
gateway address as every client's source address.

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `singbox_container_exists` | Whether the container is already present |
| `singbox_container_update` | Whether the running container's image differs from the configured tag |
| `singbox_active_clients` | Clients read back from the deployed `config.json` |
| `singbox_inactive_clients` | Deployed clients no longer listed in `singbox_clients_config.clients` |
| `singbox_new_clients` | Configured clients with no credentials yet |
| `singbox_all_clients` | Final client list rendered into both configs |

## Dependencies

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.docker` | [README.md](../docker/README.md) |

## Example inventory

```yaml
all:
  children:
    singbox:
      hosts:
        vps_host:
```

## Example playbook

```yaml
- hosts: singbox
  gather_facts: true
  roles:
    - role: flyoverhead.docker.docker
    - role: flyoverhead.docker.singbox
```

## Server config example

```yaml
singbox_server_config:
  name: '{{ inventory_hostname }}'
  tag: proxy
  listening_address: 0.0.0.0
  listening_port: 443
  public_address: '{{ ansible_host }}'
  vless_address: npo.nl
  vless_port: 443
  dns_port: 53
  flow: xtls-rprx-vision
  # dns, endpoints, outbounds, route and log: see defaults/main.yml for the full structure
```

## Clients list example

```yaml
singbox_clients_config:
  path: '{{ inventory_dir }}/files/clients'
  clients:
    - name: keenetic_router
    - name: smartphone
      platform: android
    - name: iphone
      platform: ios
    - name: macbook
      platform: macos
  dns:
    servers:
      - tag: dns-direct
        address: local
        detour: out-direct
      - tag: dns-proxy
        address: https://dns.adguard-dns.com/dns-query
        address_resolver: dns-direct
        detour: out-proxy
    rules:
      - outbound: out-direct
        server: dns-direct
    final: dns-direct
    strategy: ipv4_only
    independent_cache: true
  route:
    rules:
      - protocol: dns
        action: hijack-dns
      - domain:
          - 4pda.to
        outbound: out-proxy
    rule_set:
      - tag: geosite-category-ads
        type: remote
        format: binary
        url: https://raw.githubusercontent.com/sagernet/sing-geosite/rule-set/geosite-category-ads.srs
        action: reject
    default_domain_resolver:
      server: dns-direct
    auto_detect_interface: true
  # inbounds, outbounds, endpoints, log and experimental: see defaults/main.yml for the full structure
```

The role owns `singbox_clients_config.path` on the controller: it creates the
directory mode `0700` and writes each profile mode `0600`, because the profiles
carry the client credentials. Point it at a dedicated directory, not at a shared
one like `{{ playbook_dir }}`.

Client entries may be a plain name or a mapping. Extra keys on a mapping (such
as `platform`) are carried through to `singbox_all_clients` untouched, so
templates and per-client overrides can key off them.

Every list-valued key under `route` must be an explicit `[]` when unused. A bare
`rules:` with only comments beneath it is YAML `null`, and the client template
would render `"rules": null`.

## Per-client inbounds example

```yaml
singbox_client_inbounds:
  macbook:
    - type: mixed
      tag: mixed-in
      listen: 127.0.0.1
      listen_port: 2080
```

A client listed here gets these inbounds in its profile instead of
`singbox_clients_config.inbounds`.

## Periodic restart sidecar

The compose project includes a `restarter` sidecar that restarts the sing-box
container every `singbox_restarter.interval` seconds. It is enabled by default,
matching the deployed behaviour.

```yaml
singbox_restarter:
  enabled: true
  image: docker:cli
  interval: 3600
```

It bind-mounts `/var/run/docker.sock`, which gives that container
root-equivalent access to the host. A systemd timer running
`docker restart sing-box` achieves the same thing without exposing the socket.
Set `enabled: false` if you do not need the periodic restart.

## Predefined secrets

By default the role generates the server `REALITY` key pair
(`sing-box generate reality-keypair`) and each client's `uuid`/`short_id`
(`sing-box generate uuid` + `generate rand 8 --hex`), and reuses any values
already present on the host.

Reuse locates the role's own inbound in the deployed `config.json` by `tag`
(`singbox_server_config.tag`) and takes the private key and the client list off
it. Searching the deployed `tls` block for `vless_address` instead -- which is
what this role used to do -- stops matching the moment that address changes, and
an empty match is silent: the keypair and every client credential would be
reissued, invalidating all distributed profiles. If the inbound cannot be
located while a `config.json` exists, the role fails rather than reissuing.

Unlike `xray x25519 -i`, sing-box has no subcommand that derives a public key
from a private one, so the public half is stored in a `public_key` file next to
`config.json`. If that file goes missing while `config.json` still holds a
private key, the role fails rather than handing clients an empty public key.

To reuse previously generated secrets and skip the generation tasks, define the
values explicitly. This is useful for rebuilding a host from scratch or storing
secrets in a vault.

Server keys — when both `private_key` and `public_key` are set,
`generate reality-keypair` is skipped:

```yaml
singbox_server_config:
  private_key: SERVER_PRIVATE_KEY
  public_key: SERVER_PUBLIC_KEY
```

Client secrets — when both `uuid` and `short_id` are set, generation is skipped
for that client:

```yaml
singbox_clients_config:
  clients:
    - name: smartphone
    - name: keenetic_router
      uuid: 11111111-2222-3333-4444-555555555555
      short_id: 0123456789abcdef
```

## Check mode

`--check --diff` reports drift in `config.json`, the compose file and the client
profiles against a host where the stack is already deployed. Detection is
`stat` and `slurp` only, so it needs nothing special in a check run.

A check run never mints a secret. Whenever the REALITY keypair or a client
credential is missing -- a first deployment, a newly added client,
`singbox_force_update_secrets` -- the generator container is skipped and the
diff is rendered with `CHECK-MODE-PLACEHOLDER-*` values standing in for the
UUID, the short id and the keys. So the diff tells you *which* clients would be
added, not what their credentials will be; those are only issued on a real run.

Detection also reads the running container through
`community.docker.docker_container_info`, so a check run needs a reachable
docker daemon.

## Tags

| Tag | Purpose |
| :--- | :--- |
| `singbox.docker` | Container detection, install, update and the full config render |
| `singbox.server` | Server key readback only |
| `singbox.clients` | Client credential readback only |
| `singbox.tuning` | Kernel parameters and nofile limits |

`config.yml` is reached by `singbox.docker` and `singbox.tuning`, but the client
and key facts it consumes are set by the `detect | database` block, which is
tagged `singbox.clients`/`singbox.server`. A tagged client render therefore
needs both, e.g. `--tags singbox.clients,singbox.docker`.

## License

GPL-3.0-only

## Author Information

fLy0v3rH34d
