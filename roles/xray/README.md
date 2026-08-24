# `flyoverhead.docker.xray`

`Xray VPN` (VLESS + REALITY) docker service deployment.

Generates the server configuration on the host and one client profile per
configured client on the controller.

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `xray_docker_config` | Docker configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `xray_server_config` | Server configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `xray_clients_config` | Clients configuration | Definition example in [defaults/main.yml](defaults/main.yml) |
| `xray_client_inbounds` | Per-client `inbounds` override, keyed by client email | `Example below` |
| `xray_tuning_config` | Server kernel optimization parameters | Definition example in [defaults/main.yml](defaults/main.yml) |
| `xray_force_update_secrets` | Force regeneration of server keys and client secrets | `false` |

`xray_docker_config.network_mode` is deliberately `host` rather than
`{{ docker_network_mode }}`: a REALITY server behind a docker bridge sees the
gateway address as every client's source address.

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `xray_container_exists` | Whether the container is already present |
| `xray_container_update` | Whether the running container's image differs from the configured tag |
| `xray_active_clients` | Clients read back from the deployed `config.json` |
| `xray_inactive_clients` | Deployed clients no longer listed in `xray_clients_config.clients` |
| `xray_new_clients` | Configured clients with no credentials yet |
| `xray_all_clients` | Final client list rendered into both configs |

## Dependencies

| Name | Description |
| :--- | :--- |
| `flyoverhead.docker.docker` | [README.md](../docker/README.md) |

## Example inventory

```yaml
all:
  children:
    xray:
      hosts:
        vps_host:
```

## Example playbook

```yaml
- hosts: xray
  gather_facts: true
  roles:
    - role: flyoverhead.docker.docker
    - role: flyoverhead.docker.xray
```

## Server config example

```yaml
xray_server_config:
  name: '{{ inventory_hostname }}'
  tag: proxy
  listening_address: 0.0.0.0
  listening_port: 443
  public_address: '{{ ansible_host }}'
  vless_address: vk.com
  target_port: 443
  dns_port: 53
  flow: xtls-rprx-vision
  stream_settings:
    network: tcp
    security: reality
  # dns, outbounds, routing and log: see defaults/main.yml for the full structure
```

`stream_settings` is rendered as-is, with `realitySettings` merged in from the
generated keypair and short IDs, so any transport (`tcp`, `xhttp`, `grpc`, ...)
can be configured without touching the template.

## Clients list example

```yaml
xray_clients_config:
  path: '{{ inventory_dir }}/files/clients'
  clients:
    - keenetic_router
    - smartphone
    - macbook
  # dns, inbounds, outbounds, routing and log: see defaults/main.yml for the full structure
```

The role owns `xray_clients_config.path` on the controller: it creates the
directory mode `0700` and writes each profile mode `0600`, because the profiles
carry the client credentials. Point it at a dedicated directory, not at a shared
one like `{{ playbook_dir }}`.

Client entries may be a plain string or a mapping with an `email` key. Extra
keys on a mapping are carried through to `xray_all_clients` untouched, so
templates and per-client overrides can key off them.

## Per-client inbounds example

```yaml
xray_client_inbounds:
  macbook:
    - protocol: socks
      tag: socks-in
      listen: 127.0.0.1
      port: 1080
      settings:
        udp: true
```

A client listed here gets these inbounds in its profile instead of
`xray_clients_config.inbounds`.

## Predefined secrets

By default the role generates the server `REALITY` key pair (`xray x25519`) and
each client's `id`/`shortId` (`xray uuid` + `openssl rand`), and reuses any
values already present on the host.

Reuse reads from a single source: the deployed `config.json`. The role finds
its own inbound there by `tag` (`xray_server_config.tag`), takes the private key
and the client list off it, and derives the public key from that private key.
The `public_key` file on the host is written for convenience but never read
back, so it cannot drift from the deployed key pair. If the inbound cannot be
located while a `config.json` exists, the role fails rather than silently
reissuing every credential.

To reuse previously generated secrets and skip the generation tasks, define the
values explicitly. This is useful for rebuilding a host from scratch or storing
secrets in a vault.

Server keys — when both `private_key` and `public_key` are set, `xray x25519`
is skipped:

```yaml
xray_server_config:
  private_key: SERVER_PRIVATE_KEY
  public_key: SERVER_PUBLIC_KEY
```

Client secrets — define a client as a mapping; when both `id` and `shortId` are
set, generation is skipped for that client (plain strings still auto-generate):

```yaml
xray_clients_config:
  clients:
    - android
    - email: keenetic_router
      id: 11111111-2222-3333-4444-555555555555
      shortId: 0123456789abcdef
```

## Check mode

`--check --diff` reports drift in `config.json`, the compose file and the client
profiles against a host where the stack is already deployed.

A check run does start one short-lived container. `detect | derive public key
from private key` carries `check_mode: false` because `xray x25519 -i` computes
the public half of the deployed private key and writes nothing, and the
`-runtime` container is one-shot and cleaned up. Without it a check run reads no
public key, concludes the keypair is missing, and reports reissuing the REALITY
keypair and every client credential -- the exact reissue the tag assert in
`detect.yml` exists to prevent.

A check run never mints a secret. Whenever the REALITY keypair or a client
credential is missing -- a first deployment, a newly added client,
`xray_force_update_secrets` -- the generator container is skipped and the diff
is rendered with `CHECK-MODE-PLACEHOLDER-*` values standing in for the UUID, the
short id and the keys. So the diff tells you *which* clients would be added, not
what their credentials will be; those are only issued on a real run.

Detection also reads the running container through
`community.docker.docker_container_info`, so a check run needs a reachable
docker daemon.

## Tags

| Tag | Purpose |
| :--- | :--- |
| `xray.docker` | Container detection, install, update and the full config render |
| `xray.server` | Server key readback only |
| `xray.clients` | Client credential readback only |
| `xray.tuning` | Kernel parameters and nofile limits |

`config.yml` is reached by `xray.docker` and `xray.tuning`, but the client and
key facts it consumes are set by the `detect | database` block, which is tagged
`xray.clients`/`xray.server`. A tagged client render therefore needs both,
e.g. `--tags xray.clients,xray.tuning`.

## License

GPL-3.0-only

## Author Information

fLy0v3rH34d
